# Mooncake Store SSD Offload Bug 修复记录

> 分支: `fix/store-offload-clean` (基于 `b57d18a4` main)
> 仓库: `pingzhuu/Mooncake`
> 最后更新: 2026-07-02

## 背景

store-nvme 部署（.15 master + 4 store-server, .18 4 store-server, 8× NVMe SSD）在 AI_ERA_PRESSURE 压测中出现 offload task 大量 expired（600s TTL 超时），导致 cache hit 率断崖式下跌。经过多轮诊断和修复，共发现并修复了 A-K 共 11 个 bug。

## Bug 列表

### Bug A: evicted_count 不计 offload-deferred keys（BatchEvict 无限迭代）

**文件**: `master_service.cpp` `try_evict_or_offload` / `try_evict_group_or_object`

**根因**: BatchEvict 第二遍（second pass）用 `freed > 0 ? 1 : 0` 判断是否 evict 了 object。但当 key 被 offload-deferred（pin 到 offload queue 而非直接 evict MEMORY replica）时，`freed=0`（数据还留住，只 pin 了），所以 `evicted_objects=0`，`target_evict_num` 不递减 → 第二遍无限迭代处理远超 target 的 key。

**修复**: `try_evict_or_offload` 返回类型从 `uint64_t` 改为 `std::pair<uint64_t, bool>`（freed_bytes, progress_made）。`try_evict_group_or_object` 使用 `progress` flag 作为 `evicted_objects` 计数。

**Commit**: `ff96d065`

**诊断日志**: `[BATCHEVICT-FIRST]` / `[BATCHEVICT-SECONDA]`

---

### Bug B: offload_cap 绑定 offload_force_evict_（cap 完全失效）

**文件**: `master_service.cpp` `try_evict_or_offload`

**根因**: `if (offload_force_evict_ && queued >= cap)` — `offload_force_evict_=false` 时（默认）短路，cap 永远不生效。每轮 BatchEvict 可以无限 push key 到 offload queue。

**修复**: cap 检查与 `offload_force_evict_` 解耦。cap 独立强制执行，到达 cap 时 `return {0, false}`（跳过，不 force-evict，不丢数据）。

**Commit**: `ff96d065`

---

### Bug C: skipped offload keys 静默丢弃 → orphan offloading_tasks

**文件**: `storage_backend.cpp` `GroupOffloadingKeysByBucket` / `file_storage.cpp` `OffloadObjects`

**根因**: store-server 在 `GroupOffloadingKeysByBucket` 中对 `IsExist=true` 的 key（SSD 上已有数据）静默 `continue` 跳过，不回告 master。这些 key 的 `offloading_tasks` 悬挂在 master 端直到 TTL 过期。

触发场景: `remove_all` / master 重启清空了 master 元数据，但 store-server 的 SSD 数据保留。crossing re-put 同一个 key 后 master push 到 offload queue，store-server 发现 key 已存在 → 跳过 → 不回告 → orphan task → expired。

**修复**: `GroupOffloadingKeysByBucket` 收集 skipped keys 到 `skipped_keys` 输出参数。`OffloadObjects` 循环后对 skipped keys 调用 `BatchQuery` 获取已有 metadata，通过 `complete_handler`（即 `NotifyOffloadSuccess`）回告 master，清除 `offloading_tasks` + 减少 MEMORY replica refcnt + 重新注册 LOCAL_DISK replica。

**Commit**: `76720648`

**诊断日志**: `[OFFLOAD-SKIP]` / `[OFFLOAD-GROUP]`

---

### Bug D: partial bucket 不 flush → ungrouped 积压 → TTL expired

**文件**: `storage_backend.cpp` `GroupOffloadingKeysByBucket`

**根因**: 凑不满 `BUCKET_KEYS_LIMIT`（1000）的 key 被存入 `ungrouped_offloading_objects_` 内存池等待下轮凑满。压测高峰过后每轮 heartbeat 只来 151-491 个 key，永远凑不满 1000 → key 的 `offloading_tasks` 悬挂 → TTL 过期。

**修复**: 遍历到 `offloading_objects.cend()` 时，如果 `bucket_keys` 非空，立即输出为 partial bucket（不解散回 ungrouped 池）。

**Commit**: `fd4ecb79`

**诊断日志**: `[OFFLOAD-PARTIAL]`

---

### Bug E: orphan task 诊断日志缺失

**文件**: `master_service.cpp` `NotifyOffloadSuccess` / `PushOffloadingQueue` / `BatchEvict` cleanup

**根因**: 无法定位 orphan task 的来源。

**修复**: 添加 5 类诊断日志:
- `[OFFLOAD-NOTFOUND]` / `[OFFLOAD-RECOVERY]` / `[OFFLOAD-REGISTER]`: NotifyOffloadSuccess 中 key 不在 master
- `[OFFLOAD-NOTASK]`: key 存在但无 offloading_task
- `[PUSH-EMPTY]`: PushOffloadingQueue segment_names 为空
- `[CLEANUP-ORPHAN]`: BatchEvict cleanup 删除有 offloading_task 的 key
- `[EXPIRE-DETAIL]`: expired task 的 metadata_exists 状态

**Commit**: `9b2f7ffc`（后续 Fix G 回退了 NOTFOUND，改为 REGISTER/RECOVERY）

---

### Bug F: 串行 WriteBucket → 单盘吞吐瓶颈

**文件**: `file_storage.cpp` `OffloadObjects`

**根因**: `OffloadObjects` 串行遍历 buckets_keys，逐个调用 `BatchOffload`（→ `WriteBucket` → pwritev）。每个 bucket 是独立文件，无共享写状态，串行化是人为瓶颈。单盘 1.6 GB/s，8 盘聚合 ~12 GB/s，跟不上 10 GB/s 客户端写入。

**修复**: 添加 worker 线程池（默认 4 线程，`MOONCAKE_OFFLOAD_WRITE_THREADS` 可配），per-bucket `BatchOffload` 并行 dispatch。单盘提到 4.5-4.8 GB/s（3×），8 盘聚合 ~37 GB/s。

**Commit**: `ec2f8a12`（cherry-pick 自 `bfda1a9b`）

**效果对比**:
| | 串行 | Parallel (4线程) |
|---|---|---|
| 单盘 | 1.4-1.6 GB/s | 4.5-4.8 GB/s |
| 8盘聚合 | ~12 GB/s | ~37 GB/s |
| SSD 增长峰值 | 8.7 GB/s | 32 GB/s |

---

### Bug G: ~~跳过 AddReplica → SSD 数据孤儿~~（已回退）

**文件**: `master_service.cpp` `NotifyOffloadSuccess`

**方案**: master 无 metadata 时跳过 `AddReplica`，不创建 ghost metadata。

**问题**: SSD 数据在 re-put 前不可用（cache GET 返回 OBJECT_NOT_FOUND），与 remove_all 后 store 重启的 cache 恢复需求冲突。

**回退**: 由 Fix I 替代。`NotifyOffloadSuccess` 恢复无条件 `AddReplica`。

**Commit**: `23dd1edf`（提交）→ `4fd97655`（回退）

---

### Bug H: AddReplica 不替换 stale LOCAL_DISK replica

**文件**: `master_service.cpp` `AddReplica`

**根因**: store-server 重启后 ScanMeta 回告，`AddReplica` 发现已有 LOCAL_DISK replica（旧 client_id）时不替换，只更新 client_id 相同的。旧 replica 等 `ClearInvalidHandles` 异步清理，期间新 replica 未注册。

**修复**: `AddReplica` 先删除所有不同 client_id 的 stale LOCAL_DISK replica，再注册新的。

**Commit**: `23dd1edf`

---

### Bug I: PutStart 的 OBJECT_ALREADY_EXISTS 误拦 LOCAL_DISK-only metadata

**文件**: `master_service.cpp` `PutStart` (line ~2472)

**根因**: PutStart 检查 `metadata.HasReplica(&Replica::fn_is_completed)` — `fn_is_completed` 匹配任意类型的 completed replica（包括 LOCAL_DISK）。store 重启后 ScanMeta 通过 AddReplica 创建了只有 LOCAL_DISK replica 的 metadata，后续 PutStart 同一个 key 被 `OBJECT_ALREADY_EXISTS` 阻塞。

**修复**: 改为 `metadata.HasReplica([](r) { r.is_memory_replica() && r.is_completed() })` — 只匹配 completed MEMORY replica。LOCAL_DISK-only metadata 不阻塞新 PutStart。

**Commit**: `4fd97655`

**验证**: store 重启 + master 重启后，cache GET hit（LOCAL_DISK replica 可读）+ PutStart 成功（不被 ghost 阻塞）。

---

### Bug J: BatchEvict 跨周期重复 pin replica → refcnt 永久泄露

**文件**: `master_service.cpp` `try_evict_or_offload`（BatchEvict + tenant-quota 两个版本）

**根因**: `offloading_tasks` 以 `user_key` 为 key，每个 key 只能有一条记录。但 `try_evict_or_offload` 没有 `offloading_tasks.count(key) > 0` 前置检查。当第一轮 pin 了 replica A（offload 在路上），`FetchOffloadingTasks` 清空 `offloading_objects` 后第二轮可以再 pin replica B → `emplace` 失败（key 已有 task A）→ B 的 `inc_refcnt` 永久泄露。

触发条件: `replica_num ≥ 2`（至少 2 个 MEMORY replica 在不同 segment）。

**修复**:
1. `VisitReplicas` 前加 `offloading_tasks.count(key) > 0` 前置检查，有 in-flight task 时直接 skip
2. `emplace` 检查返回值，失败时 `dec_refcnt` 回退（defense-in-depth）

**Commit**: `a57d27eb`

---

### Bug K: PutEnd offload 路径无 early-exit → 双 replica 都被 pin

**文件**: `master_service.cpp` `PutEnd` (line ~2621, `offload_on_evict=false` 路径)

**根因**: `VisitReplicas` 的 visitor 没有 `if (queued) return;` early-exit。`replica_num=2` 时两个 replica 在不同 segment，`PushOffloadingQueue` 写入不同 `offloading_objects` → 都成功 → 都 `inc_refcnt` → 第二个 `emplace` 失败 → leak。

触发条件: `replica_num ≥ 2` + `offload_on_evict=false`。

**修复**: 加 `queued` early-exit + `emplace` 返回值检查 + `dec_refcnt` 回退。

**Commit**: `a57d27eb`

---

## 上游 cherry-pick 的修复

以下 5 个 commit 从上游分支 cherry-pick，非自研：

| Commit | PR | 描述 |
|--------|-----|------|
| `a982bb32` | PR#2529 (1/3) | promotion worker 从 heartbeat 路径剥离到独立线程 |
| `45ae658e` | PR#2529 (2/3) | tighten promotion worker |
| `1c7d3f18` | PR#2529 (3/3) | tighten promotion drain batch |
| `21ded052` | — | pwritev/preadv 按 UIO_MAXIOV 分块 + errno 诊断 |
| `0ca9f054` | — | segment 重注册后重报 SSD 容量 |

##remove_all 行为说明

`curl -X DELETE http://<prefill>:58080/api/remove_all` 的行为：

- **只删除 lease-expired 的 master 元数据**（HTTP 端点用 `force=false`）
- **不删除 store-server 的 SSD 数据** — master 只是丢掉元数据，SSD 物理文件原封不动
- **不调用 `BatchEvictDiskReplica`** — 不主动清理 LOCAL_DISK replica
- **不清理 `offloading_tasks`** — 留给 TTL sweeper 异步回收
- 客户端侧 `storage_backend_->RemoveAll()` 只删除 prefill 本地文件，不删除 store-server SSD

**与 Fix I 方案兼容**: `remove_all` 清元数据 → store 重启 → ScanMeta 重新注册 LOCAL_DISK replica → cache hit 恢复。

## 关键配置参数

| 参数 | 默认值 | 说明 |
|------|--------|------|
| `put_start_release_timeout_sec` | 600 | offload task TTL，测试时改为 120 |
| `MOONCAKE_OFFLOAD_HEARTBEAT_INTERVAL_SECONDS` | 10 | store-server heartbeat 间隔，测试时改为 5 |
| `MOONCAKE_OFFLOAD_WRITE_THREADS` | 4 | parallel WriteBucket 线程数 |
| `MOONCAKE_OFFLOAD_BUCKET_KEYS_LIMIT` | 1000 | 每 bucket 最大 key 数 |
| `MOONCAKE_OFFLOAD_BUCKET_SIZE_LIMIT_BYTES` | 2GB | 每 bucket 最大字节数 |
| `eviction_high_watermark_ratio` | 0.7 | DRAM 使用率触发 BatchEvict |
| `eviction_ratio` | 0.2 | BatchEvict 目标：DRAM 使用率降到 20% |
| `kOffloadCapRatio` | 0.5 | offload cap = offloading_queue_limit_ × 0.5 |

## 测试脚本

| 脚本 | 用途 |
|------|------|
| `tests/test_store_direct_bench.py` | 直连写入压测（batch_put_from_multi_buffers） |
| `tests/run_store_direct_bench.sh` | 上述脚本的 Docker wrapper |
| `tests/test_store_restart_cache.py` | store 重启 cache 恢复测试（write/get/put 三阶段） |
| `tests/run_restart_cache_test.sh` | 上述脚本的完整流程 wrapper |
| `tests/test_ghost_metadata.py` | ghost metadata 复现测试 |
| `tests/repro_ghost_metadata.sh` | 上述脚本的完整流程 wrapper |

## Wheel 版本

| Wheel | Fixes | 状态 |
|-------|-------|------|
| `mooncake-wheel-fix-ab-*` | A+B | 早期验证 |
| `mooncake-wheel-fix-abcd-*` | A+B+C+D | 早期验证 |
| `mooncake-wheel-fix-abcdef-*` | A+B+C+D+E+F | 性能验证 |
| `mooncake-wheel-fix-abcdefgh-*` | A+B+C+D+E+F+G+H | ghost metadata 验证 |
| `mooncake-wheel-fix-abcdefghi-*` | A+B+C+D+E+F+H+I（G 回退） | cache 恢复验证 |
| `mooncake-wheel-fix-abcdefghi+j+k` | A+B+C+D+E+F+H+I+J+K | 最新（待构建） |
