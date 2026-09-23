---
title: "Hermes 接入抖音平台实现自动发布：现状澄清与完整接入方案"
date: 2026-09-23
tags: [hermes, douyin, cron, mcp, webhook]
author: "AlwaysWannaFly01"
---

你有没有过这种瞬间：想让自己的 agent 每天定时把剪好的视频发到抖音，结果翻遍 Hermes 官方文档，messaging gateway 那页的 Platform Comparison 表里根本没有「抖音」这一行，TikTok 也没有。

这不是文档写漏了。Hermes 官方 messaging gateway 目前没有抖音原生适配器，也不支持 TikTok，那张表一共列了 28 个平台，中文生态里有钉钉、飞书/Lark、企业微信、微信、QQ、元宝，唯独缺抖音。所以「自动发布到抖音」这件事，只能走间接接入：用抖音开放平台的开放 API 做真正的内容发布，再用 Hermes 的扩展机制去触发和编排它。

这篇文章把这条链路从头讲清楚。先说结论，再给方案，最后给能直接抄的代码。

## Hermes 现状：没有抖音适配器，但有五种扩展口

先确认边界。Hermes 的 messaging gateway 是一个常驻后台进程，把同一个 agent 挂到 Telegram、Discord、Slack、WhatsApp、Email 等平台上。这些是「聊天通道」，agent 收消息、回消息。抖音不是聊天工具，它是一个内容发布平台，两者的交互模型不同，所以官方没有为它写适配器。

这里得把话说透。聊天通道和内容发布平台是两种产品形态：Telegram、Discord 是等 agent 来收消息的 IM 通道，而抖音开放平台给你的是一组 OAuth、上传、发布的 REST API，不是一条让 agent 挂着等消息的线，两者没有天然对齐点。官方文档其实提供了「新增平台适配器」的开发者指南，说明这份列表是开放可扩展的，只是抖音适配器官方还没写。本文不去重写那个适配器，而是用 Hermes 现成的扩展口拼一条更轻的链路。

Hermes 留了五个扩展口，这正是间接接入的抓手：

- **Cron 定时任务**：支持 no-agent 模式，脚本按点跑、stdout 原样投递、全程零 LLM 调用。这是「每天定时发一条」最直接的落点。
- **Plugins 插件**：在 plugins 目录注册自定义 tool 和 hook，把发布能力变成 agent 的一等工具。
- **MCP 客户端**：连外部 MCP server，启动时自动发现工具，注入成 mcp_服务名_工具名 的形式。
- **Webhook**：收外部事件的 HTTP 入口，事件一到就能触发一次发布。
- **API Server**：OpenAI 兼容的 HTTP 端点，让脚本或其它程序直接调。

有了这五个口，「自动发布抖音」在 Hermes 这一侧就不是问题。真正要下功夫的，在抖音那一侧。

## 抖音开放平台侧：权限、OAuth、发布 API

抖音开放平台有 API 域名 open.douyin.com 和开发者控制台 developer.open-douyin.com。做「自动发布自己账号的内容」属于自营开发者场景，门槛如下。

**企业认证是硬门槛。** 内容发布类权限普遍要求完成主体认证和对公验证，个人开发者基本拿不到「代替用户发布内容到抖音」这个能力。这一步过不去，后面全都不用看。

**创建一个应用，拿到 Client Key 和 Client Secret。** 这两个值相当于你在抖音开放平台的账号密码，后面所有调用都靠它们鉴权。

**申请发布权限。** 关键 scope 是 video.create.bind，中文叫「代替用户发布内容到抖音」，在控制台的「应用详情 > 能力管理 > 能力实验室」里申请，要过平台审核。上传视频、发布视频、上传图片都挂在这个 scope 下。

**OAuth 授权拿 token。** 流程是三步：

1. 让用户访问 `GET https://open.douyin.com/platform/oauth/connect/`，带上 client_key、scope、redirect_uri，浏览器出一个扫码页，用户扫码授权后回调一个 code。
2. 拿 code 换 token：`POST /oauth/access_token/`，返回 access_token（15 天）、refresh_token（30 天）、open_id。这里要注意 Content-Type 是 application/x-www-form-urlencoded，不是 JSON。
3. 不需要用户授权的接口，用 `POST /oauth/client_token/` 生成 client_token（2 小时，应用级凭证）。

这几个鉴权端点在无有效凭证时都会返回明确错误码而不是 404：换 access_token 时是 10002（参数错误）或 10007（code 已失效），client_token 报 10003（密钥无效），refresh_token 过期报 10010，发布接口缺 token 报 2190002（access_token 无效）、2190008（access_token 过期）。这套错误码本身是个信号：路由存在、参数名被识别，只是凭证不对。开发时可以反过来用它核对一个 endpoint 是否真实存在，而不是对着文档臆造。

**发布视频的主链路是两步。** 先 `POST /api/douyin/v1/video/upload_video/` 用 multipart 把视频文件传上去，拿到一个加密的 video_id；再 `POST /api/douyin/v1/video/create_video/` 用 JSON 提交 video_id 加标题 text，返回 item_id。两个接口的 Query 里都要带 open_id，请求头带 access-token。发布图片走 `upload_image` 上传拿 image_id，scope 同样是 video.create.bind。

把上面这套调用用 curl 串起来，一次真实发布的请求长这样（尖括号里的值换成你自己的）：

```bash
# 1. 用授权码换 access_token（form 编码，不是 JSON）
curl -X POST https://open.douyin.com/oauth/access_token/ \
  -d "client_key=<key>&client_secret=<secret>&code=<code>&grant_type=authorization_code"

# 2. 上传视频（multipart，Query 带 open_id）
curl -X POST "https://open.douyin.com/api/douyin/v1/video/upload_video/?open_id=<open_id>" \
  -H "access-token: <access_token>" -F "video=@/path/to/video.mp4"

# 3. 创建视频即发布（JSON，返回 item_id）
curl -X POST "https://open.douyin.com/api/douyin/v1/video/create_video/?open_id=<open_id>" \
  -H "access-token: <access_token>" -H "Content-Type: application/json" \
  -d '{"video_id":"<video_id>","text":"标题"}'
```

有个坑要提前说清楚：**发布图片/图文的那个接口，官方文档页的正文是前端动态加载的，curl 拿不到具体 endpoint 和参数，得登录控制台看。** 我在这里不编造它，视频这条主链路已经足够把方案跑通。

## 三条接入路径：谁在什么时机触发发布

抖音侧要做的调用是一样的，三条路径的区别在于「谁在什么时机、以什么形式触发发布脚本」。

| 维度 | A：cron + Python 脚本 | B：自定义 MCP server | C：webhook 触发 |
|---|---|---|---|
| 触发时机 | 定时 | 对话中随时（agent 决定） | 外部事件到达即触发 |
| LLM 是否参与 | 可完全不用 | 是，agent 编排 | 视配置而定 |
| 实现成本 | 最低 | 中 | 中 |
| 适用场景 | 每天固定发一条 | 让 agent 按内容动态决定 | 上游就绪即发 |

三条不互斥。通常 A 是起点，需要对话编排时上 B，有上游系统时加 C。

另外提一句，Hermes 的 plugin 机制也能注册一个 douyin_publish 自定义 tool，语义上和 MCP 类似，但走的是 Hermes 原生插件契约（plugin.yaml 加 register_tool）。选 MCP 还是 plugin，看你想不想让这个工具脱离 Hermes 复用：MCP server 是标准协议，别的 MCP 客户端也能连；plugin 更贴近 Hermes 本身，但换个框架就带不走。

### 路径 A：cron + 脚本（最简，零 LLM）

抖音发布本质是两组带鉴权的 HTTP 调用，一个 Python 脚本就能做完。Hermes 的 cron 负责按点触发，而且支持 no-agent 模式：脚本按时运行、stdout 原样投递、全程不碰模型。对一个「把准备好的内容发出去」这种确定性动作来说，零 LLM 才是正确的抽象，省掉 token 成本、模型故障、推理延迟。

no-agent 模式的语义很干净：脚本 stdout 非空就投递，空 stdout 就静默 tick，非零退出码就投一条错误告警。脚本必须放在 `~/.hermes/scripts/` 目录内，`.sh`/`.bash` 用 bash 跑，其它扩展名用当前 Python 解释器跑。要留意一点：cron 脚本不继承 Hermes 进程里的 provider 凭证，所以密钥得自己解决，放到 `~/.hermes/.env` 里让脚本自己读。

还有一层要自己管：token 生命周期。access_token 15 天过期，脚本每次跑之前得判断是否过期，过期就用 refresh_token 换新的；refresh_token 自己 30 天过期、最多刷 5 次，再往后只能重新引导用户授权。这段续期逻辑应该写进脚本，而不是靠 cron 的定时去猜。

### 路径 B：自定义 MCP server（工具化发布）

路径 A 是脚本被动被 cron 叫。如果你希望 agent 在对话里主动编排发布，比如对它说「把这条视频发到抖音，标题做 A/B 两个版本各发一次，先查下今天发过没」，那就得把发布能力做成一个工具。MCP 是标准做法：Hermes 内置 MCP 客户端，启动时连上 server、发现工具、注入成 `mcp_douyin_publish_video` 这种一等工具，在所有平台工具集里同时可用。

工具化是「可组合性」的分水岭。脚本是一次性的，工具可以被 agent 跟读文件、改标题、查历史这些操作串成流水线。代价是要常驻一个 MCP server 进程，以及多写一点样板代码。

### 路径 C：webhook 触发（事件驱动）

定时是「时间到了发」，工具化是「agent 觉得该发」，但真实业务里最常见的触发是「上游就绪了发」，比如内容审核通过、剪辑流水线出片、后台 CMS 点了发布。这时候轮询是笨的，webhook 让事件直接唤醒一次发布，延迟最低。

Hermes 的 webhook 给了两个档位：`--deliver-only` 纯转发通知，不跑 agent；或者路由里设 `cron_job` 指向一个已有的 cron job，事件一到就立即触发那个 job，而不是等下一个定时 tick。

## 完整操作流程：从申请到跑通

以路径 A 为主线，B/C 在对应步骤处标注替换点。

1. 登录 open.douyin.com 或 developer.open-douyin.com，完成企业开发者认证。
2. 创建应用，拿到 client_key 和 client_secret，配置 OAuth 回调地址 redirect_uri（要 https）。
3. 在「应用详情 > 能力管理 > 能力实验室」申请「代替用户发布内容到抖音」（scope=video.create.bind），等审核。
4. 引导用户授权：拼 platform/oauth/connect/ 授权 URL，扫码后 code 回调，调 oauth/access_token/ 换 access_token + refresh_token + open_id。
5. 写发布脚本，封装 upload_video → create_video 调用链。
6. 把密钥放进 ~/.hermes/.env，脚本里读取。
7. 挂 cron：`hermes cron create "0 9 * * *" --no-agent --script douyin_publish.py --deliver telegram --name "douyin-daily-publish"`。
8. 验证：`hermes cron list` 看 job，`hermes cron run <id>` 手动触发一次，确认作品出现在抖音账号。

B 路径在第 5 步换成写 MCP server 并配 config.yaml 的 mcp_servers；C 路径换成 webhook subscribe，路由指向发布 job。

跑通一次之后，这条链路的日常形态就固定下来了：脚本每天被 cron 唤醒，读 .env 里的凭证，必要时刷新 token，上传视频、创建视频，把返回的 item_id 打进 stdout 投递到你的 Telegram。人只在三种时候需要介入：审核没过、token 彻底过期需要重新授权、内容本身不合规被平台拦下。剩下的都是机器的事。

## 限制与坑：动手前必须知道

- **企业认证绕不过去。** 个人开发者拿不到发布权限，这是第一条分水岭。
- **权限要单独申请、单独过审。** video.create.bind 不是默认开通的，而且挂小程序、POI 锚点还要求企业资质关联账号。
- **token 会过期，且不能无限续。** access_token 15 天，refresh_token 30 天，最多刷 5 次，之后必须重新引导用户授权。脚本里要处理 refresh_token 过期的 10010 错误，而不是傻傻重试。
- **有配额。** 一个用户在一个应用下一天最多发布 75 个作品（错误码 2114007）；视频单条不超过 15 分钟，标题不超过 1000 字。超过 50MB 的视频建议分片上传，超过 300MB 必须分片，总大小控制在 4GB 以内，编码建议 16:9、720p 以上的竖版 mp4 或 webm。
- **发布不等于可见。** 创建后有平台审核期，期间只有自己可见，可能被拒或下架。脚本别把 HTTP 200 当成功，要处理「调用成功但审核不过」的异步结果。
- **密钥别硬编码。** access_token、refresh_token 是机密，放 ~/.hermes/.env 或密钥源，别写进代码和日志。
- **测试和正式环境要隔离。** 正式上线后，别拿正式 client_key/secret 在测试环境取 token，否则会导致线上 token 失效。

## 代码：路径 A 的完整脚本

下面是一个完整可跑的发布脚本，覆盖「拿 token、上传视频、创建视频发布」全链路。endpoint 都来自抖音开放平台官方文档，视频主链路是 upload_video → create_video（scope=video.create.bind）。拆成两步是有原因的：上传是大文件的多部分请求，耗时且容易失败需要重试；发布是轻量 JSON 调用。分开之后，失败时可以只重传视频，而不会重复发帖。

```python
#!/usr/bin/env python3
# -*- coding: utf-8 -*-
import json, os, sys, urllib.request, urllib.parse, urllib.error

BASE = "https://open.douyin.com"

def _load_env(path="~/.hermes/.env"):
    """cron 脚本不继承 provider 凭证，这里从 .env 简单加载 KEY=VALUE。"""
    p = os.path.expanduser(path)
    if not os.path.exists(p):
        return
    for line in open(p, encoding="utf-8"):
        line = line.strip()
        if not line or line.startswith("#") or "=" not in line:
            continue
        k, v = line.split("=", 1)
        os.environ.setdefault(k.strip(), v.strip().strip('"').strip("'"))

_load_env()
CLIENT_KEY = os.environ.get("DOUYIN_CLIENT_KEY", "")
CLIENT_SECRET = os.environ.get("DOUYIN_CLIENT_SECRET", "")

def _post(url, headers, body_bytes):
    req = urllib.request.Request(url, data=body_bytes, method="POST")
    for k, v in headers.items():
        req.add_header(k, v)
    try:
        with urllib.request.urlopen(req, timeout=30) as resp:
            return json.loads(resp.read().decode("utf-8"))
    except urllib.error.HTTPError as e:
        return {"error": e.code, "body": e.read().decode("utf-8")}

def get_access_token(code):
    """OAuth 授权码换用户 access_token（有效期 15 天）。"""
    body = urllib.parse.urlencode({
        "client_key": CLIENT_KEY, "client_secret": CLIENT_SECRET,
        "code": code, "grant_type": "authorization_code",
    }).encode("utf-8")
    return _post(f"{BASE}/oauth/access_token/",
                 {"Content-Type": "application/x-www-form-urlencoded"}, body)

def _open_id_url(path, open_id):
    sep = "&" if "?" in path else "?"
    return f"{BASE}{path}{sep}open_id={urllib.parse.quote(open_id)}"

def upload_video(access_token, open_id, video_path):
    """上传视频文件，返回加密 video_id（视频 ≤15 分钟）。"""
    boundary = "----douyin-upload-7f3a"
    with open(video_path, "rb") as f:
        file_bytes = f.read()
    body = (
        f"--{boundary}\r\n".encode()
        + f'Content-Disposition: form-data; name="video"; filename="{os.path.basename(video_path)}"\r\n'.encode()
        + b"Content-Type: application/octet-stream\r\n\r\n"
        + file_bytes
        + f"\r\n--{boundary}--\r\n".encode()
    )
    return _post(_open_id_url("/api/douyin/v1/video/upload_video/", open_id),
                 {"Content-Type": f"multipart/form-data; boundary={boundary}",
                  "access-token": access_token}, body)

def create_video(access_token, open_id, video_id, text):
    """创建视频（发布）：video_id + 标题 text（≤1000 字），返回 item_id。"""
    body = json.dumps({"video_id": video_id, "text": text}).encode()
    return _post(_open_id_url("/api/douyin/v1/video/create_video/", open_id),
                 {"Content-Type": "application/json",
                  "access-token": access_token}, body)

def main():
    if len(sys.argv) < 3:
        print("用法: douyin_publish.py <视频路径> <标题>", file=sys.stderr)
        return 1
    video_path, text = sys.argv[1], sys.argv[2]
    access_token = os.environ.get("DOUYIN_ACCESS_TOKEN", "")
    open_id = os.environ.get("DOUYIN_OPEN_ID", "")
    if not (access_token and open_id):
        print("缺少 DOUYIN_ACCESS_TOKEN / DOUYIN_OPEN_ID", file=sys.stderr)
        return 1
    up = upload_video(access_token, open_id, video_path)
    # 上传返回结构：data.video.video_id
    video_id = up.get("data", {}).get("video", {}).get("video_id")
    if not video_id:
        print(json.dumps(up, ensure_ascii=False))
        return 1
    result = create_video(access_token, open_id, video_id, text)
    print(json.dumps(result, ensure_ascii=False))  # stdout 会被 cron 投递
    return 0

if __name__ == "__main__":
    raise SystemExit(main())
```

挂到 cron，一条命令，零 LLM：

```bash
# 每天 09:00 跑一次，脚本 stdout 原样投递到 Telegram
hermes cron create "0 9 * * *" \
  --no-agent \
  --script douyin_publish.py \
  --deliver telegram \
  --name "douyin-daily-publish"
```

想投递到别的目标，`--deliver` 换成 `discord:#ops`、`slack:#engineering`、或者 `local`（存到 ~/.hermes/cron/output/）。

## 代码：路径 B 的 MCP server 骨架

用 FastMCP 暴露两个工具，配好 config.yaml 后，对话里说「帮我把这条视频发到抖音」就能触发。

```python
#!/usr/bin/env python3
# -*- coding: utf-8 -*-
import json, os, urllib.request
from mcp.server.fastmcp import FastMCP

mcp = FastMCP("douyin")
BASE = "https://open.douyin.com"
CLIENT_KEY = os.environ.get("DOUYIN_CLIENT_KEY", "")
CLIENT_SECRET = os.environ.get("DOUYIN_CLIENT_SECRET", "")

def _post(url, headers, body_bytes):
    req = urllib.request.Request(url, data=body_bytes, method="POST")
    for k, v in headers.items():
        req.add_header(k, v)
    with urllib.request.urlopen(req, timeout=30) as resp:
        return json.loads(resp.read().decode("utf-8"))

@mcp.tool()
def douyin_client_token() -> str:
    """生成抖音 client_token（应用级凭证，不需要用户授权，有效期 2 小时）。"""
    body = json.dumps({"client_key": CLIENT_KEY, "client_secret": CLIENT_SECRET,
                       "grant_type": "client_credential"}).encode()
    return json.dumps(_post(f"{BASE}/oauth/client_token/",
                            {"Content-Type": "application/json"}, body),
                      ensure_ascii=False)

@mcp.tool()
def douyin_publish_video(access_token: str, open_id: str,
                         video_path: str, text: str) -> str:
    """上传并发布抖音视频：upload_video -> create_video（scope=video.create.bind）。"""
    boundary = "----douyin-upload-7f3a"
    with open(video_path, "rb") as f:
        file_bytes = f.read()
    upload_body = (
        f"--{boundary}\r\n".encode()
        + f'Content-Disposition: form-data; name="video"; filename="{os.path.basename(video_path)}"\r\n'.encode()
        + b"Content-Type: application/octet-stream\r\n\r\n"
        + file_bytes + f"\r\n--{boundary}--\r\n".encode()
    )
    up = _post(f"{BASE}/api/douyin/v1/video/upload_video/?open_id={open_id}",
               {"Content-Type": f"multipart/form-data; boundary={boundary}",
                "access-token": access_token}, upload_body)
    video_id = up["data"]["video"]["video_id"]
    body = json.dumps({"video_id": video_id, "text": text}).encode()
    return json.dumps(_post(f"{BASE}/api/douyin/v1/video/create_video/?open_id={open_id}",
                            {"Content-Type": "application/json",
                             "access-token": access_token}, body),
                      ensure_ascii=False)

if __name__ == "__main__":
    mcp.run()
```

Hermes 侧配置：

```yaml
# config.yaml
mcp_servers:
  douyin:
    command: "python"
    args: ["/absolute/path/to/mcp_douyin_server.py"]
    env:
      DOUYIN_CLIENT_KEY: "aw5uktj8ot******"
      DOUYIN_CLIENT_SECRET: "7802f4e6f243e659d5******"
```

stdio server 只继承白名单环境变量，所以密钥要显式写进 env 这一项。启动后工具以 mcp_douyin_client_token 和 mcp_douyin_publish_video 的形式出现，模型可直接调用。

## 收尾

一句话选型：只想每天定时发一条，用路径 A，no-agent cron 零成本；想让 Hermes 按内容动态决定发不发、怎么发，用路径 B；有上游系统、发布时机由事件决定，用路径 C。三者叠加也常见，A 打底定时兜底，C 增量事件即时发，B 增强对话里临时干预。

真正卡人的从来不是 Hermes 这一侧，而是抖音那一侧的资质和审核。企业认证过不去，代码再漂亮也发不出去。先把 scope 申请下来、把 token 的生命周期管明白，剩下就是一次普通的 HTTP 调用加上一条 cron 命令的事。
