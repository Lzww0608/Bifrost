### 全年研发路线图

#### 第一季度：骨架搭建
**核心目标：验证架构，打通从“文件输入”到“问答输出”的数据流。**

*   **第 1 个月：C++ 向量数据库原型 (Vector DB V1)**
    *   **算法设计**：实现一个 **Brute-Force（暴力搜索/全内存扫描）** 引擎。对于几万条以内的 Chunk，暴力计算余弦相似度（配合 SIMD 指令或 Eigen 库）在毫秒级就能搞定。
    *   **接口设计**：用 C++ 配合 gRPC 实现三个接口：`Insert(Vector, Metadata)`，`Search(Vector, TopK)`，`SaveToDisk()`。
    *   **持久化**：简单的二进制文件读写，启动时全量加载到内存。
*   **第 2 个月：Go 文件系统 **
    *   引入 `fsnotify` 监听指定目录的 `.md` 和 `.txt`。
    *   实现基础的 **固定长度切片（Fixed-size Chunking）**，比如 500 Token 一切，保留 50 Token 重叠（Overlap）。
    *   调用大厂 API（如 DeepSeek/智谱的 Embedding API）将文本转向量。
    *   通过 gRPC 将向量存入 C++ 引擎，并在 Go 端（可使用 SQLite）保存对应的 Chunk 文本和文件元数据。
*   **第 3 个月：Vibe Coding 驱动前端与联调 **
    *   使用 AI（Cursor/Cline）快速生成 Next.js + TailwindCSS + shadcn/ui 的聊天界面。
    *   Go 实现大模型问答接口（调用 OpenAI 兼容格式的外部大模型 API），拼接 Retrieval 的上下文，实现 SSE (Server-Sent Events) 流式打字机效果输出。
    *   **效果测试**：你把笔记丢进文件夹，前端立刻能基于你的笔记回答问题。



#### 第二季度：V0.5 检索升级，RAG 的工程深化 (The Muscle)
**核心目标：解决“搜不准”和“纯向量检索的局限”。**

*   **第 4 个月：C++ 引擎算法升级 (Vector DB V2)**
    *   **索引升级**：在 C++ 中实现 **IVF-Flat（倒排索引+聚类）**，引入 K-Means 对向量进行分桶。这会极大锻炼你的 C++ 算法能力。
    *   **内存优化**：引入 `mmap` (内存映射文件)，不再全量加载内存，让 OS 管理缓存，支持更大规模的数据。
*   **第 5 个月：Go 文档解析与高级切片 (Advanced Parsing)**
    *   支持 PDF 解析：使用 Go 的 PDF 库提取纯文本。
    *   支持 Markdown/代码的高级切片：按二级标题、三级标题进行 **语义切片 (Semantic Chunking)**，而不是生硬的按字数切，保证知识的完整性。
*   **第 6 个月：混合检索架构 (Hybrid Search)**
    *   纯向量检索对专有名词（如报错代码、特定人名）效果很差。
    *   在 Go 端利用 SQLite 的 FTS5（全文检索）引擎实现关键词搜索。
    *   实现 **Reciprocal Rank Fusion (RRF)** 算法，将 C++ 返回的向量得分和 SQLite 返回的关键词得分进行融合（Re-ranking），大幅提升检索精准度。

#### 第三季度：V1.0 斩断网线，纯本地与极致隐私 (The Brain)
**核心目标：彻底告别 API，利用本地 CPU/GPU 跑满算力。**

*   **第 7 个月：C++ 融合 llama.cpp (Local Inference)**
    *   将 `llama.cpp` 作为子模块（Submodule）集成到你的 C++ 项目中。
    *   在 C++ 端暴露新的 gRPC 接口：`ChatCompletion` 和 `CreateEmbedding`。
    *   下载量化版的模型（如 Llama-3-8B-Instruct.gguf 和 BGE-M3-Embedding.gguf）放在本地。
*   **第 8 个月：Go 服务平滑迁移**
    *   修改 Go 的配置文件，将外部大模型 API 的请求，无缝切换到本地 C++ 提供的 gRPC 接口。
    *   调优本地推理性能（如设置正确的 CPU 线程数，或开启 CUDA/Metal 加速）。
*   **第 9 个月：打通多元私人数据源**
    *   **微信/聊天记录**：解析 PC 微信的 SQLite 数据库，将其清洗为 Q&A 对话丢入向量库。
    *   **图片/OCR**：引入轻量级 OCR 引擎（本地运行），让你截图保存的笔记也能被搜索。
    *   **里程碑**：即使拔掉网线，你的 AI 依然博学，且掌握你所有的数字隐私。

#### 第四季度：V2.0 个人智能体操作系统 (Agentic OS)
**核心目标：从“你问它答”变成“它帮你做”。**

*   **第 10 个月：工具调用框架 (Function Calling)**
    *   在 Go 端实现一套插件系统。定义标准的 Tool Schema（如 `search_local_file`, `send_email`, `execute_bash`）。
    *   利用大模型的 Function Calling 能力（Llama-3 支持），让 Go 解析大模型的意图，并在本地执行具体操作。
*   **第 11 个月：系统级自动化打通**
    *   **邮件管理 Agent**：定时读取邮件，分类并自动生成回复草稿。
    *   **终端助手**：前端输入自然语言，Go 翻译成 Shell 脚本并在沙盒中执行（比如“帮我把下载文件夹里大于 1G 的视频找出来并删除”）。
*   **第 12 个月：长记忆与主动思考 (Memory & Background Tasks)**
    *   引入 **Mem0** 的概念：让系统自动提取你日常对话中的“用户画像”（比如“主人不喜欢冗长的回答”、“主人常用 Go 语言”），作为全局 Prompt。
    *   后台驻留：Go 守护进程在系统空闲时，自动对知识库进行交叉总结，生成知识图谱卡片推送到前端。
    *   **里程碑**：系统具备了主动性，成为真正的 OS (Operating System) 级 AI 助手。

---

### 🛠️ 技术栈推荐与避坑指南

#### 1. C++ 端（追求极致性能）
*   **网络框架**：推荐 `gRPC` 或 `brpc`（百度开源，非常适合 C++）。
*   **向量计算加速**：不要自己写 for 循环算内积！使用 `Eigen` 库或者 `OpenBLAS`，开启编译器的 `-O3 -march=native` 优化。
*   **大模型推理**：`llama.cpp` 是绝对的首选，它的 C API 极其稳定，且对各平台的硬件加速支持最完善。
*   **坑点**：内存泄漏。写向量数据库时，注意多线程并发插入时的锁粒度，推荐使用 `std::shared_mutex` 做读写锁。

#### 2. Go 端（追求工程化与高并发）
*   **Web 框架**：推荐 `Fiber` 或 `Gin`。
*   **元数据存储**：直接用 `SQLite`（配合 GORM 或 sqlc），单文件、免安装，非常符合本地软件的调性。
*   **RAG 编排**：前期可以参考 `LangChain-Go` 的源码，但强烈建议**不要直接用 LangChain**，自己手写编排逻辑，代码会清爽 100 倍。
*   **坑点**：处理大量大文件（如 500 页 PDF）时，注意控制 Go 的 Goroutine 并发数，避免短时间内大模型 API 限流或内存被打爆（使用 `worker pool` 模式）。

#### 3. Frontend 端（轻量化）
*   **框架**：Next.js (App Router) + TypeScript。
*   **样式**：Tailwind CSS + shadcn/ui。
*   **状态管理**：Zustand。
*   **开发策略**：千万不要在这里耗费过多精力，把需求写清楚，直接用 Cursor 帮你生成页面代码。

### 🌟 写在最后

这个项目的魅力在于它的**厚积薄发**。
不要试图在第一个月就写出一个完美的 HNSW 数据库，也不要一开始就死磕本地 LLM。**“先让系统跑起来，哪怕它很蠢”**，这是敏捷开发的核心。

建议你现在就去建一个 Github 仓库，写下第一版 `README.md`，然后从 **Go 写一个监听文件夹的简单脚本，C++ 写一个接收 float 数组的 gRPC Server** 开始。

一年后，你不仅将精通 Go 的并发工程、C++ 的高性能计算，你还会拥有一个市面上买不到的、最懂你的 AI 第二大脑。祝你好运！遇到具体的代码实现问题（比如 IVF-Flat 怎么写，Go 怎么做语义切片），随时来探讨！