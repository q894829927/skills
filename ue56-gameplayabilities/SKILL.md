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

- `quick-reference.md`：GAS 速查表，按 ASC、GA、GE、AttributeSet、GameplayCue、常见玩法模板整理“什么时候用哪个类/函数/配置”。
- `architecture.md`：第一轮源码地图，只基于 `Build.cs` 和 `Public/Private` 目录结构。
- `core-classes.md`：核心类笔记，已覆盖 `UAbilitySystemComponent`、`UGameplayAbility`，并索引 GameplayEffect 体系。
- `call-flows.md`：调用链笔记，已覆盖 Ability 激活链、Ability 生命周期、GameplayEffect 应用链。
- `gameplay-effects.md`：第四轮 GameplayEffect 专题，覆盖 `UGameplayEffect`、`FGameplayEffectSpec`、`FActiveGameplayEffect`、Modifier、Duration/Period、Stacking、GameplayCue、Cost/Cooldown 接入和 GEComponents 概览。
- `gameplay-effect-components.md`：第十三轮 GameplayEffectComponent / GEComponents 深入专题，覆盖 `UGameplayEffectComponent` 基类 hook、10 个内置 GEComponent、GE 应用/激活/移除接入、deprecated 字段迁移、Tag/Ability/Cue/网络/编辑器衔接与配置速查。
- `calculations-captures.md`：第十四轮 ExecutionCalculation / ModMagnitudeCalculation / Attribute Capture 深入专题，覆盖 `UGameplayEffectExecutionCalculation`、`UGameplayModMagnitudeCalculation`、Attribute Capture、Snapshot/Non-Snapshot、Aggregator、Modifier Magnitude、SetByCaller、预测与编辑器衔接。
- `attributes.md`：第五轮 AttributeSet 专题，覆盖 `UAttributeSet`、`FGameplayAttributeData`、`FGameplayAttribute`、ASC 管理 AttributeSet、属性定义/复制、属性回调、GE 修改 Attribute、Base/Current、Damage/Healing 和 UI delegate 绑定。
- `ability-tasks.md`：第六轮 AbilityTask / GameplayTask 专题，覆盖 `UAbilityTask`、`UGameplayTask`、`UGameplayTasksComponent`、Ability `ActiveTasks`、常见 Task 分类、输入/事件/蒙太奇/TargetData/GE/Tag/Attribute 监听流程和预测接入点。
- `gameplay-cues.md`：第七轮 GameplayCue 专题，覆盖 `UGameplayCueManager`、`UGameplayCueSet`、`GameplayCueNotify` 类型、`FGameplayCueParameters`、ASC/GE 触发路径、Notify 生命周期、复制与加载机制。
- `networking-prediction.md`：第八轮 GAS 网络预测 / RPC / Replication / Serialization 专题，覆盖 `FPredictionKey`、`FScopedPredictionWindow`、Ability 激活 RPC、GenericReplicatedEvent、TargetData、GE/Attribute/Cue 复制、ASC ReplicationMode、ReplicationProxy、Iris/NetSerializer。
- `globals-blueprint-library.md`：第九轮 GAS 全局入口与蓝图辅助 API 专题，覆盖 `UAbilitySystemGlobals`、`UAbilitySystemBlueprintLibrary`、`IAbilitySystemInterface`、ASC 查找、EffectContext、TargetData、SetByCaller、Dynamic Tags、GameplayCueParameters 与全局配置。
- `editor-blueprint.md`：第十轮 GAS 编辑器与蓝图工具链专题，覆盖 `GameplayAbilitiesEditor` 模块、GameplayAbility 蓝图工厂、Ability 图表、AbilityTask latent K2 节点、GameplayCue 蓝图事件、GameplayEffect/Attribute details customization、资产动作、Ability Audit 与运行时/编辑器边界。
- `targeting-targetdata.md`：第十一轮 TargetActor / TargetData / Targeting 专题，覆盖 `AGameplayAbilityTargetActor`、TargetData 类型体系、`UAbilityTask_WaitTargetData`、TargetData RPC、确认/取消模式、Trace/Radius/ActorPlacement、`VisualizeTargeting`、TargetData 到 GE/Cue 的衔接。
- `gameplay-tags-response.md`：第十二轮 GameplayTag / ResponseTable / Ability Tag 条件体系专题，覆盖 ASC tag count、Loose/Replicated/Minimal tags、Ability tag requirements、GE tag components、`UGameplayTagReponseTable`、Tag AbilityTask、Cue/网络/蓝图/编辑器衔接。
- `pitfalls.md`：常见坑清单，按轮次补充 ASC、Ability、GameplayEffect、AttributeSet、AbilityTask、GameplayCue、网络预测/复制相关问题。

## 优先分析顺序

优先从 `quick-reference.md` 快速定位用户问题属于 ASC、GA、GE、AttributeSet、Cue、AbilityTask、网络预测/复制、GameplayTag/ResponseTable、GameplayEffectComponent、数值计算/属性捕获中的哪一类；需要源码依据时再回到 `architecture.md`、`core-classes.md`、`call-flows.md` 与对应专题文档复核。深入分析时，从 `UAbilitySystemComponent` 串联运行时核心路径，再按问题需要进入 `UGameplayAbility`、`UGameplayEffect`、`FGameplayAbilitySpec`、`FActiveGameplayEffect`、`UAttributeSet`、`UAbilityTask`、`UGameplayCueManager`、`UGameplayTagReponseTable`、`UGameplayEffectExecutionCalculation`、`UGameplayModMagnitudeCalculation`、网络预测/复制与序列化相关类型。

下一轮推荐入口：

- `Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Private/Tests`
- `Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Public/AbilitySystemTestPawn.h`
- `Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Private/AbilitySystemTestPawn.cpp`
- `Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Public/AbilitySystemTestAttributeSet.h`
- `Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Private/AbilitySystemTestAttributeSet.cpp`
