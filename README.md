# 自用 List

日常配置文件存放处，不是项目。所有文件按用途归档到 5 个目录。

## 01 · 联通优选 IP　`01-ip-list/`
- `IP_best.txt` — 沈阳联通实测 Cloudflare 优选 IP 主清单，自动更新
- `newip.txt` — 双源汇总的节点池，自动更新

## 02 · AdGuard 黑白名单　`02-adguard/`
- `adguardhome_white_adlist.txt` — 放行白名单，解决机票查询白屏等误拦截
- `adguardhome_ai_v4list` — AI 域名 AAAA 拦截规则，关闭 IPv6 避免查询超时

## 03 · 影视 TV 配置　`03-tv-config/`
- `tv.json` — 影视资源站配置源，TVBox 类客户端使用
- `OKvideoMac.txt` — 影视资源站精简配置，Mac 端使用

## 04 · 代理分流规则　`04-proxy-rules/`
- `shadowrocket-full.conf` — Shadowrocket 完整配置，国内直连 / 国外走家庭服务器
- `passwall-gw.txt` — PassWall 2 / Sing-box 核心分流与黑名单规则集
- `passwall2-ai.txt` — OpenAI / ChatGPT / Claude 等 AI 域名分流
- `passwall2-gn.txt` — 国内直连基础包 + DoT / DoH 防环规则
- `passwall2-google.txt` — Google 与 YouTube 全家桶分流

## 05 · 电视优化 Skill　`05-tv-skills/`
- `创维电视去AD-1 SKILL.md` — cleanyourtv：Android TV 去臃肿 + Apple TV 风格桌面
- `创维电视去AD-2 SKILL.md` — coocaa-tv：创维 / 酷开 CCOS 国行电视深度调教手册
- `小米ASTP0优化报告.md` — 小米 MiTV-ASTP0 (L55M7-EA) 去臃肿 + Aerial 桌面接管实操记录

---

新增文件按上面 5 类放进对应目录；订阅类文件请固定路径，移动会改变客户端里已配置的地址。
