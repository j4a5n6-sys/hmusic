# GitHub Actions 自动构建说明

本仓库已配置 GitHub Actions 自动构建 Android APK。

## 触发方式

### 1. 手动触发（推荐首次使用）

1. 推送代码到 GitHub 后，进入仓库的 **Actions** 标签页
2. 选左侧 **构建 Android APK**
3. 点右上角 **Run workflow**，选择构建模式：
   - `universal`：仅构建通用版（约 30–50 MB，兼容所有设备）
   - `split-per-abi`：仅构建分架构版（每个架构独立 APK，体积更小）
   - `both`：两者都构建（默认）
4. 点 **Run workflow** 开始

### 2. Tag 自动触发（用于发版）

```bash
git tag v3.0.1
git push origin v3.0.1
```

推送 tag 后会自动构建 + 创建 GitHub Release，APK 直接挂在 Release 页。

## 产物下载

构建完成后：

- **artifact 模式**（手动触发）：在 Actions 运行详情页底部 → **Artifacts** 区域下载
- **Release 模式**（tag 触发）：在仓库 Releases 页直接下载

每次构建包含：
- `HMusic-vX.X.X-android-universal.apk`：通用版
- `HMusic-vX.X.X-android-arm64-v8a.apk`：现代手机（推荐）
- `HMusic-vX.X.X-android-armeabi-v7a.apk`：旧设备
- `HMusic-vX.X.X-android-x86_64.apk`：模拟器
- `checksums.txt`：SHA-256 校验和
- `symbols/`：调试符号（崩溃分析用）

## 配置签名（可选但强烈推荐）

**首次跑可以跳过签名**——CI 会输出 debug 签名的 APK，能装能用，但每次构建签名不同，**升级时会要求卸载旧版**。

要让构建产物使用稳定的发布签名（升级体验顺畅），需要在 GitHub 配 4 个 Secret：

### 步骤 1：生成 keystore（仅首次需要）

在本机 / 任何有 JDK 的机器上：

```bash
keytool -genkey -v -keystore hmusic-release.jks \
  -keyalg RSA -keysize 2048 -validity 10000 \
  -alias hmusic
```

按提示填密码、姓名等。**妥善保管这个 jks 文件和密码**——丢失后无法升级线上版本，必须卸载重装。

### 步骤 2：转 base64

```bash
# Linux / macOS / Git Bash
base64 -w 0 hmusic-release.jks > keystore.base64.txt

# Windows PowerShell
[Convert]::ToBase64String([IO.File]::ReadAllBytes("hmusic-release.jks")) | Out-File -NoNewline keystore.base64.txt
```

### 步骤 3：在 GitHub 配 Secrets

仓库 → **Settings** → **Secrets and variables** → **Actions** → **New repository secret**

| Name | Value |
|------|-------|
| `ANDROID_KEYSTORE_BASE64` | `keystore.base64.txt` 文件全部内容 |
| `ANDROID_KEY_ALIAS` | 创建 keystore 时填的 alias，本例为 `hmusic` |
| `ANDROID_STORE_PASSWORD` | keystore 密码 |
| `ANDROID_KEY_PASSWORD` | key 密码（如果跟 store 密码相同就填同一个） |

下次构建时 workflow 会自动检测这些 Secrets 并签名 APK。日志里看到 `🔐 解码 keystore...` 和 `✅ 已写入 android/key.properties` 即说明签名生效。

## 故障排查

### Q: 构建失败提示 `Flutter SDK 版本不匹配`
A: `pubspec.yaml` 要求 `sdk: '>=3.7.2 <4.0.0'`。修改 `.github/workflows/build.yml` 顶部的 `FLUTTER_VERSION` 到匹配版本（默认 3.27.4 已兼容）。

### Q: 构建失败提示 `Gradle 内存不足`
A: 在 `android/gradle.properties` 加 `org.gradle.jvmargs=-Xmx4G -XX:MaxMetaspaceSize=1G`。

### Q: 想跑 `flutter analyze` 严格检查
A: 把 workflow 里 `flutter analyze lib/ || true` 的 `|| true` 删掉，分析失败就阻断构建。

### Q: 装的 APK 一打开就闪退
A: 八成是混淆把第三方库的关键类剔除了。看 `android/app/proguard-rules.pro` 是否漏了某些 `-keep` 规则。

## 本次修复对应的验证

CI 跑出的新 APK 装到手机后，播一首歌，预期日志：

```
📱 小爱音箱 (LX04) - 192.168.x.x        ← IP 解析成功（device.ip 字段名修复生效）
✅ [MiIoT] 设备IP与手机IP同网段，使用本地代理转发  ← 走快速通道
🔗 [ProxyServer] 代理请求 #1: ...        ← 音箱真访问了代理（关键证据）
✅ [MiIoT] 播放成功
```

如果手机和音箱不在同网段，则会看到：

```
⚠️ [MiIoT] 设备IP与手机IP不同网段，跳过本地代理（将尝试公共代理）
🔄 [MiIoT] 本地代理不可用，使用公共代理转发
✅ [MiIoT] 播放成功
```
