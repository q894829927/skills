# 核心类：UAbilitySystemComponent（第二轮）

本轮重点阅读：

- `Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Public/AbilitySystemComponent.h`
- `Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Private/AbilitySystemComponent.cpp`
- `Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Private/AbilitySystemComponent_Abilities.cpp`

以下文件在当前工作树不存在，标记为未确认：

- `Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Private/AbilitySystemComponent_GameplayEffects.cpp`：未确认
- `Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Private/AbilitySystemComponent_Replication.cpp`：未确认

## 一、类定位

- `UAbilitySystemComponent` 是 GAS 对外交互的核心 ActorComponent，源码注释称它负责 GameplayAbilities、GameplayEffects、GameplayAttributes 三类能力系统入口；源码路径：`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Public/AbilitySystemComponent.h:42`。
- `UAbilitySystemComponent` 继承自 `UGameplayTasksComponent`，并实现 `IGameplayTagAssetInterface` 与 `IAbilitySystemReplicationProxyInterface`；源码路径：`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Public/AbilitySystemComponent.h:107`。
- 源码将 `OwnerActor` 定义为“逻辑拥有者”，将 `AvatarActor` 定义为“世界中的物理表现，通常是 Pawn，也可能是 Tower/Building/Turret，二者可以相同”；源码路径：`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Public/AbilitySystemComponent.h:1546`。
- Character / PlayerState 中如何放置 ASC：本轮源码确认了 Owner/Avatar 分离模型，但没有在指定文件中规定“必须放 Character 或 PlayerState”。可确认的是 `InitAbilityActorInfo(InOwnerActor, InAvatarActor)` 会同时设置 Owner 与 Avatar，并在 Avatar 变化时通知已授予 Ability 的 `OnAvatarSet`；源码路径：`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Private/AbilitySystemComponent_Abilities.cpp:141`。Character vs PlayerState 的项目实践差异：未确认。
- `UGameplayAbility` 关系：ASC 通过 `FGameplayAbilitySpecContainer ActivatableAbilities` 保存可激活 Ability spec，`GiveAbility` 写入该容器，`TryActivateAbility` / `InternalTryActivateAbility` 根据 handle 找到 spec 并最终调用 `UGameplayAbility::CallActivateAbility`；源码路径：`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Public/AbilitySystemComponent.h:1695`、`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Private/AbilitySystemComponent_Abilities.cpp:272`、`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Private/AbilitySystemComponent_Abilities.cpp:1683`。
- `UGameplayEffect` 关系：ASC 通过 `FActiveGameplayEffectsContainer ActiveGameplayEffects` 保存 Active GameplayEffect，并提供应用、移除、查询接口；源码路径：`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Public/AbilitySystemComponent.h:1880`、`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Private/AbilitySystemComponent.cpp:798`。
- `UAttributeSet` 关系：ASC 通过 `SpawnedAttributes` 保存 AttributeSet 实例，`InitStats` / `AddSpawnedAttribute` / `AddAttributeSetSubobject` 会把 AttributeSet 纳入 ASC 管理；源码路径：`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Public/AbilitySystemComponent.h:1956`、`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Private/AbilitySystemComponent.cpp:83`、`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Private/AbilitySystemComponent.cpp:3005`。

## 二、核心职责

1. Ability 赋予、移除、查询：`GiveAbility` 只允许 authority 执行并写入 `ActivatableAbilities.Items`；`ClearAbility` / `ClearAllAbilities` 只允许 authority 移除；`FindAbilitySpecFromHandle` 在当前和 pending 列表中按 handle 查找；源码路径：`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Private/AbilitySystemComponent_Abilities.cpp:272`、`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Private/AbilitySystemComponent_Abilities.cpp:416`、`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Private/AbilitySystemComponent_Abilities.cpp:475`、`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Private/AbilitySystemComponent_Abilities.cpp:896`。
2. Ability 激活、取消、结束：`TryActivateAbility` 先检查 spec、ActorInfo、网络执行策略，再进入 `InternalTryActivateAbility`；`InternalTryActivateAbility` 调用 `CanActivateAbility` 后调用 `CallActivateAbility`；取消通过 `CancelAbility` / `CancelAbilities` 遍历 active spec 并调用 `CancelAbilitySpec`；远端结束/取消由 Server/Client RPC 与 `RemoteEndOrCancelAbility` 处理；源码路径：`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Private/AbilitySystemComponent_Abilities.cpp:1583`、`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Private/AbilitySystemComponent_Abilities.cpp:1683`、`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Private/AbilitySystemComponent_Abilities.cpp:1280`、`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Private/AbilitySystemComponent_Abilities.cpp:2099`。
3. GameplayEffect 应用、移除、查询：`ApplyGameplayEffectToSelf`/`ApplyGameplayEffectToTarget` 构造 spec 后调用 spec 版本；`ApplyGameplayEffectSpecToSelf` 检查 authority/prediction、周期预测限制、application query、CanApply、Modifier attribute，并把持续型 GE 放入 `ActiveGameplayEffects`；`RemoveActiveGameplayEffect` 委托给 `ActiveGameplayEffects.RemoveActiveGameplayEffect`；源码路径：`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Private/AbilitySystemComponent.cpp:554`、`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Private/AbilitySystemComponent.cpp:589`、`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Private/AbilitySystemComponent.cpp:777`、`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Private/AbilitySystemComponent.cpp:798`、`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Private/AbilitySystemComponent.cpp:1041`。
4. AttributeSet 管理：`InitStats` 会获取或创建 AttributeSet 子对象并可从 DataTable 初始化；`AddSpawnedAttribute` 把 AttributeSet 加入 `SpawnedAttributes` 并标记复制 dirty；`RemoveSpawnedAttribute` 移除后清理对应 Attribute aggregator；源码路径：`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Private/AbilitySystemComponent.cpp:83`、`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Private/AbilitySystemComponent.cpp:3005`、`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Private/AbilitySystemComponent.cpp:3024`。
5. GameplayTag 管理：ASC 实现 `IGameplayTagAssetInterface`，`HasMatchingGameplayTag` 等查询委托给 `GameplayTagCountContainer`；loose tag 通过 `UpdateTagMap` 修改且源码注释明确不复制；replicated loose tag 使用 `ReplicatedLooseTags`；源码路径：`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Public/AbilitySystemComponent.h:572`、`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Public/AbilitySystemComponent.h:649`、`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Public/AbilitySystemComponent.h:690`。
6. GameplayCue 触发：ASC 提供 `ExecuteGameplayCue`、`AddGameplayCue`、`RemoveGameplayCue` 和 NetMulticast RPC；实际事件调用转给 `UAbilitySystemGlobals::Get().GetGameplayCueManager()`；源码路径：`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Public/AbilitySystemComponent.h:876`、`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Private/AbilitySystemComponent.cpp:1277`、`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Private/AbilitySystemComponent.cpp:1300`。
7. 网络复制：`GetLifetimeReplicatedProps` 复制 `ActiveGameplayEffects`、`SpawnedAttributes`、`ActiveGameplayCues`、Owner/Avatar、`ReplicatedLooseTags`、`ActivatableAbilities`、`BlockedAbilityBindings`、`ReplicatedPredictionKeyMap`、`MinimalReplicationGameplayCues`、`MinimalReplicationTags` 等；源码路径：`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Private/AbilitySystemComponent.cpp:1626`。
8. 预测 Prediction：`ScopedPredictionKey` 是当前预测 key，`InternalTryActivateAbility` 在 LocalPredicted 分支创建 `FScopedPredictionWindow`、设置 `ActivationInfo.SetPredicting` 并向服务器发送激活 RPC；`ReplicatedPredictionKeyMap` 必须位于 ASC replicated properties 最后以保证 OnRep/callback 顺序；源码路径：`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Public/AbilitySystemComponent.h:267`、`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Private/AbilitySystemComponent_Abilities.cpp:1904`、`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Public/AbilitySystemComponent.h:1976`。

## 三、关键数据结构

| 成员/结构 | 定义位置 | 类型 | 作用 | 什么时候被修改 | 业务层是否通常直接访问 |
|---|---|---|---|---|---|
| `ActivatableAbilities` | `Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Public/AbilitySystemComponent.h:1695` | `FGameplayAbilitySpecContainer` | 保存已授予、可激活的 `FGameplayAbilitySpec`。 | `GiveAbility` 添加，`ClearAbility`/`ClearAllAbilities` 移除，`MarkAbilitySpecDirty` 标记复制；源码路径：`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Private/AbilitySystemComponent_Abilities.cpp:272`、`:416`、`:475`。 | 通常通过 `GiveAbility`、`TryActivateAbility`、查询函数访问；直接改数组不推荐，修改 spec 后源码提示要 `MarkAbilitySpecDirty`；源码路径：`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Public/AbilitySystemComponent.h:1142`。 |
| `ActiveGameplayEffects` | `Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Public/AbilitySystemComponent.h:1880` | `FActiveGameplayEffectsContainer` | 保存 active GE，负责聚合、持续效果、属性变化委托和查询。 | `ApplyGameplayEffectSpecToSelf` 添加/执行，`RemoveActiveGameplayEffect` 移除，`SetNumericAttributeBase` 修改 attribute base；源码路径：`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Private/AbilitySystemComponent.cpp:798`、`:1041`、`:386`。 | 通常用 ASC 的 Apply/Remove/Get API；`GetActiveGameplayEffects()` 可读容器但业务层直接改容器不推荐。 |
| `SpawnedAttributes` | `Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Public/AbilitySystemComponent.h:1956` | `TArray<TObjectPtr<UAttributeSet>>` | ASC 管理的 AttributeSet 列表，带复制和 `OnRep_SpawnedAttributes`。 | `InitStats` 通过 `GetOrCreateAttributeSubobject` 创建，`AddSpawnedAttribute` 添加，`RemoveSpawnedAttribute`/`RemoveAllSpawnedAttributes` 移除；源码路径：`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Private/AbilitySystemComponent.cpp:83`、`:104`、`:3005`、`:3024`。 | 通常用 `AddAttributeSetSubobject`、`AddSpawnedAttribute`、`GetAttributeSet`；不要绕过 API 直接改数组。 |
| `OwnedGameplayTags` | 未发现同名成员；接口返回 `GameplayTagCountContainer.GetExplicitGameplayTags()` | `const FGameplayTagContainer&` 返回值 | 表示 ASC 当前拥有的显式 GameplayTags。 | `UpdateTagMap`/`UpdateTagMap_Internal` 修改 loose tags；GE 与 GameplayCue 也会影响 tag count，具体 GE 内部更新路径本轮未完整展开，未确认；源码路径：`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Public/AbilitySystemComponent.h:597`、`:620`、`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Public/AbilitySystemComponent.h:1905`。 | 业务层通常通过 `HasMatchingGameplayTag`/`GetOwnedGameplayTags` 查询。 |
| `BlockedAbilityTags` | `Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Public/AbilitySystemComponent.h:1891` | `FGameplayTagCountContainer` | 这些 tag 对应的 Ability 不能激活。 | `BlockAbilitiesWithTags` +1，`UnBlockAbilitiesWithTags` -1，`ApplyAbilityBlockAndCancelTags` 根据 Ability 的阻塞/取消配置调用它们；源码路径：`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Private/AbilitySystemComponent_Abilities.cpp:1410`、`:1433`、`:1438`。 | 通常通过 Ability 阻塞配置或 `BlockAbilitiesWithTags` 操作，不直接改成员。 |
| `ActiveGameplayCues` | `Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Public/AbilitySystemComponent.h:1883` | `FActiveGameplayCueContainer` | 保存非 GE 直接添加的持久 GameplayCue。 | `AddGameplayCue` 添加，`RemoveGameplayCue` 移除，`RemoveAllGameplayCues` 清理；源码路径：`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Private/AbilitySystemComponent.cpp:1312`、`:1401`、`:1423`。 | 通常用 ASC 或 GameplayCueManager 包装接口；头文件注释提示不要直接调用 multicast 函数，应走 GameplayCueManager wrapper；源码路径：`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Public/AbilitySystemComponent.h:879`。 |
| `MinimalReplicationGameplayCues` | `Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Public/AbilitySystemComponent.h:1887` | `FActiveGameplayCueContainer` | minimal replication 模式下，用于复制本应来自 Active GE 的 GameplayCue。 | `AddGameplayCue_MinimalReplication`/`RemoveGameplayCue_MinimalReplication` 修改；复制条件由 `UpdateMinimalReplicationGameplayCuesCondition` 更新；源码路径：`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Private/AbilitySystemComponent.cpp:1322`、`:1406`、`:1679`。 | 通常不由业务层直接操作，除非明确处理 replication mode。 |
| `GameplayTagCountContainer` | `Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Public/AbilitySystemComponent.h:1905` | `FGameplayTagCountContainer` | 所有 gameplay tags 的加速计数容器，注释说明包含 GE 的 OwnedGameplayTags 与显式 GameplayCueTags。 | `UpdateTagMap`/`ResetTagMap`/GE 相关路径修改；GE 内部细节本轮未展开，未确认；源码路径：`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Public/AbilitySystemComponent.h:620`、`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Private/AbilitySystemComponent.cpp:763`。 | 业务层通常用接口查询，不直接改。 |
| `ReplicatedLooseTags` | `Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Public/AbilitySystemComponent.h:1966` | `FMinimalReplicationTagCountMap` | 可复制的 loose gameplay tag 容器。 | `AddReplicatedLooseGameplayTag`/`RemoveReplicatedLooseGameplayTag`/`SetReplicatedLooseGameplayTagCount` 通过 mutable accessor 修改并标记 dirty；源码路径：`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Public/AbilitySystemComponent.h:690`、`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Private/AbilitySystemComponent.cpp:3302`。 | 业务层可用公开函数操作，注意 replicated loose tag 会覆盖 simulated proxy 上本地设置的 tag count；源码路径：`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Public/AbilitySystemComponent.h:690`。 |
| `ScopedPredictionKey` | `Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Public/AbilitySystemComponent.h:267` | `FPredictionKey` | 当前预测窗口使用的 prediction key。 | `FScopedPredictionWindow` 设置；LocalPredicted 激活时生成并用于 RPC；源码路径：`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Private/AbilitySystemComponent_Abilities.cpp:1921`。 | 业务层通常不直接写，使用 ability/ASC 预测窗口相关 API。 |
| `ReplicatedPredictionKeyMap` | `Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Public/AbilitySystemComponent.h:1976` | `FReplicatedPredictionKeyMap` | 复制 prediction key，用于客户端 catch-up/确认。 | 作为 owner-only replicated property 复制；源码强调必须放在 ASC replicated properties 最后；源码路径：`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Private/AbilitySystemComponent.cpp:1651`。 | 业务层通常不直接访问。 |
| `PendingServerActivatedAbilities` | `Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Public/AbilitySystemComponent.h:320` | `TArray<FPendingAbilityInfo>` | 与服务器发起或待确认激活相关；具体完整流程本轮未展开，未确认。 | 未确认。 | 不建议业务层直接访问。 |
| `AbilityTargetDataMap` | `Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Public/AbilitySystemComponent.h:1698` | `FGameplayAbilityReplicatedDataContainer` | 按 Ability spec 跟踪 replicated target data 与回调。 | Server/Client replicated target data RPC 修改；完整流程本轮未展开，未确认；源码路径：`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Public/AbilitySystemComponent.h:1601`。 | 通常通过 AbilityTask target data API，而不是直接访问。 |

## 四、关键函数索引

### Ability 相关

| 函数 | 源码路径 | 用途 | 业务层常用 |
|---|---|---|---|
| `GiveAbility` | `Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Public/AbilitySystemComponent.h:977`；实现 `Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Private/AbilitySystemComponent_Abilities.cpp:272` | 授予 Ability，authority-only，加入 `ActivatableAbilities`，必要时创建 InstancedPerActor 实例。 | 常用，通常服务端授予。 |
| `ClearAbility` | `Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Public/AbilitySystemComponent.h:1030`；实现 `Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Private/AbilitySystemComponent_Abilities.cpp:475` | 按 handle 移除 Ability，客户端调用会报错并返回。 | 常用，通常服务端移除。 |
| `ClearAllAbilities` | `Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Public/AbilitySystemComponent.h:1012`；实现 `Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Private/AbilitySystemComponent_Abilities.cpp:416` | 清空所有已授予 Ability，authority-only。 | 偶尔用于重置/销毁。 |
| `TryActivateAbility` | `Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Public/AbilitySystemComponent.h:1068`；实现 `Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Private/AbilitySystemComponent_Abilities.cpp:1583` | 按 handle 尝试激活，检查 spec、ActorInfo、网络策略，必要时发起远端激活。 | 常用。 |
| `TryActivateAbilitiesByTag` | `Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Public/AbilitySystemComponent.h:1052`；实现 `Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Private/AbilitySystemComponent_Abilities.cpp:1541` | 找到满足标签与 tag requirements 的 Ability 并逐个尝试激活。 | 常用，适合按能力标签触发。 |
| `InternalTryActivateAbility` | `Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Public/AbilitySystemComponent.h:1220`；实现 `Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Private/AbilitySystemComponent_Abilities.cpp:1683` | 内部激活主流程，检查网络、事件响应、`CanActivateAbility`、实例化、预测并调用 `CallActivateAbility`。 | 通常不在业务层直接调用。 |
| `CancelAbility` | `Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Public/AbilitySystemComponent.h:1080`；实现 `Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Private/AbilitySystemComponent_Abilities.cpp:1280` | 按 Ability CDO 找到 spec 并取消。 | 偶尔用，更多用 handle/tag 取消。 |
| `CancelAbilities` | `Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Public/AbilitySystemComponent.h:1086`；实现 `Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Private/AbilitySystemComponent_Abilities.cpp:1305` | 按 WithTags/WithoutTags 筛选 active Ability 并取消。 | 常用。 |
| `FindAbilitySpecFromHandle` | `Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Public/AbilitySystemComponent.h:1143`；实现 `Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Private/AbilitySystemComponent_Abilities.cpp:896` | 通过 handle 查找 spec；源码提示返回指针易失，修改后要 `MarkAbilitySpecDirty`。 | 查询常用，直接修改需谨慎。 |

### GameplayEffect 相关

| 函数 | 源码路径 | 用途 | 业务层常用 |
|---|---|---|---|
| `ApplyGameplayEffectToSelf` | `Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Public/AbilitySystemComponent.h:797`；实现 `Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Private/AbilitySystemComponent.cpp:589` | 用 GE CDO/Level/Context 构造 spec 并应用到自己；需要 authority 或有效 prediction key。 | 常用。 |
| `ApplyGameplayEffectToTarget` | `Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Public/AbilitySystemComponent.h:792`；实现 `Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Private/AbilitySystemComponent.cpp:554` | 构造 spec 并调用 target 的 `ApplyGameplayEffectSpecToSelf`。 | 常用。 |
| `ApplyGameplayEffectSpecToSelf` | `Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Public/AbilitySystemComponent.h:336`；实现 `Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Private/AbilitySystemComponent.cpp:798` | 应用已构造 spec，执行 authority/prediction、CanApply、modifier attribute 检查，并写入/执行 ActiveGE。 | 常用，尤其 Ability 内部。 |
| `ApplyGameplayEffectSpecToTarget` | `Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Public/AbilitySystemComponent.h:330`；实现 `Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Private/AbilitySystemComponent.cpp:777` | 将 spec 应用到目标 ASC；可根据全局设置清掉 target GE prediction key。 | 常用。 |
| `RemoveActiveGameplayEffect` | `Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Public/AbilitySystemComponent.h:349`；实现 `Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Private/AbilitySystemComponent.cpp:1041` | 按 ActiveGE handle 移除，默认要求 authority，最终委托给 ActiveGE 容器。 | 常用，服务端为主。 |
| `GetActiveGameplayEffect` | `Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Public/AbilitySystemComponent.h:459`；实现 `Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Private/AbilitySystemComponent.cpp:336` | 按 handle 返回 const `FActiveGameplayEffect*`。 | 查询可用，不应直接改。 |
| `MakeOutgoingSpec` | `Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Public/AbilitySystemComponent.h:363`；实现 `Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Private/AbilitySystemComponent.cpp:451` | 创建带 Context/Level 的 outgoing `FGameplayEffectSpecHandle`。 | 常用。 |

### Attribute 相关

| 函数 | 源码路径 | 用途 | 业务层常用 |
|---|---|---|---|
| `GetNumericAttribute` | `Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Public/AbilitySystemComponent.h:245`；实现 `Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Private/AbilitySystemComponent.cpp:409` | 读取 AttributeSet 上的当前数值；找不到返回 0。 | 常用。 |
| `SetNumericAttributeBase` | `Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Public/AbilitySystemComponent.h:224`；实现 `Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Private/AbilitySystemComponent.cpp:386` | 通过 ActiveGE 容器设置 attribute base value，已有 modifiers 不清除。 | 可用，但业务更常通过 GE 修改属性。 |
| `GetGameplayAttributeValueChangeDelegate` | `Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Public/AbilitySystemComponent.h:534`；实现 `Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Private/AbilitySystemComponent.cpp:732` | 获取属性值变化委托。 | 常用。 |
| `InitStats` | `Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Public/AbilitySystemComponent.h:168`；实现 `Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Private/AbilitySystemComponent.cpp:83` | 创建/获取 AttributeSet 并从 DataTable 初始化；源码注释说“不太受支持，带 curve table reference 的 GE 可能更好”。 | 可用但谨慎。 |
| `AddAttributeSetSubobject` | `Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Public/AbilitySystemComponent.h:152` | 模板函数，把 ASC 子对象 AttributeSet 添加到 `SpawnedAttributes`。 | 常用，尤其组件/Actor 构造出的 AttributeSet。 |

### Tag 相关

| 函数 | 源码路径 | 用途 | 业务层常用 |
|---|---|---|---|
| `HasMatchingGameplayTag` | `Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Public/AbilitySystemComponent.h:576` | 委托 `GameplayTagCountContainer` 查询单个 tag。 | 常用。 |
| `HasAllMatchingGameplayTags` | `Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Public/AbilitySystemComponent.h:581` | 查询是否拥有全部 tags。 | 常用。 |
| `HasAnyMatchingGameplayTags` | `Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Public/AbilitySystemComponent.h:586` | 查询是否拥有任意 tag。 | 常用。 |
| `AddLooseGameplayTag` | `Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Public/AbilitySystemComponent.h:654` | 添加非 GE 支撑的 loose tag；源码注释明确不复制。 | 常用但要自己处理网络。 |
| `RemoveLooseGameplayTag` | `Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Public/AbilitySystemComponent.h:664` | 移除 loose tag；源码注释明确不复制。 | 常用但要自己处理网络。 |

## 未确认项

- `AbilitySystemComponent_GameplayEffects.cpp` 与 `AbilitySystemComponent_Replication.cpp` 在当前路径不存在，相关实现未确认是否在其他分支拆分。
- Character vs PlayerState 放置 ASC 的最佳实践，本轮指定源码只确认 Owner/Avatar 分离，不直接规定放置策略。
- `GameplayTagCountContainer` 中 GE granted tags 的完整写入链路在 `FActiveGameplayEffectsContainer` 内，本轮没有完全展开，未确认。
- `PendingServerActivatedAbilities` 的完整生命周期本轮未展开，未确认。

# 核心类：UGameplayAbility（第三轮）

本轮重点阅读：

- `Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Public/Abilities/GameplayAbility.h`
- `Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Private/Abilities/GameplayAbility.cpp`
- `Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Public/GameplayAbilitySpec.h`
- `Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Public/GameplayAbilitySpecHandle.h`

补充阅读：

- `Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Public/Abilities/GameplayAbilityTypes.h`
- `Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Public/AbilitySystemComponent.h`
- `Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Private/AbilitySystemComponent_Abilities.cpp`

## 一、类定位

- `UGameplayAbility` 定义“可由玩家或外部游戏逻辑激活的自定义玩法逻辑”；源码路径：`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Public/Abilities/GameplayAbility.h:108`。
- `UGameplayAbility` 继承自 `UObject` 并实现 `IGameplayTaskOwnerInterface`；源码路径：`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Public/Abilities/GameplayAbility.h:109`。
- `UGameplayAbility` 与 `UAbilitySystemComponent` 的关系：ASC 是授予、保存、激活、取消 Ability 的组件；`UGameplayAbility` 把 ASC 声明为 friend，激活时 ASC 查 `FGameplayAbilitySpec`，然后调用 Ability 的 `CanActivateAbility` / `CallActivateAbility`；源码路径：`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Public/Abilities/GameplayAbility.h:115`、`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Private/AbilitySystemComponent_Abilities.cpp:1683`。
- `UGameplayAbility` 与 `FGameplayAbilitySpec` 的关系：Spec 保存外部引用 handle、Ability CDO、Level、InputID、SourceObject、ActiveCount、实例数组、授予它的 GE handle 和 dynamic tags；源码路径：`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Public/GameplayAbilitySpec.h:192`、`:196`、`:200`、`:204`、`:208`、`:212`、`:252`、`:259`。
- `FGameplayAbilitySpecHandle` 是“指向特定 granted ability 的全局唯一 handle”，构造默认无效，`GenerateNewHandle` 用静态递增整数生成；源码路径：`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Public/GameplayAbilitySpecHandle.h:13`、`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Private/GameplayAbilitySpecHandle.cpp:9`。
- Cost / Cooldown 与 GameplayEffect 的关系：`CostGameplayEffectClass` 表示提交 Ability 时应用的消耗 GE；`CooldownGameplayEffectClass` 表示提交 Ability 时应用、到期前阻止再次使用的冷却 GE；源码路径：`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Public/Abilities/GameplayAbility.h:741`、`:749`。
- AbilityTask 关系：`UGameplayAbility` 实现 `IGameplayTaskOwnerInterface`，维护 `ActiveTasks`，Ability 结束时会对 active tasks 调用 `TaskOwnerEnded` 并清空数组；源码路径：`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Public/Abilities/GameplayAbility.h:530`、`:818`、`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Private/Abilities/GameplayAbility.cpp:819`。本轮不展开 AbilityTask 具体实现。

## 二、生命周期函数

| 函数 | 源码路径 | 调用时机 | 主要作用 | 业务层常重写 | Blueprint 对应 | 常见失败原因 |
|---|---|---|---|---|---|---|
| `CanActivateAbility` | 声明 `Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Public/Abilities/GameplayAbility.h:281`；实现 `Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Private/Abilities/GameplayAbility.cpp:424` | ASC `InternalTryActivateAbility` 中调用；源码路径：`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Private/AbilitySystemComponent_Abilities.cpp:1795`。 | 无副作用检查 Avatar/role、ASC、Spec、用户激活抑制、Cooldown、Cost、Tag Requirements、Input block、蓝图 CanActivate。 | C++ 可重写，常用于额外条件；但需保留父类语义，具体重写策略未确认。 | `K2_CanActivateAbility`；源码路径：`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Public/Abilities/GameplayAbility.h:567`。 | Avatar 无效、simulated proxy 或 NetSecurity 不允许、Spec 无效、激活被 UI/输入抑制、冷却/消耗不满足、Tag 缺失或被阻塞、InputID 被阻塞、蓝图返回 false；源码路径：`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Private/Abilities/GameplayAbility.cpp:428`、`:453`、`:475`、`:485`、`:495`、`:505`、`:516`。 |
| `CallActivateAbility` | 声明 `Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Public/Abilities/GameplayAbility.h:603`；实现 `Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Private/Abilities/GameplayAbility.cpp:1006` | ASC 通过该函数进入 Ability 激活；源码路径：`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Private/AbilitySystemComponent_Abilities.cpp:1897`、`:1901`、`:1948`、`:1957`。 | 固定执行 `PreActivate`，然后执行 `ActivateAbility`。 | 不常重写；它不是 virtual。 | 无直接蓝图节点。 | `PreActivate` 或 `ActivateAbility` 内部失败；本函数本身无显式失败返回。 |
| `PreActivate` | 声明 `Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Public/Abilities/GameplayAbility.h:600`；实现 `Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Private/Abilities/GameplayAbility.cpp:920` | `CallActivateAbility` 第一阶段。 | 设置 active/cancel/block 状态、写入 CurrentInfo、保存 event data、添加 `ActivationOwnedTags`、通知 ASC activated、应用 Block/Cancel tags、递增 Spec ActiveCount。 | 通常不重写；如果重写风险较高，源码未给项目推荐。 | 无直接蓝图事件。 | Spec 找不到会记录 warning 并返回；源码路径：`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Private/Abilities/GameplayAbility.cpp:988`。 |
| `ActivateAbility` | 声明 `Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Public/Abilities/GameplayAbility.h:597`；实现 `Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Private/Abilities/GameplayAbility.cpp:884` | `PreActivate` 后由 `CallActivateAbility` 调用。 | 执行 Ability 的主体逻辑；蓝图走 `K2_ActivateAbility` 或 `K2_ActivateAbilityFromEvent`；Native 子类应 override。 | 常重写，是 Ability 业务主体。 | `K2_ActivateAbility`、`K2_ActivateAbilityFromEvent`；源码路径：`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Public/Abilities/GameplayAbility.h:588`、`:591`。 | 如果蓝图只实现 FromEvent 但没有事件数据，会 warning 并取消结束；源码明确 Activate 图应调用 `CommitAbility` 和 `EndAbility`；源码路径：`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Private/Abilities/GameplayAbility.cpp:896`、`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Public/Abilities/GameplayAbility.h:577`。 |
| `CommitAbility` | 声明 `Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Public/Abilities/GameplayAbility.h:355`；实现 `Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Private/Abilities/GameplayAbility.cpp:559` | Ability 实现主动调用；ASC 不会自动调用。 | 做最后一次 `CommitCheck`，成功后执行 `CommitExecute`、蓝图 `K2_CommitExecute`，并通知 ASC commit。 | 常调用，通常不必重写；特殊资源逻辑可重写，未确认。 | `K2_CommitAbility`；源码路径：`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Private/Abilities/GameplayAbility.cpp:1389`。 | Handle/ActorInfo/Spec 无效、Cooldown 或 Cost 在提交时失败；源码路径：`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Private/Abilities/GameplayAbility.cpp:615`。 |
| `CommitCheck` | 声明 `Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Public/Abilities/GameplayAbility.h:359`；实现 `Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Private/Abilities/GameplayAbility.cpp:615` | `CommitAbility` 内部调用。 | 最后一次检查是否还能提交；不直接调用 `CanActivateAbility`，只检查 Spec 有效性、Cooldown、Cost。 | 可重写但需谨慎；源码注释强调它和 CanActivate 不完全相同。 | 无直接蓝图事件。 | 激活后状态变化导致资源/冷却条件不再满足；源码路径：`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Private/Abilities/GameplayAbility.cpp:617`。 |
| `CommitExecute` | 声明 `Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Public/Abilities/GameplayAbility.h:366`；实现 `Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Private/Abilities/GameplayAbility.cpp:651` | `CommitAbility` 的执行阶段。 | 依次 `ApplyCooldown` 与 `ApplyCost`。 | 有特殊提交效果时可重写，未确认。 | `K2_CommitExecute`；源码路径：`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Public/Abilities/GameplayAbility.h:362`。 | 通常前置失败在 `CommitCheck`；应用 GE 失败细节未展开，未确认。 |
| `CommitAbilityCost` | 声明 `Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Public/Abilities/GameplayAbility.h:357`；实现 `Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Private/Abilities/GameplayAbility.cpp:598` | 只提交 cost 时由 Ability/蓝图主动调用。 | 可受全局 `ShouldIgnoreCosts` 绕过；否则 `CheckCost` 后 `ApplyCost`。 | 可用于分离消耗和冷却。 | `K2_CommitAbilityCost`；源码路径：`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Private/Abilities/GameplayAbility.cpp:1408`。 | Cost GE 不可应用、ASC 无效、资源不足；源码路径：`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Private/Abilities/GameplayAbility.cpp:1092`。 |
| `CommitAbilityCooldown` | 声明 `Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Public/Abilities/GameplayAbility.h:356`；实现 `Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Private/Abilities/GameplayAbility.cpp:578` | 只提交 cooldown 时由 Ability/蓝图主动调用。 | 可受全局 `ShouldIgnoreCooldowns` 绕过；非 Force 时先 `CheckCooldown`，再 `ApplyCooldown`。 | 可用于先上冷却或拆分提交。 | `K2_CommitAbilityCooldown`；源码路径：`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Private/Abilities/GameplayAbility.cpp:1395`。 | Cooldown tags 已存在时失败；源码路径：`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Private/Abilities/GameplayAbility.cpp:1050`。 |
| `EndAbility` | 声明 `Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Public/Abilities/GameplayAbility.h:627`；实现 `Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Private/Abilities/GameplayAbility.cpp:769` | Ability 自然结束、取消、远端结束或业务主动结束时调用。 | 校验结束合法性，触发 `K2_OnEndAbility`，清理 latent/timer、结束 tasks、复制结束、移除 activation tags / tracked cues、清理 replicated data cache、通知 ASC ended。 | 常在业务逻辑中调用；C++ 重写需调用 Super，源码坑点由第二轮 ASC ensure 可佐证。 | `K2_EndAbility`、`K2_EndAbilityLocally`、`K2_OnEndAbility`；源码路径：`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Private/Abilities/GameplayAbility.cpp:1433`、`:1442`、`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Public/Abilities/GameplayAbility.h:620`。 | 重复 End、ASC 无效、Spec 不 active 会返回 false；忘记调用会导致 active 状态不降、任务不清；源码路径：`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Private/Abilities/GameplayAbility.cpp:738`。 |
| `CancelAbility` | 声明 `Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Public/Abilities/GameplayAbility.h:317`；实现 `Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Private/Abilities/GameplayAbility.cpp:708` | ASC `CancelAbilitySpec` 或蓝图 `K2_CancelAbility` 调用。 | 若可取消，必要时复制 cancel，广播 cancel delegate，然后以 cancelled=true 调用 `EndAbility`。 | 可重写；通常业务用取消事件/End 清理。 | `K2_CancelAbility`；源码路径：`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Public/Abilities/GameplayAbility.h:320`。 | `CanBeCanceled` false、scope lock 延迟、ActorInfo/ASC 无效导致无法复制；源码路径：`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Private/Abilities/GameplayAbility.cpp:710`。 |

## 三、实例化策略

定义位置：`EGameplayAbilityInstancingPolicy` 位于 `Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Public/Abilities/GameplayAbilityTypes.h:35`。`UGameplayAbility::InstancingPolicy` 成员位于 `Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Public/Abilities/GameplayAbility.h:713`。

| 策略 | 含义 | 成员变量安全性 | AbilityTask 适配 | 性能差异 | 常见场景 | 常见坑 |
|---|---|---|---|---|---|---|
| `NonInstanced` | 不创建实例，执行在 CDO 上；源码路径：`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Public/Abilities/GameplayAbilityTypes.h:44`。 | 不能保存 per-activation 状态；源码注释明确 NonInstanced cannot have state；源码路径：`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Public/Abilities/GameplayAbility.h:69`。 | 不适合异步 AbilityTask；源码依据：CDO 无状态、无 ability RPC；AbilityTask 需要 owner ability 维护 `ActiveTasks`；源码路径：`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Public/Abilities/GameplayAbility.h:69`、`:818`。 | 根据源码推断，分配最少，因为不 `NewObject` Ability 实例；源码路径：`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Private/AbilitySystemComponent_Abilities.cpp:1176`。 | 简单、无状态、立即完成逻辑；示例注释提到 `GameplayAbility_Montage` 是 non-instanced 示例；源码路径：`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Public/Abilities/GameplayAbility.h:64`。 | UE 5.5 标记 deprecated，`GetInstancingPolicy` 在未允许 NonInstanced 时会把 NonInstanced 视为 InstancedPerActor；源码路径：`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Public/Abilities/GameplayAbilityTypes.h:45`、`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Private/Abilities/GameplayAbility.cpp:102`。 |
| `InstancedPerActor` | 每个 actor 拥有一个 Ability 实例，可保存状态、支持复制；源码路径：`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Public/Abilities/GameplayAbilityTypes.h:47`。 | 可保存跨激活的实例状态，但要在 End/PreActivate 中正确重置；源码路径：`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Private/Abilities/GameplayAbility.cpp:943`、`:812`。 | 适合异步 AbilityTask；Ability 实例维护 `ActiveTasks` 并在 End 清理；源码路径：`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Private/Abilities/GameplayAbility.cpp:1551`、`:819`。 | 根据源码推断，授予时创建一次实例；源码路径：`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Private/AbilitySystemComponent_Abilities.cpp:299`。 | 持续 Ability、含 latent/task、需要实例状态或复制的 Ability。 | 已 active 时默认不能再次激活，除非 `bRetriggerInstancedAbility` 为 true；源码路径：`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Private/AbilitySystemComponent_Abilities.cpp:1810`。 |
| `InstancedPerExecution` | 每次执行创建新实例；复制当前不支持；源码路径：`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Public/Abilities/GameplayAbilityTypes.h:50`。 | 可保存本次执行状态，结束后实例被清理；源码路径：`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Private/AbilitySystemComponent_Abilities.cpp:1240`。 | 可用于异步 task，但输入交互有警告：InstancedPerExecution 对 input 不可靠，只能与最新实例交互；源码路径：`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Private/AbilitySystemComponent_Abilities.cpp:2799`、`:2835`。 | 根据源码推断，每次激活 `NewObject`，分配最多；源码路径：`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Private/AbilitySystemComponent_Abilities.cpp:1893`。 | 一次性并发执行、每次独立状态。 | LocalPredicted + InstancedPerExecution 只有 `ReplicateNo` 当前可预测，否则报错；源码路径：`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Private/AbilitySystemComponent_Abilities.cpp:1939`。 |

构造函数显式默认 `InstancingPolicy = InstancedPerExecution`、`NetSecurityPolicy = ClientOrServer`；本轮未发现构造函数显式设置 `NetExecutionPolicy`，默认值未确认；源码路径：`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Private/Abilities/GameplayAbility.cpp:91`、`:93`。

## 四、网络执行策略

定义位置：`EGameplayAbilityNetExecutionPolicy` 位于 `Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Public/Abilities/GameplayAbilityTypes.h:55`。`UGameplayAbility::NetExecutionPolicy` 成员位于 `Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Public/Abilities/GameplayAbility.h:733`。

| 策略 | 含义 | ASC 激活链处理 | 客户端能否直接激活 | 服务端是否确认 | 常见场景 | 常见坑 |
|---|---|---|---|---|---|---|
| `LocalOnly` | 只在拥有 local control 的客户端或服务端运行；源码路径：`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Public/Abilities/GameplayAbilityTypes.h:64`。 | 非本地时 ASC 可转发到 client；`InternalTryActivateAbility` 在非本地拒绝；源码路径：`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Private/AbilitySystemComponent_Abilities.cpp:1626`、`:1742`。 | local controlled 端可以。 | 不需要服务端确认执行；但具体业务效果是否需要服务端另行处理未确认。 | 纯本地表现、UI/镜头类能力。 | 在非本地上下文触发会失败或被转发；源码路径：`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Private/AbilitySystemComponent_Abilities.cpp:1745`。 |
| `LocalPredicted` | 本地客户端可预测执行；源码路径：`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Public/Abilities/GameplayAbilityTypes.h:61`。 | ASC 创建 `FScopedPredictionWindow`，`ActivationInfo.SetPredicting`，立即 RPC 到服务端并本地 `CallActivateAbility`；源码路径：`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Private/AbilitySystemComponent_Abilities.cpp:1904`。 | 可以在本地预测激活。 | 需要服务端通过 `InternalServerTryActivateAbility` 接受或 `ClientActivateAbilityFailed` 拒绝；源码路径：`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Private/AbilitySystemComponent_Abilities.cpp:2026`。 | 输入响应敏感且需服务端校验的 Ability。 | 预测失败清理、GE/Task 回滚需要处理；完整预测 catch-up 本轮未展开，未确认。 |
| `ServerOnly` | 只在服务端运行；源码路径：`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Public/Abilities/GameplayAbilityTypes.h:70`。 | 非 authority 会走 server activation 请求或在 internal 中拒绝；服务端分支不会通知 client 成功执行；源码路径：`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Private/AbilitySystemComponent_Abilities.cpp:1639`、`:1761`、`:1874`。 | 不能直接本地执行，只能请求服务端。 | 服务端实际执行。 | 权威判定、服务端逻辑。 | 客户端误配为直接播放表现会无本地响应；源码只确认 server-only 不在客户端执行。 |
| `ServerInitiated` | 服务端发起，但如果存在 local client 也会运行；源码路径：`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Public/Abilities/GameplayAbilityTypes.h:67`。 | 服务端创建 server initiated prediction key，并在非本地且非 ServerOnly 时通知 client succeed；源码路径：`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Private/AbilitySystemComponent_Abilities.cpp:1853`、`:1874`。 | 客户端不能自行 internal 执行；非 authority 会请求服务端。 | 服务端发起/确认。 | 服务端触发、客户端需要同步表现的 Ability。 | 结束/取消复制只对 LocalPredicted 或 ServerInitiated 策略处理；源码路径：`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Private/AbilitySystemComponent_Abilities.cpp:2099`。 |

`NetSecurityPolicy` 提供额外保护：例如 `ServerOnlyExecution` 或 `ServerOnly` 会让服务端拒绝客户端激活请求；源码路径：`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Public/Abilities/GameplayAbilityTypes.h:75`、`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Private/AbilitySystemComponent_Abilities.cpp:2058`。

## 五、Cost 和 Cooldown

- `GetCostGameplayEffect` / `GetCooldownGameplayEffect` 分别返回对应 class 的 CDO，未配置则返回 `nullptr`；源码路径：`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Private/Abilities/GameplayAbility.cpp:1026`、`:1038`。
- `CheckCooldown` 通过 `GetCooldownTags()` 取得 Cooldown GE granted tags，若 ASC 拥有任一 cooldown tag 则失败，并添加 `ActivateFailCooldownTag` 与匹配 tag；源码路径：`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Private/Abilities/GameplayAbility.cpp:1050`、`:1197`。
- `CheckCost` 通过 ASC 的 `CanApplyAttributeModifiers(CostGE, Level, Context)` 判断 Cost GE 是否可应用，失败时添加 `ActivateFailCostTag`；源码路径：`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Private/Abilities/GameplayAbility.cpp:1092`。
- `ApplyCooldown` / `ApplyCost` 都把对应 GE 应用到 owner ASC；源码路径：`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Private/Abilities/GameplayAbility.cpp:1083`、`:1115`。
- `CanActivateAbility` 阶段检查 cooldown、cost 和 tag requirements；源码路径：`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Private/Abilities/GameplayAbility.cpp:475`、`:485`、`:495`。
- `CommitAbility` 阶段再次检查 cooldown/cost，再实际应用 cooldown/cost；源码路径：`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Private/Abilities/GameplayAbility.cpp:559`、`:615`、`:651`。
- CanActivate 成功后 Commit 仍可能失败：源码注释明确 Ability 可先播放动画、等待确认/target data，再提交；期间资源或 cooldown 状态可能变化；源码路径：`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Private/Abilities/GameplayAbility.cpp:617`。
- 常见业务写法（根据源码连接点总结）：在 `ActivateAbility` / `K2_ActivateAbility` 中主动调用 `CommitAbility`；若返回 false，应调用 `EndAbility` 或取消逻辑；源码路径：`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Private/Abilities/GameplayAbility.cpp:884`。

## 六、Tag Requirements

| Tag 成员 | 定义位置 | 作用 | 生效点 |
|---|---|---|---|
| `AbilityTags` | `Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Public/Abilities/GameplayAbility.h:496` | Ability 自身 asset tags；运行时可与 Spec dynamic tags 共同成为 Ability tags。 | `DoesAbilitySatisfyTagRequirements` 用 `GetAssetTags()` 检查 ASC `BlockedAbilityTags`；源码路径：`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Private/Abilities/GameplayAbility.cpp:374`。 |
| `ActivationOwnedTags` | `Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Public/Abilities/GameplayAbility.h:765` | Ability active 期间加到 owner 上的 tags，可按全局设置复制。 | `PreActivate` 添加，`EndAbility` 移除；源码路径：`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Private/Abilities/GameplayAbility.cpp:963`、`:837`。 |
| `ActivationRequiredTags` | `Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Public/Abilities/GameplayAbility.h:769` | 激活者 ASC 必须拥有全部这些 tags。 | `DoesAbilitySatisfyTagRequirements` 检查 ASC owned tags；源码路径：`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Private/Abilities/GameplayAbility.cpp:386`。 |
| `ActivationBlockedTags` | `Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Public/Abilities/GameplayAbility.h:773` | 激活者 ASC 拥有任一这些 tags 时阻止激活。 | `DoesAbilitySatisfyTagRequirements` 检查 ASC owned tags；源码路径：`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Private/Abilities/GameplayAbility.cpp:375`。 |
| `SourceRequiredTags` | `Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Public/Abilities/GameplayAbility.h:777` | SourceTags 必须满足全部 required。 | 有 `SourceTags` 时检查；源码路径：`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Private/Abilities/GameplayAbility.cpp:387`。 |
| `SourceBlockedTags` | `Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Public/Abilities/GameplayAbility.h:781` | SourceTags 命中则阻止。 | 有 `SourceTags` 时检查；源码路径：`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Private/Abilities/GameplayAbility.cpp:376`。 |
| `TargetRequiredTags` | `Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Public/Abilities/GameplayAbility.h:785` | TargetTags 必须满足全部 required。 | 有 `TargetTags` 时检查；源码路径：`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Private/Abilities/GameplayAbility.cpp:391`。 |
| `TargetBlockedTags` | `Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Public/Abilities/GameplayAbility.h:789` | TargetTags 命中则阻止。 | 有 `TargetTags` 时检查；源码路径：`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Private/Abilities/GameplayAbility.cpp:380`。 |
| `CancelAbilitiesWithTag` | `Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Public/Abilities/GameplayAbility.h:757` | 本 Ability 执行时取消带这些 tags 的 Ability。 | `PreActivate` 通过 ASC `ApplyAbilityBlockAndCancelTags` 执行取消；源码路径：`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Private/Abilities/GameplayAbility.cpp:985`。 |
| `BlockAbilitiesWithTag` | `Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Public/Abilities/GameplayAbility.h:761` | 本 Ability active 期间阻塞带这些 tags 的 Ability。 | `PreActivate` 添加阻塞，`EndAbility` 移除阻塞；源码路径：`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Private/Abilities/GameplayAbility.cpp:985`、`:865`。 |

ASC `BlockedAbilityTags` 的关系：Ability 的 `BlockAbilitiesWithTag` 通过 ASC `BlockAbilitiesWithTags` 写入 `BlockedAbilityTags`，后续其他 Ability 的 `DoesAbilitySatisfyTagRequirements` 会用自身 `AbilityTags` 与 ASC `BlockedAbilityTags` 匹配；源码路径：`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Private/AbilitySystemComponent_Abilities.cpp:1410`、`:1433`、`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Private/Abilities/GameplayAbility.cpp:374`。

常见配置错误（基于源码路径）：Ability 自身 `AbilityTags` 为空会导致 `BlockAbilitiesWithTag` 难以命中目标 Ability；ActivationRequiredTags 与当前 ASC owned tags 不匹配会失败；Cooldown GE 没有 granted tags 会让 `CheckCooldown` 没有 tag 可查；源码路径：`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Private/Abilities/GameplayAbility.cpp:374`、`:386`、`:1057`。

## 七、关键成员变量

| 成员 | 定义位置 | 类型 | 作用 | 开发者是否常用 | 是否应直接修改 |
|---|---|---|---|---|---|
| `InstancingPolicy` | `Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Public/Abilities/GameplayAbility.h:713` | `TEnumAsByte<EGameplayAbilityInstancingPolicy::Type>` | 决定 Ability 执行时是否以及如何创建实例。 | 常在 Ability 默认配置中设置。 | 应通过类默认值/蓝图 defaults 设置，不在运行时随意改。 |
| `NetExecutionPolicy` | `Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Public/Abilities/GameplayAbility.h:733` | `TEnumAsByte<EGameplayAbilityNetExecutionPolicy::Type>` | 决定 Ability 在网络上由本地预测、仅本地、服务端发起或仅服务端执行。 | 常配置。 | 应通过 defaults 设置。 |
| `NetSecurityPolicy` | `Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Public/Abilities/GameplayAbility.h:737` | `TEnumAsByte<EGameplayAbilityNetSecurityPolicy::Type>` | 保护客户端执行/结束请求。 | 有网络安全需求时配置。 | 应通过 defaults 设置。 |
| `AbilityTags` | `Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Public/Abilities/GameplayAbility.h:496` | `FGameplayTagContainer` | Ability asset tags，用于匹配、阻塞、取消、GE source tags。 | 常配置；源码已标记旧名将迁移到 AssetTags。 | 运行时不建议直接改；`SetAssetTags` 只应在构造期调用；源码路径：`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Private/Abilities/GameplayAbility.cpp:1285`。 |
| `ActivationOwnedTags` | `Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Public/Abilities/GameplayAbility.h:765` | `FGameplayTagContainer` | active 期间添加给 owner 的 tags。 | 常配置。 | defaults 配置，运行时由 `PreActivate`/`EndAbility` 管理。 |
| `CostGameplayEffectClass` | `Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Public/Abilities/GameplayAbility.h:741` | `TSubclassOf<UGameplayEffect>` | 提交 Ability 时应用的 cost GE。 | 常配置。 | defaults 配置。 |
| `CooldownGameplayEffectClass` | `Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Public/Abilities/GameplayAbility.h:749` | `TSubclassOf<UGameplayEffect>` | 提交 Ability 时应用的 cooldown GE，之后用其 granted tags 判断冷却。 | 常配置。 | defaults 配置。 |
| `CurrentActorInfo` | `Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Public/Abilities/GameplayAbility.h:877` | `mutable const FGameplayAbilityActorInfo*` | 当前实例绑定的 ActorInfo。 | 实例 Ability 中常通过 getter 使用。 | 不应直接改；`SetCurrentActorInfo` 只在 instanced ability 生效，ASC/Ability 流程维护；源码路径：`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Private/Abilities/GameplayAbility.cpp:2122`。 |
| `CurrentSpecHandle` | `Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Public/Abilities/GameplayAbility.h:886` | `FGameplayAbilitySpecHandle` | 当前实例对应的 spec handle。 | 实例 Ability 中常通过 getter 使用。 | 不应直接改；由 `SetCurrentActorInfo` 写入。 |
| `CurrentActivationInfo` | `Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Public/Abilities/GameplayAbility.h:725` | `FGameplayAbilityActivationInfo` | 当前激活的预测/确认/authority 状态。 | 业务常读，不应裸写。 | 不应直接改；由 ASC 激活与 `SetCurrentActivationInfo` 维护。 |

## 第三轮未确认项

- `NetExecutionPolicy` 构造函数默认值未在本轮读取到显式赋值，未确认。
- AbilityTask 具体子类行为、任务等待 target data/input 的完整网络流程未展开，未确认。
- GameplayEffect 内部执行、属性聚合、cooldown GE granted tags 的完整 GE 侧实现未展开，未确认。

# 核心类：GameplayEffect 体系（第四轮）

本轮完整专题见 `gameplay-effects.md`。这里保留核心索引，便于后续快速定位。

## 类定位索引

- `UGameplayEffect` 是 GE 配置定义/CDO，继承 `UObject` 并实现 `IGameplayTagAssetInterface`；源码路径：`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Public/GameplayEffect.h:2071`。
- `FGameplayEffectSpec` 是一次应用的运行时规格，保存 `Def`、Level、Context、捕获数据、动态 tags、SetByCaller 和 modifier 运行时数值；源码路径：`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Public/GameplayEffect.h:988`、`:1169`、`:1243`。
- `FActiveGameplayEffect` 是加入目标 ASC 后的 active 实例，保存 `Spec`、`Handle`、`PredictionKey`、timer、stack cache 和事件集合；源码路径：`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Public/GameplayEffect.h:1319`、`:1389`、`:1432`。
- `FActiveGameplayEffectsContainer` 是 ASC 内部的 active GE fast array 容器，应只由 `UAbilitySystemComponent` 使用；源码路径：`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Public/GameplayEffect.h:1621`、`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Public/AbilitySystemComponent.h:1881`。

## 配置与运行时分层

| 层级 | 类型 | 主要职责 | 源码路径 |
|---|---|---|---|
| 配置资产/CDO | `UGameplayEffect` | Duration、Modifiers、Executions、GameplayCues、Stacking、GEComponents 等静态配置 | `Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Public/GameplayEffect.h:2071` |
| 运行时规格 | `FGameplayEffectSpec` | 一次应用携带的 Level、Context、SetByCaller、捕获属性、动态 tags、已计算 modifier | `Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Public/GameplayEffect.h:988` |
| 已激活效果 | `FActiveGameplayEffect` | 目标 ASC 上持续存在的效果实例，维护 handle、time、stack、prediction、delegates | `Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Public/GameplayEffect.h:1319` |
| ASC 容器 | `FActiveGameplayEffectsContainer` | 应用、执行、移除、stack、aggregator、复制 ActiveGE | `Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Public/GameplayEffect.h:1621` |

## UE5.6 版本注意

- UE5.6 中很多旧 GE 字段已标记 deprecated，当前主路径是 `GEComponents`：Asset tags、Target granted tags、Blocked ability tags、Target tag requirements、ChanceToApply、CustomCanApply、Immunity、RemoveOther、GrantedAbilities 都有对应组件；源码路径：`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Public/GameplayEffect.h:2423`、`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Public/GameplayEffectComponent.h:17`。
- `GameplayCues` 本轮确认仍是 `UGameplayEffect` 上的字段，未确认存在 `GameplayCueComponent`；源码路径：`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Public/GameplayEffect.h:2299`。
- `GrantAbilitiesGameplayEffectComponent` 这个精确类名未确认；UE5.6 实际存在的是 `UAbilitiesGameplayEffectComponent`，DisplayName 为 “Grant Gameplay Abilities”；源码路径：`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Public/GameplayEffectComponents/AbilitiesGameplayEffectComponent.h:37`、`:38`。

# 核心类：AttributeSet 体系（第五轮）

完整专题见 `attributes.md`。本节保留核心索引，便于从 ASC / GE / Ability 文档跳转到属性层。

## 类定位索引

- `UAttributeSet` 是 `UObject` 子类，用来定义 gameplay attributes；源码注释要求项目继承并添加 `FGameplayAttributeData` 属性，AttributeSet 作为 Actor subobject 注册到 ASC；源码路径：`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Public/AttributeSet.h:176`、`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Public/AttributeSet.h:183`。
- `FGameplayAttributeData` 保存 `BaseValue` 与 `CurrentValue`，源码明确建议优先使用它而不是裸 `float`；源码路径：`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Public/AttributeSet.h:19`、`:34`、`:40`。
- `FGameplayAttribute` 是 AttributeSet 内属性的描述/句柄，GE modifier、ASC 查询和 attribute delegate 都以它定位属性；源码路径：`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Public/AttributeSet.h:57`、`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Private/AttributeSet.cpp:355`。
- `UAbilitySystemComponent` 通过 `SpawnedAttributes` 管理 AttributeSet，并在 `ReplicateSubobjects` 中复制有效 AttributeSet subobject；源码路径：`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Public/AbilitySystemComponent.h:1957`、`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Private/AbilitySystemComponent.cpp:1710`。
- GameplayEffect 修改属性时，Instant/periodic execute 通过 `InternalExecuteMod` 进入 AttributeSet 的 `PreGameplayEffectExecute` / `PostGameplayEffectExecute`；Duration/Infinite modifier 通常注册到 `FAggregator` 并更新 current value；源码路径：`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Private/GameplayEffect.cpp:3907`、`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Private/GameplayEffect.cpp:4347`。

## AttributeSet 开发速查

- 定义属性看 `FGameplayAttributeData` 和 `GAMEPLAYATTRIBUTE_*` 宏；`ATTRIBUTE_ACCESSORS` 只是源码注释中的项目侧示例，UE5.6 实际提供 `ATTRIBUTE_ACCESSORS_BASIC`；源码路径：`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Public/AttributeSet.h:21`、`:430`、`:467`。
- 初始化属性可看 `InitStats` / `InitFromMetaDataTable`，但 `InitStats` 注释写明“不太受支持”；源码路径：`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Public/AbilitySystemComponent.h:166`、`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Private/AttributeSet.cpp:383`。
- 修改属性优先走 GameplayEffect 或 ASC `SetNumericAttributeBase`，不要绕过 GAS 直接改成员值；源码路径：`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Private/GameplayEffect.cpp:3973`、`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Private/AbilitySystemComponent.cpp:386`。
- 监听属性变化用 ASC `GetGameplayAttributeValueChangeDelegate`，参数是 `FOnAttributeChangeData`；源码路径：`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Public/AbilitySystemComponent.h:534`、`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Public/GameplayEffectTypes.h:1002`。
- Clamp current value 放 `PreAttributeChange`，clamp base value 放 `PreAttributeBaseChange`；源码路径：`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Public/AttributeSet.h:211`、`:225`。
- Damage Meta Attribute 是开发实践推断，不是本轮源码确认的固定类型；测试 AttributeSet 把 `Damage` 当作非持久属性，并在 `PostGameplayEffectExecute` 中转为 `Health -= Damage`；源码路径：`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Public/AbilitySystemTestAttributeSet.h:36`、`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Private/AbilitySystemTestAttributeSet.cpp:100`。
- Replication 至少检查 AttributeSet 是否加入 ASC、属性是否声明复制、RepNotify 是否调用 `GAMEPLAYATTRIBUTE_REPNOTIFY`、预测属性是否使用 `REPNOTIFY_Always`；源码路径：`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Private/AbilitySystemComponent.cpp:3005`、`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Public/AttributeSet.h:404`、`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Public/GameplayPrediction.h:127`。

## 第五轮未确认项

- `FGameplayAttributeValueChange` 精确类型名未确认；源码实际确认的是 `FOnAttributeChangeData` 与 `FOnGameplayAttributeValueChange`；源码路径：`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Public/GameplayEffectTypes.h:1002`、`:1017`。
- “Meta Attribute” 不是本轮源码确认的固定类型，作为 GAS 项目实践未确认。
- 客户端预测失败后 Attribute/GE/delegate 的完整回滚路径未展开，未确认。

# 核心类：AbilityTask / GameplayTask 体系（第六轮）

完整专题见 `ability-tasks.md`。本节保留核心索引，便于从 Ability 生命周期跳转到异步任务层。

## 类定位索引

- `UGameplayTask` 是 GameplayTasks 模块的通用异步任务基类，继承 `UObject` 并实现 `IGameplayTaskOwnerInterface`；源码路径：`Engine/Source/Runtime/GameplayTasks/Classes/GameplayTask.h:145`。
- `UAbilityTask` 继承 `UGameplayTask`，是 GAS 中 Ability 执行期的 latent/asynchronous operation；源码路径：`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Public/Abilities/Tasks/AbilityTask.h:20`、`:90`。
- `UGameplayTasksComponent` 维护任务激活、deactivate、known/ticking tasks；`UAbilitySystemComponent` 继承自它，因此 AbilityTask 在 Ability 中拿到的 GameplayTasksComponent 通常就是 ASC；源码路径：`Engine/Source/Runtime/GameplayTasks/Classes/GameplayTasksComponent.h:61`、`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Public/AbilitySystemComponent.h:109`。
- `UGameplayAbility` 实现 `IGameplayTaskOwnerInterface`，在 `OnGameplayTaskInitialized` 中把 Task 的 Ability/ASC 指针写好，并用 `ActiveTasks` 跟踪活跃任务；源码路径：`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Public/Abilities/GameplayAbility.h:110`、`:820`、`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Private/Abilities/GameplayAbility.cpp:1539`。
- Ability 结束时会对所有 `ActiveTasks` 调用 `TaskOwnerEnded` 并清空数组；源码路径：`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Private/Abilities/GameplayAbility.cpp:819`。

## AbilityTask 开发速查

- 自定义 AbilityTask 继承 `UAbilityTask`，不要用 `UGameplayTask::NewTask`；源码路径：`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Public/Abilities/Tasks/AbilityTask.h:90`、`:153`。
- 静态工厂函数只创建任务和保存输入参数，真正开始等待/绑定 delegate 要放到 `Activate`；源码路径：`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Public/Abilities/Tasks/AbilityTask.h:28`、`:35`、`:44`。
- 调用方绑定输出 delegate 后调用 `ReadyForActivation`，该函数进入 GameplayTask 激活流程；源码路径：`Engine/Source/Runtime/GameplayTasks/Classes/GameplayTask.h:157`、`Engine/Source/Runtime/GameplayTasks/Private/GameplayTask.cpp:58`。
- 广播前使用 `ShouldBroadcastAbilityTaskDelegates`，确保 Ability 仍 active；源码路径：`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Private/Abilities/Tasks/AbilityTask.cpp:190`。
- `OnDestroy` 负责解绑 task 注册的 callbacks/timers/TargetActor，并最后调用 `Super::OnDestroy`；源码路径：`Engine/Source/Runtime/GameplayTasks/Classes/GameplayTask.h:291`、`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Public/Abilities/Tasks/AbilityTask.h:43`。
- 需要预测时再使用 `FScopedPredictionWindow`、generic replicated events 或 target data replicated delegates；AbilityTask 基类只提供 helper，不保证所有任务自动预测；源码路径：`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Private/Abilities/Tasks/AbilityTask.cpp:182`、`:217`、`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Public/GameplayPrediction.h:192`。

## 第六轮未确认项

- `UAbilityTask_WaitMontageNotify` 在 UE5.6 当前 Tasks 目录未找到，未确认。
- AbilityTask 与普通 Blueprint latent node 的完整差异未展开，源码只确认 `K2Node_LatentAbilityCall` 支持 AbilityTask 蓝图体验；源码路径：`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Public/Abilities/Tasks/AbilityTask.h:23`。
- 预测失败后自定义 AbilityTask 副作用的完整回滚规则未展开，未确认。

# 核心类：GameplayCue 体系（第七轮）

完整专题见 `gameplay-cues.md`。本节保留核心索引，便于从 ASC / GameplayEffect / Ability 跳转到表现事件层。

## 类定位索引

- GameplayCue 以 `GameplayCue` 分类下的 `FGameplayTag` 为事件键，`FGameplayEffectCue` 把 GE level/magnitude 与一个或多个 cue tag 关联；源码路径：`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Public/GameplayEffect.h:578`、`:613`。
- `UGameplayCueManager` 是 `UDataAsset` 类型的全局 cue 分发和 notify spawn/recycle 管理器；源码路径：`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Public/GameplayCueManager.h:128`、`:130`。
- `UGameplayCueSet` 保存 `FGameplayCueNotifyData` 数组和 tag->index 加速 map，负责把 cue tag 路由到 `UGameplayCueNotify_Static` 或 `AGameplayCueNotify_Actor`；源码路径：`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Public/GameplayCueSet.h:52`、`:92`、`:95`、`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Private/GameplayCueSet.cpp:259`。
- `UGameplayCueNotify_Static` 是非实例化 UObject notify，适合无状态 one-off；`AGameplayCueNotify_Actor` 是实例化 Actor notify，可持有状态并由 Manager 复用或 spawn；源码路径：`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Public/GameplayCueNotify_Static.h:14`、`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Public/GameplayCueNotify_Actor.h:15`、`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Private/GameplayCueManager.cpp:459`。
- UE5.6 当前源码确认存在 `UGameplayCueNotify_Burst`、`AGameplayCueNotify_BurstLatent`、`AGameplayCueNotify_Looping`；源码路径：`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Public/GameplayCueNotify_Burst.h:19`、`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Public/GameplayCueNotify_BurstLatent.h:19`、`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Public/GameplayCueNotify_Looping.h:19`。
- ASC 提供 `ExecuteGameplayCue`、`AddGameplayCue`、`RemoveGameplayCue` 和 NetMulticast RPC；手动 persistent cue 进入 `ActiveGameplayCues`，minimal replication 下 GE cue 可进入 `MinimalReplicationGameplayCues`；源码路径：`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Public/AbilitySystemComponent.h:910`、`:914`、`:921`、`:1883`、`:1887`。
- `FGameplayCueParameters` 是 cue 参数载体，包含 magnitude、EffectContext、聚合 tags、location/normal、instigator、causer、source object 等，并实现 `NetSerialize`；源码路径：`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Public/GameplayEffectTypes.h:834`、`:857`、`:865`、`:877`、`:893`、`:928`。

## GameplayCue 开发速查

- 一次性表现使用 `ExecuteGameplayCue` 或 Instant/periodic GE 的 `GameplayCues`；源码路径：`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Public/GameplayEffectTypes.h:969`、`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Private/AbilitySystemComponent.cpp:1300`。
- 持续表现使用 Duration/Infinite GE 的 `GameplayCues`，或手动 `AddGameplayCue` 并配对 `RemoveGameplayCue`；源码路径：`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Private/GameplayEffect.cpp:4411`、`:4714`、`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Private/AbilitySystemComponent.cpp:1312`、`:1401`。
- Static/Burst 用于无状态 one-off；Actor/BurstLatent 用于需要 latent 或状态的 one-off；Looping 用于持续效果；源码路径：`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Public/GameplayCueNotify_Burst.h:16`、`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Public/GameplayCueNotify_BurstLatent.h:16`、`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Public/GameplayCueNotify_Looping.h:16`。
- Cue tag 命名放在 `GameplayCue.*` 层级是开发实践推断；源码依据是 GE cue tag 属性限制 `Categories="GameplayCue"`，CueSet base tag 为 `GameplayCue`；源码路径：`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Public/GameplayEffect.h:613`、`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Private/GameplayCueSet.cpp:15`。
- 排查 cue 不播放时先查 tag 是否有效、notify 是否被 `GameplayCueNotifyPaths` 扫描、目标 Avatar 是否 ready、Manager/ASC 是否 suppression；源码路径：`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Private/GameplayCueManager.cpp:974`、`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Private/AbilitySystemGlobals.cpp:609`、`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Private/AbilitySystemComponent.cpp:1215`、`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Private/GameplayCueManager.cpp:177`。

## 第七轮未确认项

- `GameplayCue_Types.cpp` 在当前源码树不存在，未确认；源码路径：`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Private/GameplayCue_Types.cpp`。
- GameplayCue 编辑器工具、完整网络预测回滚、Niagara/Audio 底层、本轮未展开，未确认；源码路径：`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilitiesEditor/Private`、`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Private/GameplayCueNotifyTypes.cpp`。
- `GameplayCueComponent` 或等价 GameplayEffectComponent 在当前 `GameplayEffectComponents` 目录未确认；确认路径仍是 `UGameplayEffect::GameplayCues`；源码路径：`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Public/GameplayEffect.h:2299`。

# 核心类：GAS 网络预测 / 复制体系（第八轮）

完整专题见 `networking-prediction.md`。本节保留核心索引，便于从 Ability、AbilityTask、GameplayEffect、AttributeSet、GameplayCue 文档跳转到网络层。

## 类定位索引

- `FPredictionKey` 是 GAS 预测动作的标识，保存 `Current`、`Base`、server initiated 标记，并提供 reject / caught-up delegate；源码路径：`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Public/GameplayPrediction.h:296`、`:304`、`:308`、`:324`。
- `FScopedPredictionWindow` 以 RAII 方式临时设置 ASC 的 `ScopedPredictionKey`，服务端窗口析构时可把确认 key 写入 `ReplicatedPredictionKeyMap`；源码路径：`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Public/GameplayPrediction.h:479`、`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Private/GameplayPrediction.cpp:493`、`:509`。
- `FGameplayAbilityActivationInfo` 保存 Ability 激活模式和激活时 prediction key，模式包括 Authority、NonAuthority、Predicting、Confirmed、Rejected；源码路径：`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Public/GameplayAbilitySpec.h:25`、`:113`、`:156`。
- `FReplicatedPredictionKeyMap` 是 ASC owner-only 复制的 fast array，用于服务端确认 prediction key 后触发客户端 catch-up；源码路径：`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Public/GameplayPrediction.h:595`、`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Public/AbilitySystemComponent.h:1978`、`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Private/GameplayPrediction.cpp:575`。
- `FGameplayAbilitySpecHandle` 是指向已授予 Ability spec 的 handle，作为 Ability 激活 RPC、TargetData 和 replicated event 的基础标识；源码路径：`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Public/GameplayAbilitySpecHandle.h:15`、`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Public/AbilitySystemComponent.h:1753`。
- `FGameplayAbilityReplicatedDataContainer` 按 AbilitySpecHandle + PredictionKey 缓存 TargetData 和 GenericReplicatedEvent 数据；源码路径：`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Public/Abilities/GameplayAbilityTypes.h:602`、`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Private/GameplayAbilityTypes.cpp:413`。
- `FAbilityReplicatedData` 是源码实际确认的 GenericReplicatedEvent 缓存类型；`FGameplayAbilityReplicatedData` 同名类型本轮未确认；源码路径：`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Public/Abilities/GameplayAbilityTargetTypes.h:681`。
- `FServerAbilityRPCBatch` / `FScopedServerAbilityRPCBatcher` 只批处理 ASC 明确支持的 TryActivate、TargetData、EndAbility 三类 server RPC；源码路径：`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Public/Abilities/GameplayAbilityTypes.h:448`、`:486`、`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Public/AbilitySystemComponent.h:1328`。
- `EGameplayEffectReplicationMode` 控制 ActiveGE 复制粒度：Minimal、Mixed、Full；源码路径：`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Public/AbilitySystemComponent.h:81`。
- `IAbilitySystemReplicationProxyInterface` 让 Actor 作为 ASC 复制代理，主要转发 GameplayCue RPC；源码路径：`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Public/AbilitySystemReplicationProxyInterface.h:21`、`:39`。
- GameplayAbilities 模块通过 `SetupIrisSupport(Target)` 接入 Iris，并提供 `PredictionKey`、TargetData、EffectContext、RepAnimMontage、MinimalCue、MinimalTag 等 NetSerializer；源码路径：`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/GameplayAbilities.Build.cs:42`、`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Private/Serialization/PredictionKeyNetSerializer.cpp:194`。

## GAS 网络预测速查

- LocalPredicted Ability 必查：`NetExecutionPolicy`、ActorInfo/Avatar、locally controlled、`FScopedPredictionWindow`、`ClientActivateAbilitySucceed` / `ClientActivateAbilityFailed`；源码路径：`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Private/AbilitySystemComponent_Abilities.cpp:1904`、`:1922`、`:2251`、`:2339`。
- ServerOnly Ability 必查：客户端不要直接执行权威逻辑，激活应走服务端 `InternalTryActivateAbility`；源码路径：`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Private/AbilitySystemComponent_Abilities.cpp:1761`、`:2026`。
- TargetData 必查：`FGameplayAbilityTargetDataHandle::NetSerialize`、PredictionKey、`CallServerSetReplicatedTargetData`、服务端 delegate 绑定和 `ConsumeClientReplicatedTargetData`；源码路径：`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Private/GameplayAbilityTargetTypes.cpp:183`、`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Private/AbilitySystemComponent_Abilities.cpp:3945`、`:3838`。
- WaitInputPress / WaitInputRelease 必查：输入 replicated event 是否发送、Task 是否绑定 `AbilityReplicatedEventDelegate`、是否调用 `ConsumeGenericReplicatedEvent`；源码路径：`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Private/Abilities/Tasks/AbilityTask_WaitInputPress.cpp:26`、`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Private/Abilities/Tasks/AbilityTask_WaitInputRelease.cpp:26`、`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Private/AbilitySystemComponent_Abilities.cpp:3849`。
- GameplayEffect 预测必查：periodic GE 不预测，predicted instant GE 临时作为 infinite ActiveGE，回滚依赖 prediction key delegate；源码路径：`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Private/AbilitySystemComponent.cpp:821`、`:862`、`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Private/GameplayEffect.cpp:4264`。
- Attribute 复制必查：AttributeSet subobject 复制、属性 RepNotify、`GAMEPLAYATTRIBUTE_REPNOTIFY`、预测属性 `REPNOTIFY_Always`；源码路径：`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Private/AbilitySystemComponent.cpp:1710`、`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Public/AttributeSet.h:404`、`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Public/GameplayPrediction.h:127`。
- GameplayCue 复制必查：手动 Cue RPC、`ActiveGameplayCues`、`MinimalReplicationGameplayCues`、local prediction key 去重；源码路径：`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Private/AbilitySystemComponent.cpp:1436`、`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Public/AbilitySystemComponent.h:1884`、`:1887`、`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Private/AbilitySystemComponent.cpp:1438`。
- ReplicationMode 选择建议是开发实践推断：玩家自身 ASC 常用 Mixed，AI 或非关键单位常用 Minimal，需要所有客户端完整 GE 状态时用 Full；源码依据：`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Public/AbilitySystemComponent.h:81`、`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Private/GameplayEffect.cpp:4875`。
- PredictionKey 无效排查：确认当前动作仍在 prediction window 内，跨帧 AbilityTask 是否开启新窗口，RPC 是否带正确 key；源码路径：`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Public/GameplayPrediction.h:198`、`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Private/AbilitySystemComponent_Abilities.cpp:4234`。

## 第八轮未确认项

- `FGameplayAbilityReplicatedData` 同名类型未确认，源码实际确认的是 `FAbilityReplicatedData`；源码路径：`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Public/Abilities/GameplayAbilityTargetTypes.h:681`。
- `GameplayCue_Types.cpp` 在当前源码路径未确认存在；GameplayCue 参数和容器序列化主要确认在 `GameplayEffectTypes.cpp` 与 `GameplayCueInterface.cpp`。
- TargetActor 子类的 TargetData 校验失败策略、predicted GameplayCue / GE 的完整表现回滚、Montage prediction reject 的具体 warning 路径、本轮未完整展开，未确认。

# 核心类：GAS 全局入口与蓝图 API（第九轮）

完整专题见 `globals-blueprint-library.md`。本节只保留核心索引，便于从 ASC、Ability、GE、TargetData、GameplayCue 快速跳到全局/蓝图入口。

## 类定位索引

- `UAbilitySystemGlobals` 是 GameplayAbilities 模块的全局配置、初始化与工厂入口，通过 `UAbilitySystemGlobals::Get()` 返回模块级单例；源码路径：`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Public/AbilitySystemGlobals.h:45`、`:52`、`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Private/GameplayAbilitiesModule.cpp:24`。
- `UAbilitySystemBlueprintLibrary` 是蓝图工具库，暴露 ASC 获取、GameplayEvent、Attribute 查询、TargetData、EffectContext、GameplayCueParameters、GameplayEffectSpec handle、SetByCaller 和 dynamic tag 操作；源码路径：`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Public/AbilitySystemBlueprintLibrary.h:68`、`:75`、`:83`、`:178`、`:222`、`:311`、`:411`、`:423`。
- `IAbilitySystemInterface` 要求 C++ 类实现 `GetAbilitySystemComponent() const`，且源码注释确认返回的 ASC 可以位于另一个 Actor，例如 Pawn 使用 PlayerState 的 ASC；源码路径：`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Public/AbilitySystemInterface.h:15`、`:25`、`:26`。
- Globals 查找 ASC 时优先使用 `IAbilitySystemInterface`，fallback 才是 `FindComponentByClass<UAbilitySystemComponent>()`；BlueprintLibrary 的 ASC 获取只是转发到 Globals；源码路径：`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Private/AbilitySystemGlobals.cpp:233`、`:240`、`:243`、`:247`、`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Private/AbilitySystemBlueprintLibrary.cpp:77`。
- Globals 初始化全局 curve/attribute/cue/tag 配置，并初始化 TargetData / EffectContext 的 NetSerialize struct cache；源码路径：`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Private/AbilitySystemGlobals.cpp:64`、`:76`、`:82`、`:84`、`:87`、`:370`。

## GAS 全局入口与蓝图 API 速查

- 获取 ASC：蓝图侧用 `UAbilitySystemBlueprintLibrary::GetAbilitySystemComponent`，项目侧优先实现 `IAbilitySystemInterface::GetAbilitySystemComponent`；源码路径：`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Private/AbilitySystemBlueprintLibrary.cpp:77`、`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Public/AbilitySystemInterface.h:26`。
- Character / PlayerState 谁实现接口：源码未规定固定架构；开发实践推断是外界常传入谁查 ASC，谁就应该能返回正确 ASC，PlayerState 持有 ASC 时 Pawn/Character 可转发 PlayerState ASC；源码依据：`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Public/AbilitySystemInterface.h:25`。
- 创建 GE Spec：蓝图库提供 `MakeSpecHandleByClass` 和 `CloneSpecHandle`，旧 `MakeSpecHandle(UGameplayEffect*)` 已 deprecated；ASC 另有 `MakeOutgoingSpec`；源码路径：`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Public/AbilitySystemBlueprintLibrary.h:249`、`:255`、`:259`、`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Public/AbilitySystemComponent.h:363`。
- 应用 GE：`ApplyGameplayEffectToSelf/Target` 与 `ApplyGameplayEffectSpecToSelf/Target` 在 ASC 上，不是蓝图库同名函数；源码路径：`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Public/AbilitySystemComponent.h:327`、`:333`、`:790`、`:795`。
- 设置 SetByCaller：用 `AssignSetByCallerMagnitude` 或 `AssignTagSetByCallerMagnitude` 在应用 GE 前写入 Spec；源码路径：`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Public/AbilitySystemBlueprintLibrary.h:423`、`:427`、`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Private/AbilitySystemBlueprintLibrary.cpp:985`、`:1004`。
- 操作 EffectContext：蓝图库确认能读写 HitResult、Origin，并读取 Instigator / OriginalInstigator / EffectCauser / SourceObject；源码路径：`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Public/AbilitySystemBlueprintLibrary.h:319`、`:327`、`:331`、`:335`、`:339`、`:343`、`:347`、`:351`。
- 操作 TargetData：先用 `GetDataCountFromTargetData` 和 `TargetDataHasActor/HitResult/Origin/EndPoint` 检查，再读取 actor、hit result、origin、endpoint；源码路径：`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Public/AbilitySystemBlueprintLibrary.h:230`、`:275`、`:279`、`:287`、`:295`。
- 触发 GameplayCue：`GameplayCueFunctionLibrary` 提供 Actor 级 Execute/Add/Remove 辅助，内部优先找 ASC，authority 时走 ASC cue 路径；源码路径：`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Public/GameplayCueFunctionLibrary.h:33`、`:39`、`:45`、`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Private/GameplayCueFunctionLibrary.cpp:25`、`:49`、`:78`。
- 排查空 ASC：确认 Actor 非空、接口已实现、接口返回不为空、Actor 身上有 ASC fallback，或者 PlayerState/Character 转发关系正确；源码路径：`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Private/AbilitySystemGlobals.cpp:235`、`:240`、`:247`、`:254`。

## 第九轮未确认项

- 用户列出的 `Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Public/GameplayAbilityTargetTypes.h` 未确认存在；实际确认路径为 `Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Public/Abilities/GameplayAbilityTargetTypes.h:193`。
- `UAbilitySystemBlueprintLibrary` 当前未确认存在直接 `ClearTargetData`、`EffectContextAddSourceObject`、`EffectContextAddInstigator` 或读取 SetByCaller magnitude 的蓝图函数；底层 C++ handle/spec 具备部分能力；源码路径：`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Public/Abilities/GameplayAbilityTargetTypes.h:215`、`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Public/GameplayEffectTypes.h:544`、`:642`、`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Public/GameplayEffect.h:1082`。
- 是否必须自定义 `UAbilitySystemGlobals` 类未确认；源码只确认可通过 developer settings 配置 Globals class，并提供 virtual factory/hook；源码路径：`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Public/GameplayAbilitiesDeveloperSettings.h:37`、`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Public/AbilitySystemGlobals.h:57`、`:78`、`:81`、`:84`。

# 核心类：GameplayAbilitiesEditor / Blueprint / K2 工具链（第十轮）

完整专题见 `editor-blueprint.md`。本节只保留索引和开发速查，便于从运行时主题跳到编辑器工具链。

## 类定位索引

- `GameplayAbilitiesEditor` 是编辑器模块，依赖 `GameplayAbilities` 运行时模块，并额外依赖 `AssetTools`、`ClassViewer`、`PropertyEditor`、`BlueprintGraph`、`KismetCompiler`、`GraphEditor`、`Sequencer`、`MovieScene`、`DataRegistryEditor` 等编辑器模块；源码路径：`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilitiesEditor/GameplayAbilitiesEditor.Build.cs:9`、`:22`、`:25`、`:28`、`:30`、`:31`、`:40`、`:43`、`:45`。
- `FGameplayAbilitiesEditorModule` 是编辑器模块入口，负责注册 details、资产动作、GameplayCue editor tab、GameplayCue tag tree 刷新、GameplayAttribute pin factory、Sequencer GameplayCue track editor 和 AbilitySystemGlobals 调试回调；源码路径：`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilitiesEditor/Private/GameplayAbilitiesEditorModule.cpp:61`、`:171`、`:175`、`:185`、`:190`、`:206`、`:213`、`:232`、`:239`。
- `UGameplayAbilityBlueprint` 是运行时模块中的专用 Blueprint 资产类型，继承 `UBlueprint`；Factory 和 AssetTypeActions 位于编辑器模块；源码路径：`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Public/GameplayAbilityBlueprint.h:18`、`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilitiesEditor/Private/GameplayAbilitiesBlueprintFactory.cpp:300`。
- 当前源码实际存在 `UGameplayAbilitiesBlueprintFactory` 和 `FAssetTypeActions_GameplayAbilitiesBlueprint`，用户列出的单数 `UGameplayAbilityBlueprintFactory` / `FAssetTypeActions_GameplayAbilityBlueprint` 未确认；源码路径：`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilitiesEditor/Public/GameplayAbilitiesBlueprintFactory.h:13`、`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilitiesEditor/Private/AssetTypeActions_GameplayAbilitiesBlueprint.h:11`。
- `UGameplayAbilityGraph` / `UGameplayAbilityGraphSchema` 是 Ability 蓝图专用图表和 K2 Schema；当前 Schema 的 variable get/set 仍转调父类，没有确认特殊节点限制；源码路径：`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilitiesEditor/Public/GameplayAbilityGraph.h:11`、`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilitiesEditor/Public/GameplayAbilityGraphSchema.h:11`、`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilitiesEditor/Private/GameplayAbilityGraphSchema.cpp:15`、`:20`。
- `UK2Node_LatentAbilityCall` 只处理 `UAbilityTask`，只允许在 `UGameplayAbility` 蓝图的 Ubergraph/Macro 中出现，并通过 `GameplayTasksEditor` 的 `UK2Node_LatentGameplayTaskCall` 展开到工厂调用、delegate 绑定和 `ReadyForActivation`；源码路径：`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilitiesEditor/Private/K2Node_LatentAbilityCall.cpp:30`、`:35`、`:77`、`Engine/Source/Editor/GameplayTasksEditor/Private/K2Node_LatentGameplayTaskCall.cpp:53`、`:593`、`:657`、`:899`。
- `UK2Node_GameplayCueEvent` 只对实现 `UGameplayCueInterface` 的蓝图兼容，并根据 `GameplayCue` 根 tag 的子 tag 生成事件节点；源码路径：`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilitiesEditor/Private/K2Node_GameplayCueEvent.cpp:23`、`:56`、`:84`、`:87`、`:94`。
- `FGameplayEffectDetails`、`FGameplayEffectModifierMagnitudeDetails`、`FGameplayEffectExecutionDefinitionDetails`、`FGameplayEffectExecutionScopedModifierInfoDetails` 构成 GE 细节面板核心定制；源码路径：`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilitiesEditor/Private/GameplayAbilitiesEditorModule.cpp:179`、`:180`、`:186`、`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilitiesEditor/Private/GameplayEffectDetails.cpp:20`。
- `FAttributePropertyDetails`、`FAttributeDetails`、`SGameplayAttributeWidget`、`SGameplayAttributeGraphPin` 构成 Attribute 编辑器选择链；源码路径：`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilitiesEditor/Private/AttributeDetails.cpp:34`、`:175`、`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilitiesEditor/Private/SGameplayAttributeWidget.cpp:268`、`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilitiesEditor/Private/SGameplayAttributeGraphPin.cpp:21`。
- `FGameplayTagRequirementsDetails`、`FAbilityAudit`、`FAssetTypeActions_GameplayEffect`、`FAssetTypeActions_GameplayCueNotify` 在当前扫描范围未命中，未确认；实际 Ability Audit 类型是 `FGameplayAbilityAuditRow`，GE/CueNotify 创建分别走 `UGameplayEffectCreationMenu` 和 `SGameplayCueEditor`；源码路径：`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilitiesEditor/Public/GameplayAbilityAudit.h:46`、`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilitiesEditor/Private/GameplayEffectCreationMenu.cpp:148`、`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilitiesEditor/Private/SGameplayCueEditor.cpp:1389`。

## GAS 编辑器与蓝图工具速查

- 创建 GameplayAbility 蓝图：看 `UGameplayAbilitiesBlueprintFactory`、`UGameplayAbilityBlueprint`、`FAssetTypeActions_GameplayAbilitiesBlueprint`、`FGameplayAbilitiesEditor`；源码路径：`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilitiesEditor/Public/GameplayAbilitiesBlueprintFactory.h:13`、`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Public/GameplayAbilityBlueprint.h:18`、`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilitiesEditor/Private/AssetTypeActions_GameplayAbilitiesBlueprint.h:11`、`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilitiesEditor/Public/GameplayAbilitiesEditor.h:11`。
- 创建 GameplayEffect 资产：看 `UGameplayEffectCreationMenu`，它从 Project Settings 的 `Definitions` 构建 Content Browser 右键菜单，并用 `UBlueprintFactory` 创建 GE 父类蓝图；源码路径：`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilitiesEditor/Public/GameplayEffectCreationMenu.h:30`、`:47`、`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilitiesEditor/Private/GameplayEffectCreationMenu.cpp:74`、`:93`、`:148`。
- 创建 GameplayCueNotify：看 `SGameplayCueEditor::CreateNewGameplayCueNotifyDialogue`、`IGameplayAbilitiesEditorModule` 的 cue notify class/path delegates、`FGameplayCueTagDetails`；源码路径：`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilitiesEditor/Private/SGameplayCueEditor.cpp:1389`、`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilitiesEditor/Public/GameplayAbilitiesEditorModule.h:73`、`:78`、`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilitiesEditor/Private/GameplayCueTagDetails.cpp:145`。
- AbilityTask 蓝图节点找不到：检查工厂函数是否返回 `UAbilityTask` 子类、当前图是否是 Ability 蓝图 Ubergraph/Macro、节点是否被 `RegisterClassFactoryActions<UAbilityTask>` 收录；源码路径：`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilitiesEditor/Private/K2Node_LatentAbilityCall.cpp:30`、`:35`、`:77`。
- GameplayCue 蓝图事件找不到：检查蓝图类是否实现 `UGameplayCueInterface`，以及 `GameplayCue` 根 tag 是否存在；源码路径：`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilitiesEditor/Private/K2Node_GameplayCueEvent.cpp:56`、`:84`。
- Runtime 模块不要引用 `GameplayAbilitiesEditor` 类型；开发实践推断，打包报 Editor 类型依赖时先在项目 Runtime 模块中搜索 `GameplayAbilitiesEditor`、`K2Node_LatentAbilityCall`、`FGameplayEffectDetails`、`FGameplayAbilitiesEditor`；源码依据：`GameplayAbilitiesEditor` Build.cs 依赖多项 Editor 模块；源码路径：`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilitiesEditor/GameplayAbilitiesEditor.Build.cs:25`、`:28`、`:30`、`:40`。

## 第十轮未确认项

- `GameplayAbilitiesBlueprintEditor` 目录当前不存在，独立 BlueprintEditor 模块未确认；源码路径：`Engine/Plugins/Runtime/GameplayAbilities/Source`。
- `UGameplayAbilityBlueprintFactory`、`FGameplayAbilityGraphPanelNodeFactory`、`FGameplayTagRequirementsDetails`、`FAbilityAudit`、`FAssetTypeActions_GameplayAbilityBlueprint`、`FAssetTypeActions_GameplayEffect`、`FAssetTypeActions_GameplayCueNotify` 这些请求名在当前源码未命中，未确认；实际替代类型见 `editor-blueprint.md`。
- `UGameplayEffect` 的 `GEComponents` 是否有专门组件列表 customization，本轮未看到独立 details 类，未确认；源码路径：`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Public/GameplayEffect.h:2421`、`:2422`。

# 核心类：GameplayAbilityTargetActor / TargetData / Targeting（第十一轮）

完整专题见 `targeting-targetdata.md`。本节只保留索引和速查，便于从 AbilityTask、网络预测、GameplayEffect、GameplayCue 快速跳到 Targeting 体系。

## 类定位索引

- `AGameplayAbilityTargetActor` 是 Ability targeting 辅助 Actor，由 AbilityTask 生成，用于产生或确定传递给后续逻辑的 TargetData；源码注释也提醒默认形式每次 Ability 激活都会 spawn，效率不高，多数游戏需要重写或改成项目侧实现；源码路径：`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Public/Abilities/GameplayAbilityTargetActor.h:19`、`:22`、`:23`。
- `UAbilityTask_WaitTargetData` 是等待 TargetActor 返回 valid / cancel 数据的 AbilityTask，暴露 `ValidData` 与 `Cancelled` BlueprintAssignable delegate；源码路径：`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Public/Abilities/Tasks/AbilityTask_WaitTargetData.h:16`、`:28`、`:31`。
- `FGameplayAbilityTargetDataHandle` 是 TargetData 多态容器，源码注释明确用于蓝图引用传递、多态 TargetData 和客户端/服务端按值复制；源码路径：`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Public/Abilities/GameplayAbilityTargetTypes.h:179`、`:181`、`:182`、`:183`、`:192`。
- 常用 TargetData 派生类型包括 `FGameplayAbilityTargetData_ActorArray`、`FGameplayAbilityTargetData_SingleTargetHit`、`FGameplayAbilityTargetData_LocationInfo`；源码路径：`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Public/Abilities/GameplayAbilityTargetTypes.h:448`、`:550`、`:380`。
- `AGameplayAbilityTargetActor_Trace` 是 trace 类 TargetActor 中间基类，`SingleLineTrace` 和 `GroundTrace` 分别实现直线命中和地面落点；源码路径：`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Public/Abilities/GameplayAbilityTargetActor_Trace.h:20`、`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Public/Abilities/GameplayAbilityTargetActor_SingleLineTrace.h:13`、`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Public/Abilities/GameplayAbilityTargetActor_GroundTrace.h:17`。
- `AGameplayAbilityTargetActor_Radius` 以 `StartLocation` 为圆心收集半径内 Pawn，并默认 `ShouldProduceTargetDataOnServer = true`；源码路径：`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Public/Abilities/GameplayAbilityTargetActor_Radius.h:15`、`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Private/Abilities/GameplayAbilityTargetActor_Radius.cpp:25`。
- `UAbilityTask_VisualizeTargeting` 复用 TargetActor 做可视化，到时间只广播 `TimeElapsed`，没有 TargetData 输出；源码路径：`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Public/Abilities/Tasks/AbilityTask_VisualizeTargeting.h:21`、`:26`、`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Private/Abilities/Tasks/AbilityTask_VisualizeTargeting.cpp:168`、`:174`。
- TargetData RPC 由 ASC 的 `AbilityTargetDataMap` 缓存，key 是 AbilitySpecHandle + PredictionKey；源码路径：`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Public/AbilitySystemComponent.h:1698`、`:1699`、`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Public/Abilities/GameplayAbilityTypes.h:500`、`:510`。

## TargetData / TargetActor 速查

- 需要玩家瞄准、点击地面、选 actor、确认/取消并把客户端结果传给服务端时，用 `WaitTargetData`；源码路径：`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Public/Abilities/Tasks/AbilityTask_WaitTargetData.h:46`、`:48`。
- 目标已经由投射物、服务端逻辑、UI 或其他系统算好时，可以只用 `AbilitySystemBlueprintLibrary` 创建 TargetDataHandle；源码路径：`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Private/AbilitySystemBlueprintLibrary.cpp:393`、`:401`、`:515`、`:380`。
- 单目标命中点用 `SingleTargetHit`，多个 actor 用 `ActorArray`，纯位置/方向用 `LocationInfo`；源码路径：`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Public/Abilities/GameplayAbilityTargetTypes.h:550`、`:448`、`:380`。
- 直线瞄准用 `SingleLineTrace`，地面 AOE 或放置用 `GroundTrace`，范围 actor 收集用 `Radius`，放置预览用 `ActorPlacement`；源码路径：`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Public/Abilities/GameplayAbilityTargetActor_SingleLineTrace.h:13`、`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Public/Abilities/GameplayAbilityTargetActor_GroundTrace.h:17`、`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Public/Abilities/GameplayAbilityTargetActor_Radius.h:15`、`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Public/Abilities/GameplayAbilityTargetActor_ActorPlacement.h:14`。
- 客户端预测 TargetData 必查：PredictionKey、`CallServerSetReplicatedTargetData`、TargetData native NetSerialize、服务端 delegate 是否绑定、是否 consume；源码路径：`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Private/Abilities/Tasks/AbilityTask_WaitTargetData.cpp:280`、`:288`、`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Private/GameplayAbilityTargetTypes.cpp:183`、`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Private/AbilitySystemComponent_Abilities.cpp:3838`。
- 服务端校验 TargetData：ASC RPC validate 只检查 TargetData item 是否有效，项目级距离、视线、阵营、碰撞等合法性应在 `OnReplicatedTargetDataReceived` 或服务端 Ability 逻辑中做；源码路径：`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Private/AbilitySystemComponent_Abilities.cpp:3971`、`:3974`、`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Public/Abilities/GameplayAbilityTargetActor.h:64`。

## 第十一轮未确认项

- 用户给出的 `Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Private/Abilities/GameplayAbilityTargetTypes.cpp` 当前未确认存在；实际实现路径是 `Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Private/GameplayAbilityTargetTypes.cpp:1`。
- `AGameplayAbilityTargetActor_ActorPlacement` 的 `PlacedActorClass` 注释写 replication purpose not implemented yet，因此其放置 Actor 复制支持未确认；源码路径：`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Public/Abilities/GameplayAbilityTargetActor_ActorPlacement.h:25`。

# 核心类：GameplayTag / GameplayTagResponseTable / Ability Tag 条件体系（第十二轮）

完整专题见 `gameplay-tags-response.md`。本节只保留索引和速查，便于从 ASC、Ability、GE、Cue、网络复制快速跳到 tag 驱动体系。

## 类定位索引

- ASC 的 owned tag 主账本是 `FGameplayTagCountContainer GameplayTagCountContainer`，`HasMatchingGameplayTag`、`HasAllMatchingGameplayTags`、`HasAnyMatchingGameplayTags`、`GetOwnedGameplayTags` 都围绕它查询；源码路径：`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Public/AbilitySystemComponent.h:576`、`:581`、`:586`、`:597`、`:1905`。
- `FGameplayTagCountContainer` 维护层级 tag count、explicit tag count、tag event delegate；0/非0 变化触发 `NewOrRemoved`，任意 count 变化触发 `AnyCountChange`；源码路径：`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Public/GameplayEffectTypes.h:1043`、`:1259`、`:1298`、`:1301`、`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Private/GameplayEffectTypes.cpp:582`、`:597`。
- Loose GameplayTags 不复制；Replicated Loose Tags 通过 `FMinimalReplicationTagCountMap ReplicatedLooseTags` 复制；Minimal replication mode 下 GE granted tags 通过 `MinimalReplicationTags` 补充给非 owner；源码路径：`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Public/AbilitySystemComponent.h:649`、`:690`、`:719`、`:1966`、`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Private/AbilitySystemComponent.cpp:1642`、`:1657`。
- Ability tag 条件集中在 `UGameplayAbility`：`AbilityTags`/AssetTags 表达 Ability 分类，`ActivationOwnedTags` 表达激活期间拥有状态，Required/Blocked tags 决定 CanActivate，Cancel/Block tags 决定激活时对其他 Ability 的取消/阻塞；源码路径：`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Public/Abilities/GameplayAbility.h:496`、`:757`、`:761`、`:765`、`:769`、`:773`。
- GE tag 体系在 UE5.6 中主要由 GEComponents 承载：`UAssetTagsGameplayEffectComponent` 配 Asset Tags，`UTargetTagsGameplayEffectComponent` 配 Granted Tags，`UTargetTagRequirementsGameplayEffectComponent` 配 Application/Ongoing/Removal requirements，旧字段已 deprecated；源码路径：`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Public/GameplayEffectComponents/AssetTagsGameplayEffectComponent.h:13`、`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Public/GameplayEffectComponents/TargetTagsGameplayEffectComponent.h:13`、`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Public/GameplayEffectComponents/TargetTagRequirementsGameplayEffectComponent.h:15`、`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Public/GameplayEffect.h:2311`、`:2316`、`:2326`。
- GameplayTagResponseTable 的实际类名是 `UGameplayTagReponseTable`，源码历史拼写少了一个 `s`；它监听 ASC tag count，并用 positive/negative count 差应用、移除或更新响应 GE level；源码路径：`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Public/GameplayTagResponseTable.h:19`、`:41`、`:55`、`:62`、`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Private/GameplayTagResponseTable.cpp:98`、`:120`、`:181`。
- `UAbilitySystemGlobals` 从 `UGameplayAbilitiesDeveloperSettings::GameplayTagResponseTableName` 加载 ResponseTable；ASC 在 `InitAbilityActorInfo` 过程中注册表的 tag event；源码路径：`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Public/GameplayAbilitiesDeveloperSettings.h:102`、`:104`、`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Private/AbilitySystemGlobals.cpp:625`、`:630`、`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Private/AbilitySystemComponent_Abilities.cpp:186`。

## GameplayTag / ResponseTable 速查

- 判断状态：优先查 ASC owned tags，即 `HasMatchingGameplayTag` / `GetGameplayTagCount`；源码路径：`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Public/AbilitySystemComponent.h:576`、`:686`。
- 监听状态：C++ 用 `RegisterGameplayTagEvent`，蓝图用 `UAbilitySystemBlueprintLibrary::BindEventWrapperToGameplayTagChanged`，Ability 内异步等待用 WaitGameplayTag 系列 task；源码路径：`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Public/AbilitySystemComponent.h:743`、`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Public/AbilitySystemBlueprintLibrary.h:99`、`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Public/Abilities/Tasks/AbilityTask_WaitGameplayTagBase.h:15`。
- 授予持续状态：优先用 GE Granted Tags；源码路径：`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Public/GameplayEffect.h:2147`、`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Private/GameplayEffect.cpp:4373`。
- Ability 激活期间的临时状态：用 `ActivationOwnedTags`，结束时自动清；源码路径：`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Private/Abilities/GameplayAbility.cpp:963`、`:837`。
- 不需要 GE 的手动状态：本地用 Loose Tags，需要复制时用 Replicated Loose Tags；源码路径：`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Public/AbilitySystemComponent.h:649`、`:690`。
- 全局 tag count 到 GE 响应：用 `UGameplayTagReponseTable`，注意触发 tag 与响应 GE granted tags 避免互相递归；源码路径：`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Public/GameplayTagResponseTable.h:55`、`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Private/GameplayTagResponseTable.cpp:66`、`:181`。
- 表现路由：用 `GameplayCue.*` tag，不建议把 GameplayCueTag 当核心状态 tag；开发实践推断，源码依据是 Cue tag 用于 CueSet/Notify 路由，而状态查询来自 ASC tag count；源码路径：`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Public/GameplayCueSet.h:18`、`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Public/AbilitySystemComponent.h:597`。

## 第十二轮未确认项

- `UGameplayTagReponseTable` 是否有专门编辑器 details customization，本轮未确认。
- `UBlockAbilityTagsGameplayEffectComponent` 在 ActiveGE add/remove 时具体如何把 `CachedBlockedAbilityTags` 接入 ASC `BlockedAbilityTags` 的完整调用链，本轮未完全展开，未确认。
- `UAbilityTask_WaitGameplayTagQuery` 的完整内部实现本轮未逐行展开，未确认。
- GameplayTagResponseTable 响应 GE 与 GE stacking policy 的全部组合行为，本轮未完整展开，未确认。
- 预测失败后 tag 变化的完整回滚路径，本轮未展开，未确认。

# 核心类：GameplayEffectComponent / GEComponents（第十三轮）

完整专题见 `gameplay-effect-components.md`。本节只保留索引，便于从 GameplayEffect、GameplayTag、网络预测和编辑器配置快速跳转。

## 类定位索引

- `UGameplayEffectComponent` 是 `UGameplayEffect` 内的配置子对象基类，源码注释明确 GEComponents 住在 GameplayEffect 内，并且一个 GEComponent 会被所有应用实例共享，因此不应保存每次应用的运行时状态；源码路径：`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Public/GameplayEffectComponent.h:17`、`:23`、`:24`。
- `UGameplayEffect` 用 `GEComponents` 数组保存组件，属性是 `Instanced`，编辑器显示名为 `Components`；源码路径：`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Public/GameplayEffect.h:2421`、`:2422`、`:2423`。
- GEComponents 的核心 hook 是 `CanGameplayEffectApply`、`OnActiveGameplayEffectAdded`、`OnGameplayEffectExecuted`、`OnGameplayEffectApplied`、`OnGameplayEffectChanged` 和编辑器 `IsDataValid`；源码路径：`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Public/GameplayEffectComponent.h:47`、`:54`、`:60`、`:66`、`:71`、`:81`。
- `UGameplayEffect::CanApply`、`OnAddedToActiveContainer`、`OnExecuted`、`OnApplied` 会遍历 `GEComponents` 并调用对应 hook；源码路径：`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Private/GameplayEffect.cpp:881`、`:895`、`:911`、`:924`。
- UE5.6 实际扫描到 10 个 GEComponent：AssetTags、TargetTags、TargetTagRequirements、BlockAbilityTags、Immunity、RemoveOther、AdditionalEffects、ChanceToApply、CustomCanApply、Abilities；源码路径：`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Public/GameplayEffectComponents`、`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Private/GameplayEffectComponents`。
- 当前扫描范围未确认存在 `UGameplayCueComponent` 或等价 GEComponent；`UGameplayEffect::GameplayCues` 字段仍存在，并仍由 GE/ActiveGE 路径触发；源码路径：`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Public/GameplayEffect.h:2299`、`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Private/GameplayEffect.cpp:4414`。

## 组件职责索引

| 组件 | 什么时候看它 | 源码路径 |
|---|---|---|
| `UAssetTagsGameplayEffectComponent` | GE 自身 asset tags / 查询分类 | `Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Public/GameplayEffectComponents/AssetTagsGameplayEffectComponent.h:13`、`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Private/GameplayEffectComponents/AssetTagsGameplayEffectComponent.cpp:63` |
| `UTargetTagsGameplayEffectComponent` | GE 授予目标 ASC 的 granted tags | `Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Public/GameplayEffectComponents/TargetTagsGameplayEffectComponent.h:13`、`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Private/GameplayEffect.cpp:4373` |
| `UTargetTagRequirementsGameplayEffectComponent` | Application/Ongoing/Removal tag requirements | `Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Public/GameplayEffectComponents/TargetTagRequirementsGameplayEffectComponent.h:15`、`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Private/GameplayEffectComponents/TargetTagRequirementsGameplayEffectComponent.cpp:19`、`:143` |
| `UBlockAbilityTagsGameplayEffectComponent` | GE 存在期间阻塞一类 Ability | `Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Public/GameplayEffectComponents/BlockAbilityTagsGameplayEffectComponent.h:13`、`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Private/GameplayEffect.cpp:4377`、`:4674` |
| `UImmunityGameplayEffectComponent` | ActiveGE 存在期间免疫 incoming GE | `Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Public/GameplayEffectComponents/ImmunityGameplayEffectComponent.h:16`、`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Private/GameplayEffectComponents/ImmunityGameplayEffectComponent.cpp:24`、`:66` |
| `URemoveOtherGameplayEffectComponent` | 本 GE 应用成功后移除其他 ActiveGE | `Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Public/GameplayEffectComponents/RemoveOtherGameplayEffectComponent.h:15`、`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Private/GameplayEffectComponents/RemoveOtherGameplayEffectComponent.cpp:16`、`:39` |
| `UAdditionalEffectsGameplayEffectComponent` | 应用成功或 ActiveGE 移除时附加 GE | `Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Public/GameplayEffectComponents/AdditionalEffectsGameplayEffectComponent.h:13`、`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Private/GameplayEffectComponents/AdditionalEffectsGameplayEffectComponent.cpp:22`、`:98` |
| `UChanceToApplyGameplayEffectComponent` | GE 概率应用 | `Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Public/GameplayEffectComponents/ChanceToApplyGameplayEffectComponent.h:14`、`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Private/GameplayEffectComponents/ChanceToApplyGameplayEffectComponent.cpp:24` |
| `UCustomCanApplyGameplayEffectComponent` | 自定义应用条件 | `Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Public/GameplayEffectComponents/CustomCanApplyGameplayEffectComponent.h:14`、`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Private/GameplayEffectComponents/CustomCanApplyGameplayEffectComponent.cpp:18` |
| `UAbilitiesGameplayEffectComponent` | ActiveGE 存在期间授予 Ability | `Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Public/GameplayEffectComponents/AbilitiesGameplayEffectComponent.h:38`、`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Private/GameplayEffectComponents/AbilitiesGameplayEffectComponent.cpp:44`、`:98` |

## 第十三轮未确认项

- GEComponents 本身作为 GE asset subobject 的 cook/load/asset replication 细节未展开，未确认。
- 当前扫描范围未发现 `UGameplayCueComponent`；是否存在项目侧或其他模块扩展，未确认。
- `GameplayEffectDetails` 是否提供 GEComponents 专门添加/删除/排序 UI，当前在指定编辑器文件中未确认。
- Prediction reject / rollback 时，由 AdditionalEffects、ChanceToApply、Immunity query 引发的所有表现清理细节未完整展开，未确认。
