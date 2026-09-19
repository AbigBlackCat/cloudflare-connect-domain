<div align="center">

# 域名接入 Cloudflare Skill

> 网站已经上线后，把你买好的域名安全接入 Cloudflare DNS，再绑定到 Cloudflare Worker。

[![Agent Skills](https://img.shields.io/badge/Agent%20Skills-Compatible-16a34a)](https://agentskills.io)
[![skills CLI](https://img.shields.io/badge/skills%20CLI-Compatible-2563eb)](https://github.com/vercel-labs/skills)
[![Cloudflare DNS](https://img.shields.io/badge/Cloudflare-DNS%20·%20Custom%20Domain-f38020?logo=cloudflare&logoColor=white)](https://developers.cloudflare.com/dns/)
[![Multi Agent](https://img.shields.io/badge/Agent-Codex%20·%20Claude%20Code%20·%20Cursor%20·%20Gemini-7c3aed)](#按不同-agent-安装)

**先保证 `workers.dev` 能访问，再迁移 DNS 和绑定正式域名。**

[它能做什么](#它能做什么) · [安装](#安装) · [怎么使用](#怎么使用) · [接入流程](#skill-会怎样工作) · [常见问题](#常见问题)

</div>

---

## 它能做什么

这个 Skill 用于“网站已经部署、后来才有自己的域名”的阶段。安装后，Agent 会协助你：

- 确认域名注册商、当前 nameservers 和目标 Cloudflare 账号。
- 盘点并保留原有 DNS 记录，尤其是邮箱和第三方验证记录。
- 把根域名加入 Cloudflare，指导你在注册商修改 nameservers。
- 正确处理旧 DNSSEC，避免换 nameservers 后域名无法解析。
- 等待并验证 Cloudflare zone 真正变为 Active。
- 把 Worker 绑定到根域名、`www` 或指定子域名。
- 同步修改应用里的站点网址、trusted origins、cookie、CORS、OAuth callback 等配置。
- 设置根域名与 `www` 的访问或跳转关系。
- 验证 DNS、HTTPS 证书、页面、登录、API、sitemap 和邮件记录。
- 保留 `workers.dev` 作为故障排查备用入口。

它通常做的是 **DNS 托管迁移**，不是把域名从原注册商转移到 Cloudflare Registrar。

## 适合哪些情况

| 你的情况 | 是否适合 |
| --- | --- |
| 网站已经能通过 `workers.dev` 访问 | 必要前提 |
| 已经购买并拥有一个域名 | 适合 |
| 域名在 Namecheap、GoDaddy、阿里云等注册商 | 适合，需要在注册商修改 nameservers |
| 域名直接买自 Cloudflare Registrar | 适合，但会跳过 nameserver 迁移 |
| 想使用根域名、`www` 或应用子域名 | 适合 |
| 还没有部署网站 | 请先使用 [cloudflare-deploy-website](https://github.com/AbigBlackCat/cloudflare-deploy-website) |
| 还没有购买域名 | 本 Skill 不负责购买 |

## 小白先理解三个概念

| 概念 | 通俗解释 |
| --- | --- |
| 域名注册商 | 你购买和续费域名的地方，例如 Namecheap、GoDaddy、阿里云 |
| DNS 托管商 | 保存域名解析记录、告诉浏览器去哪里找网站的服务商 |
| 网站托管平台 | 真正运行网站代码的地方；本流程中是 Cloudflare Workers |

把 nameservers 改为 Cloudflare，通常只是把 **DNS 管理权** 交给 Cloudflare。域名仍然在原注册商续费，除非你另外申请注册商转移。

## 使用前准备

1. 一个已经能访问的 Cloudflare Worker 网站，最好有可用的 `workers.dev` 地址。
2. 一个已注册并归你控制的域名。
3. 能登录域名注册商后台，用于修改 nameservers。
4. 能登录网站所在的 Cloudflare 账号。
5. Node.js 和一个支持 Agent Skills 的 AI 编程工具。

如果域名已经用于邮箱、企业验证或其他服务，不要直接切换。Skill 会先盘点 MX、TXT、SPF、DKIM、DMARC 等记录。

---

## 安装

### 方式一：自动识别 Agent（最适合小白）

打开终端，复制下面一行：

```bash
npx skills add AbigBlackCat/cloudflare-connect-domain -g
```

- `npx` 第一次运行时可能询问是否安装 `skills`，输入 `y` 并回车。
- `-g` 表示全局安装，以后其他网站也能使用。
- 安装器会识别兼容 Agent，并让你选择安装目标。

只想安装到当前网站项目时，去掉 `-g`：

```bash
npx skills add AbigBlackCat/cloudflare-connect-domain
```

### 按不同 Agent 安装

| Agent | 一键安装命令 | 全局安装位置 |
| --- | --- | --- |
| Codex | `npx skills add AbigBlackCat/cloudflare-connect-domain -g -a codex -y` | `~/.codex/skills/` |
| Claude Code | `npx skills add AbigBlackCat/cloudflare-connect-domain -g -a claude-code -y` | `~/.claude/skills/` |
| Cursor | `npx skills add AbigBlackCat/cloudflare-connect-domain -g -a cursor -y` | `~/.cursor/skills/` |
| Gemini CLI | `npx skills add AbigBlackCat/cloudflare-connect-domain -g -a gemini-cli -y` | `~/.gemini/skills/` |
| OpenCode | `npx skills add AbigBlackCat/cloudflare-connect-domain -g -a opencode -y` | `~/.config/opencode/skills/` |
| OpenClaw | `npx skills add AbigBlackCat/cloudflare-connect-domain -g -a openclaw -y` | `~/.openclaw/skills/` |
| GitHub Copilot | `npx skills add AbigBlackCat/cloudflare-connect-domain -g -a github-copilot -y` | `~/.copilot/skills/` |
| Hermes Agent | `npx skills add AbigBlackCat/cloudflare-connect-domain -g -a hermes-agent -y` | `~/.hermes/skills/` |

同时安装到多个 Agent：

```bash
npx skills add AbigBlackCat/cloudflare-connect-domain -g -a codex claude-code cursor -y
```

### 手动安装

Codex：

```bash
git clone https://github.com/AbigBlackCat/cloudflare-connect-domain.git ~/.codex/skills/cloudflare-connect-domain
```

Claude Code 把安装路径换成 `~/.claude/skills/cloudflare-connect-domain`；Cursor 换成 `~/.cursor/skills/cloudflare-connect-domain`。其他 Agent 请使用它自己的 skills 目录。

### 不安装，临时使用一次

```bash
npx skills use AbigBlackCat/cloudflare-connect-domain@cloudflare-connect-domain
```

不支持 Agent Skills 的工具，可以打开 [`SKILL.md`](SKILL.md)，把内容连同域名接入请求一起粘贴给 AI。

### 检查安装结果

```bash
npx skills list -g
```

列表里应出现 `cloudflare-connect-domain`。如果没有，再运行 `npx skills list` 检查是否安装成了当前项目级 Skill。

---

## 怎么使用

先进入已经部署的网站项目目录，再对 Agent 说：

```text
请使用 $cloudflare-connect-domain，把我已有的域名接入 Cloudflare，
并绑定到当前已经部署的 Worker。
先保存现有 DNS 记录和检查 DNSSEC，涉及注册商登录时再叫我操作。
```

建议同时告诉 Agent 三项信息：

```text
域名：example.com
当前可用网站：https://my-site.example-account.workers.dev
希望正式使用：https://example.com，并把 www.example.com 跳转到根域名
```

更多说法：

```text
我的网站已经在 Cloudflare Workers 上了。现在买了域名，请帮我接入，邮箱 DNS 记录不要弄丢。
```

```text
把 app.example.com 绑定到当前 Worker，根域名保持现状，不要影响其他子域名。
```

```text
检查这个域名是否已经正确托管到 Cloudflare，并修复自定义域名和 HTTPS 问题。
```

## Skill 会怎样工作

```text
验证 workers.dev 网站仍然可用
        ↓
识别注册商、当前 DNS、nameservers 和 DNSSEC
        ↓
保存 A / CNAME / MX / TXT 等记录
        ↓
把根域名加入正确的 Cloudflare 账号
        ↓
你在注册商修改 nameservers
        ↓
等待并验证 zone = Active
        ↓
绑定 Worker Custom Domain
        ↓
更新站点网址、OAuth/CORS/cookie 等配置
        ↓
处理根域名与 www 的跳转关系
        ↓
验证 DNS、HTTPS、页面、登录、API 和邮箱记录
```

### 为什么不能一上来就改 nameservers

Cloudflare 的自动扫描不一定能找到全部 DNS 记录。如果域名已经有邮箱或验证服务，遗漏 MX、SPF、DKIM、DMARC、TXT 等记录可能导致收不到邮件或第三方服务失效。正确顺序是先保存和复核，再切换。

### 哪些步骤通常需要你本人完成

- 登录域名注册商。
- 关闭旧 DNSSEC 或删除旧 DS 记录。
- 把 nameservers 改成 Cloudflare 分配的两个地址。
- 完成账号二次验证或邮件确认。
- 登录 GitHub、Google 等第三方控制台修改 OAuth callback。

Agent 应在需要时只给你一个清晰动作，例如“请在注册商的 Nameservers 页面，把原来的两个地址替换为下面两个”，完成后再继续。

## 安全边界

- Worker 还不能稳定访问时，不切换正式域名。
- DNS 清单不完整时，不贸然更换 nameservers。
- 不删除邮件、验证或其他无关 DNS 记录。
- 不把 DNS 托管误说成域名注册商转移。
- zone 仍为 Pending 时，不宣称接入完成。
- 不手工创建与 Worker Custom Domain 冲突的 CNAME。
- 自定义域名验证完成前，保留 `workers.dev` 备用入口。

## 最终你会拿到什么

- Cloudflare zone、账号和域名注册商信息。
- Cloudflare 分配并已验证的 nameservers。
- 正式主域名和备用域名的处理方式。
- 绑定的 Worker Custom Domain。
- 更新过的站点 URL、trusted origins、OAuth/CORS/cookie 等配置清单。
- DNSSEC 状态和 HTTPS 证书验证结果。
- 页面、资源、登录、API、sitemap/feed 和邮件 DNS 检查结果。
- DNS 传播等待说明和剩余人工事项。

---

## 常见问题

### 我需要把域名转移到 Cloudflare Registrar 吗？

不需要。大多数情况下只需要在原注册商修改 nameservers，让 Cloudflare 托管 DNS。域名续费仍在原注册商完成。

### 修改 nameservers 会立刻生效吗？

不一定。注册商和公共 DNS 缓存需要传播时间。Skill 会同时检查 Cloudflare zone 状态和权威 NS，不会只看浏览器一次结果就下结论。

### Cloudflare 自动导入了 DNS，还需要检查吗？

需要。自动扫描可能遗漏记录，尤其是邮件、验证和不常见的子域名记录。

### 根域名和 www 会自动同时生效吗？

不会。Worker Custom Domain 精确匹配主机名。需要明确决定两者都运行网站，还是其中一个跳转到另一个。

### HTTPS 证书需要自己购买吗？

正常的 Worker Custom Domain 会由 Cloudflare 创建所需 DNS 记录和证书。仍需等待证书签发完成并实际访问验证。

### 如何更新 Skill？

```bash
npx skills update cloudflare-connect-domain
```

更新全部全局 Skills：

```bash
npx skills update -g
```

### 如何卸载？

```bash
npx skills remove -g cloudflare-connect-domain
```

## 仓库结构

```text
cloudflare-connect-domain/
├── README.md
├── SKILL.md
├── agents/
│   └── openai.yaml
└── references/
    └── domain-workflow.md
```

- `README.md`：给人阅读的中文安装和使用说明。
- `SKILL.md`：Agent 执行域名接入时遵守的核心规则。
- `agents/openai.yaml`：Codex 等客户端使用的界面信息。
- `references/domain-workflow.md`：DNS 迁移、Custom Domain、验收和回退流程。

## 依据与致谢

- [Cloudflare Full Zone Setup](https://developers.cloudflare.com/dns/zone-setups/full-setup/setup/)
- [Cloudflare Worker Custom Domains](https://developers.cloudflare.com/workers/configuration/routing/custom-domains/)
- [Cloudflare DNSSEC](https://developers.cloudflare.com/dns/dnssec/)
- [Agent Skills 标准](https://agentskills.io)
- [Vercel Labs skills CLI](https://github.com/vercel-labs/skills)
