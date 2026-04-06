# PDF RAG Agent

基于 RAG (Retrieval-Augmented Generation) 的英文 PDF 技术文档问答与代码生成工具。支持在线 API 和本地离线 LLM 两种模式。

---

## 快速开始

### 在线 API 模式

在 `.env` 文件中配置 API 凭据：

```bash
LLM_API_BASE=https://api.example.com/v1
LLM_API_KEY=sk-xxx
LLM_MODEL_NAME=gpt-4
```

运行：

```bash
python pdf_rag_agent.py --doc manual.pdf
```

### 本地离线模式

无需 API，使用内置 Ollama + Qwen2.5:3b 模型：

```bash
python pdf_rag_agent.py --doc manual.pdf --local
```

### 便携包模式

将 `dist/pdf_rag_agent/` 整个文件夹复制到目标机器，直接运行：

```bash
pdf_rag_agent.exe --doc manual.pdf --local
```

无需安装任何软件，无需网络连接。

---

## 命令行参数完整说明

### 基础参数

| 参数 | 类型 | 默认值 | 说明 |
|------|------|--------|------|
| `--doc` | 文件路径 (多个) | `manual_en.pdf` | 输入的 PDF 文件路径 |
| `--api-base` | 字符串 | .env: `LLM_API_BASE` | LLM API 基础 URL |
| `--api-key` | 字符串 | .env: `LLM_API_KEY` | LLM API 密钥 |
| `--model` | 字符串 | .env: `LLM_MODEL_NAME` | LLM 模型名称 |

**`--doc`** : 指定一个或多个 PDF 文件。多个文件会合并建立统一的向量库。

```bash
# 单个文件
python pdf_rag_agent.py --doc datasheet.pdf

# 多个文件
python pdf_rag_agent.py --doc manual1.pdf manual2.pdf appendix.pdf
```

**适用场景**：所有场景必须指定至少一个 PDF 文件。多个文件适合需要跨文档查询的情况。

---

**`--api-base`** / **`--api-key`** / **`--model`** : 覆盖 .env 中的 API 配置。

```bash
python pdf_rag_agent.py --doc doc.pdf --api-base https://api.openai.com/v1 --api-key sk-xxx --model gpt-4o
```

**适用场景**：需要临时切换 API 提供商或模型时使用，不想修改 .env 文件。

---

### 本地 LLM 参数

| 参数 | 类型 | 默认值 | 说明 |
|------|------|--------|------|
| `--local` | 开关 | 关闭 | 强制使用本地 Ollama LLM |
| `--local-model` | 字符串 | `qwen2.5:3b` | Ollama 模型名称 |

**`--local`** : 启用本地 LLM 模式，忽略所有 API 配置。自动启动 Ollama 服务。

```bash
python pdf_rag_agent.py --doc manual.pdf --local
```

**适用场景**：无网络环境、数据安全要求高（不希望数据离开本机）、或 API 不可用时。

---

**`--local-model`** : 指定 Ollama 模型。若模型未下载，非便携模式下会自动拉取。

```bash
# 使用更大的模型获得更好的回答质量
python pdf_rag_agent.py --doc manual.pdf --local --local-model qwen2.5:7b

# 使用更小的模型以提高速度
python pdf_rag_agent.py --doc manual.pdf --local --local-model qwen2.5:1.5b
```

**适用场景**：默认的 3b 模型在精度和速度之间取得平衡。内存充裕（>8GB）时可选用 7b 获得更详细的回答；内存紧张或需要快速响应时可选 1.5b。

---

### Ollama CPU 推理优化参数

以下参数仅在 `--local` 模式下生效，用于精细控制 Ollama 推理引擎的资源分配。

| 参数 | 类型 | 默认值 | 说明 |
|------|------|--------|------|
| `--num-thread` | 整数 | `8` | CPU 推理线程数 |
| `--num-gpu` | 整数 | `0` | GPU 层数 (0=纯 CPU) |
| `--num-ctx` | 整数 | 按模型自动选择 | 上下文窗口大小 (token) |
| `--ollama-timeout` | 整数 | `300` | 请求超时时间（秒） |
| `--num-parallel` | 整数 | `1` | Ollama 并发请求槽位 |
| `--max-loaded-models` | 整数 | `1` | 内存中同时保持的最大模型数 |
| `--no-flash-attention` | 开关 | 关闭 (默认启用) | 禁用 flash attention 优化 |
| `--local-top-k-cap` | 整数 | `4` | 本地 LLM 自动调优 top_k 上限 |
| `--local-codegen-top-k-cap` | 整数 | `8` | 本地 LLM 自动调优 codegen_top_k 上限 |

---

**`--num-thread`** : 控制 Ollama 推理使用的 CPU 线程数。应设置为**物理核心数**（非超线程逻辑核心数），超线程可能引起竞争反而降速。

```bash
# i7-9700: 8 物理核心
python pdf_rag_agent.py --doc manual.pdf --local --num-thread 8

# Ryzen 5 5600: 6 物理核心
python pdf_rag_agent.py --doc manual.pdf --local --num-thread 6

# 低功耗笔记本: 4 核
python pdf_rag_agent.py --doc manual.pdf --local --num-thread 4
```

**调优策略**：
- 通过任务管理器或 `lscpu` 查看物理核心数
- 若系统同时运行其他重负载程序，可适当减少 1-2 个线程
- 设置过高（超过物理核心）不会加速，反而增加上下文切换开销

---

**`--num-gpu`** : 控制模型有多少层卸载到 GPU 运行。`0` 表示纯 CPU 推理，`99` 表示尽可能全部卸载到 GPU。

```bash
# 纯 CPU（无独立 GPU 或仅集显）
python pdf_rag_agent.py --doc manual.pdf --local --num-gpu 0

# 有 NVIDIA GPU，全部层卸载
python pdf_rag_agent.py --doc manual.pdf --local --num-gpu 99

# 部分卸载（GPU 显存不足以放下整个模型时）
python pdf_rag_agent.py --doc manual.pdf --local --num-gpu 20
```

**调优策略**：
- 无独立 GPU 或仅有集显：保持 `0`
- 有 NVIDIA GPU (4GB+ VRAM)：设为 `99` 让 Ollama 自动分配
- GPU 显存不足时会报错，此时逐步减小数值直到不报错
- 每层约占 100-200MB 显存（取决于模型大小）

---

**`--num-ctx`** : 覆盖上下文窗口大小（以 token 计）。不指定时使用内置的模型级别配置：

| 模型 | 默认 num_ctx |
|------|-------------|
| qwen2.5:3b / qwen3:4b / phi4-mini / gemma3:4b | 4096 |
| gemma4:e2b / gemma4:e4b | 2048 |

```bash
# 扩大上下文（需要更多 RAM，适合长文档检索）
python pdf_rag_agent.py --doc manual.pdf --local --num-ctx 8192

# 缩小上下文（减少内存占用，加速推理）
python pdf_rag_agent.py --doc manual.pdf --local --num-ctx 1024
```

**调优策略**：
- 上下文窗口越大，能容纳的检索块越多，但 RAM 消耗和推理延迟也越高
- 公式估算：`num_ctx` 需容纳 `top_k * chunk_size / 4 + max_tokens + system_prompt`（约 500 token）
- 8GB RAM 建议 <=2048，16GB 建议 <=4096，24GB+ 可尝试 8192
- 设置过大导致 OOM 时，Ollama 会自动降级但性能不可预测

---

**`--ollama-timeout`** : 单次 LLM 请求的超时时间（秒）。CPU 推理较慢时需适当增大。

```bash
# 较慢的 CPU，增大超时
python pdf_rag_agent.py --doc manual.pdf --local --ollama-timeout 600

# 快速失败检测（调试用）
python pdf_rag_agent.py --doc manual.pdf --local --ollama-timeout 60
```

**调优策略**：
- 默认 300 秒适合大多数场景
- 如果频繁出现超时错误，说明模型/上下文对当前 CPU 过大，优先减小 `--num-ctx` 或换用更小模型
- 设置过短会导致长回答被截断

---

**`--num-parallel`** : Ollama 服务端的并发处理槽位。每增加 1 个槽位，额外占用约 `num_ctx * model_size_factor` 的内存。

```bash
# 单用户使用（默认，最省内存）
python pdf_rag_agent.py --doc manual.pdf --local --num-parallel 1

# 允许 2 个并发请求
python pdf_rag_agent.py --doc manual.pdf --local --num-parallel 2
```

**调优策略**：内存不足时保持 `1`。仅当 RAM 充裕（>32GB）且需要并发查询时增大。

---

**`--max-loaded-models`** : 同时在内存中保持的模型数量。多模型占用大量 RAM。

```bash
# 单模型（默认，推荐）
python pdf_rag_agent.py --doc manual.pdf --local --max-loaded-models 1

# 同时保持 2 个模型（需要大量 RAM）
python pdf_rag_agent.py --doc manual.pdf --local --max-loaded-models 2
```

**调优策略**：除非需要频繁在不同模型间切换且 RAM 充裕（>32GB），否则保持 `1`。

---

**`--no-flash-attention`** : 禁用 flash attention 优化。Flash attention 默认开启，可减少内存占用并加速推理。

```bash
# 如果 Ollama 报 flash attention 相关错误，可禁用
python pdf_rag_agent.py --doc manual.pdf --local --no-flash-attention
```

**调优策略**：正常情况下无需使用。仅在 Ollama 日志报告 flash attention 兼容性问题时使用。

---

**`--local-top-k-cap`** / **`--local-codegen-top-k-cap`** : 控制本地 LLM 模式下自动调优的 top_k 和 codegen_top_k 上限值。

```bash
# 放宽检索上限（更多上下文，但可能超出 num_ctx）
python pdf_rag_agent.py --doc manual.pdf --local --local-top-k-cap 6 --local-codegen-top-k-cap 12

# 收紧上限（加速推理，减少 token 消耗）
python pdf_rag_agent.py --doc manual.pdf --local --local-top-k-cap 3 --local-codegen-top-k-cap 6
```

**调优策略**：
- 上限值仅影响自动调优的结果，通过 `--top-k` 显式指定的值不受此限制
- 增大上限前需确保 `num_ctx` 足够容纳更多检索块
- 计算公式：`top_k * chunk_size / 4` 应 < `num_ctx * 0.6`（留 40% 给输出和 prompt）

---

### 典型硬件配置推荐

| 硬件配置 | 推荐参数 | 说明 |
|----------|----------|------|
| **i7-9700 (8核/24GB/无GPU)** | 默认配置即可 | 无需额外参数 |
| **Ryzen 5 5600 (6核/16GB/无GPU)** | `--num-thread 6 --num-ctx 2048 --max-tokens 1024` | 减少线程和上下文匹配资源 |
| **低内存 (4核/8GB/无GPU)** | `--num-thread 4 --num-ctx 1024 --local-top-k-cap 3 --max-tokens 512 --local-model qwen2.5:1.5b` | 最小化内存压力 |
| **有 GPU (8核/16GB/RTX3060 12GB)** | `--num-gpu 99 --num-ctx 8192 --local-top-k-cap 8` | 全部层卸载 GPU，扩大上下文 |

---

### Embedding 参数

| 参数 | 类型 | 默认值 | 说明 |
|------|------|--------|------|
| `--embedding` | `local` / `api` | `local` | Embedding 模型类型 |
| `--embedding-model` | 字符串 | 见下方 | 自定义 embedding 模型名称 |

**`--embedding`** : 选择 embedding 模型来源。

- `local`（默认）：使用 HuggingFace 本地模型 `all-MiniLM-L6-v2`，无需网络。
- `api`：使用 OpenAI 兼容 API 的 embedding 模型。

```bash
# 本地 embedding（默认，离线可用）
python pdf_rag_agent.py --doc manual.pdf --embedding local

# API embedding（需要网络，质量可能更高）
python pdf_rag_agent.py --doc manual.pdf --embedding api
```

**适用场景**：离线环境必须用 `local`。在线环境下，若 PDF 内容为非英文或专业领域，API embedding 可能效果更好。

---

**`--embedding-model`** : 覆盖默认的 embedding 模型名称。

```bash
# 使用其他本地模型
python pdf_rag_agent.py --doc manual.pdf --embedding local --embedding-model paraphrase-MiniLM-L6-v2

# 使用其他 API embedding 模型
python pdf_rag_agent.py --doc manual.pdf --embedding api --embedding-model text-embedding-3-small
```

| 模式 | 默认模型 |
|------|----------|
| local | `all-MiniLM-L6-v2` |
| api | `text-embedding-ada-002` |

---

### 向量库参数

| 参数 | 类型 | 默认值 | 说明 |
|------|------|--------|------|
| `--rebuild` | 开关 | 关闭 | 强制重建向量库 |
| `--vector-db-dir` | 路径 | `./pdf_vector_db` | 向量库存储目录 |

**`--rebuild`** : 强制重建向量库，即使已有缓存。

```bash
python pdf_rag_agent.py --doc manual.pdf --rebuild
```

**适用场景**：更换了 embedding 模型、修改了分块参数、或 PDF 文件内容已更新但文件名未变时。正常情况下，工具会自动检测 PDF 变化并重建。

---

**`--vector-db-dir`** : 指定向量库的存储位置。

```bash
python pdf_rag_agent.py --doc manual.pdf --vector-db-dir ./my_vectors
```

**适用场景**：需要为不同项目维护独立的向量库，或需要将向量库存储到特定磁盘。

---

### 分块参数（支持自动调优）

| 参数 | 类型 | 默认值 | 说明 |
|------|------|--------|------|
| `--chunk-size` | 整数 | 自动调优 | 文本分块大小（字符数） |
| `--chunk-overlap` | 整数 | 自动调优 | 分块之间的重叠字符数 |

这两个参数如果不指定，会根据 PDF 总大小自动选取最优值（见 [自动调优机制](#自动调优机制)）。

**`--chunk-size`** : 控制每个文本块的大小。

```bash
# 小块 - 适合精确检索简短信息（如寄存器定义、参数表）
python pdf_rag_agent.py --doc register_map.pdf --chunk-size 500

# 大块 - 适合需要完整上下文的长段落（如流程说明、架构描述）
python pdf_rag_agent.py --doc architecture.pdf --chunk-size 3000
```

**适用场景**：
- **小 chunk (400-800)**：PDF 包含大量表格、寄存器映射、API 参数列表等结构化内容
- **中 chunk (1000-2000)**：通用技术文档，段落长度适中
- **大 chunk (2000-3000)**：PDF 内容为连续的长篇叙述，上下文关联性强

---

**`--chunk-overlap`** : 控制相邻块之间的重叠量，防止关键信息被切断。

```bash
# 大重叠 - 减少信息截断风险
python pdf_rag_agent.py --doc manual.pdf --chunk-size 2000 --chunk-overlap 400
```

**适用场景**：通常设为 chunk-size 的 10%-20%。如果检索结果经常缺少上下文，可以增大 overlap。

---

### 检索参数（支持自动调优）

| 参数 | 类型 | 默认值 | 说明 |
|------|------|--------|------|
| `--top-k` | 整数 | 自动调优 | 每次查询检索的文档块数 |
| `--search-type` | `similarity` / `mmr` | `similarity` | 检索算法类型 |
| `--score-threshold` | 浮点数 (0-1) | 无 (禁用) | 最低相似度阈值 |

**`--top-k`** : 控制每次查询返回多少个相关文档块给 LLM。

```bash
# 少量检索 - 回答简洁，速度快
python pdf_rag_agent.py --doc manual.pdf --top-k 3

# 大量检索 - 回答详尽，覆盖面广
python pdf_rag_agent.py --doc manual.pdf --top-k 12
```

**适用场景**：
- **top-k=3-4**：问题明确且 PDF 较小，只需少量上下文
- **top-k=6-8**：通用推荐值，平衡质量和速度
- **top-k=10-15**：PDF 很大或问题涉及多个章节的信息汇总
- **本地 LLM 模式**：自动调优默认限制为 6，但可通过显式指定覆盖

---

**`--search-type`** : 选择检索算法。

```bash
# 余弦相似度（默认）- 返回最相似的块
python pdf_rag_agent.py --doc manual.pdf --search-type similarity

# MMR（最大边际相关性）- 返回相似但多样化的块
python pdf_rag_agent.py --doc manual.pdf --search-type mmr
```

**适用场景**：
- **similarity**（默认）：适合大多数场景，精确匹配问题关键词
- **mmr**：当检索结果重复度高（多个块内容雷同）时使用，MMR 会在相关性和多样性之间取得平衡，适合综述性问题

---

**`--score-threshold`** : 丢弃低于阈值的检索结果（仅 `--search-type similarity` 有效）。

```bash
# 只保留高度相关的块
python pdf_rag_agent.py --doc manual.pdf --score-threshold 0.5

# 更严格的过滤
python pdf_rag_agent.py --doc manual.pdf --score-threshold 0.7
```

**适用场景**：当 PDF 内容量大但查询话题集中时，过滤低质量检索结果可减少噪声，提升回答准确性。值越高过滤越严格。**注意**：设置过高可能导致无结果返回。

---

### LLM 生成参数

| 参数 | 类型 | 默认值 | 说明 |
|------|------|--------|------|
| `--temperature` | 浮点数 (0-1) | `0.1` | LLM 温度（创造性 vs 确定性） |
| `--max-tokens` | 整数 | API: 模型默认 / 本地: `4096` | LLM 最大输出 token 数 |

**`--temperature`** : 控制 LLM 输出的随机性。

```bash
# 几乎确定性的回答（适合技术文档问答）
python pdf_rag_agent.py --doc manual.pdf --temperature 0.0

# 稍有创造性（默认）
python pdf_rag_agent.py --doc manual.pdf --temperature 0.1

# 较高创造性（适合需要推断或总结的场景）
python pdf_rag_agent.py --doc manual.pdf --temperature 0.5
```

**适用场景**：
- **0.0-0.1**：查询事实性信息（寄存器值、API 参数、协议定义），需要严格忠实于文档
- **0.2-0.4**：需要 LLM 做一定程度的归纳总结
- **0.5+**：开放性问题，需要 LLM 发挥推理能力

---

**`--max-tokens`** : 限制 LLM 单次回答的最大长度。

```bash
# 简短回答
python pdf_rag_agent.py --doc manual.pdf --max-tokens 512

# 详尽回答
python pdf_rag_agent.py --doc manual.pdf --max-tokens 8192
```

**适用场景**：
- **512-1024**：简单的是非问题或查找特定参数值
- **2048-4096**（本地默认）：标准问答，包含解释和上下文
- **4096-8192**：需要 LLM 列举大量信息或生成长篇回答

---

### 代码生成参数

| 参数 | 类型 | 默认值 | 说明 |
|------|------|--------|------|
| `--language` | 字符串 | `verilog` | 目标编程语言 |
| `--codegen-top-k` | 整数 | 自动调优 | 代码生成模式的检索块数 |
| `--codegen-temperature` | 浮点数 | `0.2` | 代码生成模式的 LLM 温度 |

**`--language`** : 指定代码生成的目标语言。影响 prompt 模板和输出格式。

```bash
python pdf_rag_agent.py --doc npi_manual.pdf --language tcl --query "生成读取信号的代码" --mode code
python pdf_rag_agent.py --doc spec.pdf --language python --query "实现协议解析" --mode code
python pdf_rag_agent.py --doc design.pdf --language verilog --query "实现 FSM" --mode code
```

**适用场景**：根据文档类型选择对应语言。支持 verilog、c、python、tcl 等。

---

**`--codegen-top-k`** : 代码生成模式下的检索块数。通常需要比 Q&A 模式更多的上下文。

```bash
# 需要大量 API 参考的代码生成
python pdf_rag_agent.py --doc api_ref.pdf --codegen-top-k 15 --query "实现完整的初始化流程" --mode code
```

**适用场景**：代码生成需要参考更多 API 定义和示例代码。默认值通过自动调优机制设定，通常为 Q&A 模式 top-k 的 1.5-2 倍。

---

**`--codegen-temperature`** : 代码生成专用的温度值，独立于 Q&A 的 `--temperature`。

```bash
# 严格参照文档生成代码（默认）
python pdf_rag_agent.py --doc manual.pdf --codegen-temperature 0.1 --query "生成初始化代码" --mode code

# 允许更多变体
python pdf_rag_agent.py --doc manual.pdf --codegen-temperature 0.4 --query "实现数据处理" --mode code
```

**适用场景**：代码生成通常需要较低温度以保证语法正确性。默认 0.2 比 Q&A 的 0.1 稍高，允许一定的灵活性。

---

### 单次查询模式

| 参数 | 类型 | 默认值 | 说明 |
|------|------|--------|------|
| `--query` | 字符串 | 无 (交互模式) | 单次查询，执行后退出 |
| `--mode` | `qa` / `code` | `qa` | 查询模式 |

**`--query`** : 非交互式运行，执行单次查询后退出。适合脚本集成。

```bash
# 问答模式
python pdf_rag_agent.py --doc manual.pdf --query "NPI 的初始化流程是什么"

# 代码生成模式
python pdf_rag_agent.py --doc manual.pdf --query "生成 NPI 初始化代码" --mode code
```

**适用场景**：CI/CD 管道集成、批量查询脚本、Qoder skill 集成。不带 `--query` 时进入交互式问答循环。

---

## 自动调优机制

当 `--chunk-size`、`--chunk-overlap`、`--top-k`、`--codegen-top-k` 未显式指定时，工具会根据 PDF 总大小自动选取最优值：

| 档位 | PDF 总大小 | chunk_size | chunk_overlap | top_k | codegen_top_k |
|------|-----------|------------|---------------|-------|---------------|
| Small | < 1 MB | 800 | 100 | 4 | 6 |
| Medium | 1 - 10 MB | 1500 | 200 | 6 | 10 |
| Large | 10 - 50 MB | 2000 | 300 | 8 | 14 |
| XLarge | > 50 MB | 2500 | 400 | 10 | 18 |

### 本地 LLM 模式限制

本地 LLM (Ollama) 的上下文窗口相对有限，因此自动调优的值会被进一步限制：

| 参数 | 最大值 |
|------|--------|
| chunk_size | 1200 |
| chunk_overlap | 200 |
| top_k | 6 |
| codegen_top_k | 10 |

**重要**：此限制仅作用于自动调优的值。如果通过命令行显式指定了参数值，该值不会被覆盖。例如：

```bash
# top_k 将使用你指定的 10，不会被限制为 6
python pdf_rag_agent.py --doc manual.pdf --local --top-k 10
```

---

## 使用场景示例

### 基础技术文档问答

```bash
python pdf_rag_agent.py --doc datasheet.pdf
```

使用默认参数，自动调优分块和检索参数。进入交互式问答。

### 大型 PDF 高精度检索

```bash
python pdf_rag_agent.py --doc large_manual.pdf --chunk-size 1000 --top-k 12 --search-type mmr --score-threshold 0.4
```

小分块 + 多检索 + MMR 多样性 + 相似度过滤，适合 100+ 页的大型手册。

### 基于文档的代码生成

```bash
# 交互式代码生成（输入时以 ? 前缀触发）
python pdf_rag_agent.py --doc npi_reference.pdf --language tcl

# 单次代码生成
python pdf_rag_agent.py --doc npi_reference.pdf --language tcl --query "读取模块顶层信号" --mode code
```

### 完全离线使用

```bash
python pdf_rag_agent.py --doc manual.pdf --local --embedding local
```

LLM 和 embedding 都使用本地模型，无需任何网络连接。

### 批量脚本集成

```bash
for pdf in docs/*.pdf; do
  python pdf_rag_agent.py --doc "$pdf" --local --query "summarize this document" --max-tokens 2048
done
```

### 高创造性回答 + 详尽输出

```bash
python pdf_rag_agent.py --doc spec.pdf --temperature 0.4 --max-tokens 8192 --top-k 10
```

适合需要 LLM 综合多处信息进行推理的复杂问题。

---

## 验证经验 (Skill) 系统

在测试点分解模式 (`/tp`) 下，可以通过 `/s` 命令教 agent 验证经验。这些经验会持久化保存，在后续分析任何设计文档时自动匹配并注入到 LLM 的 system prompt 中，帮助生成更高质量的测试点。

### 添加 Skill

在交互模式下，使用 `/s <经验描述>` 保存一条验证经验：

```
/s 对于SPI模块，CPOL和CPHA的四种组合模式都需要独立覆盖测试
```

输出示例：
```
[SKILL] 已保存验证经验 S001
  关键词: SPI, CPOL, CPHA, 模块, 组合, 覆盖
  累计 skill: 1 条
```

更多添加示例：

```
/s FIFO深度边界必须测试空、满、半满三种状态，以及上溢和下溢异常处理
/s 中断控制器需要验证中断优先级仲裁、中断嵌套、中断屏蔽和中断清除的完整组合
/s AXI总线协议验证时，outstanding transaction的最大数量边界和乱序响应都要覆盖
/s 低功耗模式切换需要验证进入和退出时序，以及切换过程中寄存器状态保持
/s 时钟域交叉(CDC)信号需要验证亚稳态防护逻辑，使用多周期路径约束
```

### 查看所有 Skill

```
/skill list
```

输出示例：
```
[SKILL] 验证经验列表 (共 5 条):
  S001 | 对于SPI模块，CPOL和CPHA的四种组合模式都需要独立覆盖测试
         关键词: SPI, CPOL, CPHA, 模块, 组合, 覆盖 | 2025-04-05
  S002 | FIFO深度边界必须测试空、满、半满三种状态，以及上溢和下溢异常处理
         关键词: FIFO, 深度, 边界, 上溢, 下溢, 异常 | 2025-04-05
  S003 | 中断控制器需要验证中断优先级仲裁、中断嵌套、中断屏蔽和中断清除的完整组合
         关键词: 中断, 控制器, 优先级, 仲裁, 嵌套, 屏蔽 | 2025-04-05
  ...
```

### 删除 Skill

按 ID 删除单条经验：

```
/skill delete S002
```

输出：
```
[SKILL] 已删除 S002
```

### 搜索 Skill

按关键词搜索已有经验：

```
/skill search 中断
```

输出示例：
```
[SKILL] 搜索结果 (1 条):
  S003 | 中断控制器需要验证中断优先级仲裁、中断嵌套、中断屏蔽和中断清除的完整组合
```

### 清空所有 Skill

```
/skill clear
```

会要求确认：
```
[SKILL] 确认清空所有 5 条 skill? (y/N): y
[SKILL] 已清空所有 skill (共 5 条)
```

### Skill 撰写最佳实践

#### 撰写建议

- **粒度**：每条 skill 聚焦一个具体验证要点，不要在一条里堆叠多个不相关要点
- **措辞**：使用具体的技术术语（如 "FIFO 空满状态"、"SPI CPOL/CPHA"），避免泛泛描述
- **长度**：建议 20-100 字，过短匹配不精准，过长关键词被稀释
- **关键名词**：模块名、协议名、信号名等专有名词是自动匹配的核心

**示例对比**：

| 质量 | 示例 | 问题/优势 |
|------|------|-----------|
| 差 | "注意边界条件" | 太泛泛，几乎匹配所有查询 |
| 中 | "FIFO 需要测试边界" | 有模块名但缺少具体条件 |
| 好 | "FIFO 深度边界必须测试空、满、半满三种状态，以及上溢和下溢异常处理" | 具体模块 + 具体条件 + 具体场景 |

#### 分组策略

- **按项目/芯片分组**：`/sg ADC_v2 ...` — 同一项目的经验归档在一起
- **按验证领域分组**：`/sg 时钟验证 ...`、`/sg 总线协议 ...` — 按技术领域归档
- **默认分组**（不指定 `/sg`）：自动以当前 PDF 文件名分组，适合单文档积累

#### 自动匹配 vs 手动载入

| 场景 | 推荐方式 | 原因 |
|------|----------|------|
| Skill 总数 <= 5 条 | 无需操作 | 系统自动注入全部 skill |
| Skill 多 + 查询主题明确 | 自动匹配即可 | 关键词能准确命中相关 skill |
| Skill 多 + 跨领域查询 | `/skill load @分组` 手动载入 | 确保相关经验一定被注入 |
| 需要精确控制 | `/skill load` 仅指定需要的 | 只注入手动载入的，排除干扰 |

**重要机制**：自动匹配仅在有手动载入的 skill 时才同时执行。如果当前会话没有载入任何 skill（`/skill load`），则不会自动匹配注入。载入至少一条 skill 后，系统会同时执行自动匹配，将关键词命中的 skill 一并注入。

---

### 自动调用机制

在测试点模式下提问时，系统会自动根据问题和上下文匹配相关 skill，并注入到 LLM 的 system prompt 中，无需手动操作。

**工作流示例**：

1. 分析第一份文档时积累经验：
```bash
python pdf_rag_agent.py --doc spi_design_spec.pdf
```
```
> /tp
[模式已切换到 testpoint]
> 分解SPI主机模块的测试点
... (LLM 生成测试点) ...
> /s 对于SPI模块，CPOL和CPHA的四种组合模式都需要独立覆盖测试
[SKILL] 已保存验证经验 S001
> /s SPI片选信号的时序间隔需要验证最小间隔边界
[SKILL] 已保存验证经验 S002
```

2. 分析第二份文档时自动生效：
```bash
python pdf_rag_agent.py --doc another_spi_module.pdf
```
```
> /tp
> 分解这个SPI从机模块的验证测试点
```
此时 S001、S002 会自动匹配（关键词"SPI"命中），LLM 会参考这些经验生成更全面的测试点。

### 手动载入 Skill

当自动匹配不够精确、或想强制使用特定经验时，可以手动载入 skill。载入的 skill 在当前会话中**始终注入** prompt，不依赖关键词匹配。

#### 载入 Skill

先查看可用列表，再按 ID 载入：

```
/skill list
/skill load SK_001 SK_003
```

输出示例：
```
  [OK]   SK_001 已载入
  [OK]   SK_003 已载入
[SKILL] 当前已载入 2 条: SK_001, SK_003
```

载入后，`/skill list` 中会标记已载入的 skill：
```
[SKILL] 验证经验列表 (共 5 条, 已载入 2 条):
  SK_001 [已载入] | 对于SPI模块，CPOL和CPHA的四种组合模式都需要独立覆盖测试
         关键词: SPI, CPOL, CPHA, 模块, 组合, 覆盖 | 2025-04-05
  SK_002 | FIFO深度边界必须测试空、满、半满三种状态...
         关键词: FIFO, 深度, 边界, 上溢, 下溢, 异常 | 2025-04-05
  SK_003 [已载入] | 中断控制器需要验证中断优先级仲裁、中断嵌套...
         关键词: 中断, 控制器, 优先级, 仲裁, 嵌套, 屏蔽 | 2025-04-05
  ...
```

#### 卸载 Skill

```
# 卸载单个
/skill unload SK_001

# 批量卸载
/skill unload SK_001 SK_003

# 卸载所有
/skill unload all
```

#### 典型使用场景

当积累了大量 skill 但只想针对当前模块使用特定几条经验时：

```
> /skill list                        # 查看所有经验
> /skill search 时钟                  # 搜索时钟相关经验
> /skill load SK_005 SK_012           # 载入与当前分析最相关的
> 分析时钟分频模块的测试点              # 载入的 skill 始终参与生成
```

手动载入的 skill 会与自动匹配的 skill 合并（去重），载入的优先显示。

### 存储说明

- Skill 保存在程序目录下的 `skills.json` 文件中
- 跨文档、跨会话持久生效
- 便携包模式下，skill 文件随 exe 所在目录一起迁移

#### skills.json 文件格式

```json
{
  "version": 1,
  "next_id": 5,
  "skills": [
    {
      "id": "SK_001",
      "content": "FIFO 深度边界必须测试空、满、半满三种状态",
      "keywords": ["fifo", "深度", "边界", "空满", ...],
      "created_at": "2025-04-05T01:09:06",
      "source_doc": "design_spec.pdf",
      "group": "FIFO验证"
    }
  ]
}
```

| 字段 | 说明 |
|------|------|
| `version` | 文件格式版本号（当前为 1） |
| `next_id` | 下一条 skill 的自增序号 |
| `id` | 唯一标识符，格式 `SK_NNN` |
| `content` | 验证经验的自然语言描述 |
| `keywords` | 自动提取的关键词列表（用于匹配，无需手动编辑） |
| `created_at` | ISO 格式创建时间戳 |
| `source_doc` | 创建时加载的 PDF 文件名 |
| `group` | 分组名（可选，`null` 时以 `source_doc` 为分组） |

**关键词提取算法**：
- 英文：单词分词 + 小写化 + 去停用词
- 中文：2-4 字符 n-gram 滑动窗口 + 去停用词（无需 jieba 分词依赖）

**手动编辑**：
- 可直接编辑 `content` 和 `group` 字段，修改后重启程序生效
- `keywords` 字段由系统自动生成，手动删除后需触发一次写操作（如 `/s` 添加新 skill）才会重新生成
- 备份/迁移只需复制 `skills.json` 文件

### Skill 分组

Skill 支持分组存储和批量操作。默认以当前 PDF 文件名作为分组名，也可手动指定分组。

**自动分组（默认按 PDF 名）**：
```
/s 验证时注意寄存器复位值需在时钟稳定后检查
→ [SKILL] 已保存验证经验 SK_001 → 分组: datasheet.pdf
```

**指定分组保存**：
```
/sg 时钟验证 PLL lock 信号需在 100us 内拉高，否则报 timeout
→ [SKILL] 已保存验证经验 SK_002 → 分组: 时钟验证
```

**查看所有分组**：
```
/skill groups
→ [SKILL] 分组列表 (共 3 个分组, 8 条 skill):
    @datasheet.pdf  — 4 条 (SK_001, SK_003, SK_005, SK_007)
    @时钟验证        — 2 条 (SK_002, SK_004)
    @未分组          — 2 条 (SK_006, SK_008)
```

**按分组批量载入/卸载**：
```
/skill load @datasheet.pdf          # 载入整个分组
/skill load @时钟验证 SK_006        # 混合: 分组 + 单个 ID
/skill unload @datasheet.pdf        # 卸载整个分组
```

**按分组搜索**：
```
/skill search @时钟验证             # 搜索指定分组的所有 skill
/skill search PLL                   # 关键词搜索（也匹配分组名）
```

旧版本保存的 skill（无 group 字段）会自动使用 source_doc 作为分组，若都为空则归入"未分组"。

### 命令速查

| 命令 | 说明 |
|------|------|
| `/s <经验描述>` | 添加一条验证经验（自动以当前 PDF 名分组） |
| `/sg <组名> <经验描述>` | 添加一条验证经验到指定分组 |
| `/skill list` | 查看所有已保存的经验（标记已载入状态和分组） |
| `/skill groups` | 查看所有分组及其 skill 数量 |
| `/skill loaded` | 查看当前已载入的 skill 列表 |
| `/skill unloaded` | 查看当前未载入的 skill 列表 |
| `/skill load <ID\|@组名> [...]` | 手动载入指定 skill 或整个分组 |
| `/skill unload <ID\|@组名\|all>` | 卸载已载入的 skill、分组或全部 |
| `/skill delete <ID>` | 删除指定 ID 的经验 |
| `/skill search <关键词\|@组名>` | 搜索经验（支持关键词和分组名） |
| `/skill clear` | 清空所有经验（需确认） |

---

## 构建便携包

```bash
# 完整构建（PyInstaller + Ollama 打包）
python build.py

# 仅重新打包 Ollama（跳过 PyInstaller）
python build.py --skip-pyinstaller

# 构建并压缩为 7z
python build.py --compress
```

输出目录 `dist/pdf_rag_agent/`，包含 exe + Ollama 运行时 + Qwen2.5:3b 模型，可直接复制到目标机器使用。

---

## CPU 性能调优策略

### 为什么便携版较慢？

便携版仅捆绑了 CPU 推理运行时（约 8MB），而系统安装的 Ollama 会使用 CUDA GPU 加速（cuda_v12: 2.4GB，cuda_v13: 865MB）。GPU 推理速度通常是 CPU 的 5-20 倍。

若目标机器有 NVIDIA GPU，可安装完整版 Ollama 获得 GPU 加速。便携版适合无 GPU 或需要离线便携的场景。

---

### 参数调优建议

以下参数对 CPU 推理性能影响最大，按影响程度排序：

| 参数 | CPU 优化值 | 默认值 | 影响说明 |
|------|-----------|--------|----------|
| `--local-model` | `qwen2.5:1.5b` | `qwen2.5:3b` | **影响最大**。小模型推理速度快 2-3 倍 |
| `--num-gpu` | `99` (有 GPU 时) | `0` | GPU 卸载可加速 5-20 倍 |
| `--num-ctx` | `1024-2048` | 按模型(2048-4096) | 小上下文减少计算量 |
| `--max-tokens` | `512-1024` | `2048` | 限制输出长度，减少生成时间 |
| `--num-thread` | 物理核心数 | `8` | 匹配实际核心数，避免过度线程竞争 |
| `--top-k` | `3-4` | 自动(4-6) | 减少上下文，加速处理 |
| `--chunk-size` | `600-800` | 自动(800-1200) | 小块减少每次处理文本量 |

---

### 速度 vs 质量权衡

| 优化目标 | 参数组合 | 预期效果 |
|----------|----------|----------|
| **极速模式** | `--local-model qwen2.5:1.5b --num-ctx 1024 --max-tokens 512 --top-k 3` | 响应快，回答简短，可能丢失细节 |
| **平衡模式** | `--local-model qwen2.5:1.5b --num-ctx 2048 --max-tokens 1024 --top-k 4` | 速度与质量折中（推荐） |
| **质量优先** | `--local-model qwen2.5:3b --num-ctx 4096 --max-tokens 2048 --top-k 6` | 回答详尽，速度较慢（默认） |
| **GPU 加速** | `--num-gpu 99 --num-ctx 8192 --max-tokens 4096 --top-k 8` | 有 GPU 时，全面提升 |

---

### CPU 优化命令示例

**极速问答（适合简单查询）**：
```bash
pdf_rag_agent.exe --doc manual.pdf --local --local-model qwen2.5:1.5b --num-ctx 1024 --max-tokens 512 --top-k 3
```

**平衡模式（推荐日常使用）**：
```bash
pdf_rag_agent.exe --doc manual.pdf --local --local-model qwen2.5:1.5b --num-ctx 2048 --max-tokens 1024 --top-k 4
```

**代码生成优化**：
```bash
pdf_rag_agent.exe --doc api_ref.pdf --local --local-model qwen2.5:1.5b --max-tokens 1024 --codegen-top-k 6 --mode code --query "生成初始化代码"
```

**GPU 加速模式（有 NVIDIA GPU 时）**：
```bash
pdf_rag_agent.exe --doc manual.pdf --local --num-gpu 99 --num-ctx 8192 --max-tokens 4096 --local-top-k-cap 8
```

**低内存机器 (8GB RAM)**：
```bash
pdf_rag_agent.exe --doc manual.pdf --local --local-model qwen2.5:1.5b --num-thread 4 --num-ctx 1024 --max-tokens 512 --local-top-k-cap 3
```

---

### 进一步优化

1. **关闭其他程序**：CPU 推理占用率高，关闭浏览器、IDE 等可提升响应速度
2. **使用 SSD**：模型加载和向量库访问受磁盘 I/O 影响
3. **增加内存**：充足内存可避免模型换页到磁盘
4. **考虑安装 Ollama**：若目标机器有 GPU，安装完整版 Ollama 可获得显著加速
5. **调整 `--num-ctx`**：对性能影响仅次于模型大小，优先尝试缩小上下文窗口

---

### Skill 与 CPU 推理的注意事项

Skill 注入会增加 system prompt 长度，每条 skill 约消耗 50-200 token。在 CPU 推理 + 小上下文窗口场景下需要注意：

- **`--num-ctx 2048` 时**：建议手动载入的 skill 控制在 3-5 条以内，避免挤压检索内容空间
- **`--num-ctx 4096` 时**：可载入 10 条左右 skill
- **`--num-ctx 1024` 时**：建议仅载入 1-2 条最关键的 skill

**推荐做法**：CPU 模式下使用分组载入，精确控制注入数量：

```bash
python pdf_rag_agent.py --doc spec.pdf --local --num-ctx 2048 --local-top-k-cap 3
# 进入交互后:
# /skill load @关键分组    (仅载入当前最需要的分组)
```
