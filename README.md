# godot-gdscript-to-csharp

把 Godot 3 GDScript 项目迁到 Godot 4（GDScript 和/或 C#）的实战 skill，支持三条路径：3 GDScript → 4 GDScript、4 GDScript → 4 C#、3 GDScript → 4 C#。

> English: a skill for porting Godot 3 GDScript to Godot 4 C#, slice by slice —
> conventions, silent-failure traps, and verification. See [SKILL.md](./SKILL.md).

## 它是干什么的

给 AI 编程助手（或人）用的迁移执行手册：拿到一个 `.gd` 文件，按里面的规则译成
`.cs`，并且**编译通过还不够**——必须保证运行时行为与原来一致。

## 适用范围

- ✅ Godot 3 GDScript → Godot 4 GDScript（换引擎不换语言）
- ✅ Godot 4 GDScript → Godot 4 C#（换语言不换引擎）的玩法代码迁移
- ✅ 渐进式迁移：GD 与 C# 共存、一次只迁一片、删 GD 前后各验证一次
- ✅ 排查"编译全绿但运行时静默失效"的问题（信号没连上、反射调不到、布局塌陷……）
- ✅ 重压场景性能定位（headless 量不到渲染时的替代方案）

## 不适用

- ❌ 资源/场景的 3→4 格式转换（那是另一套工具链，与语言无关）
- ❌ 纯编辑器插件、导出模板配置
- ❌ 从零学 GDScript 或 C# 语法——默认你已经会两种语言，只缺迁移时的坑位图

## 怎么用

把本仓库（或仅 `SKILL.md`）放进你的 skill 目录即可，例如：

```text
<你的 skill 目录>/godot-gdscript-to-csharp/SKILL.md
```

之后在相关任务开始前加载这个 skill（按你所用工具的 skill 调用方式）。

## 内容一览（`SKILL.md`）

| 部分 | 内容 |
|---|---|
| Order of work | 迁移顺序：行为唯一归属、自底向上、不重写场景、四件套验证 |
| Bridging conventions | GD 调用方清零前的双 API 桥、`_Get/_Set`、信号与 `await` 规范 |
| Silent-failure traps | 11 条"编译全绿、运行犯错"陷阱：事件泄漏、锚点塌陷、索引判界、构造时序、钩子位置、页面状态机、调用闭环、统计口径、哈希判空、版本差异、场景定位残留 |
| Performance | 为什么 headless 量不到渲染、游戏内浮层读数、对象池/批量绘制/并发上限 |
| Probe discipline | 探针写法、隔离用户目录、清理纪律 |

## 版本说明

- v1：初始版本，来自两个完整迁移项目的实机验证经验（引擎覆盖 Godot 4.6–4.8）。
- v2：新增第 11 条陷阱（Godot 3 场景定位残留压塌 Release 布局）+ 验证纪律（日志 ≠ 渲染，UI 修复须真实 Release 包截图闭环）。
- 内容与任何具体游戏无关，可直接用于你自己的项目。
