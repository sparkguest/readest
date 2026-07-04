# Readest Android Build - Changes Summary

## 项目概览
Readest 是一个跨平台电子书阅读器，基于 **Next.js 16 + Tauri v2** 构建，使用 pnpm monorepo 管理。

---

## 主要修改内容

### 1. ZIP-with-TXT 格式支持
**文件:** `apps/readest-app/src/libs/document.ts`

- 在 `DocumentLoader.open()` 方法中新增 ZIP-with-TXT 处理逻辑（第 388-413 行）
- 从 ZIP 中提取所有 `.txt` 文件，用 8 个换行符连接，通过 `TxtToEpubConverter` 转换为 EPUB 后打开
- 使用递归调用 `DocumentLoader` 处理转换后的 EPUB

```typescript
const txtEntries = entries.filter(
  (entry) => /\.txt$/i.test(entry.filename) && !entry.directory,
);
if (txtEntries.length > 0) {
  const rawParts: Blob[] = [];
  const separator = new Blob(['\n'.repeat(8)]);
  for (const entry of txtEntries) {
    const blob = await loader.loadBlob(entry.filename, 'application/octet-stream');
    if (blob) {
      if (rawParts.length > 0) rawParts.push(separator);
      rawParts.push(blob as Blob);
    }
  }
  // ...转换为 EPUB 并打开
}
```

### 2. BookFormat 类型更新
**文件:** `apps/readest-app/src/types/book.ts`

```typescript
export type BookFormat =
  | 'EPUB' | 'PDF' | 'MOBI' | 'AZW' | 'AZW3'
  | 'CBZ' | 'FB2' | 'FBZ' | 'TXT' | 'MD'
  | 'ZIP';  // 新增
```

### 3. 文件格式注册
**文件:** `apps/readest-app/src/libs/document.ts`

```typescript
export const EXTS: Record<BookFormat, string> = {
  // ...原有格式
  TXT: 'txt',
  MD: 'md',
  ZIP: 'zip',  // 新增
};

export const MIMETYPES: Record<BookFormat, string[]> = {
  // ...原有格式
  TXT: ['text/plain'],
  MD: ['text/markdown', 'text/x-markdown'],
  ZIP: ['application/zip'],  // 新增
};
```

### 4. Android 文件关联（Intent Filters）
**文件:** `apps/readest-app/src-tauri/gen/android/app/src/main/AndroidManifest.xml`

为以下格式添加了 `<intent-filter>`：

| 格式 | MIME 类型 | pathPattern |
|------|-----------|-------------|
| EPUB | `application/epub+zip` | `.*\\.epub` |
| MOBI | `application/x-mobipocket-ebook` | `.*\\.mobi` |
| AZW | `application/vnd.amazon.ebook` | `.*\\.azw` |
| AZW3 | `application/vnd.amazon.mobi8-ebook` | `.*\\.azw3` |
| FB2 | `application/x-fictionbook+xml` | `.*\\.fb2` |
| CBZ | `application/vnd.comicbook+zip` | `.*\\.cbz` |
| PDF | `application/pdf` | `.*\\.pdf` |
| TXT | `text/plain` | `.*\\.txt` |
| ZIP | `application/zip` | `.*\\.zip` |
| ZIP | `application/x-zip-compressed` | `.*\\.zip` |

ZIP 格式的完整 intent-filter 配置（最终版）：

```xml
<!-- scheme 方式匹配 content:// URI -->
<intent-filter>
    <action android:name="android.intent.action.VIEW" />
    <category android:name="android.intent.category.DEFAULT" />
    <data android:scheme="content" />
    <data android:pathPattern=".*\\.zip" />
</intent-filter>

<!-- scheme 方式匹配 file:// URI -->
<intent-filter>
    <action android:name="android.intent.action.VIEW" />
    <category android:name="android.intent.category.DEFAULT" />
    <data android:scheme="file" />
    <data android:pathPattern=".*\\.zip" />
</intent-filter>

<!-- MIME type 方式匹配 -->
<intent-filter>
    <action android:name="android.intent.action.VIEW" />
    <action android:name="android.intent.action.SEND" />
    <action android:name="android.intent.action.SEND_MULTIPLE" />
    <category android:name="android.intent.category.DEFAULT" />
    <data android:mimeType="application/zip" />
</intent-filter>
<intent-filter>
    <action android:name="android.intent.action.VIEW" />
    <action android:name="android.intent.action.SEND" />
    <action android:name="android.intent.action.SEND_MULTIPLE" />
    <category android:name="android.intent.category.DEFAULT" />
    <data android:mimeType="application/x-zip-compressed" />
</intent-filter>
```

### 5. Android 构建配置
**文件:** `apps/readest-app/src-tauri/gen/android/app/build.gradle.kts`

```kotlin
missingDimensionStrategy("store", "foss")
```

### 6. BuildTask.kt 优化
**文件:** `apps/readest-app/src-tauri/gen/android/buildSrc/src/main/java/com/bilingify/readest/kotlin/BuildTask.kt`

跳过 Rust 重编译（当 `.so` 已存在时）并复制前端资源：

```kotlin
if (abiDir != null) {
    val jniLib = File(project.projectDir, "src/main/jniLibs/$abiDir/libreadestlib.so")
    if (jniLib.exists()) {
        logger.lifecycle("Skipping Rust build for $targetName, .so already exists")
        copyFrontendAssets()
        return
    }
}
```

前端资源复制逻辑：`out/` → `src/main/assets/`

### 7. Turso 扩展 crate 修复
**文件:** `patches/turso_ext/build.rs`

修复 `turso_ext` crate 在 Android 交叉编译时错误链接 `advapi32` 的问题：

```rust
// 修复前：错误地在 Android 上链接 Windows 库
// 修复后：正确判断目标平台
if cfg!(target_os = "windows") && !cfg!(target_os = "android") {
    println!("cargo:rustc-link-lib=advapi32");
}
```

**文件:** `Cargo.toml`（根目录）

```toml
[patch.crates-io]
turso_ext = { path = "patches/turso_ext" }
```

---

## 构建流程

### 环境变量
```powershell
$env:JAVA_HOME="D:\Program Files\Java\jdk-22"
$env:ANDROID_HOME="C:\Users\spark\AppData\Local\Android\Sdk"
```

### 构建命令
```bash
# 1. 构建 Next.js 前端（生成 out/ 目录）
pnpm build

# 2. 构建 Android APK
$env:Path="$env:JAVA_HOME\bin;$env:Path"
Set-Location src-tauri/gen/android
.\gradlew.bat assembleArm64Release --no-daemon

# 3. 签名 APK
& "C:\Users\spark\AppData\Local\Android\Sdk\build-tools\37.0.0\apksigner.bat" sign `
  --ks "C:\Users\spark\.android\debug.keystore" `
  --ks-pass pass:android `
  --ks-key-alias androiddebugkey `
  --out app-arm64-release-signed.apk `
  app-arm64-release-unsigned.apk
```

### 输出文件
- APK 路径: `src-tauri/gen/android/app/build/outputs/apk/arm64/release/app-arm64-release-signed.apk`
- 大小: ~71.8 MB

---

## 已知问题

### Next.js 导出阶段挂起
`pnpm build` 在 `output: 'export'` 模式下，"Exporting using 19 workers (0/4)" 阶段会挂起。

**临时解决方案:** 使用之前构建好的 `src/main/assets/` 目录，跳过前端重构建。`BuildTask.kt` 中的 `copyFrontendAssets()` 在 `out/` 不存在时会跳过复制。

### Files by Google 中 ZIP 文件显示问题
Files by Google 会内置处理 ZIP 文件（作为归档格式）。用户需要通过以下方式打开：
1. 三个点菜单 → "Open with" → 选择 Readest
2. 长按文件 → "Share" 或 "Open with"

---

## 测试验证

### 已验证的构建产物
- `app-arm64-release-signed.apk` - 已签名的 release APK
- `app-arm64-debug.apk` - 调试版本
- `Readest_0.11.12_x64_en-US.msi` - Windows 安装包

### 文件格式支持状态
| 格式 | 打开方式 | Android 关联 | 状态 |
|------|----------|--------------|------|
| EPUB | 直接打开 | ✅ | ✅ |
| PDF | 直接打开 | ✅ | ✅ |
| MOBI | 直接打开 | ✅ | ✅ |
| AZW/AZW3 | 直接打开 | ✅ | ✅ |
| CBZ | 直接打开 | ✅ | ✅ |
| FB2 | 直接打开 | ✅ | ✅ |
| TXT | 转换为 EPUB | ✅ | ✅ |
| ZIP (含 TXT) | 提取+转换 | ✅ | ✅ |
