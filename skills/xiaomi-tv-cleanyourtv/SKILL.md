---
slug: xiaomi-tv-cleanyourtv
name: xiaomi-tv-cleanyourtv
displayName: 小米电视 CleanYourTV 净化部署
version: 1.2.1
summary: 让 AI 带你给小米电视装上 Apple TV 风格桌面，按需精简预装应用，也能排查按主页键黑屏。
description: 适用于小米 Android 电视和兼容盒子。把型号、系统版本和局域网 IP 告诉电脑上的 AI，它会指导你开启 ADB、备份现状、安装或复用 CleanYourTV，并验证遥控器主页键和开机桌面。可按需停用广告和预装应用，保留还原记录。
license: MIT
tags: [android-tv, xiaomi, hyperos, debloat, launcher, adb, cleanyourtv]
homepage: https://github.com/Gathin23/cleanyourtv
---

# 小米电视 CleanYourTV 净化部署

想给小米电视换一个 Apple TV 风格的桌面，或排查按主页键后黑屏？这个 skill 让电脑上的 AI 带你完成操作。电视和电脑连到同一局域网，准备好目标电视或盒子的型号、系统版本和局域网 IP。找不到这些信息时，可以先让 AI 指导你用遥控器查看。

你可以只安装桌面，也可以尝试将它设为默认桌面，或按需精简广告和预装应用。AI 会先检查兼容性、备份当前设置，再安装或复用已有桌面。能否接管主页键和开机画面取决于固件，两项分别实机验证。精简通过停用应用完成，可按变更记录还原。

[项目仓库与部署指南](https://gitee.com/ice-colaa/cleanyourtv)

## 红线（先读）

- 只用 `pm disable-user --user 0`，**永不 uninstall**，一个包一条命令。
- 永不禁用 `com.android.*` / `com.google.android.*` 核心包、`com.android.tv.settings`、`com.android.systemui`、providers（详见 cleanyourtv 仓库 AGENTS.md，`tools/disable.sh` 会拒绝这类清单）。
- 不禁用：电视/输入切换服务（HDMI/信号源）、蓝牙遥控、输入法、应用商店、**电视管家**（小爱"打开应用"依赖它的 provider）、系统更新。
- 不启动名字含 factory/test/fac 的包（会触发工厂模式，部分机型无法撤销）。
- 改动前必须备份；分批禁用、批间截图验证。
- 完成后建议关闭 ADB（`ro.adb.secure=0` 时局域网内任何设备可免授权连接）。

## 1. 连接电视

电视端：设置 → 关于 → 连按"产品型号"7 次（开启开发者模式）→ 账号与安全 → **ADB 调试 → 允许**。

```bash
adb connect <TV_IP>:5555     # HyperOS 弹"允许 USB 调试"时勾选"一律允许"
adb devices                  # 确认 device
```

坑：
- **该固件每次重启后 ADB 自动关闭**，需重开；待机只断连不关授权，重连即可。
- 待机连不上时可发 WoL 魔术包唤醒（使用本机网络信息）。
- ping 不通不代表电视离线（小米电视常禁 ICMP），以 `adb connect` 的 refused/timeout 区分：refused=电视在线端口关，timeout=可能待机。

## 2. 同步与备份

从 https://github.com/Gathin23/cleanyourtv 获取代码。已有工作区先检查分支、远端及本地修改，记录实际提交号。阅读对应提交的 AGENTS.md、PROMPT.md 和本机型说明。安装任务只需本地同步；维护 fork 时按机主指定的远端和分支单独处理。

任何变更前建立独立的时间戳备份目录。保存 getprop、用户 0 的完整包列表和已停用列表、三类 settings、dumpsys tv_input、桌面包启用状态、HOME 角色、候选和解析组件。逐项记录本次变更及原值，先生成还原脚本，并在每次成功变更后更新记录。备份、截图与设备标识只保存在本机。

## 3. 选择安装或诊断路径

先核对型号、完整固件号和 Android SDK。通过已授权的 ADB 连接执行以下只读查询：

```sh
adb shell cmd role get-role-holders --user 0 android.app.role.HOME
adb shell cmd package query-activities --brief --user 0 -a android.intent.action.MAIN -c android.intent.category.HOME
adb shell cmd package resolve-activity --brief --user 0 -a android.intent.action.MAIN -c android.intent.category.HOME
adb shell dumpsys window
```

已有 CleanYourTV 时，检查已知桌面组件能否直接打开、截屏是否正常。若主页黑屏，在按键后立即查 mCurrentFocus 和 logcat。如果进入 FallbackHome，优先检查默认 HOME 的选择。保留可用原厂桌面及设置入口，通过系统支持的默认桌面设置尝试切换。

首次安装时按当前检出版本的 launcher 构建说明核对 JDK、SDK 和签名。安装一次并验证实际画面。安装被固件拒绝或无法选择为默认桌面时，保留可用桌面，按机主要求继续精简。

## 4. 验证桌面

分别记录直接打开的画面、从其他应用按实体主页键后的前台窗口，以及重启后首次进入的桌面。模拟按键可辅助排查。只有实测过的项目才标记通过；ADB 重启后关闭时，请机主在电视设置中重开。若主页进入原厂桌面，记录“主页可用，自定义桌面尚未接管”。

MiTV-MFFU0 的 OS4.0.18.0.UFFMATV 案例中，系统已解锁，主页键却进入 FallbackHome。设置 HOME 角色后角色持有者为自定义桌面，两次模拟主页键均进入原厂桌面；直接打开自定义桌面正常。当次未做重启验收，不能据此断定固件升级是原因。

## 5. 精简系统

用仓库清单作模板（`examples/<机型>/debloat.txt`，逐包核对当前固件和原始禁用状态），按类别分 2-3 批：

```bash
bash tools/disable.sh ../work/batch1.txt   # 遥测/云控
tools/tv.py --screenshot shots/01.png      # 批间截图验证
bash tools/disable.sh ../work/batch2.txt   # 广告/购物/推广
bash tools/disable.sh ../work/batch3.txt   # 偏好类，先问机主！
```

- 禁用前用 `dumpsys package <名>` 确认用途；流媒体/语音助手/投屏/系统更新**先列清单问机主**再动。
- HyperOS 待机广告 = `com.xiaomi.mitv.hyper.screensaver`（普通 Activity 非 DreamService），禁用即断广告。
- 完成后 `pm list packages -d` 复核；重启一次对比，防固件回滚。

## 6. 定制（改 `launcher/.../values/config.xml` 后重新构建安装）

- `dock_apps`：底栏 4-7 个应用（注意改名重打包应用的真实包名，用 `query-activities` 反查）。
- `input_action` + `input_sources`：小米 CN 固件 HDMI 切换需 `com.xiaomi.mitv.tvplayer.EXTSRC_PLAY` + 端口号（HDMI1/2/3=23/24/25，AV=2；`dumpsys tv_input` 核实，幽灵端口用 `-` 屏蔽）。
- `app_banners`：无横幅应用的磁贴美化，`包名=banner_xxx`（drawable 放 `res/drawable-nodpi/`，`res/raw/keep.xml` 已有 `tools:keep="@drawable/banner_*"` 防 R8 删除）。PatchWall 风格做法：从 APK（`pm path` → `adb pull` → aapt badging 找 icon）提取官方图标，取图标底色平铺磁贴、图形去底放大居中。
- `prefer_vendor_settings`（见本仓库 fork 的 SystemSettings.java）：让"设置"按钮直达小米完整设置而非 AOSP 设置。
- 壁纸：推 `config.json` + `wallpapers/` 到 `/sdcard/Android/data/<启动器包名>/files/`，重启启动器生效；`ambient_glow:false` 可关掉聚焦染色。

## 7. 还原

先恢复并验证一个可用的桌面和设置入口，再用变更记录逐项还原本次停用的包与修改的设置。原来就停用的包维持原状态。若初始 HOME 是 FallbackHome，明确标注原值可能重现黑屏，并提供恢复可用原厂桌面的单独入口。重启后复核结果。

HyperOS 3 的历史机型参数见 [实测案例](references/xiaomi-s-pro-finch.md)。固件升级后重新采集状态并核对差异。
