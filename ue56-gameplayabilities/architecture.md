# UE 5.6 GameplayAbilities 架构笔记（第一轮）

本轮只阅读 `Build.cs` 与核心 `Public/Private` 目录结构；未阅读类实现细节。所有结论仅覆盖以下源码路径：

- `Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities`
- `Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilitiesEditor`
- `Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilitiesBlueprintEditor`

## 模块边界

- `GameplayAbilities` 是运行时主模块，源码路径：`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/GameplayAbilities.Build.cs`。
- `GameplayAbilities` 公开依赖 `Core`、`CoreUObject`、`NetCore`、`Engine`、`GameplayTags`、`GameplayTasks`、`MovieScene`、`PhysicsCore`、`DeveloperSettings`、`DataRegistry`，说明运行时模块同时覆盖对象系统、网络、任务、标签、MovieScene、物理与数据注册表集成；源码路径：`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/GameplayAbilities.Build.cs`。
- `GameplayAbilities` 私有依赖 `Niagara`，Build.cs 注释指出用途是 Gameplay Cue Notify 的 Niagara 支持；源码路径：`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/GameplayAbilities.Build.cs`。
- `GameplayAbilities` 在 `Target.bBuildEditor == true` 时额外私有依赖 `EditorFramework`、`UnrealEd`、`Slate`、`SequenceRecorder`，说明运行时模块包含少量仅编辑器构建启用的代码路径；源码路径：`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/GameplayAbilities.Build.cs`。
- `GameplayAbilities` 调用 `SetupGameplayDebuggerSupport(Target)` 与 `SetupIrisSupport(Target)`，说明模块接入 Gameplay Debugger 与 Iris 支持；具体接入点未确认；源码路径：`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/GameplayAbilities.Build.cs`。
- `GameplayAbilitiesEditor` 是编辑器模块，源码路径：`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilitiesEditor/GameplayAbilitiesEditor.Build.cs`。
- `GameplayAbilitiesEditor` 公开依赖 `GameplayTasks`，私有依赖 `GameplayAbilities`、`GameplayTags`、`GameplayTagsEditor`、`GameplayTasksEditor`、`BlueprintGraph`、`Kismet`、`KismetCompiler`、`GraphEditor`、`PropertyEditor`、`Sequencer`、`MovieSceneTools`、`DataRegistryEditor` 等，说明该模块负责 GAS 资产、蓝图图表、细节面板、Sequencer 与编辑器菜单集成；源码路径：`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilitiesEditor/GameplayAbilitiesEditor.Build.cs`。
- 当前工作树的 `Source` 下只存在 `GameplayAbilities` 与 `GameplayAbilitiesEditor` 两个目录，没有发现 `GameplayAbilitiesBlueprintEditor` 目录或 Build.cs；因此独立 `GameplayAbilitiesBlueprintEditor` 模块在当前 UE 5.6 工作树中未确认；源码路径：`Engine/Plugins/Runtime/GameplayAbilities/Source`。

## 运行时目录结构

- `GameplayAbilities/Public` 有 51 个顶层头文件，集中暴露 Ability System、Attribute、Gameplay Ability、Gameplay Effect、Gameplay Cue、Prediction、ScalableFloat、DeveloperSettings 与 Debugger 入口；源码路径：`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Public`。
- `GameplayAbilities/Private` 有 50 个顶层源/私有头文件，与 `Public` 顶层 API 基本对应，说明主要运行时实现集中在模块根 Private 下；源码路径：`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Private`。
- Ability System 的核心入口从文件名看包括 `AbilitySystemComponent.h`、`AbilitySystemGlobals.h`、`AbilitySystemInterface.h`、`AbilitySystemBlueprintLibrary.h`、`AbilitySystemReplicationProxyInterface.h`；具体职责边界未确认；源码路径：`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Public`。
- Attribute 相关入口从文件名看包括 `AttributeSet.h`、`TickableAttributeSetInterface.h`、`AbilitySystemTestAttributeSet.h`；具体属性聚合与复制流程未确认；源码路径：`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Public`。
- Gameplay Ability 相关顶层入口从文件名看包括 `GameplayAbilityBlueprint.h`、`GameplayAbilitySet.h`、`GameplayAbilitySpec.h`、`GameplayAbilitySpecHandle.h`；具体授予、激活、实例化流程未确认；源码路径：`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Public`。
- Gameplay Effect 相关顶层入口从文件名看包括 `GameplayEffect.h`、`GameplayEffectTypes.h`、`GameplayEffectComponent.h`、`GameplayEffectCalculation.h`、`GameplayEffectExecutionCalculation.h`、`GameplayModMagnitudeCalculation.h`、`GameplayEffectAggregator.h`；具体修饰器、聚合器、执行计算流程未确认；源码路径：`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Public`。
- Gameplay Cue 相关顶层入口从文件名看包括 `GameplayCueManager.h`、`GameplayCueSet.h`、`GameplayCueInterface.h`、`GameplayCueFunctionLibrary.h`、`GameplayCueNotify_*`、`GameplayCue_Types.h`；具体 Cue 路由、加载、复制流程未确认；源码路径：`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Public`。
- `GameplayAbilities/Public/Abilities` 有 15 个头文件，覆盖 `GameplayAbility.h`、Ability Target Actor、Target Types、Reticle、`GameplayAbility_Montage.h`、`GameplayAbility_CharacterJump.h` 等 Ability 侧类型；源码路径：`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Public/Abilities`。
- `GameplayAbilities/Private/Abilities` 有 13 个实现文件，与 `Public/Abilities` 基本对应；源码路径：`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Private/Abilities`。
- `GameplayAbilities/Public/Abilities/Tasks` 有 41 个头文件，覆盖根运动、Montage/Anim、等待输入、等待 GameplayTag、等待 GameplayEffect、等待 TargetData、等待事件、等待属性变化等 AbilityTask；源码路径：`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Public/Abilities/Tasks`。
- `GameplayAbilities/Private/Abilities/Tasks` 有 41 个实现文件，和公开 AbilityTask 头文件数量对应；源码路径：`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Private/Abilities/Tasks`。
- `GameplayAbilities/Public/Abilities/Async` 有 7 个头文件，文件名均为 `AbilityAsync_*` 或 `AbilityAsync.h`，从命名看是 AbilitySystem 相关异步等待封装；具体蓝图暴露方式未确认；源码路径：`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Public/Abilities/Async`。
- `GameplayAbilities/Private/Abilities/Async` 有 7 个实现文件，和公开 Async 头文件数量对应；源码路径：`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Private/Abilities/Async`。
- `GameplayAbilities/Public/GameplayEffectComponents` 有 10 个头文件，覆盖额外效果、资产标签、目标标签、阻塞 Ability 标签、免疫、概率应用、自定义 CanApply、移除其他效果等组件化 GameplayEffect 行为；源码路径：`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Public/GameplayEffectComponents`。
- `GameplayAbilities/Private/GameplayEffectComponents` 有 10 个实现文件，和公开 GameplayEffectComponent 头文件数量对应；源码路径：`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Private/GameplayEffectComponents`。
- `GameplayAbilities/Public/Sequencer` 有 `MovieSceneGameplayCueSections.h` 与 `MovieSceneGameplayCueTrack.h`，说明运行时暴露 GameplayCue 的 MovieScene/Sequencer 轨道数据类型；源码路径：`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Public/Sequencer`。
- `GameplayAbilities/Private/Sequencer` 有对应的 2 个实现文件；源码路径：`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Private/Sequencer`。
- `GameplayAbilities/Public/Serialization` 有 8 个头文件，文件名集中在 GameplayEffectContext、GameplayAbilityTargetData、GameplayAbilityRepAnimMontage、MinimalGameplayCue 与 MinimalReplicationTagCount 的 NetSerializer/ReplicationFragment；源码路径：`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Public/Serialization`。
- `GameplayAbilities/Private/Serialization` 有 17 个文件，包含 `Internal*NetSerializer`、`PredictionKeyNetSerializer`、MinimalReplicationTagCountMap 复制片段等私有实现；源码路径：`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Private/Serialization`。
- `GameplayAbilities/Private/Tests` 有 6 个测试文件，覆盖 AbilitySystemComponent、PredictionKey、GameplayTagQuery、GameplayTagCountContainer、GameplayEffect、GameplayCue 相关测试；具体测试覆盖面未确认；源码路径：`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Private/Tests`。

## 编辑器目录结构

- `GameplayAbilitiesEditor/Public` 有 10 个头文件，公开入口包括模块类、编辑器类、蓝图工厂、Ability 图表/Schema、GameplayCue K2 节点、Latent Ability K2 节点、Ability Audit 与 AttributeDetails；源码路径：`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilitiesEditor/Public`。
- `GameplayAbilitiesEditor/Private` 有 36 个文件，覆盖资产类型动作、蓝图工厂实现、图表面板 Pin 工厂、Ability 图表与 Schema、Attribute/GameplayEffect/GameplayTag 细节定制、GameplayCue 编辑 Slate 控件、K2 节点实现等；源码路径：`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilitiesEditor/Private`。
- `GameplayAbilitiesEditor/Private/Sequencer` 有 2 个文件，分别为 `GameplayCueTrackEditor.h` 与 `GameplayCueTrackEditor.cpp`，说明编辑器模块提供 GameplayCue 轨道编辑器集成；源码路径：`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilitiesEditor/Private/Sequencer`。

## 当前未确认项

- `GameplayAbilitiesBlueprintEditor` 在当前工作树中没有目录，是否被合并进 `GameplayAbilitiesEditor` 或在其他分支/安装形态中存在，未确认；源码路径：`Engine/Plugins/Runtime/GameplayAbilities/Source`。
- `AbilitySystemComponent` 与 `GameplayAbility`、`GameplayEffect`、`GameplayCueManager` 的调用链、生命周期、网络预测细节，本轮未读实现，未确认；源码路径：`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Public` 与 `Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Private`。
- `SetupIrisSupport(Target)` 对 `Serialization` 目录中 NetSerializer 文件的具体启用方式，本轮未追踪 UBT 内部逻辑，未确认；源码路径：`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/GameplayAbilities.Build.cs` 与 `Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Public/Serialization`。
- `GameplayAbilitiesEditor` 中 K2 节点、GraphSchema、Details 定制之间的注册顺序和扩展点，本轮未读实现，未确认；源码路径：`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilitiesEditor/Public` 与 `Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilitiesEditor/Private`。

## 下一轮建议

下一轮建议优先分析 `UAbilitySystemComponent`，因为它是运行时主入口，文件同时出现在 `AbilitySystemComponent.h`、`AbilitySystemComponent.cpp`、`AbilitySystemComponent_Abilities.cpp`，最适合串起 Ability 授予/激活、GameplayEffect 应用、Tag/Attribute、复制与预测等核心路径；源码路径：`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Public/AbilitySystemComponent.h`、`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Private/AbilitySystemComponent.cpp`、`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Private/AbilitySystemComponent_Abilities.cpp`。
