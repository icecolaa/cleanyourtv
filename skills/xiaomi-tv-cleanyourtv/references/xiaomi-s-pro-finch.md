# 实测案例：小米电视 S Pro Mini LED 65（2026-10-26/27 部署）

机型：L65MB-SP，实际型号 **MiTV-MFFU0**（代号 finch，MediaTek mt5877，4GB RAM）
固件：OS3.0.115.0.UFFMATV（HyperOS 3，Android 14 / SDK 34，中国版）
与 cleanyourtv 仓库 `examples/xiaomi-mitv-mffu0/`（S Pro 100）同代号同固件，清单直接套用。

> 2026-11-12 更新：定制已 rebase 到上游 v3.4.0（长按 OK 排序/5 套新主题/442 机型适配），fork `icecolaa/cleanyourtv` main=`fbf41d5c`；v3.4 定制 APK（versionCode 13）已构建，`git diff v3.4.0 HEAD --stat` 验证仅定制 6 文件。装机验证待电视在线后进行——下方 TV 端实测数据仍来自 v3.0 定制（原 commit `a2455fb5`）。

## 历史固件范围

以下结果来自 HyperOS 3 单机实测。其他固件请按 SKILL.md 重新核对。

## 已禁用的 13+1 个包

遥测/云控：`com.miui.analytics`、`com.miui.tv.analytics`（~100MB 常驻）、`com.miui.misightservice`、`com.xiaomi.tvqs`（~86MB）、`com.mitv.cloudcontrol`
广告/购物/推广：`com.duokan.videodaily`（含广告 SDK）、`com.mitv.shoplugin`、`com.xiaomi.mitv.shop`、`com.xm.webcontent`、`com.xiaomi.tv.gallery`
偏好类（机主确认）：`com.xiaomi.aitranslate`（~117MB）、`com.xiaomi.mitv.health`、`com.xiaomi.gamecenter.sdk.service.mibox`
待机广告：`com.xiaomi.mitv.hyper.screensaver`（HyperOS 屏保本体是普通 Activity，由 `system screen_saver_timeout=480000` 触发；禁用后待机直接息屏，无广告）

**保留**：小爱语音（voicecontrol/wakeupservice）、投屏（miplay/smartshare/mi_connect/hyper_connect）、米家、OTA、应用商店、`com.mitv.alarmcenter`（本机不存在未处理）、**电视管家 `com.xiaomi.mitv.tvmanager`**（小爱"打开应用"走它的 aicr provider，禁用后语音开应用会假应答）、`com.xiaomi.mitv.smartpanel`（菜单键面板）、搜狗输入法、`mitv.service`、全部 `com.mediatek.*` 电视服务。

**禁绝触碰**：`com.android.*`/`com.google.android.*` 核心、`com.xiaomi.mitv.settings`（唯一完整设置 UI）、`com.xiaomi.mitv.tvplayer`（信号源）、`com.mi.bluetooth`、`com.mitv.codec.update`、`com.xiaomi.mitv.mediaexplorer`。

## 屏保真相（三轮实测）

该固件**不执行标准 DreamService 管道**：`secure screensaver_enabled/components/default_component/timeout` 全配好、空闲超时后仍无 dream 窗口；`am startservice -a android.service.dreams.DreamService` 报需要 BIND_DREAM_SERVICE 权限。结论：待机 = 无广告直接息屏；想看航拍需手动打开 aerial-tv 应用（打开后 10 秒空闲自动全屏连播 Apple 航拍，HOME 退出）。

## HDMI / 信号源

`dumpsys tv_input` 实测：HDMI1=HW5、HDMI2=HW6、HDMI3=HW7、HW8 为幽灵端口（无物理口）、AV=HW2、DTMB=HW0、ATV=HW1。
`config.xml` 配置：

```xml
<string name="input_action">com.xiaomi.mitv.tvplayer.EXTSRC_PLAY</string>
<string-array name="input_sources"><item>HW5=23</item><item>HW6=24</item><item>HW7=25</item><item>HW8=-</item><item>HW2=2</item></string-array>
```

## 底栏与磁贴

- 底栏：影视仓（TVBox 改名版，真实包名 `com.huawei.himovceie`）、云视听小电视 `com.xiaodianshi.tv.yst`、网易云音乐 `com.netease.cloudmusic.tv`、芒果TV `com.hunantv.license`（也是改名版）。**改名应用用 `pm path` 查不到原包名，要用 `cmd package query-activities -c LEANBACK_LAUNCHER` 反查**。
- 自定义横幅（app_banners，PatchWall 风格）：`banner_netease_music`（亮红 #DD001B+白logo）、`banner_qqmusic`（白底+黄绿音符）、`banner_changba`（珊瑚 #FF5046+白"唱"）、`banner_quark`（蓝 #0A90DD+白云，云取蓝色像素反白）。图标来源：`pm path` → `adb pull base.apk` → `aapt dump badging` 找 application-icon 最大密度 → 解压提取。
- 设置直达：`SystemSettings.java` 加 `prefer_vendor_settings` 开关（config.xml 中 true），单击"设置"直达 `com.xiaomi.mitv.settings/.entry.MainActivity`（AOSP 设置无"账号与安全"页，ADB 开关在里面找不到）。
- 壁纸：aerial 全屏帧（玛琳湖星空）经 OpenCV inpaint 抹掉烙印的时钟/字幕，作为唯一壁纸推到 `/sdcard/Android/data/com.thunderstone.idun.dev/files/wallpapers/`；config.json：`{"wallpaper":true,"wallpaper_rotation_minutes":60,"wallpaper_true_color":true,"ambient_glow":false}`。

## 网络/连接坑

- 固件禁 ICMP（ping 不通属正常）；ADB 关闭时 connect 报 refused，待机时 timeout。
- `ro.adb.secure=0`：ADB 开着= 局域网任何设备可免授权 shell，用完务必关。
- Git Bash 下 adb 的 `/sdcard` 路径要加 `MSYS_NO_PATHCONV=1`。
- Gradle 官方源被断连（SSL reset），用腾讯镜像。
- 上游大版本 fetch 走 SSH 30 秒内卡死属常见：`git remote set-url origin https://github.com/Gathin23/cleanyourtv.git` 后 HTTPS fetch（86MB 约 40 分钟）。大 push 期间并发 `git ls-remote` 偶报 "Could not read from remote repository"，push 本身不受影响，结束后再验证哈希。

## 结果

- 可用内存 1.57GB → 2.15GB
- Home 三重验证：resolve-activity、mFocusedApp、截图全指向新启动器；重启后开机直达
- 全部变更可逆：`restore-all.sh`（restore.sh + 清单 + com.mitv.tvhome）
