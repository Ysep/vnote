# Code Change Log

## 2026-08-22 同步章节编号补丁（51e61c5e）到 master

### 背景
将 `v4.5.0_zyx` 分支的 `51e61c5e`
（feat(pack): restore markdown section numbering at pack time via patch）
同步到 `master`，使主分支打包时同样恢复被移除的 Markdown 章节编号功能。

### 发现的问题
补丁 `package/restore-section-numbering.patch` 基于旧代码生成，无法应用到当前
`master`：`6c8ace5e`（feat(outline)）重写了
`MarkdownEditor::updateHeadings`，从 `ElementRegion` + `matchHeader()`
改为 cmark AST 的 `HeadingInfo`，导致补丁中 `markdowneditor.cpp/h` 的
hunk 上下文不匹配，CI 的 `git apply --check` 会直接失败。

### 变更
- cherry-pick `51e61c5e` → `b6970367`（3 个 CI yml + `pack-win.ps1` + 补丁文件）
- 应用补丁中其余 17 个兼容文件（controller、config、global.h、
  markdownwebglobaloptions、htmltemplateservice、web 资源、
  markdownviewwindow2、outlineprovider、测试等）
- 将章节编号功能手动移植到当前 `markdowneditor.cpp/h`：
  - `Heading` 结构增加 `m_sectionNumber` 字段与构造参数
  - `init()` 增加 `m_sectionNumberTimer`（1s 单次定时，留出撤销时间）
  - `updateHeadings` 增加章节编号开关逻辑（读模式关闭、编辑模式开启、
    支持 `overrideSectionNumber` 强制开关、On→Off 时清理）
  - 新增 `updateHeadingSectionNumber` / `updateSectionNumber` /
    `overrideSectionNumber` 三个函数
- 适配点：新版 `HeadingInfo` 无 `match.m_sequence`，改为在循环内用
  `vte::MarkdownUtils::matchHeader(block.text())` 解析已有序列号，
  行为与旧版一致（仅在编号漂移时重写）
- 重新生成 `package/restore-section-numbering.patch`（996 → 1001 行）

### 验证
- lint：全部改动文件 0 错误
- 补丁双向验证通过：
  - 干净 master 源码上 `git apply --check` 成功（临时 worktree）
  - 已应用工作树上 `git apply --reverse --check` 成功

---

## 2026-08-22 修复 Windows CI 安装 Qt 失败（cache service 400 → python 254）

### 现象
`ci-win.yml` 的 `Install Qt Official Build` 步骤失败：
- `Warning: Failed to restore: Cache service responded with 400`
- `Error: ... python.exe failed with exit code 254`

该步骤失败导致 build 目录从未创建，后续诊断步骤
`Re-run failed tests`（`if: failure()`）在空目录 `D:\a\vnote\build`
启动 `cmd.EXE` 时报 "The directory name is invalid"（次生误导性错误）。

### 根因
`jurplel/install-qt-action@v3` 的 `cache: 'true'` 使用旧版
`@actions/cache` 实现，在 Windows runner 上 restore 时被 GitHub 缓存
服务拒绝（400），随后直接中止安装（python 254）。
而工作流第 71-76 行已有独立的 `Cache Qt` 步骤
（`actions/cache@v4`，缓存 `${{runner.workspace}}/Qt`），
action 内部缓存是**重复且损坏**的第二套缓存。

### 变更
- `install-qt-action` 的 `cache: 'true'` → `'false'`：
  关闭内部缓存，改用工作流自带的 `Cache Qt` 步骤（actions/cache@v4）
- Qt6 matrix 的 `qt_tools: tools_opensslv3_x64` → `""`：
  Qt 6 使用 Schannel 做 TLS，不需要捆绑 OpenSSL；
  减少一次 aqt 下载和一个潜在失败点（Qt5 保留不变）

---

## 2026-08-22 修复 CI 发布 continuous-build 失败（Update Continuous Build Release exit 1）

### 现象
`ci-win.yml` / `ci-linux.yml` / `ci-macos.yml` 的
`Update Continuous Build Release` 步骤报
`Error: Process completed with exit code 1.`

### 根因
该步骤用 GITHUB_TOKEN 通过 `gh` 创建/更新 `continuous-build` release 并上传产物，
但三个 workflow 均无 `permissions` 声明。官方仓库配置了写权限所以能发布；
fork（如 `Ysep/vnote`）上 GITHUB_TOKEN 默认只读，
`gh release create` 403 被 `|| true` 吞掉后，
末尾的 `gh release upload`（无 `|| true`，处于 `set -e` 下）必然失败退出。

### 变更
- 三个 workflow 顶层新增：
  ```yaml
  permissions:
    contents: write
  ```
  显式授予 release/tag 写权限，使 fork 上也能发布 continuous-build。

### 验证
- YAML 语法校验通过（结构对齐，无缩进破坏）
- 权限声明为 workflow 级，优先于仓库默认的只读设置

---

## 2026-08-22 移植工具栏 # 小节序号 按钮并修复设置页翻译

### 背景
- v4.2.0 / v4.1.1 中，打开 md 文档后在编辑器工具栏有
  `#` 小节序号按钮（下拉：跟随配置 / 启用 / 禁用），
  可在当前缓冲区强制覆盖小节序号模式。
- master（重构后的 ViewWindow2 体系）中该按钮丢失：
  `viewwindowtoolbarhelper2.cpp` 的 34 个 case 无 `SectionNumber`，
  `overrideSectionNumber` 无调用者。
- 设置界面 编辑器 -> Markdown编辑器 的 `小节序号` 行显示英文：
  master 的 `vnote_zh_CN.ts` / `vnote_ja.ts` 缺 `Section number` 等词条
  （label 为 `markdowneditorpage.cpp` 的 `tr("Section number")`）。

### 变更
- `viewwindowtoolbarhelper2.h/.cpp`：
  - 枚举新增 `SectionNumber`
  - 新增按钮 case：图标 `section_number_editor.svg` + 下拉菜单
    （Follow Configuration / Enabled / Disabled，data 为 `OverrideState`）
- `viewwindow2.h/.cpp`：
  - 新增虚函数 `handleSectionNumberOverride(OverrideState)`（默认空）
  - `addAction` 的 switch 新增 `SectionNumber` case：菜单 triggered →
    `handleSectionNumberOverride(state)`
- `markdownviewwindow2.h/.cpp`：
  - 工具栏 `TypeTable` 后加入该按钮（与 v4.2.0 位置一致，
    Tag 之后、InplacePreview 之前）
  - 实现 `handleSectionNumberOverride` → `m_editor->overrideSectionNumber(state)`
- 新增图标 `src/data/core/icons/section_number_editor.svg`（# 网格，取自 v4.2.0）
- `vnote_zh_CN.ts` / `vnote_ja.ts` 补全词条：
  - 设置页：`Section number`（小节序号）、`Section number mode`、
    `Base level to start section numbering in edit mode`、`Section number style`、`1.1.`/`1.1`
  - 工具栏菜单：`Follow Configuration`（跟随配置）、`Enabled`（启用）、`Disabled`（禁用）

### 验证
- 全量 lint 0 错误
- `overrideSectionNumber` 为 public（markdowneditor.h:132），
  `m_editor` 类型 `MarkdownEditor *` 且 view 已 include 头文件
