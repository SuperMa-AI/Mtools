---
name: cleanyourtv
description: 通过 Wi-Fi 用 ADB 给 Android TV / Google TV 去臃肿（禁用厂商遥测、广告、推荐流），并把原生桌面换成 Apple TV 风格的干净启动器（自带可编译的 280KB 原生 Launcher + 主题）。发行版 v2.6.0，含 213 款已测试机型清单，全程可一键还原、不卸载系统包。触发词：电视去广告、电视去臃肿、Android TV 去广告、Google TV 精简、小米电视广告、PatchWall、换电视桌面、电视启动器、cleanyourtv、debloat TV、Apple TV 风格桌面。
agent_created: true
---

# CleanYourTV：电视去臃肿 + 换 Apple TV 风格桌面

上游仓库 `Gathin23/cleanyourtv`（本技能装的是**发行版 v2.6.0**，213 款机型清单），已内置 Python 依赖。
核心承诺：**不用 root、不用接线、不卸载任何系统包、随时可还原**。

## 路径约定

```bash
SKILL="$HOME/.workbuddy-ai/skills/cleanyourtv"
V="$SKILL/vendor/cleanyourtv-2.3.0"     # 发行版原始内容（tools/ launcher/ examples/ themes/ docs/）
PY="$V/tools/venv/bin/python"           # 已装好 adb-shell / rsa / cryptography
cd "$V"                                 # 所有 tools/*.sh 必须在 $V 下运行
```

- 机型清单：`$V/examples/<机型目录>/debloat.txt`，索引见 `references/devices.md`
- 上游给 AI 的原始任务书：`$V/PROMPT.md`（用户问「原版怎么要求」时读它）
- 排障：`$V/docs/TROUBLESHOOTING.md`；主题键值：`$V/themes/README.md`

## 适用性先判断（省时间）

- ✅ 支持：Android TV / Google TV（小米、Redmi、TCL、海信、创维、索尼、飞利浦、OnePlus、Realme、Nvidia Shield、CHiQ、XGIMI 等）
- ❌ 不支持：Samsung Tizen、LG webOS、Roku、以及 Fire TV 的完整改造（Fire OS 可做去臃肿，换桌面受限）
- 前置：电视与电脑同一 Wi-Fi；电视上开「开发者选项 → USB 调试 / 网络调试」，保持亮屏

## 标准流程

### 1. 连接

```bash
cd "$V"
TV_HOST=<电视IP> $PY -I tools/tv.py --probe          # 打印机型 / Android 版本
TV_HOST=<电视IP> $PY -I tools/tv.py "pm list packages -3"
```

- 找 IP：电视「设置 → 网络」看，或 `arp -a`、路由器后台。
- 首次连接电视会弹「允许 USB 调试吗？」，**先提醒用户去按允许**。
- macOS 上系统 `adb` 报 `No route to host`（端口其实开着）= 本地网络隐私权限拦截，不要去修 adb，
  直接用内置的 `tools/tv.py`（adb-shell 实现，不依赖 adb 二进制）。
- 大文件推送卡住时用 `tools/b64push.py <本地> <远端>`（base64 分块 + md5 校验）。

### 2. 先备份，再动手（不可跳过）

```bash
TV_HOST=<ip> $PY -I tools/tv.py "pm list packages" > backup/packages.txt
TV_HOST=<ip> $PY -I tools/tv.py "pm list packages -d" > backup/disabled.txt
TV_HOST=<ip> $PY -I tools/tv.py "getprop" > backup/getprop.txt
TV_HOST=<ip> $PY -I tools/tv.py "settings list global" > backup/settings-global.txt
```

同时把要用的 debloat 清单复制一份到 backup 目录——它就是还原脚本的输入。

### 3. 去臃肿

优先用现成清单（命中机型最稳）：

```bash
TV_HOST=<ip> tools/disable.sh examples/xiaomi-mi-tv-4a/debloat.txt   # 禁用
TV_HOST=<ip> tools/restore.sh examples/xiaomi-mi-tv-4a/debloat.txt   # 还原
```

清单不匹配时自己整理，按类别分批禁用（**每批之间截图确认**）：

```bash
TV_HOST=<ip> $PY -I tools/tv.py "pm disable-user --user 0 <包名>"
```

- **只 `disable-user`，永不 `uninstall` 系统包。**
- **一条 `pm` 一个 shell 调用**——链式命令可能在第一条后静默中断。
- 绝不能动：TV/HDMI 输入与调谐器相关、信号源菜单、遥控与蓝牙配对、输入法、Play 商店与 Play 服务。
- 厂商包名往往看不出用途，先 `dumpsys package <包名>` 看它的 activity/service 再判断，仍不清楚就问用户。
- 动流媒体 App、语音助手、Chromecast/Miracast、系统更新服务之前，**必须先问用户哪些在用**（勾选式提问），只禁用没勾的。

### 4. 换桌面（Launcher）

`$V/launcher/` 是完整的原生 Android TV Launcher 工程（Java + RecyclerView，debug 构建即可）：

```bash
cd "$V/launcher"
./build-and-install.sh <电视IP>       # 或 build-and-install.ps1
```

- 装前先验证电视收不收 APK：中国区机型常见 `INSTALL_FAILED_VERIFICATION_FAILURE`，或拒收任何声明
  `android.intent.category.HOME` 的非厂商签名包。命中就直接说「这台只能做去臃肿，换桌面做不到」，**别反复重试**。
- Android 14 上 `adb install` 会挂在等待屏幕确认，改用「push APK → `pm install -r`」。
- 设默认桌面：

```bash
$PY -I tools/tv.py "cmd package query-activities -a android.intent.action.MAIN -c android.intent.category.HOME"
$PY -I tools/tv.py "cmd package set-home-activity --user 0 <pkg>/<activity>"
```

- 禁用原生桌面之前先确认还有谁声明了 HOME：Google TV 上 `com.google.android.tungsten.setupwraith` 会抢回桌面。
- Android 11+ 的 Launcher 必须声明 `<queries>` 里的 `LEANBACK_LAUNCHER`，否则应用网格全空。

#### 4b. 酷开 Coocaa（创维国行）专用流程 ⚠️ 与上面完全不同

**酷开没有 Android HOME 选择器**，`query-activities -c android.intent.category.HOME` 返回
`No activities found`，`set-home-activity` 也无效。桌面由**系统属性**决定，由 `com.tianci.system` 读取：

```bash
$PY -I tools/tv.py "getprop persist.service.homepage.pkg; getprop persist.service.homepage.cls"
# 出厂值：com.tianci.movieplatform / com.coocaa.homepage.vast.HomePageActivity
```

换桌面必须走 **shim 方案**（完整步骤见 `$V/examples/skyworth-7s60-a5/README.md`）：

1. **改包名编译 launcher**。酷开内存管理会在应用切走后约 2 秒强杀第三方应用，只放过包名第二段是
   `coocaa` / `skyworth` / `tianci` / `android` 的包，所以必须编译成 `com.coocaa.tvhome`：

```bash
cd "$V/launcher"
./gradlew assembleRelease -PlauncherApplicationId=com.coocaa.tvhome
$PY -I tools/b64push.py app/build/outputs/apk/release/app-release.apk /data/local/tmp/tvhome.apk
$PY -I tools/tv.py "pm install -r /data/local/tmp/tvhome.apk; rm -f /data/local/tmp/tvhome.apk"
$PY -I tools/tv.py "am start -W -n com.coocaa.tvhome/com.gathin.tvhome.MainActivity"
```

2. **推送 config.json**（`theme.sh` 硬编码 `com.gathin.tvhome`，这里要手动推）：

```bash
$PY -I tools/b64push.py examples/skyworth-7s60-a5/config.json /sdcard/Android/data/com.coocaa.tvhome/files/config.json
$PY -I tools/tv.py "am force-stop com.coocaa.tvhome; am start -n com.coocaa.tvhome/com.gathin.tvhome.MainActivity"
```

3. **装 shim 并指向它**（shim 包名必须是 `com.ccos.tv.homepage`，因为 `com.tianci.system` 每次开机
   会把 Home 重置成这个包；shim 再从后台把启动器拉起来）：

```bash
# APK 来自 https://github.com/VincentYe123/coocaa-home-shim/releases
$PY -I tools/b64push.py coocaa-home-shim.apk /data/local/tmp/shim.apk
$PY -I tools/tv.py "pm install -r /data/local/tmp/shim.apk; rm -f /data/local/tmp/shim.apk"
$PY -I tools/tv.py "setprop persist.service.homepage.pkg com.ccos.tv.homepage; setprop persist.service.homepage.cls com.ccos.tv.homepage.LaunchActivity; input keyevent KEYCODE_HOME"
```

4. **最后**才禁用原桌面（否则开机流程没东西可开）：

```bash
$PY -I tools/tv.py "pm disable-user --user 0 com.tianci.movieplatform"
```

坑：
- 开机时框架偶尔把 `home[0]`/`home[1]` 字面量写进属性，Home 失效——重跑第 3 步的 `setprop` 即可。
- 还原顺序：**先删 shim**（`pm uninstall com.ccos.tv.homepage`），否则每次开机都被指回去；
  再 `pm enable com.tianci.movieplatform` 并 `setprop` 回原值。
- 编译需要 JDK 17 + Android SDK（compileSdk 34、minSdk 26、Gradle 8.9）。
- 改 dock 顺序/壁纸/列数：改 `launcher/app/src/main/res/values/config.xml` 重新编译，或用主题（见下）。

### 5. 主题（免编译换外观）

```bash
TV_HOST=<ip> tools/theme.sh themes/aurora       # 推送主题并重启 Home
TV_HOST=<ip> tools/theme.sh --reset             # 回到内置 config.xml
```

内置 `default/`、`aurora/`、`six-columns/`；自定义就复制一份改 `config.json`。

### 6. 验证

每改一步都**截图看**，不要靠猜：

```bash
$PY -I tools/tv.py "input keyevent KEYCODE_DPAD_DOWN"
$PY -I tools/tv.py "screencap -p /sdcard/s.png" && $PY -I tools/tv.py --pull /sdcard/s.png ./s.png
```

遥控按键若依赖无障碍服务，先确认服务已启用并绑定当前用户，并请用户按一次实体键验证（注入按键不算证据）。

### 7. 收尾

- 交付：还原脚本（`restore.sh` + 本次清单）、构建安装脚本、一份说明「怎么重新启用某 App / 怎么改 dock」的 README。
- 提醒用户：USB 调试开着时部分机型会断掉 USB 口供电（插着的音箱/U 盘会失效），**改造完记得关调试**。
- 可选加广告 DNS（Android 9+）：`settings put global private_dns_specifier dns.adguard-dns.com` +
  `private_dns_mode hostname`；告知它不拦 YouTube 广告，撤销用 `private_dns_mode opportunistic`。

## 红线

1. 改动前必须备份包列表 + settings，并准备好还原命令。
2. 不卸载系统包、不 root、不刷机。
3. 涉及用户偏好的（哪些 App 在用、dock 放哪 4 个、留不留语音助手/投屏/系统更新）先问再做。
4. 小批量禁用 + 每批截图，出问题能定位到哪一批。
5. 只在用户自己的设备上操作，操作前说明会改什么。

## 参考

- **创维 / 酷开（Coocaa）机型请先看 `coocaa-tv` 技能**——它有完整的实战流程（本地网络权限绕过、
  安装确认框自动化、shim 换桌面、关开机广告），比这里的通用流程更具体。
- `references/devices.md` — 54 款机型清单索引（目录、包数、机型/系统）
- `$V/PROMPT.md` — 上游完整任务书（含设计规格：纯黑背景、dock 玻璃高亮、4 列网格、磨砂胶囊按钮等）
- `$V/docs/TROUBLESHOOTING.md` — 已知坑
- `$V/docs/modules.md` — 各包类别说明
