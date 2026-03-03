# scrcpy 升级最简执行版（Checklist）

适用场景：快速把 sonic-agent 的 scrcpy 升级到指定版本（示例：`3.3.4`）。

1. 在 `sonic-android-scrcpy` 拉取上游 tag 并切换到目标版本。

```bash
git remote add upstream https://github.com/Genymobile/scrcpy.git
git fetch upstream --tags
git checkout -b sonic-upgrade-v3.3.4 v3.3.4
```

2. 构建 server 插件。

```bash
./gradlew :server:assembleRelease
```

3. 用构建产物替换 agent 插件。

```bash
cp server/build/outputs/apk/release/server-release-unsigned.apk <sonic-agent>/plugins/sonic-android-scrcpy.jar
```

4. 修改 `src/main/java/org/cloud/sonic/agent/tests/android/scrcpy/ScrcpyLocalThread.java`：

- `Server <version>` 改为目标版本（示例：`3.3.4`）
- 推荐参数保留：`video=true audio=false raw_stream=true`

5. 构建 agent（macOS arm64 示例）。

```bash
mvn -Dplatform=macosx-arm64 -DskipTests package
```

6. 部署时同时替换两个文件（缺一不可）：

- `target/sonic-agent-macosx-arm64.jar`
- `plugins/sonic-android-scrcpy.jar`

7. 发布前校验宿主机插件版本。

```bash
aapt dump badging plugins/sonic-android-scrcpy.jar | grep -E "versionName|versionCode"
```

8. 清理设备端旧缓存后再测。

```bash
adb -s <udid> shell rm -f /data/local/tmp/sonic-android-scrcpy.jar
```

9. 如报版本不一致，回读设备端实际文件。

```bash
adb -s <udid> pull /data/local/tmp/sonic-android-scrcpy.jar /tmp/sonic-android-scrcpy.jar
aapt dump badging /tmp/sonic-android-scrcpy.jar | grep versionName
```

10. 若看到 `server version (X) != client (Y)`：

- 先确认运行目录是否正确
- 再确认 plugin 和 agent 是否都替换到同一部署目录
