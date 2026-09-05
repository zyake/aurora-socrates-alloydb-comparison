# クラウドネイティブOLTPアーキテクチャ比較
## Aurora vs. Socrates / Azure SQL Hyperscale vs. AlloyDB

**焦点:** 書き込みパス、WALの耐久性、ページのマテリアライズ、ストレージ分離、エラスティシティ、耐久性/可用性、コスト、およびこのセッションで得られたアーキテクチャ上の教訓。

**スコープに関する注意。** 本レポートは、商用製品そのものだけではなく、*アーキテクチャ上の考え方*を比較する。特に、2025年の *OLTP in the Cloud* 論文は、モデル化した **Aurora-like** / **Socrates-like** アーキテクチャと、実際の商用サービスを明確に区別している。

---

## エグゼクティブサマリー

3つのシステムに共通する最も重要な原理は、WALの基本原則である。

> トランザクションをコミットするために、変更済みデータページそのものを同期的に永続化する必要はない。必要なのは、対応するWAL/REDOが耐久性のある境界まで到達することである。

各システムの違いは、主として **その耐久性境界がどこに置かれているか**、そして **どの程度の追加処理がその境界に結合されているか** にある。

| アーキテクチャ | 同期コミット境界 | WAL / REDO配布 | REDO / ページマテリアライズ | 永続ページ容量 | 中核思想 |
|---|---|---|---|---|---|
| 従来型DB + リモートブロックストレージ | リモートブロックデバイス上のWAL | DBエンジン | DBエンジン | SAN/EBS系リモートストレージ | 従来DBをほぼそのままクラウドへ移行 |
| **Aurora** | Protection Groupの4/6ストレージクォーラム | Primary/Storageプロトコルが6メンバーへREDOをfan-out | Auroraストレージノード | Aurora分散ストレージ | **Compute/Storage分離 + Smart Storage** |
| **Socrates / Hyperscale** | **Landing Zone (LZ)** | 永続化後に **XLOG** で非同期配布 | **Page Server** | **XStore / Backup Storage** | **耐久性、ログ配布、ページサービスを分離** |
| **AlloyDB** | **低レイテンシのRegional Log Storage** | Regional WAL Pipeline | **Log Processing Service (LPS)** | **Sharded Regional Block Storage** | **ストレージ層内部まで分離** |

最も重要なアーキテクチャ進化は、次のように捉えられる。

```text
Traditional DB
    |
    v
Compute + WAL + Buffer Manager + Checkpoint
    |
    | Remote Block Storage
    v
SAN / EBS-like Storage


Aurora
    |
    v
Compute
    |
    | REDOのみ
    v
Smart Replicated Storage
  - Log Durability
  - REDO Processing
  - Page Materialization
  - Page Serving
  - Repair / Recovery


Socrates
    |
    +---- Sync WAL ----> Landing Zone
    |                    Durability
    |
    +---- Async WAL ---> XLOG
                         Distribution / Lifecycle
                              |
                              v
                         Page Servers
                         REDO / Hot Pages
                              |
                              v
                            XStore
                         Cheap Capacity


AlloyDB
    |
    v
Regional Log Storage
    |
    | Asynchronous WAL Consumption
    v
Log Processing Service
    |
    v
Regional Block Storage
```

このセッションで得られた最も重要な結論は次の通りである。

> **AuroraはComputeとStorageを分離した。SocratesとAlloyDBはさらに、機能とNFRの特性に応じてストレージサブシステムそのものを分離した。**

この細粒度の分離は、リソースのエラスティシティやコスト効率を改善し得る。一方で、分散システムとしての複雑性は増加する。また、レプリケーション、REDO適用、ページ書き込みそのものが消えるわけではない。

---

# 1. 研究上の位置づけ

添付された2024年IEEE Survey、*Cloud-Native Databases: A Survey* は、クラウドネイティブDBの主要技術として **Compute-Storage Disaggregation** と **“Log is the Database”** を挙げ、クラウドネイティブOLTPのアーキテクチャ選択を整理している。

ただし、**“Log is the Database”** という表現は慎重に解釈する必要がある。

- ページイメージを一切生成しなくてよい、という意味ではない。
- 従来型DBがコミットのたびにダーティページを同期flushしている、という意味でもない。
- 本質は、**耐久性のあるログを更新の権威的な差分表現とし、ページ生成やリカバリ処理をトランザクション実行Computeから別の層へ移せること** にある。

Aurora、Socrates、AlloyDBを正しく比較するうえで、この区別は極めて重要である。

---

# 2. ベースライン: 従来型WAL DBがすでに行っていること

一般的なWAL/ARIES型DBMSでは、通常次の順序で処理される。

```text
UPDATE
  |
  +--> Buffer Pool上のPageを変更
  |
  +--> WALを生成
          |
          v
      WALを永続化
          |
          v
       COMMIT

       ... 後で ...

Dirty Buffer
   |
Checkpoint / Eviction
   |
   v
Data Page Write
```

したがって、

> **従来型WAL DBでも、通常はダーティページの永続化そのものはトランザクションコミットのクリティカルパスには存在しない。**

ただし、リモートブロックデバイスを用いたクラウドホスト型DBの場合、最終的にはWALもページもCompute/Storage境界を越える。

```text
DB Compute
   |
   +---- WAL -----------> Remote Block Storage
   |
   +---- Dirty Pages ---> Remote Block Storage
                          (later)
```

その結果、次の2つの問題が分かれる。

1. **Commit Latency** はWALの耐久化経路に依存する。
2. **総ネットワーク/ストレージトラフィック** はWALと後続のページ書き込みの両方を含む。

クラウドネイティブなLog-based設計は、主として後者を改善し、ページ再構築処理をCompute/Storage境界のStorage側へ移す。

---

# 3. 「Write Amplification」の3つの意味

このセッションで得られた大きな教訓の一つは、**Write Amplification（書き込み増幅）** という語は、どの層の増幅なのかを区別しなければ曖昧すぎるということである。

## 3.1 Compute側のWrite Amplification

DB Computeプロセス自身がどの程度の書き込み作業を担当するか。

```text
Traditional:
    WAL generation
    + Buffer management
    + Dirty-page writeback / Checkpoint I/O

Aurora / Socrates / AlloyDB:
    WAL generation
    + DB ComputeからStorageへのPage Writebackを大幅に削減/排除
```

## 3.2 Compute→Storage間のNetwork Amplification

DB ComputeとStorageの境界を越えるバイト量。

たとえば8 KiBページに小さな変更を行う場合:

```text
Traditional Remote-Storage DB
    Compact WAL
    +
    eventual 8 KiB Page Transfer

Log-shipping Architecture
    Compact WAL
    only from Compute
```

## 3.3 システム全体のPhysical Write Amplification

サービス全体では、依然として次の処理が必要になる。

- WALレプリケーション
- WAL永続化
- REDO replay
- ページのマテリアライズ
- Multi-ZoneでのPage/Blockレプリケーション
- Checkpoint
- Garbage Collection
- Backup/Archiveトラフィック

ある設計が **Compute側** や **Compute/Storage境界** の増幅を劇的に減らしても、**システム全体** の仕事量は依然として大きい可能性がある。

Aurora、Socrates、AlloyDBはいずれもこの特徴を持つ。

---

# 4. Auroraアーキテクチャ

## 4.1 元々の設計目標

Aurora SIGMOD論文では、ComputeとStorageを分離したクラウド環境では、主要なボトルネックが **Network** に移ると論じている。そこでAuroraは、ダーティページではなく **REDO Record** を送信し、REDO処理を専用の分散ストレージサービスへ押し込む。

概念的には次の通りである。

```text
Aurora Writer
     |
     | REDO
     v
+--------------------------------+
| Aurora Smart Storage           |
|                                |
|  Replication                   |
|  REDO Persistence              |
|  REDO Application              |
|  Page Materialization          |
|  Page Serving                  |
|  Repair / Recovery             |
+--------------------------------+
```

これは、汎用Network Block Storage上で従来DBをそのまま動作させる設計からの大きな進歩であった。

---

## 4.2 Protection GroupとWrite Path

Auroraはストレージを小さな論理チャンク/Protection Groupに分割する。各Protection Groupは3つのAvailability Zoneにまたがる6つのStorage Memberに対応する。

```text
                    Aurora Writer
                         |
                  Page-specific REDO
                         |
                         v
                Protection Group
                         |
       +---------+-------+-------+---------+---------+
       |         |       |       |         |         |
       v         v       v       v         v         v
      S1        S2      S3      S4        S5        S6
      AZ-A      AZ-A    AZ-B    AZ-B      AZ-C      AZ-C
       \         \       \       \         \         \
        +---------------- wait for 4 ----------------+
                         |
                         v
                    Durable WAL
                         |
                         v
                       COMMIT
```

重要な特徴:

- **6つのWAL/REDO destination**
- **4/6 Write Quorum**
- **3 AZ** に分散
- 故障モデルは、AZ全体の相関故障を明示的に考慮
- DB Writerはダーティページイメージを同期送信しない

後期のAuroraストレージ最適化では、6コピーすべてに完全なページイメージを保持するわけではない。2025年のコスト論文では、概ね **3つのFull Page Copy + 6-way Log Replication** としてモデル化している。

---

# 4.3 Auroraが排除したもの

Auroraは、通常のページ永続化経路として、

```text
DB Compute --> Dirty Page --> Remote Storage
```

を排除する。

これにより、次を削減できる。

- DB PrimaryのPage Write I/O
- Compute/Storage間のFull Page Network Traffic
- Foreground Checkpoint Pressure
- Primary上のCrash Recovery負荷

代わりに、Storage ServiceがREDOからPage Stateを再構築する。

---

# 4.4 Auroraが排除していないもの

更新済みページは、最終的にはどこかで生成されなければならない。

Storage側の処理は概ね次のようになる。

```text
REDO Receive
    |
REDO Persist
    |
Sort / Deduplicate / Coalesce
    |
Apply REDO
    |
Materialize Page Version
    |
Persist / Replicate Page State
    |
GC / Repair / Backup
```

したがって、正確な表現は次の通りである。

> **Auroraはページメンテナンス処理をDB ComputeからStorage Serviceへ移したのであり、処理そのものを消したわけではない。**

これは“Log is Data”系アーキテクチャ全体に共通する重要な教訓である。

---

# 5. AuroraのForeground Write Overhead

このセッションで検討した重要な仮説の一つは、AuroraのPrimary Write Pathは、Dedicated Log Service型アーキテクチャよりもStorage Topologyを意識する必要がある、という点である。

概念的には1つのREDO Recordに対し:

```text
Page Identifier
     |
     v
Protection-Group Mapping
     |
     v
Protection-Group Membership
     |
     v
Six Destination Streams
     |
     v
ACK Processing / 4-of-6 Quorum
     |
     v
Durable Frontier
```

ただし、**Mapping処理自体** が重いとは限らない。Protection Group Membershipはキャッシュ可能であり、Page→Group Mappingも決定的な演算で実装できる。

より本質的なコスト候補は次である。

1. 6-way Network Fan-out
2. 多数のDestination Queue/Stream
3. Cross-AZ Transmission
4. ACK Processing
5. Quorum Progress Bookkeeping
6. Flow Control / Slow Member Handling

### 重要なエビデンス上の限界

2025年の *OLTP in the Cloud* の分析モデルは、Auroraの概ね **6× WAL Network Amplification** を明示的に考慮している。一方で、次のような独自実装の細かいCPU/Protocolコストまではモデル化していない。

- RecordごとのPG Lookup CPU
- 6 StreamのQueue Management
- ACK Processing CPU
- Quorum Bookkeeping CPU
- Storage-side REDO CPU
- Proprietary Packet/Flow-control Contention

したがって、これらは合理的なアーキテクチャ上の懸念ではあるものの、AuroraのThroughput Limitの直接原因として測定・実証されたものではない。

---

# 6. Aurora Smart StorageでLocal SSDが合理的な理由

Aurora Storage Nodeは単なる受動的なRemote Diskではない。次の処理を行う。

- REDO ingestion
- REDO processing
- Page reconstruction
- Page reads
- Version maintenance
- Repair

そのため、低レイテンシのLocal Storageを使うことは自然である。

仮にAuroraの各Storage Nodeの下に、さらにEBSのようなReplicated Block Deviceを置くと、概念的にはレプリケーションが二重化する。

```text
Aurora Replication
     |
     +--> Storage Node A --> Underlying Block-Storage Replication
     |
     +--> Storage Node B --> Underlying Block-Storage Replication
     |
     +--> Storage Node C --> Underlying Block-Storage Replication
     ...
```

これは耐久性処理を重複させ、さらにNetwork Service Boundaryを追加する可能性がある。

ただし、これは「すべてのDisaggregated DBがLocal SSDだけを使うべき」という意味ではない。Socratesはより階層化された設計を示している。

```text
Fast Page-Processing Tier --> Local SSD / Cache
Cheap Durable Tier        --> Remote Cloud Storage
```

---

# 7. Socrates / Azure SQL Hyperscaleアーキテクチャ

Socratesは、システムをAuroraよりさらに細かく分解する。

```text
                         Primary
                            |
              +-------------+-------------+
              |                           |
          Sync WAL                    Async WAL
              |                           |
              v                           v
        Landing Zone                    XLOG
           (LZ)                           |
       Fast Durability                    |
              |                           v
              |                      Page Servers
              |                     REDO / Page Cache
              |                           |
              |                           v
              +-----------------------> XStore
                                  Durable / Cheap Capacity
```

重要な概念は次である。

```text
Durability != Log Dissemination != Page Materialization
```

---

# 8. Socrates Landing Zone (LZ)

## 8.1 目的

LZは次のNFRに最適化される。

- 同期的で耐久性のあるWAL Append
- 非常に低いCommit Latency
- 強いIntegrity/Consistency
- 小さいWorking Set
- 予測しやすいWrite Behavior

元のSocrates実装ではAzure Premium Storage/XIOを利用し、**3 replicas** を保持すると説明されている。

論文ではLZを次のように位置づけている。

- Fast
- Durable
- Expensiveであってもよい
- 意図的にSmall
- Circular Bufferとして構成

---

## 8.2 Commit Path

クリティカルパスは次の通りである。

```text
Transaction
    |
Generate WAL Block
    |
    v
Landing Zone
    |
Durable Write / Write Quorum
    |
    v
WAL Hardened
    |
    v
COMMIT ACK
```

最も重要なのは、

> **SocratesはCommit時にXLOGではなくLZのDurabilityを待つ。**

ということである。

---

# 9. LZがすでにDurableなのにXLOGが必要な理由

LZとXLOGは同じ論理WALを扱うが、そのWorkload/NFR特性は大きく異なる。

| Dimension | LZ | XLOG |
|---|---|---|
| Primary Role | Commit Durability | Log Distribution / Lifecycle |
| Synchronization | **Synchronous** | **Asynchronous** |
| Critical Latency | 極めて重要 | Commit Pathには直接入らない |
| Workload | Append-heavy | Fan-out / Read / Serve / Filter |
| Capacity | Small | より大きいWorking Hierarchy |
| Reader Count | 最小 | 多数のConsumer |
| Consumer Lag | 主目的ではない | Lagging Consumerを扱う必要あり |
| Ordering/Gaps | Durable Storage Semantics | Reorder / Gap Handling |
| Cost Model | SmallならExpensiveでも許容 | 経済的にScaleする必要あり |
| Long-term Retention | しない | XStoreへDestage |

これらを1つのServiceにまとめると、同時に次の要求を満たす必要がある。

```text
Very Low Tail-Latency Synchronous Writes
+ Many Concurrent Readers
+ Filtering
+ Consumer Lag
+ Cache Hierarchy
+ Long Retention
+ Cheap Capacity
+ Horizontal Distribution
```

Socratesは、このNFR競合を避けるためにLZとXLOGを分離している。

---

# 10. LZとXLOGへのParallel Send

PrimaryはLog Blockを **LZとXLOGの両方へ並列に送る**。

```text
                         Primary
                            |
                    Produce Block B
                            |
               +------------+------------+
               |                         |
               v                         v
              LZ                    XLOG Pending
          Synchronous                 Async
               |                         |
               |                         X
          B Hardened                 Not Publishable
               |                         |
               +---- Hardened Notice ----+
                                         |
                                         v
                                  LogBroker / Consumers
```

これにより、Durability LatencyとDissemination LatencyをOverlapできる。

直列なら:

```text
LZ Durable
   |
Then Send XLOG
```

となり、Downstream Propagation Latencyは両者の時間を足し合わせる形になる。

並列化すれば、LZのDurability Barrierが完了した時点でXLOGはすでにデータ本体を持っている可能性が高い。

---

# 11. XLOGが受信WALをすぐPublishできない理由

もしXLOGがまだLZで永続化されていないWALをPage Server/Secondaryへ送ってしまい、その後PrimaryがLZへのDurable Write完了前にFailした場合、不可能な履歴が生じ得る。

```text
Durable History:
100 -> 101 -> 102

Consumer History:
100 -> 101 -> 102 -> 103
                    ^
                    Not Durable
```

Socrates論文では、この危険な方式を **Speculative Logging** と呼ぶ。

したがって、XLOGでは概念的に次のInvariantが必要になる。

```text
Received(B) AND Hardened(B) AND Ordering-valid(B)
            |
            v
       Publishable(B)
```

Primaryは、どのBlockがLZでDurableになったかをXLOGへ通知する。XLOGはそのBlockを **Pending Area** から **LogBroker** へ移し、必要に応じてGap FillingやReorderingも行う。

これは分散版のWAL原則と考えられる。

> **Consumerは、まだ耐久化されていない履歴を意味のある状態として観測してはならない。**

---

# 12. XLOG Storage HierarchyとDestaging

XLOGは、小さく高価なDurability Tierを、経済的なLog Hierarchyへ接続する役割も持つ。

```text
             Recent WAL
                 |
                 v
                LZ
          Fast / Expensive
                 |
              XLOG
            Destaging
           /         \
          v           v
    Local SSD       XStore
      Cache       Cheap Long-term
```

これにより、次を実現する。

- 低Commit Latency
- Recent WALへの高速アクセス
- 低コストな長期Retention
- Point-in-Time Recovery
- 有限なLZ Circular Bufferの再利用

Socrates論文では、DestageされていないLogでLZが満杯になれば、最終的にUpdate Processingを停止せざるを得ないと説明している。

したがってXLOGは、**即時のTransaction Critical Path** には存在しないが、**継続的なWrite Availability** には必要である。

---

# 13. Socrates Page Server

Page Serverは、PrimaryがWALだけを書けば消えてしまうわけではないページ処理を担当する。

```text
XLOG
  |
  | Filtered WAL
  v
Page Server
  |
  +--> Replay WAL / REDO
  +--> Maintain Page State
  +--> Serve Page Reads
  +--> Local Memory/SSD Cache
  +--> Checkpoint Modified Pages
           |
           v
         XStore
```

したがって、

> **Socratesでもダーティページ/Checkpoint処理は存在する。これをPrimaryからPage Serverへ移し、Transaction Commitに対して非同期化している。**

この区別は非常に重要である。

---

# 14. SocratesのNFR分解

Socratesの各Componentは、NFRごとにSpecializeしたTierとして理解できる。

```text
LZ
    Latency + Synchronous Durability

XLOG
    Throughput + Fan-out + Ordering + Lifecycle

Page Server
    REDO CPU + Random Page Access + Fast Cache

XStore
    Capacity + Durability + Low Cost

Primary
    Transaction Execution
```

ここから一般化できる設計原則は次である。

> **ComponentはFunctionだけでなく、競合するNFR Profileによっても分離すべきである。**

---

# 15. SocratesのPerformance Evidence

元のSocrates SIGMOD論文には、Write-heavyなLogging Experimentが含まれている。

CDB workloadでLoggingを飽和させた実験では:

| Architecture | Log Throughput | CPU |
|---|---:|---:|
| Previous SQL HADR | **56.9 MB/s** | 46.2% |
| Socrates | **89.8 MB/s** | 73.2% |

これはLogging Throughputで約 **58%向上** に相当する。

論文では、従来HADRアーキテクチャではCompute NodeがLog/DB Backup処理にも関与していたのに対し、SocratesではBackup/Storage責務を下位層へ押し下げたことが差の一因とされている。

### 重要な注意

このBenchmarkは:

```text
Socrates vs. Previous Azure SQL HADR
```

であり、

```text
Socrates vs. Aurora
```

ではない。

したがって、Socratesの分離設計が旧Microsoft Architectureより有利であることは示すが、Hyperscaleが常にAuroraより高速にWriteできることを証明するものではない。

---

# 16. Aurora vs. Socrates: Foreground Write Path

両者の違いは次のように要約できる。

## Aurora

```text
Generate REDO
     |
     v
Identify Protection Group
     |
     v
Use Six-member Topology
     |
     v
Fan-out REDO
     |
     v
Track 4/6 Durability
     |
     v
COMMIT
```

## Socrates

```text
Generate WAL
     |
     v
One Logical Durable-write Interface
     |
     v
Landing Zone Durability
     |
     v
COMMIT

         meanwhile / asynchronously
                    |
                    v
                  XLOG
                    |
                    v
              Page Servers
```

したがってAuroraでPrimary/Foreground Protocolが担ういくつかの責務は、Socratesでは **Primaryから除去** されている。ただし、システム全体から消滅しているわけではない。

| Concern | Aurora Primary / Foreground Protocol | Socrates Primary | System-wide |
|---|---|---|---|
| Page→Storage Shard Mapping | Yes | Commit Durabilityには不要 | Page Partitioning自体は存在 |
| Replica Membership Awareness | より直接的 | 大幅に抽象化 | 下位Serviceには存在 |
| Six Storage Streams | Yes | No | LZ自体は複製される |
| Quorum Tracking | Aurora Storage Protocol | LZへ委譲 | 物理的には必要 |
| Page Server Dissemination | Storage設計に密接 | XLOGでAsync | 依然必要 |

より正確には:

> **Socratesはこれらの複雑性をPrimary Commit Pathから排除または下位層へ委譲する。Distributed Storageの複雑性そのものを消すわけではない。**

---

# 17. SocratesのCost / Elasticity Argument

2025年の *OLTP in the Cloud: Architectures, Tradeoffs, and Cost* は、Socrates-likeをAurora-likeより **さらにDisaggregated** されたArchitectureと表現する。理由はLog ServiceとPage Serviceを分離しているためである。

モデルでは、Small Datasetに対するVery High Transaction Rate領域で **Socrates-likeがCost-optimalになり得る** とし、一部領域ではAurora-likeが約 **70%高コスト** となる。主な理由はAurora-likeではPage ResourceとLog Resourceを独立にScalingしにくいことである。

また、Socrates-likeはModerate Durability Requirementの広い領域で合理的なDefault Choiceになり得るとされる。

これは、次のResource Coupling仮説を支持する。

```text
Write-heavy Workload Needs:
    More WAL Bandwidth
    More REDO CPU

But May NOT Need:
    More Database Capacity

Socrates:
    Scale Log / Page Processing Separately

Aurora-like:
    Log / Page / Storage Resources are More Coupled
```

---

# 18. AuroraのDurabilityとCost Trade-off

同じ2025年論文では、Aurora-likeの大きな利点としてDurabilityを挙げている。

6-way Log Replicationにより、モデル上はAurora-likeは約 **20 ninesのDurability** を持つと推定される。

その一方で、Replicated Database Stateのコストも大きい。

> LargeかつMostly ColdなDatasetでは、3-fold Full-DB ReplicationがAurora-likeのコストを支配し、モデル上は代替案の2倍以上になる場合がある。

ここから、顧客価値は次のように捉えられる。

```text
Customer First Needs:

Durability >= Business Requirement
Availability >= Business Requirement

Once Those Constraints Are Satisfied:

Cost
Latency
Throughput
Elasticity
Often Become Dominant Differentiators
```

したがって、Alternative Architectureがすでに業務上十分なDurabilityを満たしている場合、追加の極端なDurabilityには限界効用がある。

---

# 19. Real Aurora Benchmark Evidenceとその限界

2025年のコスト論文では、Commercial Aurora PostgreSQLによるModel Validationも行っている。

Updateを含むWorkloadでは、いくつかのケースでAnalytical Modelがより高いThroughputを予測していたにもかかわらず、実Auroraでは約 **0.5 million tx/s** を超えられなかったと報告している。

1 TBのケースでは、モデル上は約1.1 GB/sのPage Readを予測したのに対し、実測は約630 MB/sで、CPUやNetworkに余裕があるように見えた。

著者らは **I/O StackのContention/Overhead** を疑っているが、Aurora内部をProfileできないため原因は特定できていない。

この点は、本セッションの議論に対して重要である。

### 論文が示したこと

- Real AuroraのUpdate-heavy behaviorがSimple Resource Modelから乖離した。
- 実験上、Scaling Limit/Overheadが観測された。

### 論文が証明していないこと

- Protection Group Routingが原因である。
- 4/6 ACK Bookkeepingが原因である。
- Storage-side REDO Processingが原因である。
- Six-way Fan-out単独が原因である。

これらは妥当なHypothesisではあるが、より深いInstrumentationまたはAWS内部データが必要である。

---

# 20. AlloyDBアーキテクチャ

AlloyDBは、概念的にはOriginal AuroraよりもSocratesに近い設計を採用している。

GoogleはStorage Layerを3つのComponentとして説明している。

1. **Low-latency Regional Log Storage**
2. **Log Processing Service (LPS)**
3. **Failure-tolerant Sharded Regional Block Storage**

```text
                       PostgreSQL Primary
                              |
                         Generate WAL
                              |
                              v
                 +-----------------------+
                 | Regional Log Storage  |
                 | Low-latency Durability|
                 +-----------+-----------+
                             |
                         Async WAL
                             |
                             v
                 +-----------------------+
                 | Log Processing Service|
                 | LPS                   |
                 |                       |
                 | REDO                  |
                 | Block Materialization |
                 | Page/Block Serving    |
                 +-----------+-----------+
                             |
                             v
                 +-----------------------+
                 | Regional Block Storage|
                 | Sharded + Durable     |
                 +-----------------------+
```

Googleは、Storage Service内部でもComputeとStorageをさらに分離し、**Block StorageをLog Processingとは独立にScale可能にする** と説明している。

---

# 21. AlloyDB Commit Path

Write Lifecycleは次の通りである。

```text
SQL UPDATE / INSERT / DELETE
        |
        v
Primary Modifies In-memory State
        |
        v
Generate PostgreSQL WAL
        |
        v
Synchronously Persist WAL
To Regional Log Storage
        |
        v
Transaction Durable / Commit
        |
        | asynchronous
        v
LPS Consumes WAL
        |
        v
REDO + Block Materialization
        |
        v
Regional Block Storage
```

したがって:

```text
COMMIT Requires:
    Durable WAL

COMMIT Does NOT Require:
    LPS REDO Completion
    Block Materialization Completion
```

これはSocratesで確認した基本的な分離と同じである。

---

# 22. AlloyDB LPS Elasticity

AlloyDBで特に重要なのは、**REDO/Page Processing Tier自体がHorizontally Elastic** であることである。

Googleは次を説明している。

- PersistenceをBlock ShardにPartition
- 各ShardをLPSへ割当
- 1 LPSが複数Shardを担当可能
- Shard→LPS Mappingを動的に変更
- Durable DataをCopyせずにLPS Resourceを追加/削除可能

```text
                 Regional WAL
                      |
           +----------+----------+
           |          |          |
           v          v          v
         LPS-1      LPS-2      LPS-3
           |          |          |
        Shard A    Shard B     Shard C
           \          |          /
            +---------+---------+
                      |
                      v
              Regional Block Store
```

これはAuroraについて議論したResource Coupling懸念に直接対応する。

```text
Need More REDO CPU?
    --> Add / Rebalance LPS Capacity

Need More Durable Capacity?
    --> Scale Block Storage

Need More SQL CPU?
    --> Scale Database Compute
```

各Resourceを同じ割合でScaleする必要はない。

---

# 23. AlloyDBにも相当量のReplication/Write Workは残る

DisaggregationはMulti-Zone Durabilityのコストを消さない。

Googleは次のように説明している。

- Storageは **3 Zones** に分散
- 各Zoneに **Complete Database State** が存在
- 各Log Recordは各Zoneで処理される
- BlockはStorage LayerでZone間Replicationされる

概念的には:

```text
                   Regional WAL
                       |
          +------------+------------+
          |            |            |
          v            v            v
        Zone A       Zone B       Zone C
          |            |            |
         LPS          LPS          LPS
          |            |            |
          v            v            v
      DB State A   DB State B   DB State C
       Complete     Complete     Complete
```

したがってAlloyDBを「絶対的にWrite Amplificationが小さい」と評価するのは不正確である。

より正確には:

> **ReplicationとREDO処理を、独立にScalingできるServiceの後段に配置し、Latency-sensitiveなDB Primary Pathから外している。**

---

# 24. Socrates vs. AlloyDB

対応関係は非常に明確である。

| Socrates | AlloyDB | Role |
|---|---|---|
| Primary | PostgreSQL Primary | Transaction Execution |
| **LZ** | **Regional Log Storage** | Synchronous WAL Durability |
| **XLOG** | Log Pipeline/Service Functionの一部 | WAL Dissemination / Lifecycle |
| **Page Server** | **LPS** | REDO + Page/Block Materialization |
| **XStore** | **Regional Block Storage** | Durable DB Blocks / Capacity |
| RBPEX / Local SSD | Local Cache / LPS Cache | Fast Working Set |

大きな違いは、Socratesでは独立した **XLOG Dissemination/Lifecycle Layer** が公開Architectureとして現れるのに対し、AlloyDBの公開Architectureはより直接的に:

```text
Regional Log Store -> LPS -> Regional Block Store
```

として表現される点である。

またAlloyDBはLPS Resourceの独立ScalingとDynamic Shard-to-LPS Mappingを強く打ち出している。

---

# 25. アーキテクチャ世代

歴史的には、次のように分類すると理解しやすい。

## Generation 0 — Traditional Cloud-hosted DB

```text
DB Engine
  + WAL
  + Buffer Pool
  + Checkpoint
       |
       v
Generic Remote Block Storage
```

**Strength:** 移行モデルが単純  
**Weakness:** DB内部のモノリシックなI/O/Network特性をそのまま引きずる。

---

## Generation 1 — Aurora-style Smart Storage

```text
DB Compute
    |
    | REDO Only
    v
Smart Distributed Storage
```

**Innovation:** Page Maintenance/RecoveryをNetwork境界のStorage側へ移動。

**Strengths:**

- 非常に強いFault Tolerance
- DB ComputeからのPage Traffic削減
- Fast Recovery
- Self-healing Storage

**Trade-offs:**

- Log/Page Functionが比較的Coupled
- 6-way WAL Fan-out
- 複数のFull Data Copy
- Write/Storage ProtocolがTopologyをより意識する

---

## Generation 2 — Socrates-style Functional Disaggregation

```text
Compute
  |
  +--> Fast Durable Log Tier
  |
  +--> Async Log Distribution
             |
             v
        Page-processing Tier
             |
             v
        Cheap Durable Tier
```

**Innovation:** 異なるNFRに応じてStorage Functionを分離。

**Strengths:**

- Primaryに対するDurability Abstractionが単純
- Page WorkをCommit Pathから排除
- より安価なDurable Capacity
- Resourceを独立にSizing可能

**Trade-offs:**

- Distributed Component数が増える
- Progress Watermark / Lease / Backpressure管理
- XLOG/Page ServerのFailure Management
- Async Backlogの複雑性

---

## Generation 2+ — AlloyDB Multi-layer Storage Disaggregation

```text
Compute
  |
  v
Regional Log Store
  |
  v
Elastic LPS
  |
  v
Regional Block Store
```

**Innovation:** Storage Layer内部のREDO Processing Computeを独立にElastic化。

**Strengths:**

- WAL Appendを個別最適化
- REDO CPUを独立Scale
- Block Capacityを独立Scale
- DB ComputeはStorage Topologyを意識しにくい

**Trade-offs:**

- 3-ZoneのComplete DB Stateは依然コストが大きい
- 高度なDistributed Block/REDO Coordination
- Proprietary Internalのため外部検証に限界

---

# 26. Auroraが最初からSocrates/AlloyDB型を採用しなかった理由

AWSが「Socrates型を検討して却下した」と明言する公開論文はないため、この節ではEvidenceとInferenceを分ける必要がある。

## 公開資料から確認できる事実

Auroraの初期論文が強く重視したのは:

- NetworkがBottleneckになること
- Compute→Storage Trafficの削減
- REDOをSmart Storageへ移すこと
- Correlated AZ Failureへの耐性
- Fast Recovery / Self-healing

Socratesは2019年に、AlloyDBはさらに後の2022年に登場している。

## 合理的なArchitecture Inference

Auroraが解いた最初のCloud問題は:

```text
"How do we remove Dirty-page / Network / Recovery pressure
 from a conventional DB Compute node?"
```

である。

Socratesが次に問うたのは:

```text
"After moving Storage work out of Compute,
 why should Durability, Log Distribution,
 REDO, Page Serving, and Cheap Capacity
 still be one Storage Architecture?"
```

であり、AlloyDBはその考えをさらに洗練したと解釈できる。

したがってSocrates/AlloyDBを **Later-generation Disaggregation** と呼ぶのは妥当であるが、Auroraが当初の目的に対して悪い設計だったという意味ではない。

---

# 27. なぜAuroraは後からStorage Architectureを単純に置き換えなかったのか

この点は必然的にInferenceを多く含む。

```text
Primary -> Aurora Storage
```

を、

```text
Primary -> Log Service -> REDO Service -> Page Store
```

へ置き換えることは、事実上新しいStorage Engineを導入することに近く、互換性とCorrectness Riskが非常に大きい。

再検証対象には次が含まれる。

- Crash Consistency
- Failover
- Protection-group Recovery
- Backup/PITR
- Global Database Behavior
- Encryption
- Repair
- Version Upgrade
- Engine-specific REDO Semantics
- Existing Customer Fleet

AWSは、公開されている範囲では、Core 6-member Protection-group Modelを置き換えるよりも、既存Storage Architectureを前提にその周辺を進化させてきたように見える。

これは妥当なProduct/Engineering Strategyと考えられるが、AWS内部の意思決定理由として証明されたものではない。

---

# 28. DurabilityとAvailabilityを混同しない

このセッションで繰り返し修正した重要点は、次の2つを分離することである。

### Durability（耐久性）

> ACK済みCommitted Dataが、Failure後も存在し続けるか。

### Availability（可用性）

> Failure中/後でもServiceがRequestを処理し続けられるか。

Auroraの6-way Storage Quorumは、非常に強いStorage Durability/Fault Toleranceを提供する。

Socratesは意図的に **DurabilityとAvailabilityを分離** する。

- LZ/XStore: Durable Truth
- Compute/Page Server: Availability/Performance
- XLOG: Dissemination / Log Lifecycle

AlloyDBも同様に、WAL Durability、REDO Compute、Replicated Block Stateを分離する。

したがって、

> 「AuroraはDurabilityが強い」

は比較的強く支持できるが、

> 「AuroraはSocratesより常にAvailabilityが高い」

という一般化は支持できない。

---

# 29. Infrastructure / AZ仮説

このセッションでは、Aurora/Socratesの違いがAWSとAzureのInfrastructure Historyに由来する可能性も検討した。

強い形の主張:

> 「AWS AZは実際のData Center Failure Domainだが、Azure AZはそうではない」

は現在では誤りである。両ProviderともAvailability Zoneを、独立した物理インフラを持つ隔離されたData Center Location/Groupとして定義している。

より防御可能な歴史的仮説は次である。

```text
AWS
  Early First-class AZ Failure Boundary
       |
       v
Aurora Explicitly Internalizes
3-AZ Durability Topology


Azure
  Historically Strong Regional Storage / Fault-domain Primitives
       |
       v
Socrates Delegates Durability
To Cloud-storage Services
```

Auroraの初期論文は、6-member設計をAZ-correlated failure modelから明示的に導いている。

SocratesはAzure Premium StorageとXStoreを明示的に活用する。

ただし:

> **このセッションで確認した資料の中に、Data Center TopologyがDB Architectureを直接的に「原因」として決定したことを証明する資料はない。**

この因果関係はResearch Hypothesisとして扱うべきである。

---

# 30. 包括比較マトリクス

| Dimension | Traditional + RBD/SAN | Aurora | Socrates / Hyperscale | AlloyDB |
|---|---|---|---|---|
| DB ComputeがDirty PageをRemote Storageへ書く | Yes | **No** | **No** | **No** |
| Commit時にWALの同期Durabilityが必要 | Yes | Yes | Yes | Yes |
| Synchronous Durability Target | Remote Block WAL | 4/6 Storage PG | **LZ** | **Regional Log Store** |
| WAL Fan-outがDB Primaryから見える | Low / Logical Device | **High: 6 members** | **Low: Logical LZ** | **Low: Logical Regional Log Service** |
| Commit DurabilityがPage Placementに依存 | Storage Deviceが暗黙に処理 | **Protection-group specific** | **No** | **No** |
| REDO適用場所 | DB Compute | Storage Node | Page Server | LPS |
| Page Materialization | DB Compute | Storage Node | Page Server | LPS |
| Checkpoint/Page Writeが消えるか | No | No; transformed | **No; Page Serverが実行** | No; LPS/Block Tierが実行 |
| Fast Local SSD Tier | DB/Host依存 | Storage Nodes | Page Servers / RBPEX | Cache / LPS |
| Cheap Remote Capacity Tier | RBD itself | 分離度は低い | **XStore** | **Regional Block Storage** |
| Separate Log Distribution Tier | No | No | **XLOG** | 公開Architecture上は独立同等Tierではない |
| Independent REDO Compute Scaling | No | Limited/Coupled | Page Server Scale-outで改善 | **Strong: Elastic LPS** |
| Multi-zone Page State | Storage-dependent | Modelでは~3 Full Copies | Page Copies + Durable Backup Model | **3 ZonesそれぞれComplete State** |
| PrimaryのForeground Topology Awareness | Low | **Higher** | **Low** | **Low** |
| Architectural Component Count | Low | Medium | **High** | High |
| Extreme Durability Emphasis | Storage-dependent | **Very High** | Strong but Different Model | Strong Multi-zone |
| Functional/NFR Decomposition | Low | Medium | **Very High** | **Very High** |
| Best Conceptual Strength | Familiarity | Durability + Smart Storage | Cost/Elasticity Decomposition | Elastic Storage-layer Processing |
| Main Conceptual Weakness | Network/I/O Coupling | Storage-function Coupling | Distributed-service Complexity | Complex Multi-zone Replicated Pipeline |

---

# 31. Customer-oriented Interpretation

顧客は通常、単独で「最大のDurability」を最大化しようとはしない。

より現実的なObjectiveは:

```text
Minimize:
    Cost
    Latency

Maximize:
    Throughput
    Elasticity

Subject to:
    Durability >= Requirement
    Availability >= Requirement
```

となる。

これは、Alternative Systemが必要なDurability/Availabilityをすでに満たすなら、追加のReplicationに対する顧客価値が限定的になることを意味する。

### Workloadごとの傾向

| Workload | Architecturally Attractive Direction |
|---|---|
| Extreme Durability / RPO-sensitive | Aurora-like Redundancyが魅力的になり得る |
| High-write, Small/Medium DB | Fine Log/Page Separationが魅力的 |
| Large Mostly-cold DB | Cheap Durable Capacity + Smaller Fast Tier |
| Highly Variable Write Load | Independently Scalable Log/REDO Tiers |
| Read-scale Workload | Shared Storage + Cheap Read Compute |
| Development/Test | Extreme DurabilityよりCost優先になりやすい |

これは特定商用製品が常に優れていることを意味しない。評価時に見るべきWorkload Dimensionを示している。

---

# 32. 「Cheaper and Faster」Evidenceが実際に示すもの

このセッションで確認した最も強いAcademic/Model Evidenceは次の通りである。

1. **Socrates Paper**
   - Real Socrates vs. Previous Azure HADR
   - Write-heavy Logging: 89.8 MB/s vs 56.9 MB/s
   - Storage/Backup WorkをComputeから移す効果を実測

2. **2025 OLTP Cost Paper**
   - Aurora-like / Socrates-like Architectureをモデル化
   - Moderate DurabilityではSocrates-likeがCost-optimalまたはNear-optimalになりやすい
   - Small Dataset + Very High Transaction Rateでは、モデル上Aurora-likeが~70%高コストとなる領域がある
   - Aurora-likeの3-fold DB ReplicationがLarge Cold DataでCost Dominantになり得る
   - Aurora-likeのModel Durabilityは極めて高い
   - Real Aurora Update Workloadでは説明できないScaling Gapを観測

3. **AlloyDB Official Architecture**
   - GoogleはElasticityのためStorage-layer Disaggregationを明示的に採用
   - LPSは独立Scale可能
   - WAL DurabilityとREDO/Page Processingを分離

### 依然不足しているもの

次の条件を揃えたCleanなPeer-reviewed Experimentは存在しない。

```text
Same DB Engine
Same Transaction Semantics
Same Hardware Budget
Same Durability Target
Same Availability Target

Aurora-style Storage
        vs
Socrates-style Storage
        vs
AlloyDB-style Storage
```

したがって、製品レベルの「XがYより速いのはArchitectureのため」という主張には慎重であるべきである。

---

# 33. このセッションでの主要結論: Claim Audit

| Claim | Assessment |
|---|---|
| Traditional WAL DBは通常CommitでDirty Pageをsyncしない | **Correct** |
| Log-based Cloud DBはDirty-page Workを排除する | **Incorrect** |
| Compute-side Dirty-page Network Writeを排除する | **Correct** |
| Storage/Page Nodeは依然PageをMaterialize/Checkpointする | **Correct** |
| AuroraはCompute→StorageでREDO-only Writeを行う | **Supported** |
| Auroraの6-way REDO Replicationは実際のNetwork Amplificationを持つ | **Supported** |
| Page→PG Lookup自体が必ず非常に高コスト | **Not Established** |
| Six-stream Fan-out/Quorum HandlingがForeground Overheadを追加し得る | **Architecturally Plausible; CPU Costは公開定量化されていない** |
| SocratesはCommitでXLOGを待つ | **Incorrect** |
| SocratesはCommitでLZ Durabilityを待つ | **Correct** |
| XLOGはWALをAsync受信し、Hardened WALのみPublishする | **Correct** |
| LZとXLOGは異なるWorkload/NFRのため分離される | **Strongly Supported Interpretation** |
| Socrates Page ServerもCheckpoint/Page Writeを行う | **Correct** |
| SocratesはReplication/Quorum WorkをSystem-wideに排除する | **Incorrect** |
| SocratesはそれらをPrimaryからRemove/Delegateする | **Correct** |
| SocratesはAuroraよりFunctionally Disaggregated | **2025 PaperでSupported** |
| Socratesは常にAuroraよりWriteが速い | **Not Proven** |
| Socrates-likeはModerate DurabilityでしばしばCost-efficient | **Supported by Model** |
| AuroraはModel上非常に高いDurabilityを持つ | **Supported** |
| Auroraの3× Full DB ReplicationはCold Datasetで高コストになり得る | **Supported** |
| AlloyDBはCommit/Page分離という点でSocratesに近い | **Strongly Supported** |
| AlloyDBはMulti-zone Write Amplificationを排除する | **Incorrect** |
| AlloyDBはREDO Processingを独立にElastic化する | **Supported** |
| Azure AZはReal Physical DC Failure Domainではない | **Incorrect Today** |
| Cloud Infrastructure HistoryがDB Architectureに影響した可能性 | **Plausible Hypothesis, Not Proven Causally** |

---

# 34. 最も深いArchitecture Lesson

この進化は、Synchronous Transaction Pathに残す仕事を段階的に減らしていく過程として理解できる。

```text
Traditional Cloud DB
--------------------
Commit:
    Durable WAL

Background on Same DB Instance:
    Page Management
    Checkpoint
    Page Writes


Aurora
------
Commit:
    Partition-aware REDO
    Six-way Storage Fan-out
    Quorum Durability

Background Storage:
    REDO
    Pages


Socrates
--------
Commit:
    One Logical LZ Durable Append

Background:
    XLOG Distribution
    Page Server REDO
    Checkpoint
    XStore


AlloyDB
-------
Commit:
    Regional Durable WAL Append

Background:
    Elastic LPS REDO
    Regional Block Materialization
```

したがって、新しいDesign Metricとして次を考えられる。

> **Transaction Commit Pathから、Physical Page Placement、Replica Topology、REDO Processing、Consumer Disseminationがどの程度見えているか。**

この観点では:

- Traditional RBD: Page WorkはCommit外だがDB Compute自身に残る。
- Aurora: Page WriteはComputeから消えたが、REDO Commit ProtocolはStorage Shard/Quorum-aware。
- Socrates: Page Placement/Disseminationの大部分はCommit Durability Boundaryの外側。
- AlloyDB: DB ComputeはRegional Log Abstractionに対してCommitし、REDO/Block PlacementはElastic Storage Serviceの後段。

---

# 35. 2つ目の深い教訓: “Move” と “Eliminate” を区別する

Cloud-native Optimizationを評価するとき、常に次を問うべきである。

```text
What work disappeared?
What work merely moved?
What work became asynchronous?
What work became independently scalable?
What work became cheaper because it moved to a different tier?
```

たとえば:

| Operation | Aurora | Socrates | AlloyDB |
|---|---|---|---|
| DB PrimaryからのDirty-page Network Traffic | Eliminated | Eliminated | Eliminated |
| Page Materialization | **Moved** to Storage | **Moved** to Page Server | **Moved** to LPS |
| WAL Replication | Still Required | Still Required below LZ | Still Required below Regional Log/Storage |
| Checkpoint-like Work | Transformed / Storage-managed | Page Server → XStore | Storage-layer Materialization |
| Multi-zone Redundancy | Explicit Protection-group Scheme | Underlying Services / Architecture | Explicit 3-zone Block State |
| Foreground Complexity | Conventional Page I/Oより削減、ただしQuorum-aware | Primaryで大幅削減 | Primaryで大幅削減 |

この語彙を使うことで、「Log-based SystemはWrite Amplificationを消す」といった誤解を避けられる。

---

# 36. 3つ目の深い教訓: NFRによるSpecialization

SocratesとAlloyDBの設計から、次の一般原則を導ける。

```text
Do Not Force One Storage Tier to Optimize:

  Synchronous WAL Latency
  + REDO CPU
  + Random Page Serving
  + Cheap Capacity
  + Long Retention
  + High Fan-out
  + Multi-zone Durability
```

代わりに:

```text
Fast Append Tier
    |
REDO / Distribution Compute
    |
Hot Page / Cache Tier
    |
Cheap Durable Capacity Tier
```

へSpecializeする。

これはMemory/Storage Hierarchyに似ているが、**Distributed Database ServiceとそのNFR** に適用したものと考えられる。

---

# 37. 4つ目の深い教訓: DurabilityにはPrice/Performance Frontierがある

AuroraはAggressive Replicationによって非常に高いDurabilityを得ることを示した。

Socrates/AlloyDBは、別の分離点により次を提供し得ることを示した。

- 低いResource Coupling
- より安価なCapacity
- Workload-specific Scaling

したがって、正しい設計上の問いは:

> 「どのDBが最もDurableか？」

ではなく、

> **「要求されるDurability、Availability、Latency、Throughputを同時に満たす最小コストのArchitectureは何か？」**

である。

これは2025年 *OLTP in the Cloud* 論文の多次元的な問題設定そのものである。

---

# 38. Open Research Questions

このセッションでは、公開Literatureで十分に解決されていない領域がいくつか明らかになった。

## 38.1 Aurora Foreground Protocol Cost

次を定量化する必要がある。

```text
CPU(Page -> PG Mapping)
CPU(Destination Dispatch)
CPU(ACK Processing)
CPU(Quorum Frontier)
PPS Overhead
Cross-AZ Serialization Cost
```

そしてWrite-heavy Loadで各要素がどの程度寄与するかを特定する。

---

## 38.2 Storage-side REDO Cost

システム全体で次を追跡する。

```text
Logical WAL Byte
   ->
Network Bytes
   ->
Durable Log Bytes
   ->
REDO CPU Cycles
   ->
Materialized Block Bytes
   ->
Replicated Page Bytes
   ->
GC / Backup Bytes
```

これをAurora、Socrates、AlloyDBで比較する。

単純なLogical TPS比較よりもはるかに有益である。

---

## 38.3 Durability-normalized Price/Performance

公平なBenchmarkでは次をNormalizeすべきである。

```text
Same RPO
Same Durability Probability
Same AZ-failure Guarantee
Same Number of Readable Replicas
Same Backup Retention
Same DB Size
Same Engine Semantics if Possible
```

そうでなければ、「安いArchitecture」が単に弱いFailure Guaranteeを提供しているだけかもしれない。

---

## 38.4 Cloud Infrastructure → DB Architectureの因果関係

有用なResearch Frameworkは次のChainを追跡することである。

```text
Physical DC Topology
        |
Failure-domain Abstraction
        |
Cloud Storage Primitives
        |
Database Durability Boundary
        |
Replication Topology
        |
Write Amplification
        |
Cost / Latency / Elasticity
```

これをAWS、Azure、Google Cloudで比較する。

この因果Chainは直感的には非常に魅力的だが、このセッションで確認した論文からはまだ十分に確立されていない。

---

# 39. 推奨評価フレームワーク

新しいCloud Databaseを比較するときは、「NoSQL」「NewSQL」「Disaggregated」といったラベルだけではなく、次のMatrixを埋めるべきである。

## A. Commit Path

- 同期Durability Boundaryは正確にどのComponentか？
- DB PrimaryはいくつのLogical Network Destinationへ接続するか？
- ACK前に何個のPhysical Replicaが必要か？
- DB PrimaryはReplica Topologyを理解しているか？
- CommitはPage Ownership/Shardingに依存するか？

## B. Log Path

- 誰がWALをPersistするか？
- 誰がWALをDistributeするか？
- DistributionはSyncかAsyncか？
- WAL Filteringは行われるか？
- Lagging Consumerをどう処理するか？
- 高価なStorageと安価なStorageにそれぞれ何日分のWALを保持するか？

## C. Page Path

- 誰がREDOを実行するか？
- 誰がPageをMaterializeするか？
- Hot PageはどこにCacheされるか？
- 誰がCheckpointするか？
- Page Serviceは独立Scale可能か？

## D. Durability / Availability

- どのFailure Domainを許容するか？
- DurabilityはDB-level ReplicationかUnderlying Storageか？
- Complete DB Copyはいくつあるか？
- Availability CopyとDurability Copyは同一か？
- Page Server/LPS Failure時に何が起こるか？

## E. Economics

- どのResourceが独立Scaleするか？
- Cold Dataが高価なMediaに置かれていないか？
- Write Throughput増加時にCapacityもScaleさせられるか？
- Capacity増加時にREDO ComputeもScaleさせられるか？
- Cross-AZ Trafficはどれくらい発生するか？

このTaxonomyは、現代のCloud Databaseを比較するうえで単純な **NoSQL vs. NewSQL** より有用である。

---

# 40. 最終評価

## Aurora

**最も適切な説明**

> DB ComputeからDatabase Page Writeを取り除き、REDO、Page Reconstruction、Recovery、DurabilityをSmart Distributed Storageへ押し込んだ、先駆的な第一世代Cloud-native OLTP設計。

**主な強み**

> 非常に強いMulti-AZ Storage Durabilityと成熟したSelf-healing Smart Storage。

**主なTrade-off**

> 6-way WAL Replicationと、比較的CoupledなLog/Page/Storage Functionにより、コストとIndependent Scalingに制約が生じ得る。

---

## Socrates / Azure SQL Hyperscale

**最も適切な説明**

> Synchronous Log Durability、Asynchronous Log Distribution、Page Processing、Fast Local Cache、Cheap Durable Storageを分離した、より細粒度のArchitecture。

**主な強み**

> Function/NFR-based Disaggregationにより、実Workloadに近い形でResourceをProvisionできる。

**主なTrade-off**

> Distributed Component、Progress State、Backpressure、Failure/Recovery Workflow、Service Coordinationが増える。

---

## AlloyDB

**最も適切な説明**

> Dedicated Regional WAL Service、Horizontally ElasticなLog-processing/Page Server、Sharded Regional Block Storageを備えた、より後発のMulti-layer Disaggregated PostgreSQL Architecture。

**主な強み**

> Primaryからは単純なDurable-WAL Abstractionを見せつつ、REDO ComputeとBlock Capacityを独立にScaleできる。

**主なTrade-off**

> 強い3-Zone Replicated Database Stateのため、System-wideでは依然大きなStorage/REDO Workが必要。

---

# 41. 一文での総括

> **AuroraはPage WorkをDB Computeから外した。SocratesはDurability、Log Distribution、Page Processing、Cheap Storageを別Serviceへ分離した。AlloyDBはさらにREDO/Page-processing Tierを明示的にElastic化した。**

Architecture shorthandでは:

```text
Aurora:
    Compute | Smart Storage

Socrates:
    Compute | Durable Log | Log Distribution | Page Service | Cheap Store

AlloyDB:
    Compute | Regional Log | Elastic REDO/Page Compute | Regional Block Store
```

全体としての教訓は、後発ArchitectureがAuroraの処理を「消した」のではないということである。

> **同じ基本的なDurability/Page-maintenance Responsibilityを、より適切なService Boundaryの後ろへ移し、Latency-sensitive Work、CPU-intensive Work、Capacity-intensive Workを独立に最適化・Scalingできるようにした。**

---

# 参考文献

1. Haowen Dong, Chao Zhang, Guoliang Li, Huanchen Zhang. **Cloud-Native Databases: A Survey.** IEEE TKDE 36(12), 2024. DOI: 10.1109/TKDE.2024.3397508.  
   Survey framing: Compute-Storage Disaggregation、“Log is the Database”、Cloud-native OLTP/OLAP Architecture。

2. Alexandre Verbitski et al. **Amazon Aurora: Design Considerations for High Throughput Cloud-Native Relational Databases.** SIGMOD 2017.  
   https://www.amazon.science/publications/amazon-aurora-design-considerations-for-high-throughput-cloud-native-relational-databases

3. Panagiotis Antonopoulos et al. **Socrates: The New SQL Server in the Cloud.** SIGMOD 2019.  
   https://www.microsoft.com/en-us/research/wp-content/uploads/2019/05/socrates.pdf

4. Michael Haubenschild, Viktor Leis. **OLTP in the Cloud: Architectures, Tradeoffs, and Cost.** The VLDB Journal 34, Article 42, 2025.  
   https://link.springer.com/article/10.1007/s00778-025-00913-z

5. Ravi Murthy, Gurmeet Goindi. **AlloyDB for PostgreSQL under the hood: Intelligent, database-aware storage.** Google Cloud, 2022.  
   https://cloud.google.com/blog/products/databases/alloydb-for-postgresql-intelligent-scalable-storage

6. Jack Hu, Eric Lee, Prashanth Purnananda, Hanuma Kodavalla. **Scaling and Hardening XLOG: The SQL Azure Hyperscale Log Service.** ICDE 2025, pp. 4211–4221. DOI: 10.1109/ICDE65448.2025.00314.

---

## 本レポート内で暗黙的に使用しているEvidence Label

- **Supported** — 引用した論文/ベンダーArchitecture資料で直接記述または測定されている。
- **Modeled** — 2025年の分析コストモデルの結果であり、必ずしも商用製品Benchmarkではない。
- **Inference** — 公開Mechanismから導いたArchitecture Reasoning。測定事実として主張されたものではない。
- **Not proven** — 妥当な仮説ではあるが、確認した公開Evidenceでは不十分。
