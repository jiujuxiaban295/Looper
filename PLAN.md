# Looper 项目计划

> 本文件交给 Claude Code 执行，是这个仓库的需求来源。

## 0. 给 Claude Code 的工作约定

- 按第 6 节的里程碑顺序做。每完成一个里程碑：跑测试、提交、用几句话汇报，**然后停下来等确认**再做下一个。
- 第 2 节是已经定下的设计决策。觉得哪条有问题，先提出来讨论，不要直接改方向。
- 凡是涉及 Claude Code hooks 和插件的字段名、输入输出格式，实现前先对照官方文档核实：
  - Hooks reference：https://code.claude.com/docs/en/hooks
  - Hooks guide：https://code.claude.com/docs/en/hooks-guide
  - Plugins：https://code.claude.com/docs/en/plugins/overview
- 开发在云端进行，**没有 GPU，也不调用真实模型**。所有测试用假审核器（`FakeReviewer`）、临时 git 仓库和样例数据。真实模型联调由作者在自己的服务器上完成。

## 1. 项目定位

一句话：**Claude Code 每次改完代码、准备结束时，先在本地跑测试；测试通过后，再交给一个干净上下文的独立模型审核测试的范围和逻辑，重点抓"为了让测试通过而改测试"的情况。审核不合格时先停下来，把审核报告交给用户，由用户决定是让 Claude 按意见接着改，还是回头重写需求（spec）。**

- 为什么要独立审核：同一个上下文里刚写完代码的模型，回头检查自己时很难跳出自己的思路。审核者只看需求、代码改动和测试结果，**刻意不看实现者的思考过程**。
- 为什么可以用便宜模型：审核任务范围窄、标准明确，20B–35B 的本地模型就能胜任。
- 用途：作者的个人开源项目，放简历（目标岗位：AI agent 开发）。
- 部署：作者的 Linux 服务器（RTX 5090，32GB 显存），用本地开源模型做审核。
- 和 Claude Code 自带 agent 类型 hook 的区别：审核模型可以是便宜的本地模型、和实现者不同家；每次审核都有记录可查；评审标准专门针对测试完整性；有一套评测集量化审核效果（见 M5）。

## 2. 已确定的设计决策

1. **只做一个场景：代码改动后的测试审核。** 不做通用评测。
2. **"测试是否通过"不交给模型。** 在用户机器上直接跑测试命令，看退出码。测试失败就直接拦回，不调模型。
3. **测试通过后才调审核模型**，审核测试的范围和逻辑。
4. **审核者的输入只有四样：用户的需求、代码 diff、测试文件的 diff、测试运行结果摘要。** 不传实现者的思考过程，也不传它最后的答复，保证上下文干净。
5. **本地部分用 command 类型的 hook 脚本**（diff 和测试都在用户机器上，HTTP hook 拿不到）。服务端只负责审核和记录。
6. **审核模型接口统一为 OpenAI 兼容格式**（`base_url` / `api_key` / `model` 可配置），默认指向作者服务器上的本地 vLLM。
7. **v1 审核器是单次模型调用**，输出结构化结果；v2 再升级为能多步查证的审核 agent。
8. **一切异常都放行（fail-open）**：服务器不可达、超时、解析失败，都不能卡住 Claude Code，只给用户一条提示。
9. **v1 只完整支持 Python + pytest。** 测试命令可配置，其他语言能跑测试，但静态规则（见 M3）先只覆盖 pytest。
10. **审核不合格时交给用户决定，不自动退回 Claude。** 理由：便宜的审核模型会有误报；而且问题可能出在需求本身，而不是实现，这时让 Claude 接着改只会越改越偏。具体做法：Stop hook 放行，用提示消息把审核结论告诉用户，完整报告写到本地；用户可以用插件提供的命令让 Claude 按报告修改，也可以自己重写需求后重新提问。
    - 测试失败不在此列：它是确定性的、意见明确，默认仍自动退回 Claude。
    - 两种行为都可配置：`on_test_fail` / `on_review_fail`，取值 `block`（自动退回 Claude）或 `ask_user`（交给用户）。默认 `on_test_fail=block`、`on_review_fail=ask_user`。无人值守的运行（如 `claude -p`）可以都设成 `block`。

## 3. 架构

```
用户机器（Claude Code）
  ├─ UserPromptSubmit hook（本地脚本）
  │    记下用户需求 + 给工作区拍一个快照
  └─ Stop hook（本地脚本）
       1. 再拍一个快照，和开头的快照对比，得到本轮 diff；没有改动 → 放行
       2. 跑测试命令；失败 → 拦回（附测试输出尾部），不调模型
       3. 测试通过 → 把需求 + diff + 测试结果发给服务器
       4. 根据服务器的结论：
          合格   → 放行
          不合格 → 放行，用提示消息告诉用户结论和建议，完整报告写到本地
                │
  用户看完报告后二选一：
       ├─ /looper:fix  → Claude 读取报告，按意见修改（改完结束时再次走测试和审核）
       └─ 自己重写需求，重新提问
                │
                ▼
作者的 Linux 服务器：looper server（FastAPI）
  ├─ 静态信号：在 diff 里找新增的 skip、被删的断言等
  ├─ 审核模型：OpenAI 兼容接口 → 本地 vLLM / 云端 API
  └─ 记录：每次审核入库（SQLite）
```

## 4. 技术栈

- **本地 hook 脚本**：Python 3，**只用标准库**（用户机器上不能要求 pip 安装任何东西）
- **服务端**：Python 3.11+，`uv` 管理依赖；FastAPI + uvicorn；pydantic / pydantic-settings；SQLite；`openai` SDK
- pytest + ruff；全部代码带类型注解

## 5. 仓库结构（建议）

```
looper/
├─ PLAN.md
├─ README.md
├─ pyproject.toml
├─ Dockerfile
├─ plugin/                         # Claude Code 插件（M4）
│  ├─ .claude-plugin/plugin.json
│  ├─ hooks/hooks.json
│  └─ scripts/
│     ├─ on_prompt.py             # UserPromptSubmit：记需求、拍快照
│     └─ on_stop.py               # Stop：取 diff、跑测试、请求审核
├─ src/looper/                # 服务端
│  ├─ server.py
│  ├─ config.py
│  ├─ store.py
│  ├─ redact.py
│  ├─ signals.py                  # 静态信号
│  └─ reviewer/
│     ├─ base.py                  # 接口与结果数据结构
│     ├─ llm.py
│     ├─ fake.py
│     └─ prompts.py
├─ benchmark/                      # 评测集（M5）
├─ examples/claude-settings.json   # 不用插件时的手动配置
└─ tests/
```

## 6. v1 里程碑

### M1：本地快照与 diff

- `on_prompt.py`（UserPromptSubmit）：
  - 从 stdin 读 hook 输入，取 `session_id`、`prompt_id`、`cwd`、`prompt`
  - 不是 git 仓库就直接退出
  - 拍快照：用**临时 index 文件**（设 `GIT_INDEX_FILE` 指向临时文件，`git add -A` 后 `git write-tree`）得到包含未跟踪文件的 tree hash。**绝不能改动用户的工作区和真实 index。**
  - 把需求和快照 hash 写进状态文件：`<git_dir>/looper/<session_id>.json`（放在 `.git` 里，不会被提交）
  - **stdout 必须为空、退出码 0**：这个事件下 stdout 的纯文本会被加进 Claude 的上下文
- `on_stop.py` 的前半部分：再拍一个快照，和状态文件里的快照 diff，把文件分成"测试文件"和"其他代码"（按路径规则：`tests/`、`test_*.py`、`*_test.py` 等，可配置）。diff 为空就放行。
- 验收：在临时 git 仓库里模拟一轮"开头快照 → 改文件（含新建未跟踪文件）→ 结束快照"，diff 正确；用户的 index 和工作区没有任何变化。

### M2：本地跑测试，失败即拦

- 配置：仓库根目录的 `.looper.toml`（测试命令、测试超时、测试文件路径规则、服务器地址、token），也支持环境变量。没配测试命令就跳过测试和审核，直接放行。
- 跑测试命令，有超时；记录退出码和输出的最后若干行。
- 测试失败（不调服务器）：
  - `on_test_fail=block`（默认）：输出 `{"decision": "block", "reason": "<测试输出尾部 + 一句修改要求>"}`
  - `on_test_fail=ask_user`：放行，用提示消息告诉用户测试没通过
- 防死循环：
  - 状态文件里按 `prompt_id` 记拦截次数，同一轮最多拦 N 次（默认 3，可配置），超过就放行并给用户提示
  - 同时参考输入里的 `stop_hook_active` 字段
  - 注意：Claude Code 在 Stop hook 连续拦截 8 次、中间没有工具调用时会强制放行
- 验收：测试失败 → 输出 block JSON；超过次数上限 → 放行；测试超时 → 放行并提示。

### M3：服务端审核

- `POST /review`，请求体：需求、代码 diff、测试文件 diff、测试结果摘要（命令、退出码、通过/失败数，能解析出来的话）。可选 Bearer token 鉴权（配置了就校验）。请求体大小上限可配置。
- 入库前脱敏：常见密钥格式（`sk-...`、`Bearer ...`、`AKIA...` 之类）替换为 `[REDACTED]`。
- **静态信号**（`signals.py`，确定性规则，只针对 pytest）：在测试文件 diff 里找
  - 新增的 `@pytest.mark.skip` / `skipif` / `xfail`、`pytest.skip(`
  - 被删除或注释掉的 `assert`
  - 被删除的测试函数
  - 断言里的预期值被改动
  - pytest 配置被改动（可能让部分测试不被收集）

  静态信号既直接写进结果，也交给审核模型作为参考。
- **审核模型的评审标准**：
  1. **测试被动手脚**（high）：上面那些信号是否真的削弱了测试；是否用 mock 替换了被测对象本身；断言是否被放宽
  2. **测试范围**（medium）：代码 diff 里新增或改动的行为，有没有对应的测试
  3. **测试逻辑**（medium）：断言是否真的检验了需求，而不是只检查"没抛异常"之类
  4. **偏离需求**（low）：有没有和需求无关的改动
  5. **需求本身的问题**：代码和测试暴露出需求含糊、矛盾或有缺漏时，列出需要用户澄清的问题
- 输出结构：
  ```json
  {
    "verdict": "pass|fail",
    "recommendation": "fix_implementation|clarify_spec",
    "issues": [
      {"severity": "high|medium|low", "category": "...", "file": "...", "evidence": "...", "suggestion": "..."}
    ],
    "spec_questions": ["..."],
    "summary": "..."
  }
  ```
  - 有任何 `high` 级问题即为 `fail`（阈值可配置）。
  - `recommendation` 帮用户做选择：问题出在实现上 → `fix_implementation`；需求有含糊或矛盾 → `clarify_spec`，同时给出 `spec_questions`。
- `LLMReviewer`：`openai` SDK + `base_url`，要求输出 JSON；解析失败重试一次，仍失败就返回"无法审核"，客户端放行。
- diff 太长时截断，优先保留测试文件的 diff。
- 每次审核入库：时间、仓库标识、diff 摘要、静态信号、结论、用的模型。
- 验收：用样例 diff 测试静态信号；`FakeReviewer` 下接口返回正确结构；鉴权、大小限制、超时都有测试。

### M4：串起来，打包成插件

- `on_stop.py` 后半部分：测试通过后调用 `/review`；网络错误或超时一律放行并提示。按结论处理：
  - 合格：放行
  - 不合格且 `on_review_fail=ask_user`（默认）：
    - 把完整报告写成 Markdown：`<git_dir>/looper/reviews/<prompt_id>.md`，并复制一份为 `latest.md`
    - 放行，用 `systemMessage` 给用户一段简短提示：结论、最重要的 1–3 个问题、建议（改实现还是澄清需求）、下一步怎么做（运行 `/looper:fix`，或重写需求后重新提问）
    - 实现前先核对文档，确认 Stop 事件下 `systemMessage` 会显示给用户
  - 不合格且 `on_review_fail=block`：输出 `{"decision": "block", "reason": ...}`，沿用 M2 的防死循环计数
- 插件里提供一个命令（用插件的 skill 或 command 实现，按插件文档核实写法）：
  - `/looper:fix`：让 Claude 读取 `latest.md`，按报告里的问题修改代码；改完结束时会再次走测试和审核
  - 报告里的 `spec_questions` 不交给 Claude 自行回答，由用户决定是否重写需求
- 打包成 Claude Code 插件：`hooks/hooks.json` 注册两个 command hook，脚本路径用 `${CLAUDE_PLUGIN_ROOT}`；服务器地址和 token 做成插件配置项（按插件文档的方式实现）。Stop hook 的 `timeout` 要覆盖"测试超时 + 审核超时"。
- `examples/claude-settings.json`：不用插件时的手动配置。
- `Dockerfile`（服务端）。
- 验收：在临时仓库里用脚本模拟两个 hook 的完整调用（stdin 喂 JSON），服务端用 `FakeReviewer`，以下路径都走通：
  - 测试失败 → 被拦回 Claude
  - 审核不合格（默认配置）→ 放行、输出提示消息、生成报告文件
  - 审核不合格（`on_review_fail=block`）→ 被拦回 Claude
  - 合格 → 放行

### M5：评测集与 README

- `benchmark/`：一组手工构造的案例，每个包含需求、代码 diff、测试 diff、测试结果和标注（是否动了手脚、属于哪一类）。至少覆盖：老实的改动、删断言、加 skip、改预期值、mock 掉被测对象、只加了不检验任何东西的测试。
- 一个脚本对任意 OpenAI 兼容模型跑这套评测集，输出每类的检出率和误报率。云端只用 `FakeReviewer` 验证脚本能跑；真实数字由作者在服务器上用不同模型跑出来。
- `README.md`：要解决的问题、架构图、和 Claude Code 自带 agent hook 的区别、评测集结果表（留空待作者填）、快速开始（服务器上用 vLLM 起 OpenAI 兼容服务，模型选 20B–35B 档；启动服务端；安装插件；写 `.looper.toml`）、配置项表。

## 7. v2 方向（只记录，v1 不做）

- 审核器升级为 agent：能向客户端要更多上下文（比如被改函数的完整代码）再判断
- 更多语言和测试框架的静态规则（Jest、Go test、JUnit 等）
- MCP 工具：让 agent 中途主动请求一次审核
- 异步审核、按用户额度
- 公网部署：HTTPS、反向代理；服务器在家里的话要解决公网暴露问题

## 8. 约束

- **绝不修改用户的工作区、index 和提交历史。** 快照只用临时 index。
- **任何异常都放行**，并用 `systemMessage` 告诉用户发生了什么。
- 本地脚本只用标准库。
- diff 含用户代码。v1 默认只在内网或本机使用，README 里要写明。
- 小步提交，每个 commit 只做一件事。

## 9. 启动提示词

在 Claude Code 里打开这个仓库后，发送：

> 阅读仓库根目录的 PLAN.md，从里程碑 M1 开始实现。每完成一个里程碑就跑测试、提交、简要汇报，然后停下来等我确认。涉及 Claude Code hooks 和插件字段的地方，先对照 PLAN.md 里给的官方文档核实。
