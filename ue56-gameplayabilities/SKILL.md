---
name: ue56-gameplayabilities
description: UE 5.6 GameplayAbilities/GAS source-study assistant for Unreal Engine. Use when Codex needs to help analyze, explain, navigate, or extend knowledge about GameplayAbilities, GameplayAbilitiesEditor, AbilitySystemComponent, GameplayAbility, GameplayEffect, GameplayCue, AbilityTask, AttributeSet, GAS networking/prediction/serialization, or related editor tooling in the local UE 5.6 source tree.
---

# UE 5.6 GameplayAbilities 学习 Skill

## 工作边界

只读分析 UE 源码，默认不要修改 `Engine/` 下任何文件。除非用户明确要求生成项目侧代码，否则不要生成 Ability 模板、GameplayEffect 模板、插件模板或示例业务类。

分析结论必须标注源码路径。无法从当前已读源码确认的内容，写“未确认”。

## 源码根路径

- 运行时模块：`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities`
- 编辑器模块：`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilitiesEditor`
- BlueprintEditor 路径：`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilitiesBlueprintEditor`，当前架构笔记中记录为未确认

## 使用流程

1. 先读取 `architecture.md`，确认当前已知模块边界、目录结构与未确认项。
2. 根据用户问题读取对应专题文档，再只打开必要源码文件复核；优先用 `rg` 定位类、函数、宏、委托、配置项和调用点。
3. 输出中文结论，并在每条关键结论后标注源码路径。
4. 区分“源码已确认”“根据命名/目录推断”“未确认”。
5. 如果需要补充本 skill 的知识库，只更新 `.codex/skills/ue56-gameplayabilities/` 下的文档，不修改 UE Engine 源码。

## 当前参考资料

- `quick-reference.md`：最终速查入口，用于按 ASC、Ability、GE、Attribute、Cue、TargetData、Tag、Prediction、Debug、Tests、Performance 快速定位文档。
- `final-templates.md`：生成或审查 GAS 项目侧代码前的规则、模板结构和检查清单；只给规则，不给完整业务代码。
- `debugging-logging.md`：调试、日志、VisualLogger、GameplayDebugger、CVar 和常见排错入口。
- `tests-practices.md`：官方 Tests 覆盖点、未覆盖点和可借鉴的测试方式。
- 专题文档：`architecture.md`、`core-classes.md`、`call-flows.md`、`gameplay-effects.md`、`gameplay-effect-components.md`、`calculations-captures.md`、`attributes.md`、`ability-tasks.md`、`gameplay-cues.md`、`networking-prediction.md`、`globals-blueprint-library.md`、`editor-blueprint.md`、`targeting-targetdata.md`、`gameplay-tags-response.md`、`pitfalls.md`。

## 最终使用顺序

1. 普通问题先查 `quick-reference.md`。
2. 代码生成或改造前先查 `final-templates.md`。
3. 具体机制再查对应专题文档，并按需回到源码复核。
4. 排错先查 `debugging-logging.md`。
5. 官方测试验证与测试写法先查 `tests-practices.md`。

源码分析阶段已完成，不再建议继续按轮次扩展。未来只在遇到具体项目问题、源码版本差异或未确认项需要复核时，按主题局部更新现有文档。
