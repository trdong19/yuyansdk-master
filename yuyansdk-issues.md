---
name: yuyansdk-issues
description: 雨燕输入法开发过程中遇到的问题和解决方案
metadata: 
  node_type: memory
  type: project
  originSessionId: 8bcc436f-f3fd-44a8-a7b5-9cde0f925bf2
---

## 项目结构

- `yuyansdk-master/` — SDK 库模块（修改源码）
- `YuyanIme-master/` — 完整 App 项目（包含 yuyansdk + app 模块，生成 APK）
- GitHub: `trdong19/yuyansdk-master`（库）、`trdong19/YuyanIme-master`（App）

## 遇到的问题和解决方案

### 1. GitHub Actions 构建失败：找不到 Android Gradle Plugin
**原因**: `settings.gradle` 缺少 `pluginManagement` 配置
**解决**: 添加 `pluginManagement` + `dependencyResolutionManagement`，指定 AGP 8.7.3、Kotlin 2.0.0 等版本

### 2. 构建失败：AndroidX 未启用
**原因**: 缺少 `gradle.properties`，没有 `android.useAndroidX=true`
**解决**: 创建 `gradle.properties`，设置 `android.useAndroidX=true`

### 3. App 构建失败：keystore 路径为 null
**原因**: `app/build.gradle` 的 signing config 在 keystore 不存在时直接 `rootProject.file(null)` 报错
**解决**: 加 `if (keystoreFilePath)` 判断；debug 构建移除 `signingConfig signingConfigs.release`

### 4. 剪贴板导入报错"压缩包中未找到词库文件"
**原因**: 用户的 zip 是文件夹内容直接压缩（没有外层 `*.userdb/` 文件夹），代码只检查一级子目录
**解决**: `importDictionary()` 改用 `walkTopDown()` 递归搜索，额外检测 `CURRENT` 文件兼容直接压缩格式

### 5. 打不了字 / build 目录为空
**原因**: `build/` 目录存放 Rime 的 schema、prism.bin、table.bin，缺失则 Rime 无法初始化
**根因 A**: `.gitignore` 里的 `build/` 规则把 `yuyansdk/src/main/assets/rime/build/` 也忽略了
**根因 B**: `Launcher.kt` 只检查 `dataDictVersion` 版本号，不检查文件是否存在
**解决 A**: `.gitignore` 改为 `/build/`（只忽略根目录），用 `git add -f` 强制添加 build 文件
**解决 B**: `Launcher.kt` 增加 `buildDir.listFiles().isNullOrEmpty()` 检查，空目录强制重新复制

### 6. yuyansdk 模块 assets 未合并到 APK
**原因**: Android 多模块项目中库模块 assets 默认应自动合并，但被 `.gitignore` 的 `build/` 规则排除
**解决**: 修复 `.gitignore`（同问题 5）

### 7. 异步处理后 Executor 崩溃：RejectedExecutionException
**原因**: `RimeEngine.destroy()` 调用 `rimeExecutor.shutdownNow()`，但 `resetIme()` 会重新初始化，executor 没有重建
**解决**: 添加 `ensureExecutor()` 方法，每次 submit 前检查 `rimeExecutor.isShutdown`，关闭了就重建

### 8. 异步处理后候选词不显示
**原因**: `RimeEngine` 在后台线程调用 `onCandidatesReady?.invoke()`，callback 中的 `candidatesLiveData.value = ...` 必须在主线程
**解决**: InputView 的 callback 改为 `handler.post { updateCandidate() }`，确保 LiveData 更新在主线程

## 关键注意事项

- **Rime 引擎**: 完全基于 Rime 1.11.2，通过 JNI 调用，userdb 格式为 LevelDB
- **九键 userdb 名**: `pinyin.userdb`（取自 `translator.dictionary` 字段，不是 `schema_id`）
- **线程安全**: Rime JNI 调用不线程安全，异步处理必须用单线程 Executor
- **LiveData**: `setValue()` 必须主线程，`postValue()` 可以在任意线程
- **assets 合并**: 库模块 assets 会合并到 APK，但被 `.gitignore` 影响
- **ADB 路径**: Git Bash 下 `adb push` 路径会被错误转换，需要 `MSYS_NO_PATHCONV=1`
