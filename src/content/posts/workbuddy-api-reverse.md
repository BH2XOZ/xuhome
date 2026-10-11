---
title: WorkBuddy AI 国际版 API 逆向全解析：把免费的 DeepSeek V4.1 Flash 变成自己的 OpenAI 接口
keywords: ["WorkBuddy", "DeepSeek V4.1 Flash", "API逆向", "Electron", "asar", "OpenAI兼容", "New API", "多账号token池", "Python"]
description: 实测 WorkBuddy AI 国际版 v2.137.1：Electron 壳里藏着一个 CodeBuddy CLI sidecar，模型请求其实就是一个普通的 OpenAI 接口。附 asar 解包、token 收割、网关四个怪癖的完整解决方案，以及一个零依赖的 Python 代理（含 Web 控制台 + 多账号池）。
published: 2026-10-11 17:40:00
tags: ["逆向", "AI", "Python", "白嫖"]
categories:
  - 技术
---

> 前排提醒：本文仅讨论**研究自己已购服务的技术实现**，所有样本均来自本人账号且文中全部脱敏。
> 绕过官方客户端直连接口大概率违反服务条款，账号存在被限权的风险，请自行评估后再动手。

最近 WorkBuddy AI 国际版不是送了免费的 DeepSeek V4.1 Flash 嘛，我第一时间就想把它接进我的中转站，结果发现——**它没有开放 API**。

那就只剩一条路了。

这篇文章把我实测踩通的完整链路讲清楚：这是个什么程序、模型请求打到哪里、token 藏在哪、网关有哪些奇怪的规矩、以及最后怎么把它包成一个干净的 OpenAI 接口接进 New API。结论先行——**这玩意儿本质上就是一个普通得不能再普通的 OpenAI 兼容接口，一个本地回显服务器就能把 token 捞出来，整个代理零第三方依赖**。

## 一、先看看这个 App 是个什么东西

Windows 桌面版装在 `%LOCALAPPDATA%\Programs\WorkBuddyAI`，第一眼就是熟悉的 Electron 结构：

```
WorkBuddyAI.exe          204 MB    ← Electron 主程序
resources/
  app.asar               296 MB    ← 全部业务代码在这
  app.asar.unpacked/               ← 不能打进 asar 的原生模块
```

296 MB 的 asar，解包肯定不能整包展开（我当时手贱试了一下，直接给我卡死了）。所以我先用 Python 写了个**极简 asar 读取器**——asar 的格式其实简单得可笑：

```
偏移 4    4字节   header 大小
偏移 8    4字节   头部字符串长度（含 padding）
偏移 12   ...     header（JSON，含所有文件的 offset/size）
之后             文件数据区
```

读取器核心就二十行，按需抽单个文件，秒级完成：

```python
def read(self, rel):
    node = self.header
    for part in rel.replace("\\", "/").split("/")[:-1]:
        node = node["files"][part]
    node = node["files"][parts[-1]]
    self.f.seek(self.base + int(node["offset"]))
    return self.f.read(node["size"])
```

（这里有个小坑：`offset` 在某些 asar 里是**字符串**不是数字，直接相加会报 `unsupported operand type(s) for +: 'int' and 'str'`，我在这浪费了五分钟。)

## 二、真正的模型请求是谁发的

解包后在 `main/` 里翻，很快撞见一个文件叫 `gateway-secret.js`，文件头的注释直接就把架构交代了：

> WorkBuddy Desktop spawn 的 `codebuddy --serve` sidecar 在 127.0.0.1 随机端口暴露 `/api/v1/*`……

**原来桌面端就是个壳，真正干活的是腾讯 CodeBuddy 的终端 Agent CLI**（包名 `@genie/agent-cli`，跟 Claude Code 同构）。进程树长这样：

```
WorkBuddyAI.exe (Electron)
 └─ daemon (workbuddy-server)
     └─ spawn: codebuddy --serve     ← 就是它
         ├─ 本地 ACP 网关  http://127.0.0.1:<随机端口>/api/v1/acp
         └─ 模型请求  ──────────────►  https://www.workbuddy.ai/v2/chat/completions
```

而 CLI 本体就躺在安装目录里，**可以直接跑**：

```bash
$ ls resources/app.asar.unpacked/cli/dist/
codebuddy-headless.js   19 MB
codebuddy.js            23 MB
```

这一步是整个逆向的转折点——**意味着我根本不用管桌面端，直接研究这个 CLI 就行**。

## 三、网关长什么样

在 bundle 里搜端点拼接逻辑，找到这么一行：

```js
"ot resolve model request endpoint: neither getEndpoint() nor product.endpoint is available yet. \
 Set CODEBUDDY_BASE_URL, or ensure the product configuration has finished loading."
```

顺着 `product.endpoint` 找，最终落到产品配置快照 `~/.workbuddy-ai/cache/acc-product-config-v3.json` 里：

| 项目 | 值 |
| --- | --- |
| 模型网关 | `POST https://www.workbuddy.ai/v2/chat/completions` |
| 登录态 | Keycloak JWT，realm = `copilot` |
| Token 有效期 | **约一年**（实测 iat 2026-10-11 → exp 2027-10-07） |
| 配置目录 | `%USERPROFILE%\.workbuddy-ai` |

然后我干了一件很朴素的事：**起一个本地回显服务器，把 CLI 的 base_url 指过去，让它自己把请求送上门**。CLI 支持 `CODEBUDDY_BASE_URL` 环境变量覆盖端点，于是：

```bash
# 1. 起本地回显（只记录，返回 401）
python tools/echo_server.py 19099 work/capture.jsonl

# 2. 让 CLI 往回显服务器发请求
CODEBUDDY_CONFIG_DIR="$HOME/.workbuddy-ai" \
CODEBUDDY_BASE_URL="http://127.0.0.1:19099" \
  "$LOCALAPPDATA/Programs/WorkBuddyAI/resources/app.asar.unpacked/cli/bin/codebuddy" -p "hi"
```

抓到的请求头：

```
POST /chat/completions
Authorization: Bearer eyJhb...（1353 字符的 JWT）
X-User-Id: <用户UUID>
X-Domain: www.workbuddy.ai
X-Product: SaaS
x-codebuddy-request: 1
User-Agent: CLI/2.137.1 WorkBuddy AI/2.137.1
```

**token 就这么到手了**——不用破解加密存储、不用扫内存，让 CLI 自己解密好送出来就行。而 JWT 解码后能看到 `exp` 是一年后，也就是说收割一次能用一年，根本不用折腾自动刷新。

> [!IMPORTANT]
>
> 这个方法的本质是**利用 CLI 自己的登录态**。前提是桌面端处于登录状态，而且这台机器能直连 `www.workbuddy.ai`（实测国内直连可用，不需要代理）。

## 四、然后我就被网关教做人了

拿到 token 直接 curl，第一个坑马上来了：

```json
{"code":11101,"msg":"Non-stream chat request is currently not supported"}
```

行吧，改成流式。第二个坑：

```json
{"code":11128,"msg":"first message is not system prompt"}
```

加上 system 再试，通了。所以我干脆把网关的规矩全测了一遍：

| 发送的 messages | 结果 |
| --- | --- |
| 只有 `user`，没有 system | ❌ 400 `11128` |
| `system` → `user` | ✅ 200 |
| `user` → `system`（system 在第二条） | ❌ 400 `11128` |
| 首条 `assistant` | ❌ 400 `11128` |
| 连续两条 `system` 在最前 | ✅ 200（所以无脑插一条不会冲突） |

也就是说：**`messages[0].role` 必须是 `system`，内容随便填**（我试过填 `"x"` 也能过）。

第三个坑更隐蔽——**每个 chunk 都带一个空的 `reasoning_content`**：

```json
{"delta": {"content": "9", "reasoning_content": "", "function_call": null, "refusal": "", "tool_calls": []}}
```

注意 `reasoning_content` 是**空字符串**，不是 `null`。我自己写客户端测试的时候没感觉，但接进 New API 之后，下游客户端看到每个 token 都带一个"思考字段"，就疯狂渲染思考块——**表现为思考内容疯狂闪烁**。查了半天才确认：这不是 New API 的锅，是上游把 `reasoning_content` 当固定字段每帧都发，标准的 DeepSeek 行为应该是思考阶段才有这个字段。

第四个坑是我自己写代理时踩的：流式响应发完 `[DONE]` 之后客户端**一直转圈显示"输出中"**。原因是 HTTP keep-alive 下，SSE 响应没有 `Content-Length` 也没有正确的分块终止，客户端不知道响应体到哪结束——即使应用层收到了 `[DONE]`，连接层还在等更多数据。修法是用 `Connection: close`，发完 `[DONE]` 直接关连接，客户端见 EOF 即结束。

## 五、于是有了 wb2api

四个坑理清楚之后，代理就很好写了。核心逻辑一共就四件事：

```python
def _prep_body(self, req):
    """补 system（首条必须 system）、强制上游流式。"""
    msgs = list(req.get("messages") or [])
    if not msgs or msgs[0].get("role") != "system":
        msgs = [{"role": "system", "content": default_system}] + msgs
    body = dict(req)
    body["messages"] = msgs
    body["stream"] = True          # 上游只认流式
    return body

def _clean_delta(self, delta):
    """清洗空 reasoning_content。"""
    rc = delta.get("reasoning_content")
    if rc is not None and not rc.strip():
        delta.pop("reasoning_content", None)   # 空串丢掉
    return delta
```

再往上一层是 **token 池**：多个账号轮询 / 随机 / 固定，单次请求跨账号重试，某个号连续失败到阈值就自动禁用（401/403 这类 token 失效的才计入），流式已经吐过内容的不重试——防止下游收到重复输出。

技术选型上我特意**只用 Python 标准库**（`http.server` + `urllib`），因为这东西最终要扔到服务器上跑，装依赖是部署路上最大的敌人。实际跑下来完全够用。

顺手做了个 Web 控制台，账号管理、收割 token、统计、在线测试、实时日志都在里面：

```
┌─────────────────────────────────────────────┐
│  概览    账号总数 / 可用 / 累计请求 / tokens   │
├──────────────────┬──────────────────────────┤
│  账号管理         │  设置                     │
│  ＋收割 token     │  轮询策略 / 重试 / 阈值    │
│  启用 / 停用 / 删 │  reasoning 清洗开关        │
├──────────────────┴──────────────────────────┤
│  在线测试（选模型发消息，流式实时看）           │
├─────────────────────────────────────────────┤
│  运行日志（实时滚动，按级别过滤）               │
└─────────────────────────────────────────────┘
```

控制台里的"收割"按钮就是把上面那套回显服务器流程自动化了：起回显 → 跑 CLI → 抓 `Authorization` → 解析 JWT 拿 uid/email/exp → 存进账号池。

## 六、实测结果

| 实验 | 结果 |
| --- | --- |
| 单轮流式补全 | ✅ 18 prompt tokens（走 CLI 是 ~29000，直连干净得多） |
| 多轮对话（含 assistant 轮） | ✅ 正确记住上下文 |
| 调用方自定义工具（OpenAI function calling） | ✅ 正确发起 `tool_call`，参数流式分片 |
| 跨模型 | ✅ `glm-5.3-flash`、`gpt-6-astra`、`grok-4.7` 等同列表可用 |
| 空 reasoning 清洗 | ✅ 31 个 chunk 中 0 个残留空思考字段 |
| 流式收尾 | ✅ 1.6s 内发完 `[DONE]` 并关连接，客户端正常结束 |
| 自动禁用 | ✅ 假 token 连续失败 2 次（阈值设 2）后自动停用 |
| 多账号轮询 | ✅ 连续两次请求命中不同账号 |

顺带一提，直连时**不需要** `X-Conversation-ID`、`traceparent`、`X-B3-*` 这些头，实测只带 `Authorization` + `Content-Type` 就能通——所以整个请求形态非常干净，就是一个最朴素的 OpenAI 接口。

可用模型里比较有意思的几个：`deepseek-v4.1-flash`（免费线）、`deepseek-v4.1-flash-sg`（付费线/新加坡节点）、`gpt-6-astra`、`gpt-6.1-sol`、`grok-4.7`、`gemini-3.5-flash`、`glm-5.3`、`kimi-k3`、`hy4-preview`。

## 七、部署到服务器（以及我在这里踩的坑）

本地跑通之后扔到阿里云，第一个报错来自 New API：

```
upstream error: do request failed
```

而 wb2api 的日志**是 0 字节**——说明请求压根没到 wb2api。加上服务器上 `curl 127.0.0.1:8787` 明明是 200，那问题必然在 New API → wb2api 这一跳。

排查下来是两件事：

1. **绑定地址**：wb2api 默认绑 `127.0.0.1`，容器/其他进程连不上。启动脚本改成默认 `0.0.0.0`。
2. **New API 的填法**：渠道类型选「自定义」时要填**完整 URL**（`http://host:8787/v1/chat/completions`），如果它拼出 `/v1/v1/chat/completions` 就会打不中。

还有一个**必须提醒的安全问题**：我第一版只给 `/v1/chat/completions` 加了鉴权，控制台和 `/api/*` 全是裸奔的——绑了 `0.0.0.0` 之后，任何人访问 `http://你的IP:8787/` 就能看到你全部账号、甚至收割删除。**token 等于账号身份，这比不鉴权还危险**。现在改成设了 `WB2API_KEY` 后全站鉴权（`/api/ping` 探活除外），控制台会自动弹框让你输密钥。

> [!WARNING]
>
> 对外部署请务必设置 `WB2API_KEY`，并且**不要把端口暴露到公网**（安全组/防火墙收紧），只允许内网访问。

启动脚本也顺手做扎实了：`./start.sh` 默认后台运行（`setsid` 开独立会话，SSH 断开不影响），启动前自动清理占用端口的旧进程，绑 `0.0.0.0` 但没设密钥时会告警。

## 八、写在最后

回过头看，这次逆向的难度其实很低——**没有加密对抗、没有反调试、没有内存扫描**，全部难点都在"信息差"上：

1. 桌面端只是个壳，真正干活的是安装目录里那个可以直接跑的 CLI；
2. 它支持 `CODEBUDDY_BASE_URL`，于是回显服务器就能把 token 骗出来；
3. 网关有四个不成文的规矩，不踩一遍根本不知道。

真正花时间的是工程化的部分：四个怪癖的兼容、token 池的失效重试、流式收尾、鉴权补漏、后台部署。这些才是让一个"能跑"的东西变成"能用"的东西。

目前整个项目（wb2api）是纯 Python 标准库实现，零依赖，`./start.sh` 一条命令上线，控制台管多账号。逆向笔记和协议细节都写在 `REVERSE.md` 里了。

几个待研究的点先记在这：JWT 到期后的自动续期（目前是手动重收割）、上游是否有更细的限速策略、以及 `deepseek-v4.1-flash-sg` 付费线在免费账号下的可用性。这些等用出问题再填坑。

**最后还是那句话**：这只是研究自己已购服务的技术实现，工具是把双刃剑，别拿去做违反服务条款和法律法规的事。
