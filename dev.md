# Semantica 二次开发与发布指南

本文面向需要在 Semantica 基础上进行二次开发、增加组件、构建镜像或发布 Python 包的开发者。

## 1. 项目定位与代码组织

Semantica 是一个以 Context Graph 和 Knowledge Graph 为核心的 Python 平台，同时包含 React/TypeScript 的 Knowledge Explorer 前端。一次完整的数据处理链路通常是：

```text
数据源
  -> ingest
  -> parse / normalize / split
  -> semantic_extract
  -> conflicts / deduplication
  -> kg / context
  -> ontology / reasoning / provenance / decisions
  -> graph_store / triplet_store / vector_store
  -> export / visualization / REST / MCP / CLI / Explorer
```

仓库主要目录如下：

| 目录 | 内容与职责 |
| --- | --- |
| `semantica/` | Python 核心包，包含图、管道、推理、存储、导出和服务端能力。 |
| `integrations/` | Agno、CrewAI、LangChain、Google ADK、OpenClaw 等外部框架集成。 |
| `semantica_mcp/` | MCP 相关协议适配和服务入口。核心 MCP 实现也位于 `semantica/mcp_server/`。 |
| `explorer/` | React 19 + TypeScript + Vite 的 Knowledge Explorer 前端。构建产物写入 `semantica/static/`。 |
| `plugins/` | 面向 Claude Code、Cursor、Codex、VS Code 等客户端的 skills、agents、hooks 和插件清单。 |
| `tests/` | Python 单元测试、集成测试、Explorer 后端测试和端到端测试。 |
| `cookbook/` | 可运行的教程、示例和 Jupyter notebooks。 |
| `examples/` | 独立示例和 CI 模板。 |
| `docs/` | 面向用户的文档站点内容。 |
| `deploy/` | Azure、GCP、AWS、Fly.io、Railway、Render、Kubernetes、Helm 等部署配置。 |
| `.github/workflows/` | CI、依赖安装矩阵、安全扫描、文档检查、基准测试和发布流程。 |

## 2. 开发环境

### 2.1 基础要求

- Python `3.9.2` 或更高版本；CI 当前重点验证 Python 3.9-3.12。
- Node.js 18 或更高版本，推荐 Node.js 20，用于 Explorer 开发。
- npm 9 或更高版本。
- Git。
- 需要运行完整测试或特定后端时，再安装相应的可选依赖和外部服务。

### 2.2 初始化 Python 环境

```bash
python -m venv .venv
source .venv/bin/activate                 # Windows: .venv\\Scripts\\activate
python -m pip install --upgrade pip
pip install -e ".[dev]"
pre-commit install                         # 可选
```

如果同时修改 Explorer：

```bash
cd explorer
npm ci
cd ..
```

完整功能环境可以安装：

```bash
pip install -e ".[all]"
```

`all` 不包含 GPU 和 CrewAI。GPU 依赖需要单独安装，CrewAI 需要显式执行 `pip install -e ".[crewai]"`。

### 2.3 可选依赖选择

不要为了修改一个组件安装所有依赖，优先安装组件对应的 extra：

| 开发目标 | 安装命令 |
| --- | --- |
| 文档解析 | `pip install -e ".[documents,parse-pdf]"` |
| 本地 Embedding / NLP | `pip install -e ".[embeddings-local,nlp-spacy]"` |
| LLM Provider | `pip install -e ".[llm-openai]"`，或选择 `llm-anthropic`、`llm-gemini` 等 |
| 图数据库 | `pip install -e ".[graph-neo4j]"`、`[graph-falkordb]` 等 |
| 向量数据库 | `pip install -e ".[vectorstore-faiss]"`、`[vectorstore-qdrant]` 等 |
| RDF 三元组库 | `pip install -e ".[tripletstore-oxigraph]"` |
| Explorer 后端 | `pip install -e ".[explorer]"` |
| 监控和基础设施 | `pip install -e ".[monitoring,infra]"` |
| 集成测试 | 按集成选择 `agno`、`langchain`、`google-adk` 或 `crewai` |

所有可用 extra 的权威定义以 `pyproject.toml` 的 `[project.optional-dependencies]` 为准。

## 3. 各 Component 的内容

### 3.1 核心数据流组件

| Component | 目录 | 主要内容 | 适合扩展的位置 |
| --- | --- | --- | --- |
| Ingest | `semantica/ingest/` | 文件、网页、数据库、API、Git、邮件、消息流、Parquet、Arrow、Databricks、Snowflake、SAP 等数据源连接器。 | 新增 `*Ingestor`，统一返回带来源信息的记录；不要在连接器中耦合 KG 构建。 |
| Parse | `semantica/parse/` | PDF、DOCX、HTML、表格等非结构化内容解析。 | 新增 Parser，并处理缺失 optional dependency 的清晰错误。 |
| Normalize | `semantica/normalize/` | 文本、实体、日期、数字和数据集清洗、标准化。 | 新增 Normalizer 或 DataCleaner 规则；保证输入输出可测试、可重复。 |
| Split | `semantica/split/` | recursive、token、sentence、entity-aware、relation-aware、graph-based、ontology-aware 等切分策略。 | 新增切分策略并注册到 `TextSplitter`；保留实体、关系和来源元数据。 |
| Semantic Extract | `semantica/semantic_extract/` | NER、关系、事件、Triplet、Coreference 等语义提取。 | 新增抽取器或 Provider；输出应包含稳定字段和可选置信度。 |
| Conflicts | `semantica/conflicts/` | 值、类型、关系、时间和逻辑冲突检测及解决策略。 | 新增 Detector、Resolver 或 SourceTracker 策略；不得静默覆盖冲突。 |
| Deduplication | `semantica/deduplication/` | Blocking、候选生成、相似度判断、重复实体合并和合并历史。 | 新增相似度或合并策略；保留 provenance 和 merge history。 |
| KG | `semantica/kg/` | GraphBuilder、实体关系构建、图分析、中心性、社区、路径、链路预测和时间事实。 | 修改图模型或算法时同步更新序列化和测试。 |
| Context | `semantica/context/` | `ContextGraph`、`AgentContext`、`AgentMemory`、决策记录和跨图上下文能力。 | 面向用户的高层 API；公共方法变更必须补兼容性测试。 |

### 3.2 智能、治理和审计组件

| Component | 目录 | 主要内容 |
| --- | --- | --- |
| Ontology | `semantica/ontology/` | OWL 生成、SHACL 校验、SKOS 词汇表、Ontology Hub、质量检查和对齐。 |
| Reasoning | `semantica/reasoning/` | Forward Chaining、Rete、Datalog、SPARQL、演绎/归纳/溯因推理和解释生成。 |
| Provenance | `semantica/provenance/` | W3C PROV-O 来源、实体/关系血缘、审计轨迹和导出。 |
| Change Management | `semantica/change_management/` | 图和知识数据变更记录、变更提议、审核和应用流程。 |
| Evals | `semantica/evals/` | 质量评估、基准、回归检查和能力评测。 |
| Seed | `semantica/seed/` | 初始数据、规则、词汇或演示图的种子加载能力。 |

### 3.3 存储和计算后端

| Component | 目录 | 主要内容 |
| --- | --- | --- |
| Graph Store | `semantica/graph_store/` | Neo4j、FalkorDB、Apache AGE、Neptune 等 LPG 图存储适配，以及统一接口。 |
| Triplet Store | `semantica/triplet_store/` | RDF 三元组存储和 SPARQL 适配，包括 Oxigraph、Blazegraph、Jena、RDF4J 等。 |
| Vector Store | `semantica/vector_store/` | FAISS、Qdrant、Weaviate、Pinecone、Milvus、PgVector、SQLite-vec 和内存后端。 |
| Embeddings | `semantica/embeddings/` | 文本/向量/图 Embedding、Provider、缓存、池化和 Embedding provenance。 |
| Pipeline | `semantica/pipeline/` | Pipeline DSL、步骤依赖、并行调度、执行状态、资源调度和 pipeline provenance。 |

新增后端时应优先实现已有抽象接口，并补充：初始化配置、写入、查询、删除、枚举/迁移、错误处理和最小集成测试。不要在上层组件直接依赖某一个厂商 SDK。

### 3.4 输出、服务和客户端组件

| Component | 目录 | 主要内容 |
| --- | --- | --- |
| Export | `semantica/export/` | RDF、OWL、JSON-LD、JSON、Parquet、Cypher 和审计报告导出。 |
| Visualization | `semantica/visualization/` | 图、社区、中心性、Ontology、Embedding 和时间线可视化。 |
| LLMs | `semantica/llms/` | OpenAI、Anthropic、Gemini、Ollama、DeepSeek、Novita、LiteLLM 等 Provider 封装。 |
| CLI | `semantica/cli.py` | `semantica` 主命令及 ingest、extract、kg、reason、export、doctor、explorer 等命令组。 |
| REST Server | `semantica/server.py` | FastAPI/uvicorn REST API，覆盖 graph、decision、reasoning、provenance、ontology、search 和 export。 |
| Worker | `semantica/worker.py` | 后台任务进程入口，配合服务端和队列执行异步任务。 |
| MCP Server | `semantica/mcp_server/` | 面向 MCP 客户端的 stdio 服务、工具和资源。 |
| Explorer Backend | `semantica/explorer/` | Explorer 的 Python API、会话、图加载、认证和静态资源服务。 |
| Explorer Frontend | `explorer/src/` | React 工作区、Sigma.js 图画布、时间线、决策、谱系、实体消歧、Ontology、SPARQL 和导入导出。 |
| Integrations | `integrations/` | 将 ContextGraph、决策、KG 工具和共享上下文接入外部 Agent 框架。 |
| Plugins | `plugins/` | 为各种 AI 编程客户端提供 skills、agents、hooks 和插件分发结构。 |

### 3.5 辅助目录

- `tests/`：和代码改动同提交测试；Python 测试使用 pytest，Explorer 使用 npm scripts 和 Playwright。
- `docs/`：用户文档，不放内部实现细节；API 行为变更需要同步文档。
- `cookbook/`：端到端示例，适合验证公开 API 是否易用。
- `deploy/`：部署模板。修改端口、环境变量、镜像入口或服务依赖时要同步更新。
- `.github/requirements/`：CI 使用的哈希锁定依赖，不是普通本地开发环境的安装清单。

## 4. 二次开发流程

### 4.1 创建分支

```bash
git checkout -b feature/short-description
# 或
git checkout -b fix/bug-description
```

建议一个分支只解决一个问题。公共 API、数据格式、环境变量、CLI 参数和存储接口属于高影响变更，应在 PR 中明确兼容性影响。

### 4.2 添加功能的推荐步骤

1. 先确定功能所属 component 和抽象边界。
2. 阅读同目录下相近实现，复用已有接口、异常和配置模式。
3. 先写最小单元测试，再实现功能。
4. 如果改变跨组件数据结构，同时更新序列化、导出、API 和 Explorer 适配。
5. 如果增加依赖，更新 `pyproject.toml` 对应 extra，并重新生成锁文件。
6. 如果增加公开能力，更新 README、`docs/` 或 cookbook 示例。
7. 执行与改动范围匹配的测试，再执行完整检查。

### 4.3 Python 代码检查

```bash
pytest
black semantica/ tests/
isort semantica/ tests/
flake8 semantica/ tests/
mypy semantica/
pre-commit run --all-files
```

只验证单个测试文件时：

```bash
pytest -q tests/path/to/test_file.py
pytest -q tests/path/to/test_file.py -k keyword
pytest -m "not integration"
```

需要外部服务或 API Key 的测试使用 `integration` marker，默认开发验证可以先执行 `pytest -m "not integration"`。

### 4.4 Explorer 开发与验证

终端一：启动 Python 后端：

```bash
pip install -e ".[explorer]"
semantica-explorer --graph path/to/my_graph.json --no-browser
```

终端二：启动 Vite：

```bash
cd explorer
npm ci
npm run dev
```

前端开发地址是 `http://localhost:5173`，Vite 会把 `/api` 和 `/ws` 转发到 `127.0.0.1:8000`。

常用检查：

```bash
cd explorer
npm run lint
npm run build
npm run test:graph-store
npm run test:graph-workspace
npm run test:plugin-registry
npm run test:deterministic-e2e
npm run test:graph-legend-e2e
```

`npm run build` 会把编译后的前端放入 `../semantica/static/`。Python 包发布前必须先执行该构建，否则 wheel 中可能缺少 Explorer 页面或 assets。

## 5. 启动与调试

本节介绍本地开发时如何启动各类进程、验证进程是否正常，以及如何在 VS Code 中附加断点。建议先使用可工作的命令行启动方式，再配置 IDE 调试，这样可以把“启动失败”和“断点未命中”区分开。

### 5.1 调试 REST API / Explorer 后端

先安装后端依赖：

```bash
pip install -e ".[explorer]"
```

启动 REST API：

```bash
# 方式一：使用已安装的 entry point
SEMANTICA_LOG_LEVEL=DEBUG semantica-server

# 方式二：使用 Python 模块启动
SEMANTICA_LOG_LEVEL=DEBUG python -m semantica.server
```

服务默认监听 `127.0.0.1:8000`。检查启动结果：

```bash
curl http://127.0.0.1:8000/health
curl http://127.0.0.1:8000/api/info
```

交互式 API 文档地址是 `http://127.0.0.1:8000/docs`。如果要让其他机器访问，设置 `SEMANTICA_HOST`，但生产或共享网络环境必须同时配置 `SEMANTICA_API_KEY`、反向代理和 HTTPS：

```bash
SEMANTICA_HOST=0.0.0.0 \
SEMANTICA_API_KEY="$(openssl rand -hex 32)" \
SEMANTICA_LOG_LEVEL=DEBUG \
semantica-server
```

将断点放在以下位置可以覆盖常见调试路径：

- `semantica/server.py` 的 `lifespan()`：观察应用启动、GraphSession 初始化和关闭过程。
- `semantica/server.py` 的路由函数：调试请求参数、响应和异常处理。
- `semantica/explorer/routes/`：调试 Explorer 的 graph、decision、ontology、provenance 等 API。
- `semantica/explorer/session.py`：调试图加载、查询和写操作。
- `semantica/explorer/runtime.py`：调试运行时能力和图变更广播。

### 5.2 调试 Explorer 前后端联调

准备一个图文件，例如 `my_graph.json`，然后启动后端：

```bash
semantica-explorer --graph my_graph.json --no-browser --port 8000
```

另开终端启动 Vite：

```bash
cd explorer
npm run dev
```

打开 `http://localhost:5173`。Vite 会把 `/api` 和 `/ws` 请求代理到 `127.0.0.1:8000`。此模式适合同时调试 React 状态和 Python API：

- 前端断点放在 `explorer/src/App.tsx`、`explorer/src/store/` 或对应 `workspaces/`。
- 后端断点放在 `semantica/explorer/routes/`、`session.py` 和 WebSocket 处理代码。
- 浏览器开发者工具的 Network 面板用于确认 `/api` 请求和 `/ws` 连接是否成功。
- 后端终端用于查看 Python traceback；前端终端用于查看 TypeScript、Vite 和 ESLint 错误。

如果只调试 Python 后端而不需要 Vite，先构建前端后直接访问 8000：

```bash
cd explorer
npm run build
cd ..
semantica-explorer --graph my_graph.json --no-browser
```

### 5.3 使用 VS Code Python 调试器

仓库没有强制提交 `.vscode/launch.json`。可以在 VS Code 的 `Run and Debug` 中创建一个 Python 配置，或将下面配置加入个人的 `.vscode/launch.json`。配置使用模块启动，能保持和命令行入口一致：

```json
{
  "version": "0.2.0",
  "configurations": [
    {
      "name": "Semantica REST server",
      "type": "debugpy",
      "request": "launch",
      "module": "semantica.server",
      "cwd": "${workspaceFolder}",
      "console": "integratedTerminal",
      "env": {
        "SEMANTICA_LOG_LEVEL": "DEBUG",
        "SEMANTICA_CORS_ORIGINS": "http://localhost:5173,http://127.0.0.1:5173"
      },
      "justMyCode": true
    },
    {
      "name": "Semantica Explorer backend",
      "type": "debugpy",
      "request": "launch",
      "module": "semantica.explorer",
      "args": [
        "--graph",
        "${workspaceFolder}/my_graph.json",
        "--no-browser",
        "--port",
        "8000"
      ],
      "cwd": "${workspaceFolder}",
      "console": "integratedTerminal",
      "env": {
        "SEMANTICA_LOG_LEVEL": "DEBUG",
        "SEMANTICA_ALLOW_ANONYMOUS": "true"
      },
      "justMyCode": true
    }
  ]
}
```

调试 Explorer 时，`SEMANTICA_ALLOW_ANONYMOUS=true` 只适用于本机开发；不要在远程或生产环境复用该设置。若使用 API Key 调试，把它放在本地未提交的环境配置中，不要写入仓库文件。

如果 Python 调试器提示找不到模块，确认 VS Code 选中了安装 `semantica` 的解释器：

```bash
which python
python -c "import semantica; print(semantica.__file__)"
```

### 5.4 调试 Vite 前端

前端开发服务器：

```bash
cd explorer
npm ci
npm run dev
```

在浏览器中打开 Vite 输出的地址，使用浏览器开发者工具设置断点。修改 `explorer/src/` 后 Vite 会热更新；如果修改了 Vite 代理或环境配置，需要重启 `npm run dev`。

前端单独检查：

```bash
npm run lint
npm run build
```

针对图工作区和交互行为运行测试：

```bash
npm run test:graph-store
npm run test:graph-workspace
npm run test:deterministic-e2e
npm run test:graph-legend-e2e
```

### 5.5 调试 Worker

Worker 是独立进程，需要和服务端使用相同的后端配置：

```bash
SEMANTICA_LOG_LEVEL=DEBUG semantica-worker
```

或在 VS Code 中使用 Python 配置：

```json
{
  "name": "Semantica worker",
  "type": "debugpy",
  "request": "launch",
  "module": "semantica.worker",
  "cwd": "${workspaceFolder}",
  "console": "integratedTerminal",
  "env": {
    "SEMANTICA_LOG_LEVEL": "DEBUG"
  },
  "justMyCode": true
}
```

启动顺序通常是先启动 API 服务，再启动 Worker。若任务没有被消费，先检查两边的环境变量、队列地址和日志中的连接错误。

### 5.6 调试 MCP stdio 服务

MCP 服务通过 stdin/stdout 传输 JSON-RPC，不能把普通调试信息写到 stdout，否则会破坏协议。调试日志应写到 stderr：

```bash
SEMANTICA_LOG_LEVEL=DEBUG semantica-mcp
```

用 initialize 请求做最小连通性检查：

```bash
echo '{"jsonrpc":"2.0","id":1,"method":"initialize","params":{"protocolVersion":"2024-11-05","capabilities":{},"clientInfo":{"name":"dev-test","version":"1.0"}}}' \
  | SEMANTICA_LOG_LEVEL=DEBUG semantica-mcp
```

需要持久化图时设置绝对路径：

```bash
SEMANTICA_KG_PATH="$(pwd)/my_graph.json" \
SEMANTICA_LOG_LEVEL=DEBUG \
semantica-mcp
```

MCP 断点建议放在 `semantica/mcp_server/__init__.py` 的请求分发和工具调用处。调试时使用 VS Code 的 Python 配置启动 `semantica.mcp_server`，并通过终端管道发送 JSON-RPC；不要在调试代码中使用 `print()` 输出到 stdout。

### 5.7 调试顺序和常见故障

推荐按以下顺序缩小问题范围：

1. 先执行 `python -c "import semantica"`，确认解释器和可编辑安装正确。
2. 单独启动后端，访问 `/health` 或 `/api/health`。
3. 再启动 Vite，确认浏览器可以访问页面。
4. 使用浏览器 Network 面板检查 API 和 WebSocket，再进入业务断点。
5. 最后检查外部服务，例如 FalkorDB、Neo4j、向量库、LLM Provider 或消息队列。

常见现象：

| 现象 | 排查方式 |
| --- | --- |
| `ModuleNotFoundError` | 检查 VS Code 解释器和 `pip install -e ".[dev,explorer]"`。 |
| Explorer 返回 `503` | 配置 `SEMANTICA_API_KEY`，或仅在本机开发显式使用 `SEMANTICA_ALLOW_ANONYMOUS=true`。 |
| 页面能打开但 API 失败 | 确认后端运行在 8000，检查 `explorer/vite.config.ts` 的代理目标和 CORS 配置。 |
| WebSocket 断开 | 检查 `/ws` 代理、后端日志以及浏览器 Network 面板。 |
| MCP 客户端不显示工具 | 先直接执行 `echo ... | semantica-mcp`，并确认 stdout 只有 JSON-RPC 响应。 |
| 断点不命中 | 确认以 `debugpy` 启动的是当前虚拟环境，且 `justMyCode` 没有过滤目标文件。 |

## 6. 依赖锁定与更新

`requirements-ci.txt` 是 CI 和发布使用的哈希锁定依赖集合，不应直接用于日常虚拟环境。修改 `pyproject.toml` 的依赖后，使用与 CI 一致的 uv 版本重新生成：

```bash
pip install uv==0.12.1
uv pip compile pyproject.toml \
  --python-version 3.11 \
  --extra all \
  --generate-hashes \
  -o requirements-ci.txt
```

CI 会检查锁文件是否相对 `pyproject.toml` 过期。不要只手工修改 `requirements-ci.txt`，否则下次 CI 的重新解析检查会失败。

构建系统版本在 `pyproject.toml` 中固定为：

```text
setuptools==84.0.0
wheel==0.48.0
```

## 7. 本地构建与发布前检查

### 6.1 构建 Python 分发包

```bash
rm -rf dist build *.egg-info
python -m build --no-isolation
```

正式构建前应使用 CI 的锁定依赖，并安装构建工具：

```bash
pip install -r requirements-ci.txt --require-hashes
pip install -r .github/requirements/build-tools.txt --require-hashes
python -m build --no-isolation
```

### 6.2 检查分发包

```bash
python -m twine check dist/*
python -m pip install dist/*.whl
python -c "import semantica; print(semantica.__version__)"
```

确认 wheel 包含 Explorer：

```bash
python - <<'PY'
import zipfile
from pathlib import Path

wheel = next(Path("dist").glob("*.whl"))
with zipfile.ZipFile(wheel) as archive:
    names = set(archive.namelist())
assert "semantica/static/index.html" in names
assert any(name.startswith("semantica/static/assets/") for name in names)
print("Explorer frontend is packaged")
PY
```

发布前最小检查清单：

- `pyproject.toml` 中的版本号已更新。
- `CHANGELOG.md`、`RELEASE_NOTES.md` 和必要的 README/docs 已更新。
- Python 测试、Explorer lint/build/test 已通过。
- `requirements-ci.txt` 与依赖声明一致。
- `twine check dist/*` 通过。
- wheel 中存在 `semantica/static/index.html` 和 `semantica/static/assets/`。
- 没有把 API Key、密码、私有证书或本地数据打进分发包。
- 检查 `git diff` 和 `git status`，确认 `dist/` 中只有预期产物。

## 8. 版本发布流程

### 7.1 版本来源

Python 包版本来源是 `pyproject.toml` 的 `[project] version`。Explorer 的 `package.json` 是独立的前端工作区版本，不作为 Python 包的发布版本来源；除非有明确需求，不要只修改前端版本而不更新 Python 版本。

### 7.2 推荐发布步骤

1. 从最新 `main` 创建发布分支或准备发布 PR。
2. 更新 `pyproject.toml` 的版本号、`CHANGELOG.md` 和 `RELEASE_NOTES.md`。
3. 完成本地测试、前端构建和 wheel 检查。
4. 合并到 `main`，确认 CI、CodeQL、security scan 等必需检查通过。
5. 创建并推送版本标签，标签格式必须是 `v*`，例如：

```bash
git tag -a v0.7.1 -m "Release v0.7.1"
git push origin v0.7.1
```

6. `.github/workflows/release.yml` 会在 `v*` 标签 push 时自动运行。
7. 发布 workflow 先构建 Explorer，再安装哈希锁定依赖和构建工具，执行 `python -m build --no-isolation`。
8. workflow 会检查 wheel 中的 Explorer 资源，并执行 `twine check`。
9. 通过 PyPI Trusted Publishing（OIDC）上传 wheel 和 sdist。
10. 生成 SLSA build provenance，使用 Sigstore 生成 `.sigstore.json`，最后创建 GitHub Release 并附加 wheel、sdist 和签名文件。

### 7.3 发布安全约束

- 发布 job 使用受保护的 GitHub Environment：`pypi`。需要按仓库设置完成人工审批。
- 发布不使用长期 `PYPI_TOKEN`；不要新增或复制长期 PyPI Token。
- `pypi` environment 的部署分支策略应限制为 `v*` 标签。
- 发布动作只在版本标签触发，不能从任意分支直接发布。
- 不要把 `pypa/gh-action-pypi-publish`、Sigstore 或其他 GitHub Action 改成可变 tag；仓库要求第三方 Action 使用完整 commit SHA。
- 发布 job 中 PyPI 上传必须发生在签名文件写入 `dist/` 之前，否则上传步骤会把 `.sigstore.json` 当成发行包而失败。
- 发布失败时先查看 workflow 日志和 tag 对应提交，不要重复推送同名 tag；修复后递增版本号重新发布。

### 7.4 发布后的验证

```bash
python -m pip install --upgrade semantica==0.7.1
python -c "import semantica; print(semantica.__version__)"
semantica doctor
semantica --help
```

同时检查：

- PyPI 上的 wheel 和 sdist 是否可下载。
- GitHub Release 是否包含 wheel、sdist、`.sigstore.json`。
- `semantica-explorer --help` 是否可执行。
- Explorer 的 `/api/health` 是否返回 `status: ok`。
- 文档中的版本号、安装命令和 release notes 是否一致。

## 9. Docker 与部署构建

### 8.1 构建镜像

`Dockerfile` 使用多阶段构建：第一阶段在 `explorer/` 中执行 `npm ci` 和 `npm run build`，第二阶段安装 Python 包并复制 `semantica/static/`。

```bash
docker build -t semantica-knowledge-explorer:local .
docker run --rm -p 8000:8000 \
  -e SEMANTICA_API_KEY="$(openssl rand -hex 32)" \
  semantica-knowledge-explorer:local
```

健康检查地址：

```text
http://127.0.0.1:8000/api/health
```

### 8.2 使用 Docker Compose

```bash
export SEMANTICA_API_KEY="$(openssl rand -hex 32)"
docker compose up --build
```

Compose 默认启动：

- `explorer`：8000 端口的 Explorer/API 服务。
- `falkordb`：6379 端口的 FalkorDB，并使用 `falkordb_data` volume 持久化数据。

生产环境不要设置 `SEMANTICA_ALLOW_ANONYMOUS=true`。如果只在本机临时调试，可以显式设置该变量，但不得把它作为公开部署默认值。

## 10. 扩展组件时的设计约定

### 新增 Ingestor

- 类名使用 `*Ingestor`，配置通过构造函数传入。
- 返回结果必须能携带原始来源、source id 或 provenance 元数据。
- 网络访问需要超时、错误处理和必要的认证配置。
- 凭据通过环境变量或 secrets manager 注入，不写入代码和示例。

### 新增 Store Backend

- 复用对应的抽象接口和工厂/注册表。
- 明确支持的能力：写入、查询、更新、删除、批量操作、枚举和迁移。
- 对外部服务不可用、超时和版本不兼容提供可诊断错误。
- 添加最小 mock/unit 测试；真实服务测试使用 `integration` marker。

### 新增 Reasoning Rule 或 Ontology 能力

- 规则、事实、结论和解释步骤使用稳定的数据结构。
- 不能只返回最终结果而丢失匹配事实、规则来源或 provenance。
- 对循环、冲突、空输入和非法规则增加测试。

### 修改 ContextGraph 公共 API

- 保持已有参数和返回结构的兼容性，或在 release notes 中明确 breaking change。
- 同步更新 CLI、REST、MCP、导出和 Explorer 使用点。
- 为保存/加载、时间快照、删除/更新和并发访问补回归测试。

### 修改 Explorer

- 状态和 API 类型变更要同时更新前后端。
- 组件测试优先使用已有 `npm run test:*` 脚本。
- 生产资源必须通过 `npm run build` 生成，不要手工编辑 `semantica/static/assets/`。
- 涉及图渲染、时间线或布局的变更，应运行对应 Playwright 端到端测试。

## 11. 常见问题

### 安装后找不到命令

先确认当前虚拟环境已激活：

```bash
python -m pip show semantica
python -m site --user-scripts
semantica --help
```

也可以使用模块形式：

```bash
python -m semantica.mcp_server
python -m semantica.explorer --graph my_graph.json
```

### Explorer 页面为空或 wheel 中没有页面

先从源代码构建前端：

```bash
cd explorer
npm ci
npm run build
cd ..
python -m build --no-isolation
```

然后重新检查 wheel 中是否存在 `semantica/static/index.html` 和 `semantica/static/assets/`。

### 依赖锁检查失败

确认是否修改了 `pyproject.toml` 的依赖声明。如果修改过，使用本文第 5 节的 `uv pip compile` 命令重新生成 `requirements-ci.txt`，不要直接手改版本行。

### 远程访问 Explorer

默认只监听 `127.0.0.1`。如果需要绑定 `0.0.0.0`，必须配置 `SEMANTICA_API_KEY`，并通过 `X-API-Key` 访问受保护接口。公开网络部署还应使用反向代理、HTTPS、访问控制和网络隔离。

## 12. 相关文档

- [README.md](README.md)：产品能力、快速开始、模块示例和安装入口。
- [CONTRIBUTING.md](CONTRIBUTING.md)：贡献、分支、测试和代码风格约定。
- [SECURITY.md](SECURITY.md)：漏洞报告、依赖安全和 CI/CD 供应链安全策略。
- [ARCHITECTURE.md](ARCHITECTURE.md)：系统架构和部署拓扑。
- [explorer/README.md](explorer/README.md)：Explorer 本地开发、构建和前端目录结构。
- [docs/cli-setup.md](docs/cli-setup.md)：CLI、REST、Worker、Explorer 和 MCP 命令说明。
- [docs/explorer-setup.md](docs/explorer-setup.md)：Explorer 使用和安全配置。
