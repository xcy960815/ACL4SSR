# iStoreOS、OpenClash 与 ACL4SSR DNS/分流运维记录

> 最后核对：2026-08-07
>
> 适用设备：iStoreOS `192.168.100.1`
>
> 本文不记录 SSH 密码、订阅令牌、代理认证、API Secret 或数据库密码。

## 1. 目的与范围

这台 iStoreOS 同时承担局域网网关、DNS、OpenClash、Docker 服务和 OpenList 等职责。DNS、Fake-IP、透明代理、路由器自身流量和机场订阅之间存在较强耦合，任意一层异常都可能表现为：

- Node/ioredis 报 `getaddrinfo ENOTFOUND`；
- 开启 OpenClash 后 `codexapis.com` 无法连接；
- OpenList 页面正常，但夸克视频卡住；
- 手机正常，电脑或 Docker 容器异常；
- 规则已经命中，但代理节点仍超时；
- 更新订阅或切换机场后临时修复丢失。

本文记录当前稳定架构、故障根因、修复方式、维护边界、验证方法、安全风险和回滚入口。

## 2. 当前架构

| 组件 | 职责 |
| --- | --- |
| iStoreOS | 局域网网关、DNS 和透明代理入口 |
| dnsmasq | 接收 LAN/Docker DNS 请求，保留运营商上游能力 |
| OpenClash/Mihomo | Fake-IP、DNS 劫持、规则匹配和节点选择 |
| ACL4SSR Fork | 维护可跨机场复用的个人规则和转换模板 |
| subconverter | 将机场订阅与个人模板组合成 Mihomo 配置 |
| OpenList | 路由器本地服务，HTTP 端口 `5244` |
| Redis | Docker 服务，监听端口 `6379`，公网暴露需单独审计 |
| Tailscale | 私网访问通道，地址范围属于 `100.64.0.0/10` |

流量与 DNS 链路：

```text
LAN / Docker
    |
    v
iStoreOS dnsmasq :53
    |
    +--> OpenClash DNS / Fake-IP
    |       +--> 普通域名：Mihomo DNS
    |       +--> 指定域名：223.5.5.5
    |       +--> Fake-IP 排除：返回真实地址
    |
    v
OpenClash 规则
    +--> Direct (Domain) --> DIRECT
    +--> Proxy (Domain)  --> Ai平台
    +--> ACL4SSR 公共规则
    +--> FINAL
```

订阅生成链路：

```text
机场原始订阅
  + ACL4SSR_Online_Xuchongyu.ini
  + Clash/Custom/*.list
              |
              v
        subconverter
              |
              v
 yuyunsvip.yaml / web3moe.yaml
              |
              v
       OpenClash/Mihomo
```

## 3. 当前稳定配置

### 3.1 OpenClash 全局设置

关键 UCI 设置：

```text
enable=1
enable_custom_dns=1
enable_custom_clash_rules=1
custom_name_policy=1
custom_fakeip_filter=1
router_self_proxy=0
cachesize_dns=1000
```

含义：

- 开启自定义 DNS、DNS Policy 和 Fake-IP Filter；
- 个人业务流量规则主要由 ACL4SSR 远程 Rule Provider 维护；
- `router_self_proxy=0`，路由器自身和 OpenList 发起的连接不自动走代理；
- DNS 缓存限制为 `1000`。

不要仅为了消除以下日志而开启路由器自身代理：

```text
Streaming Unlock Could not Work Because of Router-Self Proxy Disabled
```

该日志来自流媒体解锁辅助功能，不代表 OpenClash 核心失败。开启路由器自身代理可能再次代理 OpenList、订阅下载和 DNS 请求，形成循环依赖。

### 3.2 DNS 上游

当前采用混合策略：

- dnsmasq：`noresolv=0`，保留从 WAN 获取的运营商 DNS；
- OpenClash nameserver/fallback：UDP `223.5.5.5`；
- Google/Cloudflare DoH fallback 已禁用；
- 指定业务域名通过 DNS Policy 使用 `223.5.5.5`；
- dnsmasq 和 OpenClash DNS 缓存均限制为 `1000`。

相关 DoH 被禁用，是为了避免“代理节点不可用 -> DoH 不可用 -> 所有域名无法解析”的启动循环。UDP 国内 DNS 在当前环境中更稳定。

### 3.3 DNS Policy

文件：

```text
/etc/openclash/custom/openclash_custom_domain_dns_policy.list
```

业务条目：

```yaml
+.codexapis.com: '223.5.5.5'
+.quark.cn: '223.5.5.5'
+.drive.quark.cn: '223.5.5.5'
+.w.alikunlun.com: '223.5.5.5'
+.aliyuncs.com: '223.5.5.5'
xuchongyu.dns.army: '223.5.5.5'
```

dnsmasq 还保留：

```text
noresolv=0
server=/xuchongyu.dns.army/223.5.5.5
cachesize=1000
```

### 3.4 Fake-IP 排除

文件：

```text
/etc/openclash/custom/openclash_custom_fake_filter.list
```

新增业务条目：

```text
+.quark.cn
+.drive.quark.cn
+.w.alikunlun.com
+.aliyuncs.com
xuchongyu.dns.army
fuck.codexapis.com
```

这些域名应返回真实地址，而不是 `198.18.0.0/16` Fake-IP：

- `xuchongyu.dns.army` 用于数据库/服务连接，不能依赖 Fake-IP 映射；
- OpenList/夸克涉及路由器自身连接和 CDN 跳转；
- `fuck.codexapis.com` 明确要求直连。

### 3.5 ACL4SSR 个人规则

个人规则：

- [`Clash/Custom/Direct.list`](../Clash/Custom/Direct.list)
- [`Clash/Custom/Proxy.list`](../Clash/Custom/Proxy.list)

转换模板：

- [`Clash/config/ACL4SSR_Online_Xuchongyu.ini`](../Clash/config/ACL4SSR_Online_Xuchongyu.ini)

模板的前两条规则集：

```text
Direct (Domain) -> 全球直连
Proxy (Domain)  -> Ai平台
```

之后才是 ACL4SSR 公共规则和 FINAL。关键优先级：

```text
DOMAIN,fuck.codexapis.com   -> DIRECT
DOMAIN-SUFFIX,codexapis.com -> Ai平台
```

因此 `fuck.codexapis.com` 直连，其他 `*.codexapis.com` 继续走 `Ai平台`。

个人模板中的公共规则直接引用官方仓库：

```text
https://raw.githubusercontent.com/ACL4SSR/ACL4SSR/master/...
```

个人规则引用 Fork：

```text
https://raw.githubusercontent.com/xcy960815/ACL4SSR/master/Clash/Custom/...
```

这样可以持续享受官方规则更新，同时将个人改动限制在独立文件中。

### 3.6 OpenClash 订阅

当前订阅：

- `yuyunsvip`
- `web3moe`

两者均使用：

```text
template=0
custom_template_url=https://raw.githubusercontent.com/xcy960815/ACL4SSR/master/Clash/config/ACL4SSR_Online_Xuchongyu.ini
skip_cert_verify=true
```

`skip_cert_verify=true` 通过订阅转换写入代理 Provider override，避免机场节点证书与伪装 SNI 不匹配时被 Mihomo 拒绝。它只影响客户端到代理节点的握手，不会关闭浏览器对目标网站的 HTTPS 校验。

## 4. 故障、根因与修复

### 4.1 ioredis `ENOTFOUND`

现象：

```text
[ioredis] Unhandled error event:
getaddrinfo ENOTFOUND xuchongyu.dns.army
```

根因不是 ioredis，而是 OpenClash/dnsmasq DNS 链路异常：代理或 DoH 上游不可用、dnsmasq 重启期间监听短暂消失，最终返回 ENOTFOUND。

修复：

- 恢复 dnsmasq `noresolv=0`；
- 为域名指定 `223.5.5.5`；
- 加入 Fake-IP 排除；
- 在个人规则中保持 DIRECT；
- 限制 DNS 缓存为 `1000`。

修复后该域名返回真实公网地址，不再返回 `198.18.x.x`。

### 4.2 `codexapis.com` 规则命中但连接失败

最初规则已正确命中 `Ai平台`，问题实际发生在节点层：

1. 部分 `a1.ioubha.cn` 节点端口拒绝连接或超时；
2. `a6.mhlnf.cn` 节点可连接，但订阅 SNI 为 `www.bing.com`，节点实际返回 `*.cloudfisher.xyz` 证书；
3. 当时 `skip-cert-verify=false`，Mihomo 主动拒绝连接。

典型错误：

```text
tls: failed to verify certificate:
certificate is valid for *.cloudfisher.xyz, not www.bing.com
```

修复：

- 两份订阅都设置 `skip_cert_verify=true`；
- 选择能建立连接的香港中转节点；
- 将修复写入订阅设置，而不是只修改一次性 YAML；
- `fuck.codexapis.com` 放入个人直连规则；
- 其他 `codexapis.com` 子域保留代理规则。

### 4.3 OpenList/夸克视频卡住，但手机正常

OpenList 服务本身健康，`5244` 返回 `200`。视频卡住发生在 OpenList 获取夸克 CDN 数据时：

- OpenClash/dnsmasq 重启期间，OpenList 查询 `[::1]:53`；
- 本地 DNS 监听短暂不可用；
- 手机可能使用缓存、私有 DNS/DoH 或不同网络路径，因此表现正常。

修复：

- 保持 `router_self_proxy=0`；
- 将夸克及阿里 CDN 加入 DNS Policy 和 Fake-IP Filter；
- 在 `Direct.list` 中保持直连；
- 确保 `127.0.0.1:53` 和 `[::1]:53` 均有监听；
- 避免频繁重启 dnsmasq/OpenClash。

### 4.4 Rule Provider 下载失败

代理节点 TLS 失败时，通过代理下载 `api.asailor.org` 规则也会失败：

```text
节点不可用 -> 规则下载失败 -> 分流不完整 -> 更多请求失败
```

订阅级 `skip_cert_verify=true` 修复节点握手后，Rule Provider 恢复加载。

## 5. 配置职责边界

| 配置 | 负责什么 | 不负责什么 |
| --- | --- | --- |
| ACL4SSR `*.list` | DIRECT/代理/拒绝策略 | DNS 上游、Fake-IP、节点证书 |
| ACL4SSR `*.ini` | 规则顺序、策略组和转换结构 | dnsmasq 配置 |
| DNS Policy | 指定域名使用哪个 DNS | 最终走 DIRECT 还是代理 |
| Fake-IP Filter | 返回真实 IP 或 Fake-IP | 代理策略 |
| `skip_cert_verify` | 代理节点 TLS 握手 | 目标网站 HTTPS 校验 |
| 节点选择 | 代理出口可用性 | DNS 和分流规则 |

排障时必须先确定失败层级。不要用 DNS 修改掩盖节点端口超时，也不要用切换节点修复本地 DNS 监听丢失。

## 6. Fork 与上游同步

远程关系：

```text
origin   https://github.com/xcy960815/ACL4SSR.git
upstream https://github.com/ACL4SSR/ACL4SSR.git
```

同步流程：

```sh
git switch master
git fetch upstream
git merge upstream/master
git push origin master
```

原则：

- 不直接修改官方大型规则文件；
- 个人规则放在 `Clash/Custom/`；
- 个人模板使用独立文件名；
- 合并前先保证工作区干净；
- 发生冲突时优先保留上游公共规则，再重新应用个人模板入口；
- 不对包含个人提交的共享 `master` 使用无保护的强制推送。

## 7. 日常维护

### 7.1 只修改个人域名

1. 编辑 `Clash/Custom/Direct.list` 或 `Proxy.list`；
2. 确认精确规则优先于泛域名规则；
3. 提交并推送 `origin/master`；
4. Rule Provider 最长约 24 小时自动刷新；
5. 需要立即生效时，在 OpenClash 中手动更新 Rule Provider。

### 7.2 修改模板结构

修改 `ACL4SSR_Online_Xuchongyu.ini` 后：

1. 推送 GitHub；
2. 更新 `yuyunsvip`、`web3moe` 配置订阅；
3. 检查 `rules:` 第一项仍为 `Direct (Domain)`；
4. 检查 `Proxy (Domain)` 在公共 AI/GFW 规则之前；
5. 验证后再切换活动配置。

### 7.3 切换机场

切换后确认：

- 自定义模板 URL 仍存在；
- `skip_cert_verify=true`；
- `Direct (Domain)`、`Proxy (Domain)` 已生成；
- DNS Policy 和 Fake-IP Filter 保留；
- `节点选择` 与 `Ai平台` 指向可用节点。

节点名称、延迟和当前选择属于机场配置，不由 ACL4SSR 规则固定。

## 8. 验证清单

### 8.1 服务与监听

```sh
/etc/init.d/openclash status
pgrep -af 'clash|mihomo'
netstat -lntp | grep -E ':(53|5244|6379|7874|7893|7895|9090) '
```

### 8.2 DNS

```sh
nslookup xuchongyu.dns.army 127.0.0.1
nslookup fuck.codexapis.com 127.0.0.1
nslookup drive.quark.cn 127.0.0.1
```

期望返回真实地址，而不是 `198.18.x.x`。

### 8.3 本地服务

```sh
curl -sS -o /dev/null -w '%{http_code}\n' http://127.0.0.1:5244
printf 'PING\r\n' | nc 127.0.0.1 6379
```

OpenList 应返回 `200`；Redis 返回 `PONG` 或 `NOAUTH` 都能证明服务可达。

### 8.4 规则命中

```sh
tail -n 200 /tmp/openclash.log | grep -E 'fuck.codexapis.com|drive.quark.cn|xuchongyu.dns.army'
```

直连域名期望包含：

```text
match RuleSet(Direct (Domain)) using 全球直连[DIRECT]
```

其他 `*.codexapis.com` 应命中 `Proxy (Domain)` 或 `Ai平台`。

### 8.5 错误日志

```sh
tail -n 300 /tmp/openclash.log | grep -E 'certificate|context deadline exceeded|connection refused|OpenClash Stoping'
```

持续出现证书、超时或拒绝连接时，应先检查机场节点，而不是继续修改 DNS。

## 9. 备份与回滚

已保留备份：

```text
/etc/config/openclash.codex-backup-20260807-stability
/etc/config/dhcp.codex-backup-20260807-stability
/etc/openclash/custom/openclash_custom_domain_dns_policy.codex-backup-20260807-stability
/etc/openclash/custom/openclash_custom_fake_filter.codex-backup-20260807-stability
/etc/openclash/custom/openclash_custom_rules.codex-backup-20260807-stability

/etc/openclash/config/yuyunsvip.yaml.codex-backup-20260807-tls
/etc/openclash/custom/openclash_custom_rules.codex-backup-20260807-direct
/etc/openclash/custom/openclash_custom_fake_filter.codex-backup-20260807-direct

/etc/config/openclash.codex-backup-20260807-acl4ssr-template
/etc/openclash/config/yuyunsvip.yaml.codex-backup-20260807-acl4ssr-template
/etc/openclash/config/web3moe.yaml.codex-backup-20260807-acl4ssr-template
/etc/openclash/custom/openclash_custom_rules.codex-backup-20260807-before-acl4ssr-migration
```

回滚原则：

1. 确认备份时间和目标文件；
2. 恢复前再备份当前配置；
3. 优先恢复 `/etc/config/openclash` 和相关 custom 文件；
4. 必要时恢复活动 YAML；
5. 执行 `uci commit openclash`；
6. 重启 OpenClash；
7. 按第 8 节重新验证。

不要在未确认路径时使用通配符、递归删除或批量覆盖。

## 10. 安全与遗留事项

### 10.1 数据库公网暴露

`xuchongyu.dns.army` 使用公网地址，不代表 Redis/MySQL 应直接暴露到互联网。必须检查：

- WAN 防火墙是否允许任意来源访问 `6379`/`3306`；
- Redis 是否启用 ACL/强密码；
- 数据库连接是否使用 TLS；
- 是否可以仅允许固定源 IP；
- 是否可以优先通过 Tailscale/WireGuard 访问。

推荐优先使用 Tailscale 或防火墙白名单。DNS 和 DIRECT 规则不能替代数据库安全控制。

### 10.2 订阅和日志泄露

机场订阅 URL 通常包含令牌。分享以下信息前必须脱敏：

- `uci show openclash`；
- `/tmp/openclash.log` 中的下载 URL；
- OpenClash YAML、截图和诊断输出。

仓库不得提交机场订阅、代理密码、OpenClash API Secret 或数据库密钥。

### 10.3 `skip_cert_verify` 风险

当前机场节点证书与 SNI 不一致，需要 `skip_cert_verify=true`。这会降低代理节点身份校验强度。长期方案应由机场修复 SNI/证书，而不是永久依赖跳过校验。

### 10.4 OpenClash 重启警告

重启时曾出现：

```text
sh: out of range
```

当前未影响核心启动、规则加载和访问，但原因尚未定位。升级 OpenClash 或 shell 组件后应复查；若伴随 cron、订阅或启动失败，需要单独排查脚本兼容性。

## 11. 已验证结果（2026-08-07）

- OpenClash：`running`；
- `fuck.codexapis.com`：命中 `Direct (Domain)`，HTTP `200`；
- `drive.quark.cn`：命中 `Direct (Domain)`，约 `112ms`；
- OpenList：HTTP `200`；
- `xuchongyu.dns.army`：返回真实公网地址；
- Redis 本地端口：可达并要求认证；
- `yuyunsvip`、`web3moe`：个人模板更新成功；
- 两份配置均生成 `Direct (Domain)`、`Proxy (Domain)`；
- 两份配置均包含 `skip-cert-verify: true`；
- GitHub Raw 个人规则可下载；
- 路由器重复临时流量规则已移除；
- DNS Policy 和 Fake-IP Filter 已保留。
