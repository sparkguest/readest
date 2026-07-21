# 打包构建配置

> **用途**: 快速参考打包命令与关键配置
> **生成日期**: 2026-07-18

---

## 环境要求

| 工具 | 版本/路径 |
|------|----------|
| Rust | 1.77.2+ (edition 2021) |
| NDK | 30 (Android) |
| JDK | 22 (`D:\Program Files\Java\jdk-22`) |
| Android SDK | `C:\Users\spark\AppData\Local\Android\Sdk` |
| Android SDK API | 36 (compileSdk/targetSdk)，minSdk 26 |
| Gradle | 通过 gradlew wrapper |
| pnpm | monorepo root |

## 关键配置

### Rust 编译优化 (`Cargo.toml` workspace root)

```toml
[profile.release]
lto = "thin"
strip = true
```

- `lto = "thin"` — 跨 crate 链接时优化，比 `"fat"` 快且体积接近
- `strip = true` — 去除调试符号

### custom-protocol feature (src-tauri Cargo.toml)

```toml
tauri = { version = "2", features = ["custom-protocol", "protocol-asset"] }
```

> **必须**：Android 也需要 `custom-protocol`。缺少此 feature 时 `generate_context!()` 宏设 `dev=true` 走 `devUrl` 启动失败。PC/Android/Mac 一致启用，不可为省体积按平台条件移除。

### tauri.conf.json

| 字段 | 值 |
|------|-----|
| frontendDist | `../out` |
| devUrl | `http://localhost:3000` |
| beforeBuildCommand | `pnpm build` |
| beforeDevCommand | `pnpm dev` |
| identifier | `com.bilingify.readest` |

### Android override (`tauri.android.override.json`)

```json
{"build":{"beforeBuildCommand":""}}
```

跳过 Android 构建时的前端构建（手动 `pnpm build` 预构建）。

### Android build.gradle.kts

| 字段 | 值 |
|------|-----|
| compileSdk | 36 |
| minSdk | 26 |
| targetSdk | 36 |
| missingDimensionStrategy | `"store", "foss"`（解决 tauri-plugin-native-bridge 的 store flavor 维度） |
| storeFlavor（project property） | `foss`（默认） |

### BuildTask.kt 跳过逻辑

当 `jniLibs/arm64-v8a/libreadestlib.so` 已存在时跳过 Rust 重编译，并复制前端资源 (`out/` → `src/main/assets/`)。需手动注入此修改。

### Turso 扩展 patch

`Cargo.toml` workspace root:
```toml
[patch.crates-io]
turso_ext = { path = "patches/turso_ext" }
```

`patches/turso_ext/build.rs` 修复 Android 交叉编译时错误链接 `advapi32`:
```rust
if cfg!(target_os = "windows") && !cfg!(target_os = "android") {
    println!("cargo:rustc-link-lib=advapi32");
}
```

### Next.js build 临时绕过

`next.config.mjs`:
```js
typescript: { ignoreBuildErrors: true }
```

`pnpm build` 在 `output: 'export'` 模式 "Collecting page data" 阶段会间歇性挂起。改用 `node_modules\.bin\next build` 可绕过。或者跳过前端构建直接复用已有 assets。

## 构建命令

### Windows exe（Debug）

```powershell
cd apps/readest-app

# 1. 构建前端
pnpm build

# 2. Tauri 构建 exe
pnpm tauri build --debug
# 输出: src-tauri/target/debug/Readest.exe
```

### Windows exe（Release）

```powershell
cd apps/readest-app
pnpm build
pnpm tauri build
# 输出: target/release/Readest.exe（~59.59 MB）
```

### Android APK（arm64, Debug）

```powershell
# 1. 设置环境变量
$env:JAVA_HOME="D:\Program Files\Java\jdk-22"
$env:ANDROID_HOME="C:\Users\spark\AppData\Local\Android\Sdk"
$env:CC_aarch64_linux_android="C:\Users\spark\AppData\Local\Android\Sdk\ndk\30.0...\toolchains\llvm\prebuilt\windows-x86_64\bin\clang"
$env:AR_aarch64_linux_android="C:\Users\spark\AppData\Local\Android\Sdk\ndk\30.0...\toolchains\llvm\prebuilt\windows-x86_64\bin\llvm-ar"
$env:CARGO_TARGET_AARCH64_LINUX_ANDROID_LINKER="$env:CC_aarch64_linux_android"

# 2. 构建前端
cd apps/readest-app
pnpm build

# 3. 编译 Rust .so（单独编译）
cargo build --release --lib --target aarch64-linux-android -p Readest

# 4. 复制 .so 到 jniLibs
Copy-Item -Path "target/aarch64-linux-android/release/libreadestlib.so" `
  -Destination "apps/readest-app/src-tauri/gen/android/app/src/main/jniLibs/arm64-v8a/libreadestlib.so"

# 5. 构建 APK
$env:Path="$env:JAVA_HOME\bin;$env:Path"
cd apps/readest-app/src-tauri/gen/android
.\gradlew.bat assembleArm64Debug --no-daemon

# 输出: app/build/outputs/apk/arm64/debug/app-arm64-debug.apk（~70 MB）
```

### Android APK（arm64, Release + 签名）

```powershell
# 前 4 步同上...
# 5. 如果使用 keystore.properties 有签名配置则直接:
.\gradlew.bat assembleArm64Release --no-daemon
# 输出: app/build/outputs/apk/arm64/release/app-arm64-release-unsigned.apk

# 6. 手动签名
& "C:\Users\spark\AppData\Local\Android\Sdk\build-tools\37.0.0\apksigner.bat" sign `
  --ks "C:\Users\spark\.android\debug.keystore" `
  --ks-pass pass:android `
  --ks-key-alias androiddebugkey `
  --out app-arm64-release-signed.apk `
  app-arm64-release-unsigned.apk
```

## 产物体积参考

| 产物 | 大小 | 说明 |
|------|------|------|
| Windows exe (Release) | 59.59 MB | LTO=thin + strip |
| libreadestlib.so (arm64) | 48.33 MB | Release，含 custom-protocol |
| app-arm64-debug.apk | ~70 MB | debug 构建含更多符号 |

> 对比：无 LTO+strip 的 exe 为 90.73 MB。官方 exe 约 63.9 MB。

## 常见问题

### Next.js 构建挂起
- `pnpm build` 在 "Collecting page data using 19 workers" 阶段可能挂起
- 绕过：kill 进程后重试，或用本地 `node_modules\.bin\next build`
- 或完全跳过前端构建（使用 `tauri.android.override.json` + 已有 assets）

### Windows symlink 权限
- Android 构建会在 `gen/android/app/src/main/jniLibs/arm64-v8a/` 创建 so symlink
- Windows 需管理员权限或禁用 developer mode
- 替代方案：`Copy-Item` 手动复制 .so

### Android 构建缺少 .so
- 确保 `cargo build --release --lib --target aarch64-linux-android` 成功
- 确保 .so 已复制到 `jniLibs/arm64-v8a/`
- 确保 BuildTask.kt 跳过逻辑不会跳过第一次构建（.so 不存在时正常走 Rust 构建）
