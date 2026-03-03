# Sonic Agent scrcpy 升级文档与 SOP

本文用于指导后续将 `sonic-agent` 的 scrcpy 从旧版本升级到新版本（例如 `v3.3.4` 及以后），并避免出现“client/server 版本不一致”问题。

## 1. 升级目标

- 升级 `sonic-android-scrcpy.jar` 到目标版本（示例：`3.3.4`）。
- 同步更新 `sonic-agent` 中 scrcpy 启动参数与版本号。
- 重新构建 `sonic-agent-<platform>.jar` 并完成部署验证。

## 2. 核心原理

`sonic-agent` 启动 scrcpy 的命令在：

- `src/main/java/org/cloud/sonic/agent/tests/android/scrcpy/ScrcpyLocalThread.java`

关键机制：

1. agent 会把宿主机插件 `plugins/sonic-android-scrcpy.jar` 推送到设备：`/data/local/tmp/sonic-android-scrcpy.jar`
2. 然后执行：`app_process ... com.genymobile.scrcpy.Server <clientVersion> ...`
3. scrcpy server 会校验 `<clientVersion>` 与自身 `BuildConfig.VERSION_NAME`

所以必须保证：

- 启动参数里的版本（client）
- 插件 jar 内的 server 版本

两者完全一致。

## 3. 标准升级流程（SOP）

以下示例以升级到 `v3.3.4` 为例。

### 步骤 A：升级并构建 sonic-android-scrcpy 插件

1. 拉取仓库（首次）：

```bash
git clone https://github.com/SonicCloudOrg/sonic-android-scrcpy.git
cd sonic-android-scrcpy
```

2. 添加上游（首次）：

```bash
git remote add upstream https://github.com/Genymobile/scrcpy.git
git fetch upstream --tags
```

3. 切到目标版本 tag：

```bash
git checkout -b sonic-upgrade-v3.3.4 v3.3.4
```

4. 构建 server 产物：

```bash
./gradlew :server:assembleRelease
```

5. 产物路径（APK，但可直接作为 jar 使用）：

- `server/build/outputs/apk/release/server-release-unsigned.apk`

6. 替换 agent 插件文件：

```bash
cp server/build/outputs/apk/release/server-release-unsigned.apk <sonic-agent>/plugins/sonic-android-scrcpy.jar
```

### 步骤 B：修改 sonic-agent scrcpy 启动参数

编辑：

- `src/main/java/org/cloud/sonic/agent/tests/android/scrcpy/ScrcpyLocalThread.java`

将版本号改为目标版本（示例 `3.3.4`），并保持参数与当前数据流协议一致。当前推荐参数（仅视频流）如下：

```text
CLASSPATH=/data/local/tmp/sonic-android-scrcpy.jar app_process / com.genymobile.scrcpy.Server 3.3.4 log_level=info video=true audio=false max_size=0 max_fps=60 tunnel_forward=true raw_stream=true control=false show_touches=false stay_awake=false power_off_on_close=false clipboard_autosync=false
```

说明：

- `raw_stream=true`：关闭 meta/dummy/codec 头，更适配当前 agent 侧裸流读取方式。
- `audio=false`：当前链路仅视频，避免引入额外兼容问题。

### 步骤 C：构建 sonic-agent

macOS arm64 示例：

```bash
mvn -Dplatform=macosx-arm64 -DskipTests package
```

产物：

- `target/sonic-agent-macosx-arm64.jar`

## 4. 部署 SOP（非常重要）

部署时必须同时替换两个文件：

1. `sonic-agent-<platform>.jar`
2. `plugins/sonic-android-scrcpy.jar`

如果只替换了 agent jar，未替换 plugin jar，会出现：

- `The server version (X) does not match the client (Y)`

## 5. 验证清单（发布前必做）

### 5.1 宿主机插件版本校验

```bash
aapt dump badging plugins/sonic-android-scrcpy.jar | grep -E "versionName|versionCode"
```

期望：`versionName='3.3.4'`。

### 5.2 设备端缓存清理（防止残留）

```bash
adb -s <udid> shell rm -f /data/local/tmp/sonic-android-scrcpy.jar
```

### 5.3 运行时回读校验（出现异常时）

```bash
adb -s <udid> pull /data/local/tmp/sonic-android-scrcpy.jar /tmp/sonic-android-scrcpy.jar
aapt dump badging /tmp/sonic-android-scrcpy.jar | grep versionName
```

如果这里不是目标版本，说明运行目录或部署文件用错。

### 5.4 日志关键字

重点观察：

- `The server version (...) does not match the client (...)`
- `scrcpy服务启动失败`
- `scrcpy video socket closed`

## 6. 常见问题与处理

### Q1：日志显示 client=3.3.4，但 server=2.4

根因：运行时加载到的 `plugins/sonic-android-scrcpy.jar` 仍是旧版。

处理：

1. 确认当前运行目录
2. 重替换 plugin + agent 两个文件
3. 清理设备端 `/data/local/tmp/sonic-android-scrcpy.jar`
4. 重启 agent 后复测

### Q2：设备偶发 `device offline` / `not found`

通常是设备上线瞬时状态，不等价于 scrcpy 升级失败。优先看 scrcpy 启动报错行判断是否版本问题。

## 7. 版本升级变更模板

每次升级建议记录以下信息：

- 目标版本：`vX.Y.Z`
- `sonic-android-scrcpy` 来源：`tag/commit`
- `ScrcpyLocalThread` 参数改动点
- 产物校验结果（host plugin + device plugin）
- 回归结果（Android 主流版本：12/13/14/15/16）

---

维护建议：将本文纳入每次 release checklist，避免再次出现“只替换 agent jar，漏替换 plugin jar”的问题。
