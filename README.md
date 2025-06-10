## 摘要

本项目旨在将 **OpenAPI 2/3 规范文件（YAML / JSON）** 转换成 **LLM 友好** 的 Markdown / 纯文本 / 分块 JSON 等格式，并提供 **FastAPI 在线服务** 与 **可编排的自定义选项**。核心思路：

1. **解析 → 校验 → 转换 → 可选优化** 四步流水线；
2. 充分复用现有生态（`openapi-parser` 、`openapi-markdown` 、`openapi-spec-validator` 等），同时加入模板引擎和摘要算法，生成更适合 GPT-like 模型的 Prompt 片段；
3. 提供 REST + WebSocket 双通道，以及 CLI / Python SDK；
4. 支持 Docker-Compose 与⇢Serverless 两种部署模式。参考项目 `vrerv/openapi-markdown` 用于离线 Markdown 输出，在此基础上扩展为在线、异步、可自定义的云化服务。 ([github.com][1], [pypi.org][2])

---

## 1 | 项目概览

| 目标        | 说明                                                                                                                                                                                              |
| --------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **输入**    | OpenAPI 2.0 / 3.0 / 3.1 规范（`.json` 或 `.yaml`，本地上传或远程 URL）                                                                                                                                       |
| **输出**    | *markdown* / *plain-text* / *json* / *html*（Redoc 打包）                                                                                                                                           |
| **核心语言**  | Python 3.11+                                                                                                                                                                                    |
| **主要框架**  | FastAPI + Pydantic v2 + Uvicorn                                                                                                                                                                 |
| **解析/校验** | `openapi-spec-validator` 保证规范合规；`openapi-parser` 或 `openapi3-parser` 转 AST；可选 `datamodel-code-generator` 生成 Pydantic 模型 ([pypi.org][3], [pypi.org][4], [github.com][5], [docs.pydantic.dev][6]) |
| **转换器**   | 参考 `openapi-markdown` 的渲染逻辑；Jinja2 模板引擎；可选 `redocly/cli` 生成静态站点 ([github.com][1], [redocly.com][7])                                                                                             |
| **附加特性**  | Streaming 输出、分块分页、语言本地化、Prompt 模式优化、基本身份验证、速率限制                                                                                                                                                 |

---

## 2 | 目录结构建议

```text
openapi-llm-tool/
├─ app/
│  ├─ api/                # FastAPI 路由
│  │   ├─ v1/
│  │   │   ├─ endpoints.py
│  │   │   └─ schemas.py  # 请求/响应模型
│  ├─ core/               # 业务核心 (parser, converter, utils)
│  ├─ services/           # 服务层 (validation, summarizer, storage)
│  ├─ templates/          # Jinja2 模板（Markdown / Text / Prompt 片段）
│  └─ main.py             # FastAPI ASGI 应用入口
├─ tests/
├─ docs/                  # 开发/架构文档
├─ Dockerfile
├─ docker-compose.yml
└─ pyproject.toml
```

---

## 3 | 关键功能

### 3.1 上传与解析

* **`/convert`**：`POST`；支持 `UploadFile`（或 `file_url` Query）接收规范文件；示例参见 FastAPI 官方文档。([fastapi.tiangolo.com][8], [fastapi.tiangolo.com][9])
* 自动侦测 JSON/YAML；超 5 MB 启用 `asyncio.to_thread` 流式写盘，防止阻塞。

### 3.2 校验

* 先用 **openapi-spec-validator** 严格比对版本 2.0 / 3.0 / 3.1 规范；错误返回 422 JSON。([pypi.org][3])

### 3.3 转换

* **模式一 – Markdown**：调用改造后的 `openapi-markdown` 渲染；保留标题深度 & TOC；支持模板自定义。([github.com][1])
* **模式二 – LLM Prompt**：

  * 将每条 `path + method` 映射为一段 *instructions + JSON schema*；
  * 可选“精简”参数 → 仅保留必填字段 & 示例；
  * `summary_length` 参数控制字段描述压缩比（算法：TextRank / BERT 摘要）。
* **模式三 – 分块 JSON**：保留 AST，方便分步喂入模型；`chunk_size` 设定每块 token 预算。

### 3.4 语言本地化

* 通过 `Accept-Language` 或 query `lang=`（`en` / `zh` / `es` …）；在模板中调用 `gettext` 或简易字典替换。

### 3.5 输出

* 默认 `application/json`，字段：`{ "content": "<converted>" }`；
* 指定 `download=true` → 以 `text/markdown` 或 `text/plain` 流式下载。

---

## 4 | API 设计 (v1)

| Method | URI             | Auth        | Body / Query                                                                                                | 说明             |
| ------ | --------------- | ----------- | ----------------------------------------------------------------------------------------------------------- | -------------- |
| `POST` | `/v1/convert`   | 可选 `Bearer` | Multipart `openapi_file` or `file_url`; `output_format`, `template`, `lang`, `summary_length`, `chunk_size` | 解析 + 转换主入口     |
| `GET`  | `/v1/templates` | 无           | -                                                                                                           | 返回内置模板清单       |
| `GET`  | `/healthz`      | 无           | -                                                                                                           | Liveness probe |
| `GET`  | `/metrics`      | Basic       | Prom / statsd 指标                                                                                            |                |

*鉴权*：可选 API-Key / JWT（使用 `fastapi-security` 简易实现）。
*速率限制*：依赖 Redis + `slowapi` (10 req/min 默认)。

---

## 5 | 示例调用

```bash
curl -X POST \
  -H "Authorization: Bearer $TOKEN" \
  -F "openapi_file=@petstore.json" \
  -F "output_format=markdown" \
  http://localhost:8000/v1/convert
```

Python 客户端片段：

```python
import httpx, pathlib

async def convert(file: pathlib.Path):
    async with httpx.AsyncClient() as client:
        resp = await client.post(
            "http://localhost:8000/v1/convert",
            files={"openapi_file": file.open("rb")},
            data={"output_format": "prompt", "summary_length": 60}
        )
        resp.raise_for_status()
        return resp.json()["content"]
```

---

## 6 | 内部实现

| 模块                | 依赖                                                   | 说明                                                                                                   |
| ----------------- | ---------------------------------------------------- | ---------------------------------------------------------------------------------------------------- |
| **parser.py**     | `openapi-parser` / `openapi3-parser`                 | 将规范转成 Python 对象；若失败则抛 `InvalidSpecError`。([pypi.org][4], [github.com][5])                            |
| **validator.py**  | `openapi-spec-validator`                             | 版本/字段/引用检查；支持 config 忽略某些 WARNING。([pypi.org][3])                                                    |
| **renderer.py**   | `openapi-markdown`, `jinja2`, `mdx_truly_sane_lists` | Markdown 渲染 + 额外 Prompt 模板。([github.com][1])                                                         |
| **summarizer.py** | `nltk`, `sumy`, 可选 `transformers`                    | TextRank / T5 小模型摘要；保证可离线运行。                                                                         |
| **models.py**     | `pydantic` v2                                        | Request/Response 校验；生成 OpenAPI schema 供 `/docs` 使用。([docs.pydantic.dev][10], [docs.pydantic.dev][6]) |

---

## 7 | 部署指南

### 7.1 本地运行

```bash
git clone https://github.com/<you>/openapi-llm-tool
cd openapi-llm-tool
poetry install
poetry run uvicorn app.main:app --reload
```

### 7.2 Docker

```Dockerfile
FROM python:3.11-slim
WORKDIR /app
COPY . .
RUN pip install --no-cache-dir -r requirements.txt
CMD ["uvicorn", "app.main:app", "--host", "0.0.0.0", "--port", "8000"]
```

```bash
docker build -t openapi-llm-tool .
docker run -p 8000:8000 openapi-llm-tool
```

### 7.3 Serverless (可选)

* AWS Lambda + API Gateway：使用 `Mangum` 适配；打包至 `.zip`
* Vercel / Cloudflare Workers：将转换逻辑拆分为 Edge Function，并用 R2/S3 缓存结果。

---

## 8 | CI / CD

* **GitHub Actions**：

  * `pytest` + `coverage` → 阈值 90 %；
  * `docker build` & Push → GHCR；
  * 标签 `v*` 自动部署到 DigitalOcean App Platform。
* **Pre-commit**：`black`, `ruff`, `openapi-spec-validator --strict`。

---

## 9 | 安全与性能

| 维度            | 实践                                                                      |
| ------------- | ----------------------------------------------------------------------- |
| **上传限制**      | `MAX_SIZE_MB` 环境变量；超限即 413。                                             |
| **Sandbox**   | 解析过程运行在 `asyncio.subprocess` sand-box，禁止外部网络访问。                         |
| **缓存**        | 以 `spec_hash` 为键，将渲染结果存 24 h（Redis / SQLite），命中后秒级返回。                   |
| **Streaming** | 使用 `BackgroundTasks` + `await file.seek(0)`，边转换边返回 `text/event-stream`。 |

---

## 10 | Roadmap

1. **🔖 Tag-based Sub-Docs**：按 `tags` 拆分多文件；
2. **🧩 插件式模板系统**：允许用户上传自定义 Jinja2 模板包；
3. **🌐 i18n**：引入 `babel` 自动抽取 & 翻译；
4. **📊 Usage Analytics**：Prometheus + Grafana；
5. **🤖 SDK**：Python / Node / Go 调用封装；
6. **🔒 OAuth2**：对接 Auth0；
7. **📦 N-spec Batch**：一次性上传 ZIP 内多份 spec，异步批量处理。

---

## 11 | 参考与延伸阅读

* OpenAPI 官方规范 3.1.0 （Swagger.io）([swagger.io][11])
* 如何用 Markdown 打造一流 API 文档（Redocly Blog）([redocly.com][12])
* `awesome-openapi3` 工具索引（APIs-guru）([github.com][13])
* `openapi-readme` – 生成 README 的另一种思路 ([pypi.org][14])
* FastAPI 文件上传与异步处理示例 ([fastapi.tiangolo.com][8], [fastapi.tiangolo.com][9])
* Redocly CLI 打包离线文档 ([redocly.com][7])

> **备注**：若模板或摘要算法涉及较大依赖（如 `transformers`），请务必在 `optional-dependencies` 中标明，以免默认镜像体积暴涨。生产镜像可用 `--no-cache-dir` + `pip install --only-binary=:all:` 精简。

---

参考:

[1]: https://github.com/vrerv/openapi-markdown?utm_source=chatgpt.com "Convert openapi spec to markdown file - GitHub"
[2]: https://pypi.org/project/openapi-markdown/?utm_source=chatgpt.com "openapi-markdown - PyPI"
[3]: https://pypi.org/project/openapi-spec-validator/?utm_source=chatgpt.com "openapi-spec-validator - PyPI"
[4]: https://pypi.org/project/openapi-parser/?utm_source=chatgpt.com "openapi-parser - PyPI"
[5]: https://github.com/manchenkoff/openapi3-parser?utm_source=chatgpt.com "manchenkoff/openapi3-parser: OpenAPI 3 parser to use a ... - GitHub"
[6]: https://docs.pydantic.dev/latest/integrations/datamodel_code_generator/?utm_source=chatgpt.com "datamodel-code-generator - Pydantic"
[7]: https://redocly.com/docs/redoc/deployment/cli?utm_source=chatgpt.com "Use the Redoc CLI"
[8]: https://fastapi.tiangolo.com/tutorial/request-files/?utm_source=chatgpt.com "Request Files - FastAPI"
[9]: https://fastapi.tiangolo.com/reference/uploadfile/?utm_source=chatgpt.com "UploadFile class - FastAPI"
[10]: https://docs.pydantic.dev/latest/concepts/json_schema/?utm_source=chatgpt.com "JSON Schema - Pydantic"
[11]: https://swagger.io/specification/?utm_source=chatgpt.com "OpenAPI Specification - Version 3.1.0 - Swagger"
[12]: https://redocly.com/blog/markdown-in-openapi?utm_source=chatgpt.com "Build rich developer experiences with Markdown in OpenAPI"
[13]: https://github.com/APIs-guru/awesome-openapi3?utm_source=chatgpt.com "APIs-guru/awesome-openapi3 - GitHub"
[14]: https://pypi.org/project/openapi-readme/?utm_source=chatgpt.com "openapi-readme - PyPI"
