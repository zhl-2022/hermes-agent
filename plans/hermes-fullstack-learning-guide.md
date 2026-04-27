![Hermes Agent 已在 WSL 中运行](image/hermes-fullstack-learning-guide/1776871125052.png)

# Hermes Agent 全栈学习指南

## 目标

这份指南用于记录 Hermes Agent 的学习路线、当前远端部署状态、模型配置方法和后续代码阅读计划。目标不是只会运行 Hermes，而是逐步理解一个完整 Agent 产品如何把模型调用、工具系统、CLI、TUI、Gateway、记忆、技能、配置、测试和部署组织成可维护工程。

推荐学习顺序：

1. 跑通当前部署和 Feishu 入口。
2. 掌握模型、provider、压缩、视觉辅助模型的配置方式。
3. 沿 CLI/Gateway 请求链路阅读源码。
4. 理解 `AIAgent.run_conversation()` 主循环。
5. 完成一个小型 slash command 练习，并用测试验证。

## 当前部署状态

| 项目 | 当前状态 |
|---|---|
| 远端主机 | `srv4` / `11.0.16.4` |
| Hermes 项目目录 | `/root/zhl/hermes-agent` |
| Hermes 配置目录 | `/root/.hermes` |
| 主配置文件 | `/root/.hermes/config.yaml` |
| gateway 服务 | `hermes-gateway.service` |
| 消息平台 | Feishu 已接通 |
| 默认主模型 | `claude-opus-4-6` |
| 默认 provider | `custom` / `foxcode-claude` endpoint |
| 默认上下文配置 | `300000` tokens |
| 自动压缩阈值 | `0.86`，约 `258000` tokens |
| 当前视觉模型 | 本地 `Qwen3-VL-30B-A3B-Instruct` |

常用状态命令：

```bash
ssh srv4
cd /root/zhl/hermes-agent
source venv/bin/activate

hermes gateway status
journalctl -u hermes-gateway.service -n 80 --no-pager
hermes config check
```

## 当前模型配置

### 主模型

主模型当前走 foxcode Claude 中转：

```yaml
model:
  default: claude-opus-4-6
  provider: custom
  base_url: https://code.newcli.com/claude/ultra
  context_length: 300000
  max_tokens: 4096
  api_mode: anthropic_messages
```

注意：`https://code.newcli.com/claude/ultra` 需要 Claude Code 风格请求头。当前代码已在 `agent/anthropic_adapter.py` 中处理：

- 使用 `Authorization: Bearer ...`
- 添加 `user-agent: claude-cli/...`
- 添加 `x-app: cli`
- 移除 Anthropic Python SDK 的 `x-stainless-*` 指纹 headers
- 对该中转禁用不必要的 `anthropic-beta`

### Codex 中转模型

`foxcode-codex` 放在 `providers:` 下，因为 Hermes 的新 keyed schema 更适合一个 provider 下挂多个模型。

```yaml
providers:
  foxcode-codex:
    name: foxcode-codex
    base_url: https://code.newcli.com/codex/v1
    api_mode: chat_completions
    model: gpt-5.4
    models:
      gpt-5.4:
        context_length: 300000
        max_tokens: 4096
      gpt-5.5:
        context_length: 300000
        max_tokens: 4096
```

已实测：

- `gpt-5.4` 在 `/chat/completions` 路径可用。
- 约 `300000` tokens 输入的最小生成请求可成功。

### Claude 中转模型

`foxcode-claude` 当前放在 `custom_providers:` 下：

```yaml
custom_providers:
- name: foxcode-claude
  base_url: https://code.newcli.com/claude/ultra
  api_mode: anthropic_messages
  model: claude-opus-4-6
  models:
    claude-opus-4-6:
      context_length: 300000
      max_tokens: 4096
    claude-sonnet-4-6:
      context_length: 300000
      max_tokens: 4096
```

已实测：

- `count_tokens` 可用。
- `claude-opus-4-6` 约 `300000` tokens 输入的最小生成请求可成功。
- Claude 中转没有兼容的 `/models` 列表，因此已调整 Hermes 的模型切换逻辑：如果模型已显式写入 `config.yaml` 的 `models:`，则信任本地配置，不强制依赖远端 `/models` 校验。

## Feishu 常用命令

### 查看模型

已新增 gateway 命令：

```text
/models
```

它会直接列出 `config.yaml` 中配置的模型，不依赖中转站的 `/models` endpoint。

### 切换模型

会话内临时切换：

```text
/model gpt-5.5 --provider foxcode-codex
/model gpt-5.4 --provider foxcode-codex
/model claude-sonnet-4-6 --provider foxcode-claude
/model claude-opus-4-6 --provider foxcode-claude
```

持久化为全局默认模型：

```text
/model claude-sonnet-4-6 --provider foxcode-claude --global
```

注意：

- 必须使用 `--provider` 指定 provider。
- 不要写成 `/model gpt-5.5 foxcode-codex`，Hermes 不会把第二个普通参数当 provider。
- 如果只发 `/model`，飞书里可能只显示当前模型和简短用法；完整清单用 `/models`。

## 压缩机制

`compression` 是 Hermes Agent 的上下文压缩机制，不是模型服务商参数。

当前配置：

```yaml
compression:
  enabled: true
  threshold: 0.86
  target_ratio: 0.2
  protect_last_n: 20
```

触发点计算方式：

```text
压缩触发 tokens = model.context_length * compression.threshold
```

当前：

```text
300000 * 0.86 = 258000 tokens
```

也就是说，当会话上下文接近 `258000` tokens 时，Hermes 会压缩旧上下文，保留最近 `protect_last_n: 20` 条消息。

为了避免压缩模型被识别为只有 `128000` tokens，当前已显式配置辅助压缩模型：

```yaml
auxiliary:
  compression:
    provider: custom
    model: claude-opus-4-6
    base_url: https://code.newcli.com/claude/ultra
    api_mode: anthropic_messages
    context_length: 300000
    timeout: 120
    extra_body: {}
```

如果以后再次出现类似警告：

```text
Compression model (...) context is 128,000 tokens,
but the main model's compression threshold was ...
```

优先检查：

```bash
python - <<'PY'
from hermes_cli.config import load_config
cfg = load_config()
print(cfg["compression"])
print(cfg["auxiliary"]["compression"])
print(int(cfg["model"]["context_length"] * cfg["compression"]["threshold"]))
PY
```

## 视觉/VL 路由

当前图片理解不直接走主模型，而是先由辅助视觉模型分析，再把文字描述交给主模型。

当前配置：

```yaml
auxiliary:
  vision:
    provider: custom
    model: Qwen3-VL-30B-A3B-Instruct
    base_url: http://127.0.0.1:8003/v1
    timeout: 120
    download_timeout: 30
    extra_body: {}
```

结论：

- 主模型 `claude-opus-4-6` 虽然是多模态模型，但 Hermes 默认不会直接把图片交给主模型。
- Hermes 会优先使用 `auxiliary.vision` 做图片分析。
- 当前 VL 实际走本地 `Qwen3-VL-30B-A3B-Instruct`。
- 如果未来要改成 foxcode Claude/Codex 做视觉识别，需要单独验证 `tools/vision_tools.py` 和 `agent/auxiliary_client.py` 是否完整复用 foxcode 的特殊 headers。

## 常见问题和处理

| 问题 | 原因 | 处理 |
|---|---|---|
| Feishu `/model` 只显示当前模型 | `/model` 主要用于切换，不是完整清单 | 使用 `/models` |
| `/model claude-sonnet-4-6 --provider foxcode-claude` 失败 | Claude 中转没有兼容 `/models` 列表 | 已修复：信任本地 `models:` 配置 |
| Claude 中转返回 403 | 中转站拦截 Anthropic Python SDK 默认 headers | 已在 `agent/anthropic_adapter.py` 移除 SDK 指纹 headers |
| 压缩模型 128K warning | Hermes 认为压缩模型上下文小于压缩阈值 | 设置 `auxiliary.compression.context_length: 300000` |
| `/resrt` warning | slash command 拼写错误 | 用 `/reset` 或 `/new` |
| `providers.openai-cn` URL warning | `${OPENAI_CN_BASE_URL}` 没有展开成有效 URL | 暂不影响 foxcode；不用时可移除该 provider |

## 远端配置维护命令

查看当前关键配置：

```bash
ssh srv4
cd /root/zhl/hermes-agent
source venv/bin/activate

python - <<'PY'
from hermes_cli.config import load_config
cfg = load_config()
print("model =", {k: v for k, v in cfg["model"].items() if k != "api_key"})
print("compression =", cfg["compression"])
print("aux compression =", {k: v for k, v in cfg["auxiliary"]["compression"].items() if k != "api_key"})
print("aux vision =", {k: v for k, v in cfg["auxiliary"]["vision"].items() if k != "api_key"})
print("foxcode-codex =", {k: v for k, v in cfg["providers"]["foxcode-codex"].items() if k != "api_key"})
print("custom =", [{k: v for k, v in x.items() if k != "api_key"} for x in cfg.get("custom_providers", [])])
PY
```

重启 gateway：

```bash
hermes gateway restart
sleep 35
hermes gateway status
```

查看日志：

```bash
journalctl -u hermes-gateway.service -n 120 --no-pager
```

## 日常 Git 更新与服务器发布流程

本项目现在建议保持两个远程仓库：

| 名称 | 含义 | 地址 |
|---|---|---|
| `upstream` | 官方 Hermes Agent 仓库，只用于同步官方更新 | `https://github.com/NousResearch/hermes-agent.git` |
| `origin` | 你自己的 fork 仓库，用于保存你的修改 | `https://github.com/zhl-2022/hermes-agent.git` |

当前开发分支：

```text
zhl/foxcode-feishu-models
```

核心原则：

1. 本地 Windows 负责改代码、提交、同步官方、推送到你的 fork。
2. 服务器只负责拉取你 fork 上的分支并重启 gateway。
3. 服务器尽量不要直接改代码；服务器配置文件 `/root/.hermes/config.yaml` 可以单独维护。

### 每天先检查官方有没有更新

在本地 Windows 执行：

```powershell
cd E:\company_klb\hermes-agent
git fetch upstream main
git status --short --branch
git log --oneline HEAD..upstream/main
```

命令含义：

| 命令 | 作用 |
|---|---|
| `git fetch upstream main` | 只下载官方 `main` 的最新提交，不修改你当前代码 |
| `git status --short --branch` | 查看当前分支和工作区是否干净 |
| `git log --oneline HEAD..upstream/main` | 查看官方比你当前分支多了哪些提交 |

判断结果：

| 现象 | 说明 | 下一步 |
|---|---|---|
| `git log HEAD..upstream/main` 没有输出 | 官方没有新提交 | 不需要 rebase |
| 有若干提交输出 | 官方更新了 | 执行 `git rebase upstream/main` |
| `git status` 显示 `M`、`A`、`??` | 你本地有未提交改动 | 先提交或 stash，再 rebase |

### 情况一：你本地没有改代码，只同步官方更新

适用场景：你今天只是想把官方新版本同步进你的 foxcode 分支。

```powershell
cd E:\company_klb\hermes-agent
git fetch upstream main
git rebase upstream/main
python -m py_compile agent/anthropic_adapter.py gateway/run.py hermes_cli/commands.py hermes_cli/model_switch.py
git push --force-with-lease origin zhl/foxcode-feishu-models
```

命令含义：

| 命令 | 作用 |
|---|---|
| `git rebase upstream/main` | 把你的 foxcode 修改重新放到官方最新版后面 |
| `py_compile` | 做最小语法检查，避免推送明显语法错误 |
| `git push --force-with-lease` | rebase 会改写提交历史，所以需要用安全的强制推送更新 fork |

`--force-with-lease` 比 `--force` 安全：如果远端分支被别人更新过，它会拒绝覆盖。

### 情况二：你本地改了代码

适用场景：你让 Codex 或自己修改了源码、文档、测试。

先提交你的修改：

```powershell
cd E:\company_klb\hermes-agent
git status --short
git add 你修改的文件
git commit -m "简短说明这次修改"
```

再同步官方最新版：

```powershell
git fetch upstream main
git rebase upstream/main
```

如果没有冲突，继续：

```powershell
python -m py_compile agent/anthropic_adapter.py gateway/run.py hermes_cli/commands.py hermes_cli/model_switch.py
git push --force-with-lease origin zhl/foxcode-feishu-models
```

如果出现冲突，不要乱删文件，也不要执行 `git reset --hard`。把终端里的冲突信息发给 Codex，例如：

```text
CONFLICT (content): Merge conflict in gateway/run.py
CONFLICT (content): Merge conflict in hermes_cli/model_switch.py
error: could not apply ...
```

### 情况三：本地已经 push 成功，服务器更新代码

在服务器执行：

```bash
ssh srv4
cd /root/zhl/hermes-agent
source venv/bin/activate

git fetch origin zhl/foxcode-feishu-models
git checkout zhl/foxcode-feishu-models
git pull --ff-only origin zhl/foxcode-feishu-models

python -m py_compile \
  agent/anthropic_adapter.py \
  gateway/run.py \
  hermes_cli/commands.py \
  hermes_cli/model_switch.py

systemctl restart hermes-gateway.service
systemctl status hermes-gateway.service --no-pager -l
journalctl -u hermes-gateway.service -n 100 --no-pager
```

命令含义：

| 命令 | 作用 |
|---|---|
| `git fetch origin ...` | 从你的 fork 下载最新分支 |
| `git checkout ...` | 确保服务器在 foxcode 分支 |
| `git pull --ff-only ...` | 只允许快进更新，避免服务器产生额外 merge commit |
| `systemctl restart ...` | 重启 Feishu gateway，让新代码生效 |
| `journalctl ...` | 查看 gateway 最近日志 |

飞书里验证：

```text
/models
/model claude-opus-4-6 --provider foxcode-claude
/model gpt-5.5 --provider foxcode-codex
```

### Git 冲突是什么

冲突不是代码坏了，而是 Git 不知道应该保留哪一边的修改。

例如官方也改了 `gateway/run.py`，你也改了 `gateway/run.py`，而且两边改到了相邻位置，Git 就会在文件里插入类似内容：

- 第一段从 `<<<<<<< HEAD` 开始，表示官方 `main` 里的内容。
- 中间用 `=======` 分隔。
- 第二段到 `>>>>>>> 5085b209` 结束，表示你的分支里的内容。

含义：

| 标记 | 含义 |
|---|---|
| `<<<<<<< HEAD` 到 `=======` | 当前基底，也就是官方 `upstream/main` 的内容 |
| `=======` 到 `>>>>>>> ...` | 你自己的提交内容 |
| `>>>>>>> 5085b209` | 正在 rebase 的那个本地提交 |

解决冲突不是简单选上面或下面，而是看两边代码的意图，然后合成最终正确版本。

这次 foxcode 分支的冲突处理原则是：

| 文件 | 处理方式 |
|---|---|
| `agent/anthropic_adapter.py` | 保留官方新增的 Azure Anthropic 逻辑，同时保留 foxcode Claude Code headers 逻辑 |
| `hermes_cli/model_switch.py` | 保留官方新增的 `api_mode` 校验参数，同时保留“配置文件里声明的 foxcode 模型可直接切换”的逻辑 |
| `hermes_cli/commands.py` | 保留官方把 `/provider` 作为 `/model` 别名的设计，同时新增 gateway-only `/models` |
| `gateway/run.py` | 增加 `/models` 命令处理，不恢复旧的独立 `/provider` handler |

Codex 解决冲突后的标准动作：

```powershell
rg -n "^(<<<<<<<|=======|>>>>>>>)" agent/anthropic_adapter.py gateway/run.py hermes_cli/commands.py hermes_cli/model_switch.py
python -m py_compile agent/anthropic_adapter.py gateway/run.py hermes_cli/commands.py hermes_cli/model_switch.py
git add agent/anthropic_adapter.py gateway/run.py hermes_cli/commands.py hermes_cli/model_switch.py
git -c core.editor=true rebase --continue
```

其中：

| 命令 | 作用 |
|---|---|
| `rg "^(<<<<<<<\|=======\|>>>>>>>)"` | 检查是否还有真正的冲突标记 |
| `py_compile` | 检查 Python 语法 |
| `git add` | 告诉 Git 冲突已经解决 |
| `git rebase --continue` | 继续应用后续提交 |

注意：看到普通注释里的 `====` 不代表冲突，只有行首的 `<<<<<<<`、`=======`、`>>>>>>>` 才是冲突标记。

### 不推荐执行的命令

除非明确知道后果，否则不要执行：

```powershell
git reset --hard
git rebase --skip
git rebase --abort
git push --force
```

说明：

| 命令 | 风险 |
|---|---|
| `git reset --hard` | 会丢弃未提交修改 |
| `git rebase --skip` | 会跳过你的某个提交，可能把 foxcode 功能跳没 |
| `git rebase --abort` | 会取消本次 rebase，回到 rebase 前状态 |
| `git push --force` | 可能覆盖远端别人新增的提交 |

## 源码学习路线

### 第一阶段：建立产品地图

| 模块 | 文件 | 重点问题 |
|---|---|---|
| 产品概览 | `README.md`, `AGENTS.md` | Hermes 面向用户提供哪些能力？项目有哪些规则？ |
| CLI 入口 | `pyproject.toml`, `hermes_cli/main.py`, `cli.py` | `hermes` 命令如何进入 CLI？ |
| Gateway | `gateway/run.py`, `gateway/platforms/` | Feishu/Telegram/Discord 如何复用同一个 Agent？ |
| Agent 主循环 | `run_agent.py` | `AIAgent` 如何组织对话、工具调用和最终回答？ |
| 工具系统 | `model_tools.py`, `toolsets.py`, `tools/registry.py` | 工具如何被发现、注册、选择和执行？ |
| 配置系统 | `hermes_cli/config.py`, `hermes_cli/runtime_provider.py` | provider、base_url、api_mode 如何解析？ |
| 模型切换 | `hermes_cli/model_switch.py`, `gateway/run.py` | `/model` 如何切换会话模型？ |
| 辅助模型 | `agent/auxiliary_client.py`, `tools/vision_tools.py` | vision/compression 等任务如何选择模型？ |
| 状态存储 | `hermes_state.py`, `hermes_constants.py` | 会话、记忆、profile 路径如何管理？ |

检查清单：

- [x] 远端 Feishu gateway 已跑通。
- [x] foxcode Claude/Codex 模型已接入。
- [x] `/models` 可列出本地配置模型。
- [x] 压缩阈值已配置到约 `258000` tokens。
- [ ] 读懂 CLI 入口。
- [ ] 读懂 `AIAgent.run_conversation()`。
- [ ] 读懂工具 schema 到 handler 的路径。
- [ ] 做一个最小 slash command 练习。

### 第二阶段：跑通最小开发路径

在本地或 WSL 中：

```bash
source venv/bin/activate
uv pip install -e ".[dev]"
./hermes --help
hermes doctor
```

运行测试必须优先使用项目脚本：

```bash
scripts/run_tests.sh tests/agent/
scripts/run_tests.sh tests/tools/
scripts/run_tests.sh tests/hermes_cli/
```

不要直接裸跑 `pytest`，项目规则要求通过 `scripts/run_tests.sh` 保持 CI 环境一致。

### 第三阶段：沿 CLI 请求链路精读

目标链路：

```text
用户输入
-> hermes_cli/main.py
-> cli.py / HermesCLI
-> run_agent.py / AIAgent.run_conversation()
-> LLM API call
-> model_tools.handle_function_call()
-> tools/registry.py
-> tool result message
-> final assistant response
```

精读时回答：

- OpenAI 风格 `messages` 是怎么组装的？
- system prompt 在哪里构建并加入请求？
- enabled toolsets 是怎么解析出来的？
- registry 中的工具如何变成模型可见的 tool schema？
- tool handler 返回什么格式？
- Agent 什么时候继续循环，什么时候停止？
- tool result 和 final response 在哪里分开？

辅助搜索：

```bash
rg "tool_calls|messages.append|chat.completions.create|final_response|max_iterations|iteration_budget" run_agent.py
```

### 第四阶段：读懂模型和 provider 解析

重点文件：

| 文件 | 关注点 |
|---|---|
| `hermes_cli/runtime_provider.py` | `resolve_runtime_provider()` 如何决定 provider、api_key、base_url、api_mode |
| `hermes_cli/model_switch.py` | `/model` 如何解析 `--provider` 并校验模型 |
| `agent/anthropic_adapter.py` | Anthropic Messages API、Bearer auth、headers、Claude Code identity |
| `agent/auxiliary_client.py` | compression、vision 等辅助任务如何选择模型 |
| `gateway/run.py` | Feishu slash command 如何进入模型切换逻辑 |

自测问题：

| 问题 | 要点 |
|---|---|
| 为什么 `foxcode-codex` 可以放在 `providers:`？ | 新 keyed schema 适合一个 provider 多个模型。 |
| 为什么 `foxcode-claude` 的 `/model` 需要特殊处理？ | 中转没有兼容 `/models` 列表，需要信任本地配置。 |
| `api_mode: anthropic_messages` 和 `chat_completions` 有什么区别？ | 前者走 Anthropic Messages API，后者走 OpenAI Chat Completions。 |
| 为什么 Claude 中转需要移除 `x-stainless-*`？ | 中转站会拦截 Anthropic Python SDK 指纹 headers。 |

### 第五阶段：实现一个最小 slash command

建议练习命令：

```text
/learnpath
```

行为：

- 输出当前学习指南路径：`plans/hermes-fullstack-learning-guide.md`
- 输出下一步建议：阅读 `run_agent.py` 的 `AIAgent.run_conversation()`
- 不调用模型
- 不修改用户配置
- 不依赖网络

建议修改：

| 文件 | 改动 |
|---|---|
| `hermes_cli/commands.py` | 增加 `CommandDef("learnpath", ...)` |
| `cli.py` | 在 `HermesCLI.process_command()` 增加分支 |
| `gateway/run.py` | 如果希望 Feishu 支持，也增加 gateway 分支 |
| `tests/hermes_cli/` | 增加命令注册和别名解析测试 |

示例定义：

```python
CommandDef(
    "learnpath",
    "Show the Hermes full-stack learning guide path",
    "Info",
    aliases=("learn",),
)
```

验证：

```bash
scripts/run_tests.sh tests/hermes_cli/
```

## 学习复盘模板

学完一个模块后，可以按下面模板记录并发给 Codex 对照检查：

```text
1. CLI 入口
我理解的调用链是：...

2. Agent loop
我理解 run_conversation() 的流程是：...

3. 工具调用
我理解 tool schema、registry、handler 的关系是：...

4. provider / model
我理解 runtime_provider 和 model_switch 的关系是：...

5. gateway / Feishu
我理解 Feishu 消息进入 Agent 的流程是：...

6. 我还不确定的问题
...
```
