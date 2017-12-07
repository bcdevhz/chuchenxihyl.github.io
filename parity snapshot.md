parity snapshot

#  0.功能概述

模块功能：创建快照、恢复、网络服务

1. 快速同步快照格式Warp-Sync-Snapshot-Format 

- 扩展了前一版本的全状态快照协议，可快速获取给定区块的状态全拷贝。每3万个区块，节点共识区块的状态。
任何节点可通过网络，以快速同步方式获取这些快照。快照设计乱序恢复--不需要在另一chunk之前得到任何给定的chunk。
- 快照格式含三部分：`block chunks + state chunks + manifest`，其中每一个chunk为压缩后hash,大小为CHUNK_SIZE(4MB)。所有数据默认RLP编码。
- manifest格式见其结构体表示。
- block chunks，注意这里的区块分块格式为POW的Ethash，其他共识引擎各自含区块分块格式。
其含原始区块数据：区块本身+交易收据集。区块以abridged block格式存储，收据集以收据的集合形式存储。

每个区块分块为如下格式的RLP编码集合：
```
[
    number: P, // chunk的第一个区块号
    hash: B_32, // chunk的第一个区块哈希
    td: B_32, // chunk的第一个区块总体难度值
    [abridged_1: AB, receipts_1: RC], // 第一个区块的AB RLP值+收据集
    [abridged_2: AB, receipts_2: RC], // 第二个区块的AB RLP值+收据集
    [abridged_3: AB, receipts_3: RC], // ... 
    ...
]
```
区块需连续。才能在一开始使用元数据将abridged RLP整合到完整区块。
- Abridged block RLP标准区块RLP简化，可从区块分块开始的元数据重现。
- 注意Block Chunk Generation算法。
- State Chunks状态分块及Validity有效性约束。

2. 包含的组件：
1) io  快照io读写操作，支持写单文件及多文件两种方式的文件格式。
2）service  快照网络服务实现
3）account  账户状态(RLP编解码)
4）block    AB区块(原区块裁剪)
5）consensus 共识快照
6）watcher  监听快照相关的链事件
7）error    快照定制的错误类型


#  1. 快照io读写

提供快照的读写方法，快照含两种格式：一种是打包写进单个文件，称作Packed snapshots；一种是在一个目录下写到多个文件，称作loose snapshots。

1. SnapshotWriter trait 写快照，不建议重复写同一个块。

- 含write_state_chunk、write_block_chunk、finish三种方法。
- 其中finish表示写快照完成，manifest chunk列表必须与写进去的chunk一致。
ManifestData结构表示快照元数据，每个快照含唯一可标识的manifest。含字段version、state_hashes、block_hashes、state_root、block_number、block_hash，
分别表示快照格式版本、状态chunk哈希列表(非原始数据，而是压缩数据)、区块chunk哈希列表、最终预期的状态根、快照发生的区块号、快照发生的区块hash。
编解码采用RLP方法(into_rlp/from_rlp)。

2. PackedWriter结构，快照写入单个文件
- 文件格式含三部分：串联的chunk数据；manifest RLP；manifest开始的偏移   为易读会map chunk哈希到lengths+offsets），即ChunkInfo结构(hash, len, offset)。
- 结构字段：file；state_hashes；block_hashes；cur_len  (其中state_hashes、block_hashes为Vec<ChunkInfo>类型)
- new方法依据给出的参数路径，创建一个PackedWriter实例。
- 实现SnapshotWriter的三个方法，write_state_chunk/write_block_chunk依据chunk往file写，依据(hash, len, offset)填充state_hashes/block_hashes。
finish依据ManifestData结构字段获取RlpStream并分割字段写进file。

3. LooseWriter结构，快照写入一个路径下的多个文件
- 仅含一个类型为PathBuf的dir字段。
- new方法依据给出的PathBuf参数路径，创建一个LooseWriter实例。
- 实现SnapshotWriter的三个方法；write_state_chunk/write_block_chunk的写逻辑一致，都会调用本身的write_chunk：
先把hash推进dir对应路径，该路径下创建新文件；chunk写入。finish：路径加字符串MANIFEST；创建新文件；将参数manifest RLP编码后，写入该文件。

4.  SnapshotReader trait 读快照
- 两个方法：manifest从快照中获取ManifestData结构；chunk依据参数hash获取chunk数据。

5. PackedReader结构，读单文件快照
- 字段：file、state_hashes、block_hashes、manifest
- new方法依据路径，创建一个PackedReader实例，无效的单文件快照会报错。按字节读文件，按偏移算manifest_off，解码成UntrustedRlp，获取ManifestData字段填充该结构
及PackedReader结构。
- 实现SnapshotReader两方法：manifest直接返回ManifestData，chunk依据hash获取状态或区块的(len, off)，并依此读file。

6. LooseReader结构，读多文件快照
- 两个字段：PathBuf结构的dir,ManifestData结构的manifest。
- 默认实现new,实现SnapshotReader两方法。类似PackedReader。

#  2.快照网络服务实现

1. 结构Guard(bool, PathBuf)，辅助发生错误删除路径。self.0为true删路径，实现new、benign、disarm及Drop的drop方法。

2. trait DatabaseRestore外部数据库恢复处理，restore_db获取数据库的所有权并转移到新位置。

3. Restoration状态恢复管理器结构，RestorationParams状态恢复依赖的参数结构。
- new,参数RestorationParams，获取其manifest，获取raw_db+genesis构造chain，获取engine.snapshot_components()构建secondary。
- feed_state，状态snappy_buffer的len来state.feed，写文件writer.write_state_chunk，删对应hash block_chunks_left.remove。
- feed_blocks，类似feed_state。第二chunk secondary.feed,writer.write_block_chunk,block_chunks_left.remove。
- finalize完成恢复，检查final_state_root一致，检查缺失的code self.state.finalize(),检查chunk及识别链完整性self.secondary.finalize(engine)。
完成manifest元数据写writer.finish，最终guard.disarm()置为false。
- is_done，确保区块及state都处理完成，检查block_chunks_left及state_chunks_left是否为空。

4. Service快照服务实现，控制所持快照及从快照恢复；ServiceParams快照服务参数
- new，参数ServiceParams。snapshot_dir表示current快照路径，temp_snapshot_dir表示in_progress临时快照路径，restoration_dir获取restoration恢复路径，
restoration_db恢复db路径，temp_recovery_dir临时temp快照恢复路径，replace_client_db将self.db_restore.restore_db恢复成self.restoration_db()。
- reader，tick，take_snapshot，init_restore，finalize_restoration
- feed_chunk： feed_state/feed_blocks;fetch_add;finalize_restoration
- feed_state_chunk/feed_block_chunk调feed_chunk。
- 上述均为默认实现。还实现了SnapshotService trait方法。及Drop的drop方法。

5. SnapshotService trait，快照网络服务结构的接口
- 处理：快照恢复到临时数据库；快照manifests及chunks查询的响应。
- manifest返回最近元数据ManifestData。supported_versions获取支持的快照版本号范围，None表示快速同步共识不支持。
- chunk获取指定hash的chunk数据。status询问快照服务，恢复的状态值RestorationStatus。
- begin_restore依据ManifestData开始恢复。若状态为in-progress，重置。开始后之前的快照无效。
- abort_restore中止一个状态为in-progress的恢复restoration。
- restore_state_chunk/restore_block_chunk喂一条初始state/block chunk给服务，异步处理，若当前恢复则不操作。

#   3.账户状态编解码
- 定义了一个BasicAccount结构类型的空数据常量ACC_EMPTY。
- 定义枚举类型CodeState，Empty表示账户未编码、Inline表示编码源数据、Hash表示编码hash。默认简单实现from(u8转枚举)及raw(转u8)两方法。

- 编码为to_fat_rlps方法，参数account_hash+BasicAccount+AccountDB+used_code+first_chunk_size+max_chunk_size
遍历账户存储树，返回RLP编码后的条目集：含账户地址hash、账户特性(nonce,balance,[has_code, code_hash])、存储(TrieDB中节点k-v)。每个条目含最多max_storage_items条存储记录，
这些记录依据快照定义格式分割。

- 解码为from_fat_rlp方法，重新构造村存储树，返回更新代码后的账户结构。
依据AccountDBMut+storage_root恢复存储树，依据UntrustedRlp中k-v对作为节点插入到树木中。返回BasicAccount(依据树)+new_code字节序列(基于code_state)。

#  4.区块RLP压缩
- AbridgedBlock字节数组结构体，只有一个字节数组字段。
- 实现方法有from_raw创建一个对象，into_inner返回rlp数据成员。
- from_block_view依据区块RLP视图中的父hash及区块号，添加区块结构中部分字段，产生新的rlp编码值。
- to_block依据提供父hash及区块号，receipts_root及rlp字段构建一个简化区块(header+transactions+uncles)。

#  5.共识快照

#  6.监听快照相关的链事件
- 辅助的Oracle trait含两个方法：to_number(区块hash转number)、is_major_importing(检查是否同步)。
- StandardOracle 结构含client及sync_status两个字段，实现了Oracle trait。

- 辅助的Broadcast trait含方法take_at，根据所要快照的区块number广播该区块。
- IoChannel<ClientIoMessage>结构含两个字段：channel发送消息(ClientIoMessage外内部多种事件消息类型)，handlers(回调中获取消息)。
-  该结构实现Broadcast trait，send的消息类型为枚举消息中的TakeSnapshot。

- ChainNotify trait表示需要处理的actor监听链事件，含new_blocks/start/stop/broadcast/transactions_received五种。
- Watcher结构类型含oracle、broadcast、period、history四个字段。
- 默认new方法每个period区块触发快照事件，该区块后为history区块。
- 其实现ChainNotify trait的new_blocks方法，表示在某个区块触发快照事件。若未同步，找到最新区块，广播需快照的该区块(依据period+history算高度，调take_at广播)。

#  7.enum Error
- 为快照模块定义了包含错误的种类，不完整的链、无效或太多等不满足快照要求的区块、错误的状态根、共识算法不支持的快照等。
- 实现fmt::Display的fmt提供错误打印信息。io::Error、TrieError、DecoderError其他几种错误到该错误的转换(from函数)。







