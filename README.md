# 科陆用户手册 RAG 智能体

面向科陆储能产品用户手册的轻量 RAG 问答系统。

- **通用问答**：检索全部 11 份产品手册
- **指定产品问答**：按产品树选择后仅在对应手册内检索
- **混合检索**：向量 + BM25 + Cross-Encoder 重排

## 目录结构

```
manual_qa_agent/
├── 用户手册/                  # PDF 原稿（11 份，已纳入仓库）
├── rag_output_manuals/        # MinerU 解析结果（可选 reindex 用）
├── rag_data/all/              # 预构建 Chroma 知识库
├── config/products.json       # 产品树配置
├── ingest/                    # 离线入库流水线
├── core/                      # Embedding、Chroma、Rerank、LLM
├── manual_qa/                 # QA Agent
├── scripts/setup_server.sh    # 服务器一键准备脚本
├── app.py                     # Gradio UI（默认 7860）
└── api_server.py              # FastAPI（默认 8000）
```

> 仓库内 `用户手册/` 与 `rag_output_manuals/` 为**真实目录**（非 symlink），clone 后可直接使用。

## 环境要求

- Python 3.11+
- 可访问 Midea Gemini / 在线 Embedding API（生产环境）
- Reranker：`BAAI/bge-reranker-v2-m3`（约 230MB，服务器首次需下载）
- 已有 thinkdepth 环境时，通常只需 `pip install FlagEmbedding`

## 本地开发

```bash
cd manual_qa_agent
python -m venv .venv && source .venv/bin/activate
pip install -r requirements.txt
cp .env.example .env   # 填入密钥

# 本地可用 Ollama embedding，编辑 .env：
# EMBED_BACKEND=ollama
# OLLAMA_HOST=http://127.0.0.1:11434
# EMBED_MODEL=qwen3-embedding:4b

python app.py
# 或
uvicorn api_server:app --host 0.0.0.0 --port 8000
```

## 生产部署（thinkdepth / 已有 ML 环境）

知识库已预构建，**clone 后无需重新入库即可问答**。

```bash
git clone <你的仓库URL> manual_qa_agent
cd manual_qa_agent
conda activate thinkdepth

bash scripts/setup_server.sh   # 安装 FlagEmbedding + 下载 reranker

cp .env.example .env
# 编辑 .env：EMBED_BACKEND=online、API Key、RERANK_MODEL 等

uvicorn api_server:app --host 0.0.0.0 --port 8000
```

### 冒烟测试

```bash
curl "http://127.0.0.1:8000/api/v1/kb/status?token=你的PUBLIC_API_TOKEN"
```

## 重新入库（可选）

若要更新手册内容：

```bash
# 跳过 MinerU 解析（已有 rag_output_manuals）
python ingest/step1_build_knowledge.py --skip-parse --sync
python ingest/step2_json2chroma.py --sync

# 或从 PDF 全量解析
python ingest/step1_build_knowledge.py --input-dir ./用户手册
python ingest/step2_json2chroma.py
```

## API 端点

| 方法 | 路径 | 说明 |
|------|------|------|
| GET | `/api/v1/products` | 产品树 |
| POST | `/api/v1/chat` | 问答 |
| POST | `/api/v1/search` | 纯检索 |
| GET | `/api/v1/kb/status` | 知识库状态 |
| POST | `/api/v1/admin/reindex` | 触发增量入库 |

## 配置说明

| 变量 | 说明 |
|------|------|
| `EMBED_BACKEND` | `online`（生产）或 `ollama`（本地） |
| `RERANK_MODEL` | HF 模型 id 或本机目录，如 `$HOME/work/models/bge-reranker-v2-m3` |
| `CHROMA_PERSIST_DIR` | 默认 `./rag_data/all` |
| `PUBLIC_API_TOKEN` | API 鉴权 token |

`.env` 不要提交到 Git。
