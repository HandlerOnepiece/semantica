# Semantica 安装与依赖说明

本文整理 Semantica 安装、构建依赖、核心依赖和 optional extras 相关问题，内容以当前 `pyproject.toml` 为准。

## 1. 三类依赖

Semantica 的依赖主要分为三类：

| 配置位置 | 作用 | 典型内容 | 是否默认安装 |
| --- | --- | --- | --- |
| `[build-system].requires` | 构建和打包 Python 项目 | `setuptools`、`wheel` | 构建源码包时使用 |
| `[project].dependencies` | Semantica 核心运行依赖 | `numpy`、`pandas`、`rdflib` | 是 |
| `[project.optional-dependencies]` | 按功能选择的可选依赖 | `dev`、`explorer`、`gpu`、`crewai` | 否，除非显式指定 |

### 1.1 构建依赖

当前配置为：

```toml
[build-system]
requires = ["setuptools==84.0.0", "wheel==0.48.0"]
build-backend = "setuptools.build_meta"
```

构建依赖用于把源码构建成 wheel 或 sdist，不是 Semantica 的业务运行库。

执行以下命令时可能会使用它们：

```bash
pip install .
pip install -e .
python -m build
```

默认情况下，pip 会创建隔离的构建环境并自动安装 `[build-system].requires`。如果使用 `--no-build-isolation`，则需要提前安装构建依赖：

```bash
pip install setuptools==84.0.0 wheel==0.48.0
pip install --no-build-isolation .
```

发布和 CI 使用固定版本的构建工具，是为了让构建结果可重复。项目中还通过 `.github/requirements/build-tools.txt` 管理哈希锁定的构建工具。

如果从 PyPI 安装已经构建好的 wheel：

```bash
pip install semantica
```

通常不需要重新构建，因此一般不会重新安装这些构建依赖；wheel 已经在发布前构建完成。

## 2. `pip install semantica` 安装什么

执行：

```bash
pip install semantica
```

会安装：

```text
semantica
+ [project].dependencies 中的核心依赖
+ 核心依赖的传递依赖
```

当前核心依赖包括：

### 数值、数据和机器学习

```text
numpy>=2.0.2
pandas>=1.3.0
scipy>=1.13.1
scikit-learn
pyarrow>=14.0.0
```

### 图、RDF 和网络

```text
rdflib>=6.2.0
networkx>=2.8.0
requests
httpx<0.29.0
grpcio
protobuf>=5.29.1,<8.0
```

### 数据模型、CLI、配置和日志

```text
pydantic>=2.13.4
click
rich>=12.5.0
tqdm>=4.68.3
pyyaml>=6.0
toml>=0.10.0
python-dotenv>=1.2.1
loguru>=0.7.3
structlog>=22.1.0
chardet
pillow
```

部分依赖根据 Python 版本选择不同范围。例如 Python 3.9 使用兼容版本，Python 3.10 及以上可以使用更新版本。项目要求 Python `>=3.9.2`。

普通安装不会自动安装以下内容：

- pytest、Black、isort、flake8、mypy 等开发工具
- FastAPI、uvicorn 和 Explorer 后端
- OpenAI、Anthropic、Gemini 等 LLM SDK
- FAISS、Qdrant、Neo4j 等后端 SDK
- Transformers、PyTorch、spaCy
- GPU/CUDA 依赖
- CrewAI
- PDF、DOCX 等可选解析依赖

## 3. 常用安装命令

| 命令 | 安装内容 |
| --- | --- |
| `pip install semantica` | 已发布包 + 核心依赖 |
| `pip install -e .` | 当前源码 + 核心依赖，editable 模式 |
| `pip install -e ".[dev]"` | 核心依赖 + 开发工具 |
| `pip install -e ".[explorer]"` | 核心依赖 + Explorer 后端依赖 |
| `pip install -e ".[all]"` | 核心依赖 + `all` 中列出的 extras |
| `pip install -e ".[gpu]"` | 核心依赖 + GPU 依赖 |
| `pip install -e ".[crewai]"` | 核心依赖 + CrewAI |
| `pip install -e ".[all,gpu,crewai]"` | 核心依赖 + `all` + GPU + CrewAI |

源码开发常用：

```bash
python -m venv .venv
source .venv/bin/activate                 # Windows: .venv\\Scripts\\activate
python -m pip install --upgrade pip
pip install -e ".[dev]"
```

完整的跨平台开发环境：

```bash
pip install -e ".[all]"
```

## 4. Optional extras

`[project.optional-dependencies]` 中定义的 extras 是按功能拆分的可选依赖集合。安装格式为：

```bash
pip install "semantica[extra-name]"
```

多个 extras 可以组合：

```bash
pip install "semantica[dev,explorer,vectorstore-faiss]"
```

### 4.1 LLM Provider

| Extra | 内容 |
| --- | --- |
| `llm-openai` | OpenAI SDK |
| `llm-groq` | Groq SDK |
| `llm-gemini` | Google Gemini SDK |
| `llm-anthropic` | Anthropic SDK |
| `llm-ollama` | Ollama SDK |
| `llm-deepseek` | DeepSeek，使用 OpenAI SDK |
| `llm-novita` | Novita，使用 OpenAI SDK |
| `llm-litellm` | LiteLLM，仅 Python 3.10+ |
| `llm-instructor` | Instructor |
| `llm-all` | 以上 LLM extras 的组合 |

### 4.2 文档解析和校验

| Extra | 内容 |
| --- | --- |
| `documents` | DOCX、Excel、XML、HTML |
| `parse-docling` | Docling，仅 Python 3.10+ |
| `parse-pdf` | PDFPlumber |
| `shacl` | PySHACL |

### 4.3 数据库和数据源

| Extra | 内容 |
| --- | --- |
| `db-snowflake` | Snowflake |
| `db-databricks` | Databricks |
| `db-arrow` | Arrow / PyArrow |
| `db-salesforce` | Salesforce |
| `db-redshift` | Amazon Redshift |
| `db-all` | 主要数据库 extras 的组合 |
| `ingest-parquet` | Parquet |
| `ingest-arrow` | Arrow、Feather、IPC |
| `ingest-sap` | SAP OData |
| `ingest-git` | GitPython |

### 4.4 模型、Embedding 和 NLP

| Extra | 内容 |
| --- | --- |
| `models-huggingface` | Transformers、PyTorch |
| `embeddings-local` | Sentence Transformers、FastEmbed、ONNX Runtime |
| `nlp-spacy` | spaCy |

### 4.5 图、三元组和向量存储

| Extra | 内容 |
| --- | --- |
| `graph-neo4j` | Neo4j |
| `graph-falkordb` | FalkorDB、Redis |
| `graph-amazon-neptune` | AWS Neptune |
| `graph-apache-age` | Apache AGE |
| `graph-embeddings` | Gensim / 图 Embedding |
| `graph-all` | 以上图存储和图 Embedding extras 的组合 |
| `tripletstore-oxigraph` | Oxigraph RDF 三元组库 |
| `vectorstore-faiss` | FAISS CPU |
| `vectorstore-qdrant` | Qdrant |
| `vectorstore-weaviate` | Weaviate |
| `vectorstore-pinecone` | Pinecone |
| `vectorstore-milvus` | Milvus |
| `vectorstore-pgvector` | PostgreSQL + pgvector |
| `vectorstore-sqlite` | SQLite-vec |
| `vectorstore-all` | 以上向量存储 extras 的组合 |

### 4.6 基础设施、云、可观测性和可视化

| Extra | 内容 |
| --- | --- |
| `infra` | Redis、Celery、Kafka、Pulsar、RabbitMQ |
| `cloud` | AWS、Azure Blob、Google Cloud Storage |
| `monitoring` | Prometheus、OpenTelemetry |
| `watch` | Watchdog 文件监听 |
| `viz` | Pyvis、Graphviz、Matplotlib、Plotly、UMAP 等 |
| `media` | Librosa、OpenCV |
| `gpu` | FAISS GPU、CuPy |

### 4.7 Agent 框架和开发工具

| Extra | 内容 |
| --- | --- |
| `agno` | Agno |
| `crewai` | CrewAI，仅 Python 3.10+ |
| `langchain` | LangChain Core |
| `google-adk` | Google ADK，仅 Python 3.10+ |
| `dev` | pytest、pytest-cov、pytest-asyncio、Black、isort、flake8、mypy、pre-commit、Jupyter、ipykernel |
| `explorer` | FastAPI、uvicorn、WebSocket、multipart 等 Explorer 后端依赖 |
| `explorer-lite` | Streamlit、streamlit-agraph |

### 4.8 Split / Chunking

| Extra | 内容 |
| --- | --- |
| `split-tiktoken` | tiktoken |
| `split-community` | python-louvain |
| `split-topic` | BERTopic、Gensim |
| `split-all` | 以上三个 split extras 的组合 |

## 5. `all` 包含哪些 extras

`all` 的设计目标是“跨平台、可用于 CI 和完整开发的默认环境”，不是无条件安装所有 optional extras。

当前定义为：

```toml
all = [
  "semantica[dev,viz,media,infra,cloud,monitoring,watch,llm-all,models-huggingface,embeddings-local,nlp-spacy,documents,ingest-git,graph-embeddings,split-all,graph-all,tripletstore-oxigraph,vectorstore-all,parse-docling,parse-pdf,ingest-parquet,ingest-arrow,shacl,explorer,agno,langchain,google-adk]"
]
```

直接或递归展开后，`all` 包含：

```text
dev
viz
media
infra
cloud
monitoring
watch

llm-all
llm-openai
llm-groq
llm-gemini
llm-anthropic
llm-ollama
llm-deepseek
llm-novita
llm-litellm
llm-instructor

models-huggingface
embeddings-local
nlp-spacy
documents
ingest-git

graph-embeddings
graph-all
graph-neo4j
graph-falkordb
graph-amazon-neptune
graph-apache-age

split-all
split-tiktoken
split-community
split-topic

tripletstore-oxigraph

vectorstore-all
vectorstore-faiss
vectorstore-qdrant
vectorstore-weaviate
vectorstore-pinecone
vectorstore-milvus
vectorstore-pgvector
vectorstore-sqlite

parse-docling
parse-pdf
ingest-parquet
ingest-arrow
shacl
explorer
agno
langchain
google-adk
```

## 6. 为什么 `all` 不包含 GPU 和 CrewAI

### 6.1 GPU

`gpu` 定义为：

```toml
gpu = [
  "faiss-gpu>=1.7.0",
  "cupy>=10.0.0"
]
```

GPU 依赖通常要求：

- NVIDIA GPU 或对应硬件。
- CUDA Toolkit、驱动和 ABI 兼容。
- 特定操作系统和 Python 版本的二进制 wheel。
- 更大的安装体积和更长的安装时间。

没有 GPU 的开发机或 CI 环境安装 GPU 包没有实际价值，甚至可能安装失败。因此 GPU 被设计成按需安装：

```bash
pip install -e ".[gpu]"
```

也可以在完整环境上追加：

```bash
pip install -e ".[all,gpu]"
```

### 6.2 CrewAI

`crewai` 定义为：

```toml
crewai = ["crewai>=0.80.0; python_version >= '3.10'"]
```

它没有放进 `all`，主要有两个原因：

1. 项目支持 Python `>=3.9.2`，而当前 CrewAI 依赖要求 Python 3.10 或更高。放进 `all` 会破坏 Python 3.9 的通用安装。
2. CrewAI 会引入额外的依赖链，当前项目注释特别指出 `chromadb~=1.1.0` 存在安全审计风险。将它放进默认全量环境会影响 CI 和依赖安全扫描，即使用户并不使用 CrewAI。

需要 CrewAI 时使用 Python 3.10+：

```bash
python -m venv .venv
source .venv/bin/activate
pip install -e ".[dev,crewai]"
```

同时需要 `all` 和 CrewAI 时：

```bash
pip install -e ".[all,crewai]"
```

### 6.3 其他未加入 `all` 的 extras

当前没有包含在 `all` 中的主要 extras 包括：

```text
gpu
crewai
explorer-lite

db-snowflake
db-databricks
db-arrow
db-salesforce
db-redshift
db-all
ingest-sap
```

这些依赖通常具有平台、供应商 SDK、认证、系统库、外部服务、安全或体积方面的特殊要求，因此需要按实际场景选择。

## 7. 安装后的验证

确认 Semantica 已安装：

```bash
python -c "import semantica; print(semantica.__version__)"
semantica --help
```

如果安装了 Explorer：

```bash
semantica-explorer --help
```

如果安装了开发依赖：

```bash
pytest --version
black --version
mypy --version
```

查看已安装的包及版本：

```bash
python -m pip list
python -m pip show semantica
```

## 8. 一句话总结

```text
pip install semantica
= Semantica + 核心 dependencies

pip install -e ".[dev]"
= 源码 editable 安装 + 核心 dependencies + 开发工具

pip install -e ".[all]"
= 核心 dependencies + 项目筛选后的跨平台全量 extras

pip install -e ".[all,gpu,crewai]"
= all + GPU + CrewAI，但需要接受额外的平台、Python 版本和安全依赖约束
```

权威配置位置：

- [pyproject.toml](pyproject.toml)：构建依赖、核心依赖和所有 extras。
- [dev.md](dev.md)：二次开发、调试、构建和发布流程。
- [SECURITY.md](SECURITY.md)：依赖安全和发布供应链安全策略。
