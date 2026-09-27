# AgentOS Demo Apps

这个目录是一个独立的 Android Gradle 根目录，用来演示 AgentOS 的三个普通 App 能力：

- `meeting-records`：会议纪要记录，支持本地增删改查；同时保留当前 Agenriod Host 的迁移期 Plugin endpoint，并新增 AOSP bootstrap protocol v3 endpoint；打开 App 后还会暴露一个 loopback Streamable HTTP MCP server。
- `calendar`：本地日程编辑器，支持创建、编辑、删除日程，并用日期/时间选择器填写表单。
- `alarm`：本地闹钟，支持设置、启用/停用和删除一次性闹钟；触发时通过 Android 通知提醒。

这个 Gradle 根目录没有被加入主工程的 `settings.gradle.kts`。因此它不会改变主工程的模块图，也不会参与主工程默认构建。`plugin-api` 只读复用 `../plugins/api` 的迁移期契约源码，没有复制或修改主工程文件。记录 App 额外内置了 AOSP bootstrap 所需的两份最小 AIDL，当前 overlay 尚未接入可调用的 tool/resource 数据面，因此它只负责 protocol v3 握手和 descriptor 声明。

## 构建

在 `agenriod/` 仓库根目录执行（Android Gradle 命令统一经过工具脚本）：

```bash
./tools/android/android.sh gradle -p demo-apps :meeting-records:assembleDebug
./tools/android/android.sh gradle -p demo-apps :calendar:assembleDebug
./tools/android/android.sh gradle -p demo-apps :alarm:assembleDebug
```

或一次构建三个 APK：

```bash
./tools/android/android.sh gradle -p demo-apps :meeting-records:assembleDebug :calendar:assembleDebug :alarm:assembleDebug
```

APK 输出在各模块的 `build/outputs/apk/debug/`。记录 App 安装在运行中的 AgentOS 主 App 同一设备后，打开记录 App 即可激活 `meeting-records` Plugin；没有 AgentOS Host 时，记录编辑功能仍可单独使用，Plugin 注册会自动等待 Host。

要把三个 Demo 一起打进 AOSP 产品镜像，在仓库根目录执行：

```bash
python3 tools/aosp/wire-platform.py /path/to/aosp --apply \
  --demo-alarm-apk demo-apps/alarm/build/outputs/apk/debug/alarm-debug.apk \
  --demo-calendar-apk demo-apps/calendar/build/outputs/apk/debug/calendar-debug.apk \
  --demo-meeting-records-apk demo-apps/meeting-records/build/outputs/apk/debug/meeting-records-debug.apk
```

这三个 APK 会使用 AOSP `platform` 证书重新签名并安装到 product 分区；
`meeting-records` 的 system endpoint 会被 `AgentManagerService` 按包名发现。

从 APK 构建、AOSP 接线、Cuttlefish 镜像打包到设备烟测的完整顺序见
[`docs/aosp-image-demo-quickstart.md`](../docs/aosp-image-demo-quickstart.md)。
当前 Demo 烟测覆盖 APK 存在、Plugin 发现/启用和三个 Activity 的基本操作；
完整 tool/resource 调用仍不在 AOSP bootstrap 的验收范围内。

## 演示边界

这些 App 使用应用私有 JSON 文件保存数据，目的是验证跨 App 能力路径和最小用户流程，不是生产级日历或提醒服务。记录 App 的 MCP 服务只绑定 `127.0.0.1`、拒绝带 `Origin` 的请求，不把认证凭据写入 Plugin descriptor；端口和会话都由进程运行时生成，没有写入仓库。

当前 Agenriod Host 仍使用迁移期同步 `describe()/invoke()` endpoint；AOSP endpoint 只实现最新 bootstrap 的 protocol v3 descriptor 握手。上游的 `beginInvoke`、lease、cancel 和 sink 数据面还未在 AOSP overlay 中落地，因此这部分用于验证发现/握手兼容，不宣称已经完成系统级 Plugin 调用。
