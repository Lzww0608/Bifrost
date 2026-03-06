# 第 1 个月详细计划：C++ 向量数据库原型（Vector DB V1）

> 对齐 `plan.md` 的第一月目标：完成 Brute-Force 向量检索原型，提供 RPC 接口（统一使用 brpc），并支持落盘/重启恢复。

## 0. 月度目标与验收标准

### 0.1 月度目标（必须达成）
1. 完成一个可运行的 C++ 向量数据库服务（单机版）。
2. 支持 3 个 RPC 接口（基于 brpc + protobuf）：
   - `Insert(Vector, Metadata)`
   - `Search(Vector, TopK)`
   - `SaveToDisk()`
3. 检索算法为 Brute-Force + 余弦相似度，支持至少 3~5 万条向量数据。
4. 支持二进制持久化，重启后数据可恢复且检索结果一致。
5. 提供基础压测与正确性测试报告（可复现实验命令）。

### 0.2 验收门槛（Definition of Done）
- 功能完整：插入、检索、保存、加载全部可用。
- 正确性：
  - 对同一测试集，重启前后 TopK 结果一致。
  - 与 Python/Numpy 基准脚本比对，TopK 命中率 ≥ 99.9%。
- 性能：
  - 5 万向量、维度 768、TopK=10，P95 检索耗时在可接受范围（建议先以 <100ms 为阶段目标）。
- 工程性：
  - 支持日志、配置文件、错误码。
  - 关键模块有单元测试。

---

## 1. 技术方案细化

### 1.1 数据结构设计
- `VectorRecord`
  - `id: uint64`
  - `embedding: std::vector<float>`（后续可替换为对齐内存）
  - `metadata: std::string`（先用 JSON 字符串，不做结构化解析）
- `VectorStore`
  - `std::vector<VectorRecord> records_`
  - `size_t dimension_`
  - `mutable std::shared_mutex rw_lock_`

### 1.2 核心算法
- 相似度函数：
  - 使用 cosine similarity：`dot(a,b)/(||a||*||b|| + eps)`。
- 检索流程：
  1. 输入 query 向量，校验维度。
  2. 全量扫描 `records_`，计算相似度。
  3. 使用小顶堆维护 TopK（避免全量排序）。
  4. 输出按分值降序排列结果。
- 性能优化（V1 可选但建议做）：
  - 预先缓存每条向量范数 `norm`。
  - 编译选项 `-O3 -march=native`。
  - 使用 Eigen 进行向量内积计算。

### 1.3 RPC 接口草案（brpc + protobuf）
- `InsertRequest`
  - `repeated float vector`
  - `string metadata`
- `InsertResponse`
  - `uint64 id`
  - `int32 code`
  - `string message`
- `SearchRequest`
  - `repeated float query`
  - `int32 top_k`
- `SearchResponse`
  - `repeated SearchResult results`（含 `id`, `score`, `metadata`）
- `SaveRequest` / `SaveResponse`

### 1.4 持久化格式
- 文件头（magic + version + dim + count）。
- 每条记录：
  - `id`
  - `norm`
  - `vector bytes`
  - `metadata length + metadata bytes`
- 策略：
  - `SaveToDisk()` 显式触发全量快照。
  - 启动时 `LoadFromDisk()`。
  - V1 不做 WAL，先保证简单可靠。

---

## 2. 按周拆解（Week-by-Week）

## Week 1：工程骨架与最小可运行链路

### 目标
- 建好项目结构、构建系统、proto 定义，跑通“插入 + 查询”的最小链路。

### 任务清单
1. 项目初始化
   - 目录结构：`/proto`, `/src`, `/include`, `/tests`, `/bench`。
   - CMake 工程建立；接入 brpc、protobuf、Eigen（或先预留）。
2. proto 设计与代码生成
   - 完成 `vectordb.proto`。
   - 配置自动生成命令，确保 CI/本地可重复。
3. 核心类空实现
   - `VectorStore` / `SearchEngine` / `PersistenceManager`。
4. 最小功能打通
   - `Insert`：支持向内存数组追加。
   - `Search`：先写朴素全排序版本，确保功能正确。
5. Demo Client
   - 提供一个简单 CLI 客户端，验证可调用。

### Week 1 交付物
- brpc 服务可启动。
- 可插入 100 条向量并返回 TopK 结果。

---

## Week 2：算法正确性与基础性能优化

### 目标
- 将检索逻辑升级为“可用版本”：正确性稳定、性能可观测。

### 任务清单
1. 余弦相似度实现
   - 支持维度校验、零向量保护。
2. TopK 优化
   - 由全排序改为小顶堆维护 TopK。
3. 范数缓存
   - 插入时计算并存储 norm。
4. 单元测试
   - 相似度测试（已知向量对）。
   - TopK 边界测试（TopK=1, TopK>N, 空库）。
   - 维度错误测试。
5. 基准测试
   - 构造 1k/10k/50k 数据集，输出平均耗时、P95。

### Week 2 交付物
- 正确性测试通过。
- 输出首版性能报告（markdown 文档）。

---

## Week 3：持久化与恢复一致性

### 目标
- 让服务具备“可重启”的基本生产能力。

### 任务清单
1. 二进制序列化
   - 设计文件头与版本号。
2. SaveToDisk 实现
   - 写临时文件 + rename（避免半写入损坏）。
3. 启动加载
   - 服务启动时自动尝试加载快照。
4. 一致性测试
   - 插入数据 -> 保存 -> 重启 -> 检索结果比对。
5. 异常处理
   - 文件损坏、版本不匹配、维度不一致错误码。

### Week 3 交付物
- Save/Load 全流程可跑。
- 恢复后一致性测试报告。

---

## Week 4：稳定性收敛、文档与里程碑验收

### 目标
- 补齐工程细节，形成可演示、可交接的 V1 原型。

### 任务清单
1. 并发控制
   - Insert / Search / Save 的读写锁策略明确化。
2. 日志与可观测性
   - 记录 QPS、查询耗时、错误码计数。
3. 配置化
   - 端口、数据目录、维度、最大 TopK 等支持配置文件。
4. 回归测试
   - 功能 + 持久化 + 并发场景。
5. 文档收敛
   - README：构建、运行、压测命令。
   - 设计文档：数据结构、协议、已知限制。

### Week 4 交付物（最终）
- 可复现的一键启动和演示脚本。
- 月度总结报告：功能、性能、问题与下月改进点。

---

## 3. 按天粒度执行建议（22 个工作日示例）

- **D1-D2**：CMake + 三方依赖 + 工程目录。
- **D3-D4**：proto 定义、代码生成、服务框架。
- **D5**：Insert MVP。
- **D6-D7**：Search MVP（朴素版）。
- **D8-D9**：余弦相似度与 TopK 小顶堆优化。
- **D10**：单元测试第一轮。
- **D11-D12**：基准测试脚本与指标落表。
- **D13-D14**：持久化文件格式 + SaveToDisk。
- **D15**：LoadFromDisk + 启动恢复。
- **D16**：重启一致性回归。
- **D17-D18**：并发读写锁与竞态修复。
- **D19**：日志、错误码、配置化。
- **D20**：端到端压测（50k 数据）。
- **D21**：文档完善、演示脚本。
- **D22**：月度验收、风险复盘、Q2 接口预研。

---

## 4. 风险清单与缓解方案

1. **性能不达预期（Brute-Force 扫描慢）**
   - 缓解：先做 norm 缓存 + TopK 堆 + O3 优化；必要时降 metadata 返回字段。
2. **内存占用过高**
   - 缓解：metadata 先只存短字符串；限制单条 metadata 大小；增加容量观测。
3. **持久化文件损坏**
   - 缓解：临时文件写完再原子替换；加 magic/version/checksum（可选）。
4. **并发读写冲突**
   - 缓解：读写锁分离；Save 时短时写锁冻结；压测发现热点。
5. **接口频繁变更影响 Go 端对接**
   - 缓解：Week 1 冻结 proto v1；新增字段仅追加不破坏兼容。

---
