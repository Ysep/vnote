# 小节序号（Section Number）功能说明

> 本文件梳理工具栏 `#` 小节序号按钮在 master 分支的完整实现，供后续开发查阅。
> 最后更新：2026-08-22（提交 `64af12f8`）

## 一、背景

- `#` 小节序号按钮存在于 **v4.2.0 / v4.1.1**（旧 `ViewWindow` 体系），通过
  `ViewWindow::handleSectionNumberOverride` 实现。
- master 重构为 **ViewWindow2** 体系后该功能丢失：工具栏 helper 没有
  `SectionNumber` action，`MarkdownEditor::overrideSectionNumber` 没有调用方。
- 已按旧链路在 ViewWindow2 框架下重新实现，并补齐设置页中文翻译。
- 恢复提交：**`64af12f8`** — feat(markdown): restore section number toolbar button
  and fix setting translation（10 文件 +194 行，已推送 fork/master）。

## 二、功能描述

打开 md 文档 → 工具栏出现 `#` 图标按钮（`Section Number`），点击弹出菜单三项：

| 菜单项 | data（OverrideState） | 含义 |
|--------|----------------------|------|
| Follow Configuration | `NoOverride` | 跟随设置页配置 |
| Enabled | `ForceEnable` | 强制启用小节序号 |
| Disabled | `ForceDisable` | 强制禁用小节序号 |

选择后调用编辑器 `overrideSectionNumber()`，对该 buffer 单独覆盖设置页里的小节序号模式。

## 三、核心定义

**`OverrideState`** — `src/core/global.h` 第 78 行：

```cpp
enum OverrideState { NoOverride = 0, ForceEnable = 1, ForceDisable = 2 };
```

**`MarkdownEditor::overrideSectionNumber`**：

- 声明：`src/widgets/editors/markdowneditor.h` 第 132 行（public）
- 实现：`src/widgets/editors/markdowneditor.cpp` 第 1648 行
- 内部成员：`m_overriddenSectionNumber`（markdowneditor.h 第 309 行）
- 另有 `m_sectionNumberEnabled`（markdowneditor.h 第 307 行）用于检测配置变更

## 四、完整实现链路（7 个改动点）

调用链：

```
工具栏点击 → ViewWindowToolBarHelper2::SectionNumber（菜单按钮）
         → ViewWindow2::handleSectionNumberOverride（switch 接线，默认空实现）
         → MarkdownViewWindow2::handleSectionNumberOverride（override，转发）
         → MarkdownEditor::overrideSectionNumber（生效）
```

| # | 文件 | 位置 | 内容 |
|---|------|------|------|
| 1 | `src/widgets/viewwindowtoolbarhelper2.h` | 第 61 行 | 枚举 `Action` 中 `InplacePreview`(58) 与 `Outline`(63) 之间插入 `SectionNumber,` |
| 2 | `src/widgets/viewwindowtoolbarhelper2.cpp` | 第 362–389 行 | `case Action::SectionNumber:` 创建图标按钮 + `QMenu`（三个 checkable QAction，`setData(OverrideState)`），`InstantPopup` |
| 3 | `src/widgets/viewwindow2.h` | 第 399 行 | 虚函数 `virtual void handleSectionNumberOverride(OverrideState p_state);` |
| 4 | `src/widgets/viewwindow2.cpp` | 第 447 行 + 第 637–640 行 | 默认空实现；switch 中连接 `menu->triggered` → `handleSectionNumberOverride(state)` |
| 5 | `src/widgets/markdownviewwindow2.h` | 第 103 行 | `void handleSectionNumberOverride(OverrideState p_state) Q_DECL_OVERRIDE;` |
| 6 | `src/widgets/markdownviewwindow2.cpp` | 第 183 行（按钮）+ 第 268–272 行（实现） | 按钮加在 `TypeTable` 之后；实现转发给 `m_editor->overrideSectionNumber(p_state)` |
| 7 | `src/data/core/icons/section_number_editor.svg` | 新文件 | `#` 网格图标（取自 v4.2.0，547 字节） |

### 4.1 helper2 的 case 代码（可复用）

`src/widgets/viewwindowtoolbarhelper2.cpp`：

```cpp
case Action::SectionNumber: {
  // Menu button letting the user override the section-number mode for this
  // buffer: follow configuration, force enable, or force disable.
  act = p_tb->addAction(generateIcon(p_services, QStringLiteral("section_number_editor.svg")),
                        QObject::tr("Section Number"));
  act->setProperty("iconName", QStringLiteral("section_number_editor.svg"));
  auto *toolBtn = dynamic_cast<QToolButton *>(p_tb->widgetForAction(act));
  Q_ASSERT(toolBtn);
  toolBtn->setPopupMode(QToolButton::InstantPopup);
  toolBtn->setProperty(PropertyDefs::c_toolButtonWithoutMenuIndicator, true);
  auto *menu = new QMenu(toolBtn);
  auto *actionGroup = new QActionGroup(menu);
  auto *followAct = actionGroup->addAction(QObject::tr("Follow Configuration"));
  followAct->setCheckable(true);
  followAct->setChecked(true);
  followAct->setData(OverrideState::NoOverride);
  menu->addAction(followAct);
  auto *enableAct = actionGroup->addAction(QObject::tr("Enabled"));
  enableAct->setCheckable(true);
  enableAct->setData(OverrideState::ForceEnable);
  menu->addAction(enableAct);
  auto *disableAct = actionGroup->addAction(QObject::tr("Disabled"));
  disableAct->setCheckable(true);
  disableAct->setData(OverrideState::ForceDisable);
  menu->addAction(disableAct);
  toolBtn->setMenu(menu);
  break;
}
```

### 4.2 viewwindow2 的 switch 接线

`src/widgets/viewwindow2.cpp` 第 637–640 行：

```cpp
connect(toolBtn->menu(), &QMenu::triggered, this, [this](QAction *p_act) {
  const auto state = static_cast<OverrideState>(p_act->data().toInt());
  handleSectionNumberOverride(state);
});
```

### 4.3 markdownviewwindow2 的转发实现

`src/widgets/markdownviewwindow2.cpp` 第 268–272 行：

```cpp
void MarkdownViewWindow2::handleSectionNumberOverride(OverrideState p_state) {
  if (m_editor) {
    m_editor->overrideSectionNumber(p_state);
  }
}
```

## 五、翻译词条（设置页显示英文问题的修复）

设置页代码本体用 `tr("Section number")`
（`src/widgets/dialogs/settings/markdowneditorpage.cpp`），显示英文的根因是 master 的
`.ts` 文件缺词条（v4.2.0 有）。已在 `src/data/core/translations/vnote_zh_CN.ts` 和
`vnote_ja.ts` 补齐。

**设置页词条**（zh_CN 对照）：

| source | zh_CN | ja |
|--------|-------|-----|
| `Section number mode` | 小节序号模式 | セクション番号モード |
| `Base level to start section numbering in edit mode` | 编辑模式中开始小节序号计数的基础层级 | 編集モードでセクション番号付けを開始するベースレベル |
| `Section number style` | 小节序号样式 | セクション番号スタイル |
| `Section number` | 小节序号 | セクション番号 |

**菜单词条**：

| source | zh_CN | ja |
|--------|-------|-----|
| `Section Number` | 小节序号 | セクション番号 |
| `Follow Configuration` | 跟随配置 | 設定に従う |
| `Enabled` | 启用 | 有効 |
| `Disabled` | 禁用 | 無効 |

## 六、相关提交

- `64af12f8` — feat(markdown): restore section number toolbar button and fix
  setting translation（本功能恢复 + 翻译补全）
- `dcb9f38b` — fix(ci): grant contents:write to publish continuous-build release
  from forks（CI 权限修复，非本功能，仅相关）

## 七、验证方式

1. 重新编译 → 打开 md 文档 → 工具栏 `#` 按钮 → 选择"启用 / 禁用 / 跟随配置"。
2. 设置 → 编辑器 → Markdown 编辑器 → 首行显示中文"小节序号"（而非英文）。

## 八、快速提问模板

> 在 master 上继续开发小节序号功能。现状：工具栏 `#` 按钮已通过
> `ViewWindowToolBarHelper2::SectionNumber`（`viewwindowtoolbarhelper2.h:61`）→
> `ViewWindow2::handleSectionNumberOverride`（`viewwindow2.cpp:637`）→
> `MarkdownViewWindow2::handleSectionNumberOverride`（`markdownviewwindow2.cpp:268`）→
> `MarkdownEditor::overrideSectionNumber`（`markdowneditor.cpp:1648`）链路实现，
> `OverrideState` 在 `core/global.h:78`，图标为
> `src/data/core/icons/section_number_editor.svg`，翻译词条已补全（提交 `64af12f8`）。
> 现在需要……
