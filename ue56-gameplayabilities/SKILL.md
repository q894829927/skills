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

- `architecture.md`：第一轮源码地图，只基于 `Build.cs` 和 `Public/Private` 目录结构。
- `core-classes.md`：核心类笔记，已覆盖 `UAbilitySystemComponent`、`UGameplayAbility`，并索引 GameplayEffect 体系。
- `call-flows.md`：调用链笔记，已覆盖 Ability 激活链、Ability 生命周期、GameplayEffect 应用链。
- `gameplay-effects.md`：第四轮 GameplayEffect 专题，覆盖 `UGameplayEffect`、`FGameplayEffectSpec`、`FActiveGameplayEffect`、Modifier、Duration/Period、Stacking、GameplayCue、Cost/Cooldown 接入和 GEComponents 概览。
- `attributes.md`：第五轮 AttributeSet 专题，覆盖 `UAttributeSet`、`FGameplayAttributeData`、`FGameplayAttribute`、ASC 管理 AttributeSet、属性定义/复制、属性回调、GE 修改 Attribute、Base/Current、Damage/Healing 和 UI delegate 绑定。
- `ability-tasks.md`：第六轮 AbilityTask / GameplayTask 专题，覆盖 `UAbilityTask`、`UGameplayTask`、`UGameplayTasksComponent`、Ability `ActiveTasks`、常见 Task 分类、输入/事件/蒙太奇/TargetData/GE/Tag/Attribute 监听流程和预测接入点。
- `pitfalls.md`：常见坑清单，按轮次补充 ASC、Ability、GameplayEffect 相关问题。

## 优先分析顺序

优先从 `UAbilitySystemComponent` 串联运行时核心路径，再按问题需要进入 `UGameplayAbility`、`UGameplayEffect`、`FGameplayAbilitySpec`、`FActiveGameplayEffect`、`UAttributeSet`、`UAbilityTask`、`UGameplayCueManager` 与序列化/预测相关类型。

下一轮推荐入口：

- `Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Public/GameplayCueManager.h`
- `Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Private/GameplayCueManager.cpp`
- `Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Public/GameplayCueNotify_Actor.h`
- `Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Public/GameplayCueNotify_Static.h`
