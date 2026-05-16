# Inference-Escort 项目说明

`inference-escort` 是一个面向大语言模型应用的安全护栏项目。它不训练模型，也不负责执行模型推理；它站在 LLM 调用链的输入端和输出端，对文本进行检测、脱敏、修复、风险评分和放行决策。

如果把一个 LLM 应用比作一条生产线，那么模型本身像“核心加工机器”，而 `inference-escort` 更像前后两道质检门：

```text
用户输入
-> input scanners 检查、脱敏、截断或拦截
-> LLM 生成回答
-> output scanners 检查、恢复、修复、脱敏或拦截
-> 返回给用户或下游系统
```

它主要帮助开发者处理这些问题：

- 用户输入里是否包含 prompt injection、密钥、PII、不可见字符、超长内容或禁止主题。
- 模型输出里是否包含敏感信息、有害链接、拒答、跑题、格式错误、偏见、毒性或乱码。
- LLM 应用能否在进入生产环境前，有一层清晰、可配置、可观测的安全检查。

## 项目定位

官方仓库将 Inference-Escort 描述为 `The Security Toolkit for LLM Interactions`。从代码角度看，它的核心并不是“再造一个模型”，而是给已有模型调用增加一层安全治理。

它适合以下场景：

- 聊天机器人：过滤恶意输入，避免输出有害内容。
- RAG 问答：检查输入攻击、输出敏感信息和回答质量。
- Agent 或工具调用系统：在执行外部动作前检查模型输出。
- 内部知识库或客服系统：防止用户输入和模型回复泄露隐私数据。
- 结构化输出任务：检查 JSON 是否可解析，避免坏格式传给下游程序。

它不适合直接承担以下职责：

- 模型训练。
- 模型推理加速。
- 完整的 MLOps 平台。
- 完整的权限系统或业务网关。
- 对 LLM 安全问题的百分百证明式防御。

更准确地说，它是“LLM 应用层安全护栏”，不是“LLM 本体安全保证”。

## 核心概念

### Scanner

`Scanner` 是项目里最重要的抽象。它可以理解成一个检测器，也可以理解成流水线上的一个工位。

每个 scanner 都会返回三类信息：

```python
sanitized_text, is_valid, risk_score = scanner.scan(...)
```

其中：

- `sanitized_text`：处理后的文本，可能与原文相同，也可能被脱敏、截断、修复或删除部分内容。
- `is_valid`：当前 scanner 是否认为文本通过检查。
- `risk_score`：风险分数，数值越高通常表示风险越大。

从数学上可以把 scanner 看成一个函数：

\[
S_i(x) = (x_i', v_i, r_i)
\]

其中 \(x\) 是输入文本，\(x_i'\) 是处理后的文本，\(v_i\) 是是否通过，\(r_i\) 是风险评分。

对应接口：

- [input_scanners/base.py](inference_escort/input_scanners/base.py)
- [output_scanners/base.py](inference_escort/output_scanners/base.py)

### Input Scanner

`input_scanner` 运行在模型调用之前，负责检查用户输入。

典型能力包括：

- `Anonymize`：识别并替换姓名、邮箱、电话、信用卡等敏感信息。
- `PromptInjection`：检测 prompt injection 攻击。
- `Secrets`：检测 API key、token、密钥等秘密信息。
- `TokenLimit`：检查输入是否超过 token 限制。
- `InvisibleText`：删除零宽字符等不可见文本。
- `BanSubstrings` / `Regex`：根据规则拦截或脱敏文本。
- `Toxicity` / `Sentiment` / `Language`：基于模型或规则判断输入属性。

入口目录：

- [inference_escort/input_scanners](inference_escort/input_scanners)

### Output Scanner

`output_scanner` 运行在模型生成之后，负责检查模型回答能否交付给用户或下游系统。

典型能力包括：

- `Deanonymize`：把输入阶段的脱敏占位符恢复成真实值。
- `Sensitive`：检测输出是否泄露敏感信息。
- `JSON`：检查并尝试修复 JSON。
- `NoRefusal`：检测模型是否拒答。
- `MaliciousURLs`：检测输出中的恶意链接。
- `Relevance`：用 embedding 相似度判断回答和问题是否相关。
- `FactualConsistency`：检查事实一致性。
- `Bias` / `Toxicity` / `Gibberish`：检测偏见、毒性或乱码。

入口目录：

- [inference_escort/output_scanners](inference_escort/output_scanners)

需要注意的是，`Relevance` 这类 scanner 并不适合所有场景。翻译、代码生成、SQL 生成、分类标签输出等任务中，prompt 和 output 本来就可能在语义空间里相差较大。因此它更适合作为特定场景下的可选质量检查，而不应被机械地理解为通用安全规则。

### Vault

`Vault` 是脱敏映射表，主要和 `Anonymize` / `Deanonymize` 配合使用。

例如输入阶段：

```text
John Doe -> [REDACTED_PERSON_1]
```

`Vault` 会保存：

```text
[REDACTED_PERSON_1] -> John Doe
```

输出阶段如果模型生成了 `[REDACTED_PERSON_1]`，`Deanonymize` 可以再把它恢复为真实姓名。

代码入口：

- [inference_escort/vault.py](inference_escort/vault.py)

## 核心流程

### 库模式

最核心的两个函数由包入口暴露：

- [inference_escort/__init__.py](inference_escort/__init__.py)
- [inference_escort/evaluate.py](inference_escort/evaluate.py)

输入扫描流程：

```python
from inference_escort import scan_prompt
from inference_escort.input_scanners import Anonymize, PromptInjection, TokenLimit
from inference_escort.vault import Vault

vault = Vault()
input_scanners = [
    Anonymize(vault),
    TokenLimit(limit=4096),
    PromptInjection(),
]

prompt = "Create a user named John Doe. Email: john@example.com"

sanitized_prompt, results_valid, results_score = scan_prompt(
    input_scanners,
    prompt,
)

print(sanitized_prompt)
print(results_valid)
print(results_score)
```

输出扫描流程：

```python
from inference_escort import scan_output
from inference_escort.output_scanners import Deanonymize, Sensitive

output_scanners = [
    Deanonymize(vault),
    Sensitive(redact=True),
]

model_output = "User [REDACTED_PERSON_1] has been created."

sanitized_output, results_valid, results_score = scan_output(
    output_scanners,
    sanitized_prompt,
    model_output,
)

print(sanitized_output)
print(results_valid)
print(results_score)
```

完整示例可以从这里开始看：

- [examples/openai_api.py](examples/openai_api.py)
- [examples/openai_streaming.py](examples/openai_streaming.py)
- [examples/langchain.py](examples/langchain.py)

### API 模式

项目还提供了一个 FastAPI 服务层，适合把 scanner 能力作为独立 HTTP 服务部署。

API 入口：

- [inference_escort_api/app/app.py](inference_escort_api/app/app.py)

请求和响应结构：

- [inference_escort_api/app/schemas.py](inference_escort_api/app/schemas.py)

scanner 加载逻辑：

- [inference_escort_api/app/scanner.py](inference_escort_api/app/scanner.py)

默认配置：

- [inference_escort_api/config/scanners.yml](inference_escort_api/config/scanners.yml)

服务启动后主要暴露这些接口：

- `POST /analyze/prompt`：扫描 prompt，并返回清洗后的 prompt。
- `POST /scan/prompt`：只扫描 prompt 风险，不返回清洗后的流水线结果。
- `POST /analyze/output`：扫描 output，并返回清洗后的 output。
- `POST /scan/output`：只扫描 output 风险。
- `GET /healthz`：健康检查。
- `GET /readyz`：就绪检查。
- `GET /metrics`：Prometheus 指标，取决于配置。

## 快速开始

### 环境要求

根包要求：

- Python `>=3.10,<3.13`
- 主要依赖包括 `torch`、`transformers`、`tiktoken`、`presidio-analyzer`、`presidio-anonymizer`、`nltk` 等。

项目元信息：

- [pyproject.toml](pyproject.toml)

### 安装开发依赖

```bash
python -m pip install -U pip
python -m pip install ".[dev,onnxruntime]"
```

如果使用仓库自带 Makefile：

```bash
make install-dev
```

### 运行测试

```bash
pytest
```

或：

```bash
make test
```

核心行为测试可以从这里读：

- [tests/test_evaluate.py](tests/test_evaluate.py)
- [tests/input_scanners](tests/input_scanners)
- [tests/output_scanners](tests/output_scanners)

### 启动 API 服务

进入 API 目录：

```bash
cd inference_escort_api
python -m pip install ".[cpu]"
uvicorn app.app:create_app --factory --host 0.0.0.0 --port 8000
```

默认配置文件是：

```text
inference_escort_api/config/scanners.yml
```

如需指定配置文件，可以设置环境变量：

```bash
CONFIG_FILE=./config/scanners.yml uvicorn app.app:create_app --factory --host 0.0.0.0 --port 8000
```

Windows PowerShell 可以使用：

```powershell
$env:CONFIG_FILE = ".\config\scanners.yml"
uvicorn app.app:create_app --factory --host 0.0.0.0 --port 8000
```

## 目录结构

```text
.
├── inference_escort/
│   ├── __init__.py              # 对外暴露 scan_prompt 和 scan_output
│   ├── evaluate.py              # scanner 顺序执行的核心编排逻辑
│   ├── input_scanners/          # 输入侧 scanner
│   ├── output_scanners/         # 输出侧 scanner
│   ├── vault.py                 # 脱敏占位符映射
│   ├── model.py                 # scanner 使用的模型配置
│   └── util.py                  # 日志、设备、风险分数、文本切分等工具函数
├── inference_escort_api/
│   ├── app/
│   │   ├── app.py               # FastAPI 应用入口
│   │   ├── scanner.py           # 根据配置加载 scanner
│   │   ├── config.py            # YAML 配置解析
│   │   └── schemas.py           # API 请求和响应模型
│   └── config/scanners.yml      # API 默认 scanner 配置
├── examples/                    # OpenAI、LangChain、Gemini、Bedrock 等示例
├── tests/                       # 单元测试
├── docs/                        # MkDocs 文档
└── pyproject.toml               # Python 包配置
```

## 推荐学习路径

初学者不建议一开始逐个阅读所有 scanner。更顺滑的路径是：

1. 先读 [pyproject.toml](pyproject.toml)，理解项目依赖和包名。
2. 再读 [inference_escort/__init__.py](inference_escort/__init__.py)，确认公开入口。
3. 重点阅读 [inference_escort/evaluate.py](inference_escort/evaluate.py)，理解 `scan_prompt` 和 `scan_output` 如何串联 scanner。
4. 阅读 [tests/test_evaluate.py](tests/test_evaluate.py)，用测试理解预期行为。
5. 选择一个简单 scanner 细读，例如 [BanSubstrings](inference_escort/input_scanners/ban_substrings.py) 或 [Regex](inference_escort/input_scanners/regex.py)。
6. 再阅读一个模型型 scanner，例如 [PromptInjection](inference_escort/input_scanners/prompt_injection.py) 或 [Toxicity](inference_escort/input_scanners/toxicity.py)。
7. 最后阅读 API 层 [inference_escort_api/app/app.py](inference_escort_api/app/app.py)，理解如何服务化。

这条路径的重点是先掌握“流水线模型”，再深入“单个检测器实现”。

## 设计上的几个重要提醒

### 1. Scanner 不一定会修改文本

有些 scanner 只检测，例如 `PromptInjection`、`Toxicity`、`Relevance`。

有些 scanner 会修改文本，例如 `Anonymize`、`Regex(redact=True)`、`TokenLimit`、`InvisibleText`、`JSON(repair=True)`。

因此调用方真正应该传给模型或返回给用户的，是 `sanitized_prompt` 或 `sanitized_output`，而不是原始字符串。

### 2. `is_valid=False` 不一定等于文本已经不可用

例如 `Anonymize` 检测到敏感信息后，会生成脱敏文本，但仍可能返回 `False`，意思是“原始输入有风险”。业务方需要根据策略决定是直接拦截，还是使用清洗后的文本继续执行。

更严谨的生产系统通常会把结果拆成：

```text
allow     # 直接放行
sanitize  # 使用清洗后的文本
block     # 阻断
review    # 进入人工或异步审核
```

### 3. 不要盲目开启所有 scanner

不同业务需要不同组合。一个轻量聊天系统、一个 RAG 系统、一个 JSON 结构化抽取系统、一个代码生成系统，风险模型并不相同。

更合理的方式是按场景组织 scanner：

```text
minimal:
  TokenLimit + PromptInjection + Sensitive

pii_safe:
  Anonymize + Secrets + Sensitive + Deanonymize

json_api:
  TokenLimit + PromptInjection + JSON + Regex

public_chat:
  TokenLimit + PromptInjection + Toxicity + BanTopics + Sensitive

rag:
  PromptInjection + Sensitive + FactualConsistency
```

### 4. API 模式要注意状态隔离

`Vault` 用来保存脱敏映射。如果服务面向多用户或多租户，使用全局 `Vault` 可能导致不同请求之间的映射混杂。生产化时建议按请求、会话或租户隔离，并设置过期和清理策略。

相关代码：

- [inference_escort_api/app/app.py](inference_escort_api/app/app.py)
- [inference_escort/vault.py](inference_escort/vault.py)

## 如何新增 scanner

新增 input scanner：

1. 在 [inference_escort/input_scanners](inference_escort/input_scanners) 下新增类。
2. 实现 `scan(self, prompt: str) -> tuple[str, bool, float]`。
3. 在 [inference_escort/input_scanners/__init__.py](inference_escort/input_scanners/__init__.py) 中导出。
4. 在 [inference_escort/input_scanners/util.py](inference_escort/input_scanners/util.py) 中加入按名称加载逻辑。
5. 在 [tests/input_scanners](tests/input_scanners) 中添加测试。
6. 如需 API 配置加载，在 [inference_escort_api/app/scanner.py](inference_escort_api/app/scanner.py) 中补充配置逻辑。

新增 output scanner：

1. 在 [inference_escort/output_scanners](inference_escort/output_scanners) 下新增类。
2. 实现 `scan(self, prompt: str, output: str) -> tuple[str, bool, float]`。
3. 在 [inference_escort/output_scanners/__init__.py](inference_escort/output_scanners/__init__.py) 中导出。
4. 在 [inference_escort/output_scanners/util.py](inference_escort/output_scanners/util.py) 中加入按名称加载逻辑。
5. 在 [tests/output_scanners](tests/output_scanners) 中添加测试。
6. 如需 API 配置加载，在 [inference_escort_api/app/scanner.py](inference_escort_api/app/scanner.py) 中补充配置逻辑。

官方扩展说明：

- [docs/customization/add_scanner.md](docs/customization/add_scanner.md)

## 与 OWASP LLM 风险的关系

Inference-Escort 的很多能力可以对应到 LLM 应用常见风险：

- Prompt Injection：`PromptInjection`、`BanSubstrings`、`Regex`
- Sensitive Information Disclosure：`Anonymize`、`Secrets`、`Sensitive`
- Insecure Output Handling：`JSON`、`Regex`、`MaliciousURLs`、`URLReachability`
- Model Denial of Service：`TokenLimit`
- Overreliance / Misinformation：`FactualConsistency`、部分质量检查类 scanner

需要强调的是，scanner 是防线之一，不是完整安全体系。生产系统还需要权限控制、日志脱敏、速率限制、隔离执行、人工审核、红队测试和监控告警。

## 项目链接

- 项目仓库：https://github.com/callme110/Inference-Escort
- 项目文档：https://callme110.github.io/Inference-Escort/
- API 文档：https://callme110.github.io/Inference-Escort/api/overview/
- OWASP Top 10 for LLM Applications：https://owasp.org/www-project-top-10-for-large-language-model-applications/

## 一句话总结

`inference-escort` 的价值不在于替代模型，而在于让模型调用链多一层可组合、可配置、可观测的安全检查。它把“用户输入能不能进模型”和“模型输出能不能交付出去”这两个问题，从零散的业务判断，变成了一套可复用的 scanner 流水线。
