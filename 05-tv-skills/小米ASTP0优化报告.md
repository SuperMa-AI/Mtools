# 小米 MiTV-ASTP0 桌面与优化报告

操作日期：2026-11-11 · 设备：`192.168.26.10:5555`

## 设备

| 项目 | 值 |
|---|---|
| 型号 | MiTV-ASTP0（codename `mulan`，L55M7-EA） |
| 品牌 | Xiaomi |
| 系统 | Android 9 / SDK 28 / 固件 `OS3.0.102.0.PSTAATV` |
| 芯片 | Amlogic / droidlogic，1.5 GB 内存，1080p |
| 原桌面 | `com.mitv.tvhome/.MainActivityUserMode` |
| 包总数 | 65（操作前已禁用 0 个） |

上游 `examples/xiaomi-mitv-astp0/debloat.txt` 正好是这台机型的现成清单，本次以它为基准。

## 已完成 ✅

### 1. 完整备份

`xiaomi-backup/20261111-150015/`：全部包列表、系统/第三方/已禁用包、getprop、settings 三张表、HOME 组件、leanback 应用、前台、屏幕、磁盘。三个应用的 APK 也拉到了 `xiaomi-backup/apps/`，用于确认名称。

### 2. 应用识别（避免误禁）

| 包名 | 应用名 | 处理 |
|---|---|---|
| `com.jumi.tv` | **剧迷TV** | 保留 |
| `com.github.tvbox.osc` | **TVBox** | 保留 |
| `z.jptv.dd.tv` | **荐片TV** | 按你的选择一起保留 |

### 3. 新桌面已安装

```
package:  com.gathin.tvhome  (launcher v2.6)
配置:     /sdcard/Android/data/com.gathin.tvhome/files/config.json
截图:     xiaomi-backup/new-launcher-final.png
```

配置内容：

```json
{
  "dock_apps": ["com.jumi.tv", "com.github.tvbox.osc", "z.jptv.dd.tv"],
  "fill_dock": false,
  "app_library": true,
  "hidden_apps": ["com.mitv.alarmcenter"],
  "stock_home_redirect": "com.mitv.tvhome"
}
```

dock 目前显示 **TVBox + 荐片TV**，「定时提醒」已被隐藏。

### 4. 禁用了 12 个包（分 3 批，每批验证）

```
com.miui.analytics                         MIUI 统计
com.miui.tv.analytics                     TV 统计
com.mitv.cloudcontrol                     云控
com.xiaomi.mitv.tvpush.tvpushservice      推送
com.mitv.shoplugin                        购物插件
com.xm.webcontent                         推广网页
com.xiaomi.gamecenter.sdk.service.mibox   游戏中心 SDK
com.xiaomi.mitv.karaoke.service           K 歌服务
com.xiaomi.mitv.healthdiagnosis           自检
com.xiaomi.mitv.smartpanel                智能面板
com.xiaomi.mitv.tvmanager                 电视管家
com.xiaomi.wakeupservice                  语音唤醒（上游记录常年占 ~30% CPU）
```

每批后都确认了四个保护包仍启用：`com.xiaomi.mitv.settings`、`com.sohu.inputmethod.sogou.tv`、`com.mitv.tvhome`、`com.gathin.tvhome`。最终 `pm list packages -d` = 12，三个影视应用全部在。

清单保存在 `xiaomi-backup/applied-debloat.txt`。

### 5. 明确保留（没动）

设置、**唯一输入法**、原桌面、新桌面、`mitv.service`、`com.droidlogic*`、`com.xiaomi.mitv.tvplayer`（HDMI 切换）、应用商店、文件管理、投屏（AirPlay）、语音助手、系统更新、蓝牙、`com.android.packageinstaller`。

上游清单里禁用了搜狗输入法，但那是先装 Gboard 才做的——这台没装，**没有照抄**。

## 还剩两步需要你在电视上点一下 ⚠️

### A. 让 Home 键进新桌面（关键）

小米固件**忽略** `set-home-activity`（实测返回 `Success` 但解析仍是 `com.mitv.tvhome`），禁用原桌面又会掉进空白 FallbackHome。上游给这台机型的方案是 `HomeKeyService`（无障碍服务）+ `stock_home_redirect`。

**但 ADB 写入的开关被小米系统自动改回去了**：

```
我们写: com.xiaomi.voicecontrol/.VoiceAccessibilityService:com.gathin.tvhome/com.gathin.tvhome.HomeKeyService
系统改回: com.xiaomi.voicecontrol/.VoiceAccessibilityService
dumpsys accessibility 里只有「小爱同学」一个服务
```

标准 `android.settings.ACCESSIBILITY_SETTINGS` 在这台小米上也拉不起来（无对应 Activity）。所以**只能手动开**：

> 电视 **设置 → 无障碍 / 辅助功能 → 服务** → 找到 **「Home button」**（新桌面带的那个，不是小爱同学）→ 打开开关。
> 开完告诉我，我立刻验证按 Home 是否直达新桌面，并补上 `stock_home_redirect` 的验证。

**重启后注意**：小米可能再次移除该开关（上游 AETP0 也有此记录），届时重新打开一次即可。

### B. 把「剧迷TV」加进桌面

剧迷TV 只声明了普通 `LAUNCHER`（`com.jumi.tv/com.shenma.tvlauncher.SplashActivity`），**没有** `LEANBACK_LAUNCHER`，桌面默认枚举不到它。配置里已给它留了 dock 第一位，只差在桌面里登记一次：

> 新桌面 → 底部 **「应用库」** → 勾选 **剧迷TV** → 它会自动出现在 dock 第一位。

（`app_library` 已开启，`dock_apps` 里也已写好。）

## 回滚

```bash
# 1) 还原全部禁用
TV_HOST=192.168.26.10 tools/restore.sh /path/to/applied-debloat.txt

# 2) 恢复原桌面为默认
TV_HOST=192.168.26.10 tools/tv.py 'cmd package set-home-activity --user 0 com.mitv.tvhome/.MainActivityUserMode'
TV_HOST=192.168.26.10 tools/tv.py 'input keyevent KEYCODE_HOME'

# 3) 卸掉新桌面（可选）
TV_HOST=192.168.26.10 tools/tv.py 'pm uninstall com.gathin.tvhome'
```

关掉无障碍开关：设置 → 无障碍 → 服务 → 「Home button」→ 关闭。

## 这台机器的两个固有脾气

1. **网络 ADB 5555 每次重启会自动关闭**，需在「设置 → 账号与安全 → ADB 调试」重新打开；待机还会断 Wi-Fi。所以本次全程没重启。
2. 遥控器没有「输入源」键，HDMI 切换走小米播放器：
   `am start -a com.xiaomi.mitv.tvplayer.EXTSRC_PLAY -n com.xiaomi.mitv.tvplayer/.ExternalSourceActivity --ei input 23`（HDMI1=23、HDMI2=24、HDMI3=25、AV=2）。新桌面右上「输入源」胶囊已接这个入口。

## Home 键接管：已穷尽方案，确认固件限制（2026-11-11 15:26–15:35）

用户反馈「按 Home 又回小米桌面」。逐项排查后确认 **这台 ASTP0 无法让 Home 键可靠进入第三方桌面**：

| 方案 | 实测结果 |
|---|---|
| 标准 `set-home-activity` | 返回 `Success`，但 `resolve-activity` 仍是 `com.mitv.tvhome`（被系统忽略） |
| 小米白名单包名 `com.ts.launcher`（只装这一个第三方桌面） | HOME 变 `ResolverActivity`，按 Home 仍回原桌面。已自动回滚并卸载 |
| 无障碍服务 + `stock_home_redirect`（原版 APK） | 服务配置文件故意 `eventTypes` 为空，重定向代码永远不执行 |
| 定制版 APK（给服务加 `typeWindowStateChanged`）+ 短格式组件名启用服务 | 服务成功绑定（`dumpsys accessibility` 显示 Home button，eventTypes 正确）；日志显示新桌面确实被拉起（`last=com.mitv.tvhome, now=com.gathin.tvhome`），**但几秒后小米系统强制切回原桌面** |
| `app_keys` 把 `KEYCODE_HOME/MENU/ALL_APPS` 映射到新桌面 | 三个键全部无效，小米不把按键交给第三方无障碍服务 |

**纠正一条之前的判断**：ADB 并非完全无法启用第三方无障碍服务——关键是组件名要用**短格式**
`com.gathin.tvhome/.HomeKeyService`（全限定名会被小米系统剔除）。用短格式写入后服务确实绑定且 5 秒未被改回。
但绑定了也没用，系统仍会拉回原桌面。

小米设置包里也**没有无障碍设置 UI 入口**（只有 General / Security 等），所以用户无法手动开开关。

### 最终状态（已清理干净）

```
新桌面          com.gathin.tvhome（已装，配置正常，可从应用列表/桌面快捷方式打开）
HOME 解析        com.mitv.tvhome/.MainActivityUserMode（原桌面）
无障碍服务      只留小爱同学（无效的 Home button 已解绑）
config.json     去掉了无效的 app_keys，恢复干净版
临时 APK        全部删除
已禁包          12 个（未变）
三个影视应用    全部保留（未变）
```

定制 APK 留在工作区备用：`launcher-with-redirect.apk`（带事件订阅）、`launcher-ts-allowlist.apk`（白名单包名）。
上游 launcher 源码已还原，未污染技能包。

**结论**：新桌面可以正常打开使用，但按 Home 会回小米桌面。这是固件层面的 HOME 管控，
不 root 的情况下没有可靠解法。可选：接受现状 / 卸掉新桌面 / 再试延迟重定向的定制版（不保证）。

## 外部/上游成功案例调研（2026-11-11 15:35–15:45）

### 上游仓库里确实有三台小米成功换桌面的实测记录

| 机型 | 系统/固件 | 成功方式 | 关键条件 |
|---|---|---|---|
| **MiTV-ANSP0**（jobs，Amlogic） | Android 9 / `23.6.5.3226` | `cmd package set-home-activity --user 0 com.gathin.tvhome/.MainActivity` 直接生效，重启后仍保持 | 原桌面 `com.mitv.tvhome` **必须保持启用**；两个 HOME filter 的 `priority` 都是 0，所以由偏好决定。禁用原桌面 → 黑屏 FallbackHome |
| **MiTV-AETP0**（dofus） | Android 9 / `22.9.2.851` | 改用白名单包名 `com.ts.launcher` 编译，再 `set-home-activity` | 小米 framework 的 `MitvSettings.isThirdpartyHomeAllowed` 有包名白名单；云控 XML 里列了 `com.ts.launcher`（`third_party_home` + `boot_autostart`） |
| **MiTV-MTEQ0**（baymax，EA Pro 55） | Android 10 / `23.6.26.4028` | 同 AETP0 的白名单包名 | 明确记录：**`com.mitv.cloudcontrol` 已禁用，它下载的白名单留在磁盘上**；`resolve-activity` 返回 `com.ts.launcher/...`，禁用原桌面、重启、force-stop 后都成立 |

### 本机（ASTP0 / mulan / `OS3.0.102.0.PSTAATV`，Android 9）与成功案例的差异

- 固件比 ANSP0 老，比 AETP0 略新；**白名单内容可能不同**。
- 云控配置在 `/data/system/mitv_security_center.xml`、`mitv_control_center.xml`（后者 11-10 刚更新过），
  但都是 `system:system 0600`，**shell 读不到**，无法确认本机放行哪个包名（AETP0 文档也这么说）。
- 按 AETP0/MTEQ0 的原始姿势重装了 `com.ts.launcher`（与标准包名并存）、设默认、再主动触发 HOME intent：
  `resolve-activity` 仍是 `ResolverActivity`，`am start -a MAIN -c HOME` **不弹选择器**，画面停在设置页。
  即本机的 `chooseBestActivity` 没有放行 `com.ts.launcher`。
- 本机 `com.mitv.cloudcontrol` 已在 12 个禁用包里，所以「白名单留在磁盘」这个条件已满足，仍不生效。

### 中文老教程的两个陷阱（不能照抄）

1. 老固件（Android 6 / MIUI TV 1.3，如 MiTV4-ANSM0）靠
   `pm uninstall --user 0 com.mitv.tvhome` 卸载原桌面让第三方桌面顶上。
   **上游三份 Android 9/10 记录一致警告：这代固件卸载原桌面会掉进黑屏 FallbackHome。**
2. CSDN 那篇「更换桌面顺便去开机广告」让人执行 `pm uninstall --user 0 mitv.service`。
   **`mitv.service` 是小米核心服务**，上游 ASTP0 debloat.txt 原文标注 "do not touch it"。绝对不能动。
   该文另外建议卸载 `com.xiaomi.mitv.upgrade`（系统更新）防止桌面被还原，本机按你的「稳妥」选择保留了它。

### 唯一还没验证的变量

上游成功案例都是在**重启后**复验的（MTEQ0 明确写了 two reboots）。本机至今没重启过
（5555 端口每次重启会自动关闭，需回「设置 → 账号与安全 → ADB 调试」重开）。
云控白名单有可能开机才完整加载。若愿意配合一次断电重启 + 重开 ADB，可以再验一次白名单方案。

调研后电视已恢复干净：卸载 `com.ts.launcher`、HOME 设回 `com.mitv.tvhome`、无障碍只留小爱同学、
临时 APK 全删；12 个禁用包和三个影视应用不变。

## 第二轮调研：LM Ultimate，同型号成功案例（2026-11-11 15:50–16:00）

用户指出网上有更多小米电视改默认桌面的成功案例。这轮找到**决定性的一条**：

### 1. ZNDS 同型号专帖（最重要）

[小米EA55（L55M7-EA）怎么把系统桌面替换当贝桌面](https://www.znds.com/tv-1233298-1-1.html)
—— L55M7-EA 就是本机 ASTP0/mulan。楼主描述的故障与本次完全一致：

> 「安装当贝桌面后打开设置一下断电开机都是当贝桌面的，但被小孩按到了**系统升级后**打开当贝桌面
> 再按主页就回到系统桌面了，怎么设置都不行，**用 ADB 把系统桌面删除后也不行**」

5 楼给出解法「下载 LM ULTIMATE，禁用一下 stock Launcher 就行了」，楼主 2023-4-4 回复
「**晚上回家一试真的好用了**」，并写明完整步骤：先装第三方桌面 → 开 ADB 调试 → 装 LM Ultimate
（打开前必须先装好第三方桌面，否则可能变砖）→ 选 `DISABLE STOCK LAUNCHER` → 确定
→ 「按遥控器主页键，可以发现不会进去 MIUI TV 界面了，**重启之后也是默认打开当贝桌面**」。
恢复用 `ENABLE STOCK LAUNCHER`。9 楼有人反馈「用一段时间遥控器没法关机」，楼主回复一直正常。

### 2. 知乎「小米电视 系统精简 与 系统重置 教程」（两台 1.5G 内存小米电视）

说清了机制和**必须的顺序**（这正是我们前面几轮失败的原因）：

- 「新版小米电视**锁定了桌面切换**，不 ROOT 如果不先锁定第三方桌面就删除 MIUITV 桌面，
  就会得到经典的蓝天白云（FallbackHome）」
- 必须 Android 7 以上（本机 9 ✓），工具源自 XDA
- **流程要点**：装第三方桌面 + LM Ultimate → 选 `disable stock launcher` → **等 10 秒重启**
  → 重启后小米会关 ADB，**重新打开 ADB 调试** → 回到小米桌面 → **再打开 LM Ultimate，
  第二次选 `disable stock launcher`** → 此时 active launcher 才显示第三方桌面 → 再重启，完成
- 特别警告：装 EmotnUI 时若弹出「选择主屏幕应用」，**不要点**，点了会灰屏、重启仍是 MIUI TV

### 3. 吾爱破解 / 系统迷（工具信息与 2026 复现）

- LM Ultimate v1.0.1，1.26 MB，免 root，Android 7+，XDA 起源，兼容海信/小米/索尼；
  52pojie 帖下有用户反馈「小米电视5使用成功」
- [系统迷 2026-03-04](https://www.xitmi.com/12293.html)：作者在**两台小米电视**上成功换成 Emotn UI，
  步骤同样是点 `DISABLE STOCK LAUNCHER`；之后才卸载 `com.mitv.tvhome` 并删应用商店防还原
- 下载源：蓝奏云 `https://luojiang.lanzouu.com/iKIM51jfm39a`、
  123pan `https://www.123pan.com/s/sXtA-BuVEh.html`、ZNDS 帖附件 `LMUltimate-TDUK-NOV.apk`

### 为什么这条路和我们前面失败的做法不同

| | 我们试过的 | LM Ultimate |
|---|---|---|
| 设默认方式 | `cmd package set-home-activity`（被小米忽略） | 工具内部「锁定」第三方桌面（具体机制未公开） |
| 原桌面 | 一直保持启用 | 由工具在锁定后禁用 |
| 重启 | **一次都没重启过** | 中间必须重启，且要点**两次** disable |

### 尝试替用户先审一遍 APK

从蓝奏云解析到 `LMUltimate-TDUK-NOV.apk`（1.3 MB，上传者 nojon），但直链最后一层
要求图形验证码（`{"zt":0,"url":"验证码错误"}`），脚本拿不到文件。
**未安装任何来路不明的 APK 到电视**。若用户自行下载后放到工作区，可先做静态检查
（包名/权限/签名/可疑网络行为）再决定是否安装，并可用 ADB 注入按键代替遥控器点按钮。

### 需要用户配合的点

1. 装 LM Ultimate（第三方闭源工具，需用户同意）
2. 重启两次；**每次重启后 5555 会关闭**，需到「设置 → 账号与安全 → ADB 调试」重开
3. 回滚：LM Ultimate 自带 `ENABLE STOCK LAUNCHER`；极端情况长按菜单键呼出侧边栏做系统重置，
   或拔电源按住「确认+返回」插电进 REC 清空数据

## 第三轮：Aerial 开源项目 —— 更优方案（用户提供的 GitHub 链接）

用户给了两个链接，其中 `https://github.com/pengchujin/aerial-tv` 是本轮最有价值的发现。

### 1. 这台机型被明确标记为支持

README 兼容表原文：`小米 MiTV-ASTP0、ASTP1、NSOP0、MFTP0 / Android 9 / ✅`
（备注只给 MFTP0 写了「被系统清理后 30–60 秒才回来」，ASTP0 无额外限制）。
项目是 MIT 开源、面向国产电视的 Apple TV 风格桌面，**不用 root、不卸载/停用系统应用、随时可还原**，
兼容表覆盖小米/TCL/海信/索尼/创维等几十个机型且带 issue 编号。

### 2. 它的机制正好绕开我们卡住的每一个点

`site/agent.md` 里针对小米写得很直白：

- 第 7 步「设为默认桌面」：**「小米也跳过」**（`pm path com.mitv.tvhome` 有输出就是小米），
  「小米、TCL 等固件会无视这一步，没关系，下一步的**桌面守护会接管**」
  → 我们前几轮一直在跟 `set-home-activity` / HOME 白名单死磕，方向就错了
- `HomeGuardService`（无障碍守护）：原桌面一出现就切回 Aerial
- `RevivalListener`（通知监听）：**被系统清理后把 Aerial 拉回来**
  → 正好解决我们实测到的「小米把 `enabled_accessibility_services` 改回、守护失效」
- `pm grant app.orchard.tv android.permission.WRITE_SECURE_SETTINGS` 等 ADB 授权，
  让桌面自己能维护这些设置
- `KEEP_ADB=1`：小米待机唤醒后会关 ADB，开了这项自动重开
- 验收标准明确：`resolve-activity` 在小米上**仍是 `com.mitv.tvhome` 属正常**；
  判成功只看按主页键后 `mCurrentFocus` 是否包含 `app.orchard.tv`

### 3. 一个重要的自我纠正

`agent.md` 验收段原文：「**有的老固件上 `input keyevent KEYCODE_HOME` 不经过守护，
第 6 条看着失败时请用户用实体遥控器按一次主页键再看**」。
我们前几轮全部用 ADB 注入按键验证，可能得到**假阴性**；更重要的是当时用的
`com.gathin.tvhome` 没有 `RevivalListener` / `WRITE_SECURE_SETTINGS` 这套复活机制，
守护一旦被小米清掉就回不来。所以「按 Home 回原桌面」不等于「固件完全不可能接管」。

### 4. 用户给的另一个链接是反面案例

`https://www.znds.com/tv-1257188-1-1.html`（2025 小米电视改桌面）用的是老方法：
`pm uninstall --user 0 com.mitv.tvhome` + `pm uninstall --user 0 mitv.service`。
- **5 楼回复原文「系统会自动恢复桌面」**，印证方法不可靠
- `mitv.service` 是小米核心服务，上游清单标注 do not touch，卸载风险高
- 这代固件卸载原桌面会掉 FallbackHome（上游三份记录一致）

### 5. 成功案例规模与诚实的失败统计

独立来源（2022–2026）：CSDN 2022/2024/2025、ZNDS 同型号专帖（13 万+浏览，楼主成功）、
ZNDS 2025/2026 帖（2026 重新汉化版 14910 浏览）、吾爱破解 2023（多人「小米电视5使用成功」）、
知乎（两台 1.5G 内存小米电视）、系统迷 2026（两台）、B 站 2025/2026 视频、百度知道、
上游 cleanyourtv 仓库三台小米实测记录、Aerial 兼容表几十机型。

已知失败/副作用（不隐瞒）：
- ZNDS 2026 汉化版楼主：Redmi X65 **澎湃 OS3 失败**，「禁用系统桌面的同时还会禁用系统设置」、
  「关机重启进去系统黑屏，最后只能还原系统」；6 楼「直接重启黑屏，小白就变砖」
- ZNDS 同型号帖 9 楼「用一段时间遥控器没法关机」（10 楼楼主说一直正常）
- 52pojie：部分机型「包解析错误」安装不了；华为智慧屏不支持
- 系统迷：不要更新系统；需卸载应用商店防桌面还原
- Aerial 自己也标了 ❌ 机型（小米电视2 / Android 4.3、华为 M330、雷鸟 75S78A Plus、
  创维 7T641 等），说明不是万能 —— 但 **ASTP0 在 ✅ 那一栏**

### 6. 建议顺序（等用户重启后）

1. 先复查重启是否让云控白名单生效（最干净，不用装任何东西）
2. 没生效 → 优先用 **Aerial**（开源可审计、ASTP0 已验证、有复活机制、`install.sh` 一行安装、
   带 `KEEP_ADB=1`），而不是 LM Ultimate（闭源、有变砖案例、澎湃 OS3 实测失败）
3. Aerial 一行命令：`curl -fsSL https://aerial.qust.me/install.sh | KEEP_ADB=1 TV=192.168.26.10 bash`
   脚本只做安装 + 授权 + 守护，**不卸载任何系统应用**；还原用 `bash -s restore --uninstall`

用户暂未重启，本轮未对电视做任何改动。

## Aerial 安装准备（2026-11-11 19:15，未执行）

> ⚠️ 本节是 19:15 的**准备阶段推断**，其中三条已被 19:22 的实测修正：
> ① 本机官方 `adb` 其实**可以直连** 192.168.26.10（`device`），不需要 TVHelper 通道；
> ② Aerial 拿到 `WRITE_SECURE_SETTINGS` 后用**全限定组件名**自己打开了守护，未被小米剔除；
> ③ 用户实际装的是 **standard 版**（`versionName=1.94`），工作正常，无需 guard 版。
> 以文末「Aerial 配置为默认桌面」一节的实测结果为准。

**结论：能装，而且这是目前最对症的方案；当时唯一阻塞是电视离线。**

### 为什么官方一行命令在这里跑不了

官方 `curl -fsSL https://aerial.qust.me/install.sh | KEEP_ADB=1 TV=<IP> bash` 依赖本机 `adb`
二进制直连电视。但本会话的进程**没有 macOS 本地网络权限**（创维那台已验证：直连一律
`EHOSTUNREACH`，既不弹窗也无法手动授权），只有 `TVHelper.app` 那条通道能连局域网。
所以把安装流程重构成了 `tv.sh -f` 下发的 ADB shell 命令（APK 用 b64push 分块推送）。

### 已下载并静态检查

| 项目 | 结果 |
|---|---|
| 版本 | **v1.94，2026-11-11 01:37 UTC 发布**（当天） |
| 文件 | `aerial-standard.apk` 3.0 MB / `aerial-guard.apk` 3.0 MB |
| sha256 | guard `64dd559f…87d2` · standard `ab44c5ab…a767` |
| 包名 | `app.orchard.tv`（两版相同） |
| SDK | `minSdk 21` / `targetSdk 34` → 本机 Android 9 (28) 兼容 |
| 入口 | 两版都有 `LEANBACK_LAUNCHER` |
| 关键权限 | INTERNET / WRITE_SECURE_SETTINGS / SYSTEM_ALERT_WINDOW / QUERY_ALL_PACKAGES / ACCESS_FINE_LOCATION / REQUEST_INSTALL_PACKAGES |
| 守护组件 | 两版都含 `HomeGuardService`、`RevivalListener`、`ConfigReceiver`、`AerialDream` |

**决定用 guard 版**：它故意**不声明 `android.intent.category.HOME`**（standard 版声明了），
因此不参与小米的 HOME 白名单校验，也不会像之前装两个桌面那样弹出 `ResolverActivity`；
完全靠 `HomeGuardService` 无障碍守护接管。这与 agent.md 第 7 步「小米也跳过，交给桌面守护」
完全一致。standard 版留作安装失败时的自动回退。

### 三条来自本机实测的适配

1. **组件名必须用短格式**：之前 `com.gathin.tvhome/com.gathin.tvhome.HomeKeyService`
   会被小米自动剔除，改成 `com.gathin.tvhome/.HomeKeyService` 才绑定成功。
   Aerial 官方写的是全限定名，脚本会先按官方格式写、检查 `dumpsys accessibility`，
   没绑定就自动改用 `app.orchard.tv/.guard.HomeGuardService` 重试。
2. **必开 `keep_adb`**：小米重启/待机唤醒会关 5555，正是这轮反复失联的原因。
3. **验收别被 ADB 按键骗了**：agent.md 明确「老固件上 `input keyevent KEYCODE_HOME`
   不经过守护」，看着失败时需用**实体遥控器**按一次主页键再看 `mCurrentFocus`。
   小米上 `resolve-activity` 仍是 `com.mitv.tvhome` 属正常，不算失败。

### 当前阻塞（19:15 实测）

```
ping 192.168.26.1    通          ← 26 网段网关正常
ping 192.168.9.254   通          ← 软路由正常
ping 192.168.26.10   0/3         ← 只有电视不通
arp -n .10           no entry
nc 5555              Network is unreachable
```

路由本身没问题（`route get` 走 .254），是**电视自己离线**——重启后待机断网或 ADB 被关。
需要用户唤醒电视、确认联网，并在「设置 → 账号与安全 → ADB 调试」重新打开后，才能执行安装。

### 已备好的脚本

```
xiaomi-backup/aerial/安装-Aerial.sh    经 tv.sh -f 执行，含 guard→standard 自动回退、
                                      原值备份、授权、keep_adb、守护绑定、9 项验收
xiaomi-backup/aerial/回滚-Aerial.sh    一键回到安装前（含恢复旧桌面与无障碍设置）
xiaomi-backup/aerial/pre-install-values.txt   执行时自动生成
```

安装脚本会**先停用**旧的 `com.gathin.tvhome`（`pm disable-user`，可 `pm enable` 还原）
以避免双 HOME 桌面；**不卸载、不停用任何系统应用**，`com.mitv.tvhome` 保持启用。
屏保接管这次先不做，避免改变使用习惯。

**本轮未对电视做任何改动。**

## 开机广告状态（2026-11-11 15:20 复查）

**这轮操作没有处理小米的开机广告**，范围是桌面 + 精简。设备上确实存在开机广告机制：

```
sys.bootad.playing = 0            当前未在播放
service.bootvideo = 0
/system/media/bootanimation.zip   开机动画（非广告）
settings system: boot_ad=0, boot_ad_sound_enable=0, boot_ad_switch_show=0, boot_auto_play=1
```

`boot_ad_switch_show=0` 说明固件把用户可见的「开机广告」开关隐藏了。小米开机广告由系统级组件/云控投放，
与桌面推荐位、屏保画报是三条独立链路，ADB 禁用包只能覆盖后两条，**不能可靠关掉开机那一段**。
本轮顺带禁用了 `com.xm.webcontent`（推广网页）、`com.mitv.shoplugin`（购物插件），
上游把它们归在 Ads / promoted content / shopping 类，可能减少推荐位，但开机广告不受影响。

可行的官方途径：

1. 电视 **设置 → 智能推荐 → 广告服务 → 开机广告**（需 MIUI TV 3.5.128+ 且登录小米账号）；
   本机开关被隐藏，若找不到就走第 2 条。
2. **联系小米在线客服，提供 MAC 让后台永久关闭开机广告**。本机 MAC：
   - 有线 `78:53:33:20:60:16`
   - Wi-Fi `a8:96:09:9f:23:d7`

未擅自改动 `boot_auto_play=1`（含义不确定，可能是「开机自动进入上次信号源」）。

对照：**创维 7S87_A9 的开机广告此前已去掉**（`persist.get.bootvi_logic.enable=0`、
`persist.sys.bvsswitch=0`、`persist.sys.bootvideo.time=1`，用户 11-06 确认消失），
`persist.*` 属性重启不丢；该机当前待机离线，未能实时复查。

## Aerial 配置为默认桌面（2026-11-11 19:22–19:35，已执行）

按用户要求**完整读完** `https://aerial.qust.me/agent.md`（321 行）后执行。
本机官方 `adb` 直连成功（`device`），全部命令按官方原文原样跑。

### 起点

| 项目 | 值 |
|---|---|
| Aerial | **已由用户装好**：`app.orchard.tv` v1.94，firstInstallTime 15:43 |
| 系统 | SDK 28 / Android 9 → 完整版（第 2½ 步） |
| 当前前台 | 已是 Aerial |
| 缺的 | 守护没开、复活监听 `null`、未设 `keep_adb` |

### 按说明执行（第 3 步 3–8 条）

- 授权 13 条：`WRITE_SECURE_SETTINGS` 等 8 条成功；`ACCESS_RESTRICTED_SETTINGS` 与
  `APP_*_START` 4 条报 `Unknown operation string`（Android 13+/TCL 专用，**文档预期**）
- 通知监听 `cmd notification allow_listener` → 回读 `app.orchard.tv/app.orchard.tv.guard.RevivalListener`
- `cmd package compile -m speed -f app.orchard.tv` → `Success`
- 屏保（`screensaver_enabled=1`）→ 改为 `app.orchard.tv/app.orchard.tv.dream.AerialDream`
  （原值 `com.android.deskclock/...Screensaver` 已记在 `aerial/pre-config-values.txt`）
- **`keep_adb true`** → 广播返回 `data="keep_adb=true ..."`
- 第 7 条：**小米，跳过**设默认桌面（`pm path com.mitv.tvhome` 有输出）
- 第 8 条：Aerial 拿到授权后**自己打开了守护**，全限定名这次未被小米剔除：
  `com.xiaomi.voicecontrol/.VoiceAccessibilityService:app.orchard.tv/app.orchard.tv.guard.HomeGuardService`，
  `dumpsys` 显示 `Service[label=Aerial 桌面守护, eventTypes=TYPE_WINDOW_STATE_CHANGED]`

### 验收 6 条原始输出（完整见 `aerial/验收输出.txt`）

```
    versionName=1.94
priority=0 preferredOrder=0 match=0x108000 specificIndex=-1 isDefault=true
com.mitv.tvhome/.MainActivityUserMode
com.xiaomi.voicecontrol/.VoiceAccessibilityService:app.orchard.tv/app.orchard.tv.guard.HomeGuardService
1
(第 5 条 input keyevent KEYCODE_HOME 无输出)
  mCurrentFocus=Window{629225c u0 app.orchard.tv/app.orchard.tv.MainActivity}
```

第 2 条仍是 `com.mitv.tvhome` —— **小米正常**（说明第 244 行），由守护接管；第 4 条 =1，
第 6 条含 `app.orchard.tv` → 判定成功。

**严格补充验证**：先切到小米设置页，再按 Home → 8 秒与 13 秒后前台均为
`app.orchard.tv/app.orchard.tv.MainActivity`，守护确实生效。

### 重启验证（用户同意后 `adb reboot`）

```
5555 恢复        约 9 秒，adb devices = device   ← keep_adb 生效，无需手动重开 ADB
uptime           72.82 秒（确认真的重启了）
开机后前台       com.xiaomi.mitv.tvplayer/ExternalSourceActivity（4 次采样约 55 秒不变）
按一次 Home 后   app.orchard.tv/app.orchard.tv.MainActivity（8 秒拉回，13 秒仍保持）
守护/复活监听/Aerial 进程    重启后全部保持
```

**开机没直达 Aerial 的原因**：小米「开机自动播放」开着（`boot_auto_play=1`、
`boot_auto_play_backup=1`、`LastCurrentSource=23`），开机直接进 HDMI 信号源，
守护检测不到原桌面窗口，因此不触发。按一次主页键即进 Aerial。

### 为什么没有直接改 `boot_auto_play`

`adapt.md` 规则 2 明确「**不要改下面没有列出的系统设置**」，该键不在允许清单；
`agent.md` 规则 3 也禁止停用任何系统应用。所以这一步需要**用户在电视菜单里操作**：

> 设置 → 通用设置 → **开机自动播放** → 关闭（不要选「上次关机时的信号源」/指定 HDMI）

关掉后开机应先出小米桌面，守护随即把 Aerial 拉起，即实现「开机进入 Aerial」。
适配文档第 93–95 行也说明：开机后原桌面先出现、Aerial 几秒后接管属正常。

### 官方诊断脚本结果（只读）

- `aerial/aerial-report-MiTV-ASTP0.md`（16 KB），日志里有守护实际工作证据：
  `guard: stock home com.mitv.tvhome/... in front, not opened on purpose; bringing Aerial back`
- `aerial/aerial-selftest-MiTV-ASTP0.md`：
  `guard PASS` / `home_on_aerial PASS` / `home_from_app SKIP`（脚本没能把设置页切到前台，
  但本会话已手工验证该等价场景通过）/ `standby`、`reboot`、`inputs` 默认 SKIP

### 规则遵守与回滚

全程**没有卸载/停用任何系统应用**（`com.mitv.tvhome` 保持启用），没有安装 Aerial 以外的 App，
没有删除电视文件，没有改动未列出的系统设置。
原值：`aerial/pre-config-values.txt`（屏保组件、无障碍服务、通知监听、版本）。
还原命令见 `agent.md`「还原」一节，另有 `aerial/回滚-Aerial.sh`。

### 待办

1. **用户**：电视上关闭「开机自动播放」
2. **用户确认后**：再重启一次，跑验收第 6 条确认开机直达 Aerial（重启需再次征得同意）
3. 收尾按说明执行 `adb disconnect`（暂留连接以便复验）

## 文件清单

```
xiaomi-backup/20261111-150015/     完整备份（19 个文件）
xiaomi-backup/apps/                三个应用 APK（用于识别名称）
xiaomi-backup/config.json          当前桌面配置
xiaomi-backup/applied-debloat.txt  实际禁用的 12 个包（还原用）
xiaomi-backup/new-launcher.png     调整前截图
xiaomi-backup/new-launcher-final.png  当前效果
```
