# 域名接入与 Worker 切换流程

网站已经能通过 Cloudflare 访问，用户之后获得域名时读取本文。

## 阶段一：保存基线与盘点 DNS

修改 DNS 前，记录可用的 `workers.dev` 地址并完成一次访问测试。查询并保存当前 nameservers 和关键 DNS 记录，尤其是 MX、SPF、DKIM、DMARC、验证 TXT 和第三方子域名。对于已经在使用的域名，必须准备可回退的原始记录。

先确认希望访问者最终使用哪个地址：

- 根域名作为主站：`https://example.com`
- `www` 作为主站：`https://www.example.com`
- 应用子域名：`https://app.example.com`

Worker Custom Domain 按精确主机名匹配。配置其中一个不会自动配置其他入口。

## 阶段二：把权威 DNS 迁移到 Cloudflare

1. 把根域名加入正确的 Cloudflare 账号。Free 或 Pro 计划通常使用 full zone setup。
2. 检查 Cloudflare 扫描到的 DNS 记录，在更换 nameservers 前补回遗漏项。
3. 如果旧委派启用了 DNSSEC，按注册商说明撤销旧 DS 记录。带着失效 DNSSEC 更换 nameservers 可能导致整个域名无法访问。
4. 在域名注册商后台，把原有权威 nameservers 替换为 Cloudflare 分配的两个地址，必须完整准确复制。
5. 等待 zone 变为 Active，并独立检查 NS 结果。公共 DNS 缓存可能存在延迟。
6. zone 激活后，如需 DNSSEC，在 Cloudflare 开启并按流程通过注册商发布新的 DS 信息。

nameserver 传播需要时间。合理间隔查询，不要反复提交同一个修改；需要用户处理时明确指出具体界面和动作。

## 阶段三：绑定 Worker

绑定前只移除会阻止 Custom Domain 创建的冲突 Web CNAME，不删除无关记录。

Cloudflare Dashboard 路径：

`Workers & Pages` → 选择 Worker → `Settings` → `Domains & Routes` → `Add` → `Custom Domain`

Wrangler 源配置形态：

```jsonc
{
  "routes": [
    {
      "pattern": "example.com",
      "custom_domain": true
    }
  ]
}
```

随后运行仓库正常的构建与部署流程。如果框架会生成最终部署配置，应修改源 Wrangler 文件并检查生成结果是否包含 route，不直接编辑生成文件。

应用通常还需要更新生产网址。先搜索再判断，不要凭感觉改：

```sh
rg -n "workers\\.dev|BASE_URL|PUBLIC_SITE_URL|SITE_URL|ORIGIN|TRUSTED_ORIGINS|CORS|CALLBACK|COOKIE" .
```

搜索输出和日志中不得出现密钥。修改第三方 OAuth callback 或 webhook 是独立的外部变更，必须拥有用户授权。

## 阶段四：主网址与验收

如果备用主机名需要跳转，按 Cloudflare 当前文档创建 Redirect Rule 和代理的 originless DNS 记录。如果两个主机名都要直接运行 Worker，则声明两个精确 Custom Domains，并确保页面只输出一个 canonical URL。

验收清单：

- Cloudflare 显示 zone 为 Active。
- 权威 NS 与 Cloudflare 分配的 nameservers 一致。
- 正式域名可以解析，TLS 握手没有证书警告。
- Worker 在自定义域名下可以正确返回页面、资源、API 和深层路由。
- 备用域名按预期访问或跳转，没有循环。
- 登录、session cookie 和 OAuth callbacks 使用新域名。
- 适用时，sitemap、feeds、Open Graph URL、canonical 标签和 robots 引用正式域名。
- 原有邮件和验证 DNS 记录仍然可以解析。
- 除非有意关闭，原 `workers.dev` 地址仍可用于诊断。

## 回退

如果 zone 已激活但 Worker Custom Domain 失败，先撤销应用网址改动和 custom-domain route，同时保留已知可用的 `workers.dev` 入口。如果 nameserver 迁移本身失败，按照基线记录恢复注册商原 nameservers 和 DNSSEC 状态，并说明 DNS 缓存可能延迟恢复。

## 权威资料

- Full zone setup：<https://developers.cloudflare.com/dns/zone-setups/full-setup/setup/>
- Worker Custom Domains：<https://developers.cloudflare.com/workers/configuration/routing/custom-domains/>
- `www` 跳转到根域名：<https://developers.cloudflare.com/rules/url-forwarding/examples/redirect-www-to-root/>
- 根域名跳转到 `www`：<https://developers.cloudflare.com/rules/url-forwarding/examples/redirect-root-to-www/>
- DNSSEC：<https://developers.cloudflare.com/dns/dnssec/>
- Universal SSL：<https://developers.cloudflare.com/ssl/edge-certificates/universal-ssl/>
