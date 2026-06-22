# 常见坑：UAbilitySystemComponent（第二轮）

## ASC 放在 Character 和 PlayerState 的区别

- 源码确认 ASC 区分 `OwnerActor` 与 `AvatarActor`：Owner 是逻辑拥有者，Avatar 是世界中的物理表现，通常是 Pawn，也可能是其他 Actor，二者可以相同；源码路径：`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Public/AbilitySystemComponent.h:1546`。
- 源码确认 `InitAbilityActorInfo` 会设置 Owner/Avatar，并在 Avatar 改变时对已授予 Ability 调用 `OnAvatarSet`；源码路径：`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Private/AbilitySystemComponent_Abilities.cpp:141`。
- Character vs PlayerState 的具体取舍不是本轮指定源码直接规定的内容，未确认。可从源码得出的保守结论是：只要 Owner/Avatar 初始化正确，ASC 支持逻辑拥有者与物理 Avatar 分离；源码路径：`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Public/AbilitySystemComponent.h:1548`。

## InitAbilityActorInfo 的调用时机

- `TryActivateAbility` 会在 ActorInfo、OwnerActor 或 AvatarActor 无效时直接返回 false；源码路径：`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Private/AbilitySystemComponent_Abilities.cpp:1607`。
- `InternalTryActivateAbility` 也会重复检查 ActorInfo、OwnerActor、AvatarActor，无效则返回 false；源码路径：`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Private/AbilitySystemComponent_Abilities.cpp:1705`。
- `InvokeGameplayCueEvent` 遇到 `AbilityActorInfo` 为空会触发 ensure，提示 ASC 的 `OnRegister` 可能还没调用；源码路径：`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Private/AbilitySystemComponent.cpp:1287`。
- 因此业务层在 Possess/OnRep_PlayerState/Avatar 切换等时机需要确保调用或刷新 `InitAbilityActorInfo`；具体项目时机本轮未确认。

## GiveAbility 只应在服务端执行

- `GiveAbility` 注释写明 actor 非 authoritative 时会被忽略；实现中客户端调用会记录 error 并返回空 handle；源码路径：`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Public/AbilitySystemComponent.h:970`、`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Private/AbilitySystemComponent_Abilities.cpp:281`。
- `ClearAbility` 与 `ClearAllAbilities` 同样 authority-only，客户端调用会记录错误并返回；源码路径：`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Private/AbilitySystemComponent_Abilities.cpp:427`、`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Private/AbilitySystemComponent_Abilities.cpp:477`。

## AttributeSet 没有复制的问题

- `SpawnedAttributes` 是 replicated 属性，并有 `OnRep_SpawnedAttributes`；源码路径：`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Public/AbilitySystemComponent.h:1956`。
- `GetLifetimeReplicatedProps` 会复制 `SpawnedAttributes`，`ReplicateSubobjects` 还会逐个复制有效 AttributeSet 子对象；源码路径：`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Private/AbilitySystemComponent.cpp:1636`、`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Private/AbilitySystemComponent.cpp:1710`。
- `AddSpawnedAttribute` 会在 registered subobject list 可用且 ready for replication 时调用 `AddReplicatedSubObject`，并标记 SpawnedAttributes dirty；源码路径：`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Private/AbilitySystemComponent.cpp:3005`。
- 常见坑是绕过 ASC API 直接管理 AttributeSet，导致 `SpawnedAttributes` 未 dirty 或 subobject 未注册。AttributeSet 内属性本身是否正确复制还取决于 AttributeSet 类的属性复制声明，本轮未展开，未确认。

## Ability 激活失败但没有日志

- `TryActivateAbility` 对 invalid handle/Ability 会打 Warning，但 ActorInfo 无效、simulated proxy、pending remove 等路径会直接返回 false；源码路径：`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Private/AbilitySystemComponent_Abilities.cpp:1585`、`:1593`、`:1607`、`:1618`。
- `CanActivateAbility` 的 cooldown/cost/tag/input/Blueprint 失败日志受 `FScopedCanActivateAbilityLogEnabler` 控制；ASC 在部分路径会创建该日志 enabler；源码路径：`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Private/AbilitySystemComponent_Abilities.cpp:1643`、`:1795`、`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Private/Abilities/GameplayAbility.cpp:475`。
- `InternalTryActivateAbility` 会把失败 tag 存到 `InternalTryActivateAbilityFailureTags`，没有 failure tag 时会补 `ActivateFailCanActivateAbilityTag`；源码路径：`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Private/AbilitySystemComponent_Abilities.cpp:1796`。

## CommitAbility 失败的问题

- `UGameplayAbility::ActivateAbility` 源码注释明确：Blueprinted ActivateAbility 必须在执行链某处调用 `CommitAbility`；Native 子类 override 也应调用并检查结果；源码路径：`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Private/Abilities/GameplayAbility.cpp:884`。
- `CommitAbility` 会先执行 `CommitCheck`，失败就返回 false；然后执行 `CommitExecute`、Blueprint commit execute，并通知 ASC commit；源码路径：`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Private/Abilities/GameplayAbility.cpp:559`。
- `CommitAbilityCost` 会再次 `CheckCost`，失败返回 false；`CommitAbilityCooldown` 除非 ForceCooldown，否则会再次 `CheckCooldown`；源码路径：`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Private/Abilities/GameplayAbility.cpp:578`、`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Private/Abilities/GameplayAbility.cpp:598`。
- 常见坑是 `CanActivateAbility` 成功后，激活过程中状态变化导致 Commit 阶段资源或 cooldown 检查失败；这是源码注释确认的场景；源码路径：`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Private/Abilities/GameplayAbility.cpp:615`。

## GameplayTag 没配好导致 Ability 无法激活

- `UGameplayAbility::CanActivateAbility` 会调用 `DoesAbilitySatisfyTagRequirements`，如果 Ability tags 被 blocked、存在 Blocking tag、缺少 Required tag，就返回 false；源码路径：`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Private/Abilities/GameplayAbility.cpp:495`。
- `CanActivateAbility` 还会检查 `IsAbilityInputBlocked(Spec->InputID)`；源码路径：`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Private/Abilities/GameplayAbility.cpp:505`。
- ASC 的 `BlockedAbilityTags` 由 `BlockAbilitiesWithTags`/`UnBlockAbilitiesWithTags` 修改，`AreAbilityTagsBlocked` 用它判断传入 tags 是否被阻塞；源码路径：`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Private/AbilitySystemComponent_Abilities.cpp:1427`、`:1433`、`:1438`。
- loose tags 默认不复制，源码注释明确需要调用方自行保证客户端/服务端都添加，若需要复制应使用 replicated loose tag 系列；源码路径：`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Public/AbilitySystemComponent.h:649`、`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Public/AbilitySystemComponent.h:690`。

# 常见坑：UGameplayAbility（第三轮）

## ActivateAbility 里忘记 CommitAbility

- 源码注释明确 `K2_ActivateAbility` 图应该调用 `CommitAbility`，C++ override 也应该主动调用并检查结果；ASC 只调用 `CallActivateAbility`，不会自动 Commit；源码路径：`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Public/Abilities/GameplayAbility.h:577`、`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Private/Abilities/GameplayAbility.cpp:884`。
- 忘记 Commit 会导致 Cost/Cooldown GE 不应用，因为默认 `CommitExecute` 才调用 `ApplyCooldown` 和 `ApplyCost`；源码路径：`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Private/Abilities/GameplayAbility.cpp:651`。

## CommitAbility 失败后没有 EndAbility

- `CommitAbility` 失败会直接返回 false，不会自动结束 Ability；源码路径：`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Private/Abilities/GameplayAbility.cpp:559`。
- 源码示例注释建议 `CommitAbility` 失败后调用 `EndAbility(..., bWasCancelled=true)`；源码路径：`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Private/Abilities/GameplayAbility.cpp:903`。

## NonInstanced Ability 使用成员变量

- NonInstanced Ability 在 CDO 上执行，源码注释明确不能有状态、不能有 replicated properties、不能在 ability class 上做 RPC；源码路径：`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Public/Abilities/GameplayAbility.h:69`。
- 访问 `CurrentActorInfo`、`CurrentActivationInfo`、`CurrentSpecHandle` 等实例状态时，源码会检查是否 instantiated；源码路径：`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Private/Abilities/GameplayAbility.cpp:1291`、`:1302`、`:1313`。
- UE 5.5 中 `NonInstanced` 已被标记 deprecated；源码路径：`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Public/Abilities/GameplayAbilityTypes.h:45`。

## AbilityTask 用在不合适的实例化策略中

- AbilityTask 依赖 `UGameplayAbility` 作为 `IGameplayTaskOwnerInterface`，Ability 会维护 `ActiveTasks`；源码路径：`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Public/Abilities/GameplayAbility.h:530`、`:818`。
- NonInstanced 无 per-activation 状态，不适合保存 ActiveTasks；这是基于源码“不能有状态”与 `ActiveTasks` 成员的推断；源码路径：`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Public/Abilities/GameplayAbility.h:69`、`:818`。
- InstancedPerExecution 对输入交互有警告：只可能与最新生成的实例交互，输入事件不可靠；源码路径：`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Private/AbilitySystemComponent_Abilities.cpp:2799`、`:2835`。

## LocalPredicted Ability 没处理预测失败

- `LocalPredicted` 分支会本地创建 prediction window、调用服务端激活 RPC，并绑定 caught-up delegate；源码路径：`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Private/AbilitySystemComponent_Abilities.cpp:1904`。
- 服务端可能因 spec 无效、安全策略、`InternalTryActivateAbility` 失败而调用 `ClientActivateAbilityFailed`；源码路径：`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Private/AbilitySystemComponent_Abilities.cpp:2026`。
- 完整预测失败回滚路径本轮未展开，未确认；业务层至少不能假设本地预测一定被服务端接受。

## Cost / Cooldown GE 没配置导致 Commit 逻辑异常

- 未配置 Cost/Cooldown class 时 `GetCostGameplayEffect` / `GetCooldownGameplayEffect` 返回 `nullptr`，对应 Check/Apply 基本变成无事发生；源码路径：`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Private/Abilities/GameplayAbility.cpp:1026`、`:1038`。
- Cooldown 检查依赖 Cooldown GE 的 granted tags；若 GE 没有 granted tags，`CheckCooldown` 没有可匹配的 cooldown tag；源码路径：`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Private/Abilities/GameplayAbility.cpp:1057`、`:1197`。
- Cost 检查依赖 ASC `CanApplyAttributeModifiers`；Cost GE attribute/modifier 配置不满足时会失败并加 cost failure tag；源码路径：`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Private/Abilities/GameplayAbility.cpp:1092`。

## Tag Requirements 配错导致 Ability 无法激活

- `DoesAbilitySatisfyTagRequirements` 会先检查 blocked tags，再检查 required tags，并把失败原因加入 OptionalRelevantTags；源码路径：`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Private/Abilities/GameplayAbility.cpp:316`。
- `ActivationBlockedTags`、`SourceBlockedTags`、`TargetBlockedTags` 命中会阻止；`ActivationRequiredTags`、`SourceRequiredTags`、`TargetRequiredTags` 缺失会阻止；源码路径：`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Private/Abilities/GameplayAbility.cpp:373`。
- `BlockAbilitiesWithTag` 通过 ASC 写入 `BlockedAbilityTags`，目标 Ability 必须有对应 `AbilityTags` 才容易被阻塞匹配；源码路径：`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Private/Abilities/GameplayAbility.cpp:985`、`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Private/Abilities/GameplayAbility.cpp:374`。

## EndAbility 没调用导致 Ability 一直 active

- `PreActivate` 会递增 `Spec->ActiveCount`，`NotifyAbilityEnded` 才递减；源码路径：`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Private/Abilities/GameplayAbility.cpp:995`、`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Private/AbilitySystemComponent_Abilities.cpp:1222`。
- `EndAbility` 负责移除 `ActivationOwnedTags`、停止/清理 `ActiveTasks`、移除 tracked GameplayCues、解除 block tags、通知 ASC ended；源码路径：`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Private/Abilities/GameplayAbility.cpp:769`。
- ASC 清空 Ability 时有 ensure 提示 active ability 仍未结束通常意味着 TryActivateAbility 与 EndAbility 不匹配，或 instanced ability 没正确走 End/Super；源码路径：`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Private/AbilitySystemComponent_Abilities.cpp:441`。

## Blueprint Ability 激活后 latent task 没结束

- 源码允许 `K2_ActivateAbility` 返回时尚未调用 Commit/End，但仅当 latent/async actions pending；逻辑结束时仍期望 Commit/End 已调用；源码路径：`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Public/Abilities/GameplayAbility.h:583`。
- `EndAbility` 会对所有 `ActiveTasks` 调用 `TaskOwnerEnded` 并清空数组；如果蓝图 latent task 没有走到 EndAbility，任务和 Ability 状态都可能悬挂；源码路径：`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Private/Abilities/GameplayAbility.cpp:819`。

# 常见坑：GameplayEffect 体系（第四轮）

## 把 GameplayEffect CDO 当成运行时状态对象使用

- `UGameplayEffect` 是配置定义/CDO，源码注释明确它是编辑器中的数据资产；运行时每次应用的状态应在 `FGameplayEffectSpec` 或 `FActiveGameplayEffect` 上，而不是写 GE CDO；源码路径：`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Public/GameplayEffect.h:2071`、`:988`、`:1319`。
- `UGameplayEffectComponent` 与 GE 一样只有一个资产级实例，源码注释明确不应保存每次执行的运行时状态；源码路径：`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Public/GameplayEffectComponent.h:17`。

## 混淆 UGameplayEffect / FGameplayEffectSpec / FActiveGameplayEffect

- `UGameplayEffect` 是静态定义，`FGameplayEffectSpec` 是应用时携带 Context/Level/SetByCaller/捕获属性的可变规格，`FActiveGameplayEffect` 是进入 ASC 容器后的 active 实例；源码路径：`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Public/GameplayEffect.h:2071`、`:988`、`:1319`。
- Instant GE 的非预测路径不会加入 `ActiveGameplayEffects`，而是直接 `ExecuteGameplayEffect`；Duration/Infinite 或预测 Instant 才进入 ActiveGE 容器；源码路径：`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Private/AbilitySystemComponent.cpp:878`、`:952`。

## Cost GE / Cooldown GE 配错导致 CommitAbility 失败

- `CommitAbility` 会在 `CommitCheck` 中重新检查 cooldown/cost，失败直接返回 false；源码路径：`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Private/Abilities/GameplayAbility.cpp:559`、`:615`。
- Cost 检查走 ASC `CanApplyAttributeModifiers`，该函数会构造 Cost GE Spec、计算 modifier，并对 additive modifier 判断当前值加 cost 是否小于 0；源码路径：`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Private/GameplayEffect.cpp:5177`、`:5190`。

## Cooldown GE 没有 Granted Tag 导致冷却判断异常

- `CheckCooldown` 读取 `GetCooldownTags()`；默认 `GetCooldownTags()` 返回 Cooldown GE 的 `GetGrantedTags()`，如果 Cooldown GE 没授予冷却 tag，后续检查没有可匹配 tag；源码路径：`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Private/Abilities/GameplayAbility.cpp:1050`、`:1197`、`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Public/GameplayEffect.h:2147`。

## SetByCaller Magnitude 没设置导致数值为 0 或报错

- `FGameplayEffectModifierMagnitude::AttemptCalculateMagnitude` 遇到 SetByCaller 会从 Spec 的 tag/name map 读取；未设置且 warning 开启时 `GetSetByCallerMagnitude` 会记录 error 并返回默认值；源码路径：`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Private/GameplayEffect.cpp:1163`、`:2216`、`:2238`。
- Ability 生成 outgoing Spec 时会把 `FGameplayAbilitySpec` 上的 `SetByCallerTagMagnitudes` 拷贝到 GE Spec，但业务仍需确保对应 tag 已设置；源码路径：`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Private/Abilities/GameplayAbility.cpp:1347`。

## Instant GE 和 Duration GE 的属性修改理解错误

- Instant/periodic execute 路径通过 `InternalExecuteMod` 修改属性 base value，并触发 `PreGameplayEffectExecute` / `PostGameplayEffectExecute`；源码路径：`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Private/GameplayEffect.cpp:3065`、`:3907`、`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Public/AttributeSet.h:198`、`:204`。
- Duration/Infinite 且无 period 的 GE 会把 modifier 注册到属性 aggregator，源码注释明确 `PreGameplayEffectExecute` 不会因类似 5 秒 +10 移速 buff 的应用型效果调用；源码路径：`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Private/GameplayEffect.cpp:4312`、`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Public/AttributeSet.h:198`。

## AttributeBased Magnitude 捕获 Source / Target 搞反

- `FGameplayEffectAttributeCaptureDefinition` 明确要求指定 `AttributeSource` 是 Source 还是 Target，并指定是否 snapshot；配置反了会让 AttributeBased/MMC 从错误一侧取值；源码路径：`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Public/GameplayEffectAttributeCaptureDefinition.h:23`。

## Modifier 和 ExecutionCalculation 混用

- `Modifiers` 是数据驱动的属性修改配置；`Executions` 是自定义执行计算，运行时通过 `FGameplayEffectCustomExecutionOutput` 产出 evaluated modifiers；源码路径：`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Public/GameplayEffect.h:2247`、`:2251`、`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Public/GameplayEffectExecutionCalculation.h:201`。
- ExecutionCalculation 只在 GE 执行路径运行，不能把它当作持续 aggregator modifier 的替代；执行路径源码：`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Private/GameplayEffect.cpp:3065`、`:3155`。

## Stacking 配置不符合预期

- ActiveGE 应用时会先查找可堆叠实例，再按 `StackLimitCount`、overflow、duration refresh、period reset 策略更新已有 ActiveGE；源码路径：`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Private/GameplayEffect.cpp:4008`、`:4032`、`:4061`、`:4080`。
- Stack 变化会重新更新 modifier magnitude、通知 tag count change 并广播 `OnStackChanged`；源码路径：`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Private/GameplayEffect.cpp:3427`。

## Periodic GE 是否首帧执行配置错误

- `bExecutePeriodicEffectOnApplication` 为 true 时，应用后会通过 TimerManager `SetTimerForNextTick` 先执行一次；否则要等第一个 period；源码路径：`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Public/GameplayEffect.h:2239`、`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Private/GameplayEffect.cpp:4224`。
- 带有效 prediction key 的 periodic GE 不允许客户端预测，客户端会直接返回空 handle；源码路径：`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Private/AbilitySystemComponent.cpp:822`。

## GameplayCue 没触发或重复触发

- `bRequireModifierSuccessToTriggerCues` 会让 Execute Cue 依赖 modifier/execution 是否成功；源码路径：`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Public/GameplayEffect.h:2290`、`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Private/GameplayEffect.cpp:3028`。
- `bSuppressStackingCues` 会抑制 stacking 场景下重复触发 cue；源码路径：`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Public/GameplayEffect.h:2294`、`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Private/AbilitySystemComponent.cpp:925`。
- Minimal/Mixed replication 下 ActiveGE 和 GameplayCue 复制路径不同，Minimal 模式下 ActiveGE 不复制而通过 minimal replication cues/tags 补充表现；源码路径：`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Private/GameplayEffect.cpp:4918`、`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Public/AbilitySystemComponent.h:1887`。

## 预测场景下 GE 应用和回滚未确认

- 客户端预测 Instant GE 会被当成 Infinite duration 加入 ActiveGE，等待 prediction key caught up 或 reject 时移除；源码路径：`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Private/AbilitySystemComponent.cpp:857`、`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Private/GameplayEffect.cpp:4258`。
- 完整预测失败、服务端纠正、GameplayCue/Attribute 回滚链路本轮未展开，未确认。

## UE5.6 GEComponents 迁移导致旧字段误用

- `InheritableGameplayEffectTags`、`InheritableOwnedTagsContainer`、`OngoingTagRequirements`、`ApplicationTagRequirements`、`GrantedApplicationImmunityQuery`、`GrantedAbilities` 等旧字段在 UE5.6 已标记 deprecated，并指向对应 GEComponent；源码路径：`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Public/GameplayEffect.h:2311`、`:2316`、`:2327`、`:2332`、`:2353`、`:2401`。

# 常见坑：AttributeSet 体系（第五轮）

## AttributeSet 创建了但没有加入 ASC 管理

- ASC 只从 `SpawnedAttributes` 查找 AttributeSet，`GetAttributeSubobject` 会遍历 `GetSpawnedAttributes()`；如果项目自己创建 AttributeSet 但没有 `AddAttributeSetSubobject` / `AddSpawnedAttribute`，ASC 查询、GE modifier 和复制路径都可能找不到它；源码路径：`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Private/AbilitySystemComponent.cpp:126`、`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Public/AbilitySystemComponent.h:152`。
- `AddSpawnedAttribute` 还负责把 subobject 加入 replicated subobject list 并标记 `SpawnedAttributes` dirty；源码路径：`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Private/AbilitySystemComponent.cpp:3005`。

## AttributeSet subobject 复制了，但具体 Attribute 没有正确复制

- `SpawnedAttributes` 复制和 `ReplicateSubobjects` 只保证 AttributeSet 对象/列表参与复制；具体 Health/Mana 属性仍要在 AttributeSet 类上配置复制；源码路径：`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Private/AbilitySystemComponent.cpp:1637`、`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Private/AbilitySystemComponent.cpp:1710`。
- GAS 预测说明示例要求属性使用 `DOREPLIFETIME_CONDITION_NOTIFY(..., REPNOTIFY_Always)`；该宏属于 UE 通用复制系统，本轮未展开；源码路径：`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Public/GameplayPrediction.h:127`、`:138`。

## 属性声明了 ReplicatedUsing 但没有 DOREPLIFETIME

- `ReplicatedUsing` / `DOREPLIFETIME` 属于 UE 通用复制系统，本轮未展开；GAS 侧确认的是 RepNotify 内应调用 `GAMEPLAYATTRIBUTE_REPNOTIFY`，且预测属性建议 `REPNOTIFY_Always`；源码路径：`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Public/AttributeSet.h:396`、`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Public/GameplayPrediction.h:138`。

## OnRep_XXX 里忘记使用 GAMEPLAYATTRIBUTE_REPNOTIFY

- `GAMEPLAYATTRIBUTE_REPNOTIFY` 会把复制来的属性值交给 owning ASC 的 `SetBaseAttributeValueFromReplication`，用于更新 base/final value 和 delegate；忘记调用会绕过 GAS 的预测修正与 attribute delegate 路径；源码路径：`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Public/AttributeSet.h:404`、`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Public/AbilitySystemComponent.h:812`、`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Private/GameplayEffect.cpp:3566`。

## 直接 Set 属性值绕过 GAS 流程

- `GAMEPLAYATTRIBUTE_VALUE_SETTER` 都是转回 ASC `SetNumericAttributeBase`，GE 修改也通过 ActiveGE 容器；直接改成员值容易跳过 aggregator、AttributeSet 回调和 delegate；源码路径：`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Public/AttributeSet.h:443`、`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Private/AbilitySystemComponent.cpp:386`、`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Private/GameplayEffect.cpp:3765`。

## 把 Damage 当成永久属性保存导致逻辑混乱

- 源码测试 AttributeSet 把 `Damage` 注释为非持久属性，只用于应用负向 Health mod；这不是引擎固定规则，但说明 Damage 常作为 transient/meta 流程而不是长期状态；源码路径：`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Public/AbilitySystemTestAttributeSet.h:36`。
- “Damage Meta Attribute” 是开发实践推断，本轮源码未确认固定 Meta Attribute 类型。

## 在 PreAttributeChange 中只 Clamp CurrentValue，却没有处理 BaseValue

- `PreAttributeChange` 适合 clamp current/final value；但源码另有 `PreAttributeBaseChange`，注释明确如果希望 base value 也被 clamp，应在这里处理；源码路径：`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Public/AttributeSet.h:211`、`:225`。

## MaxHealth 改变后 Health 没同步 Clamp

- GAS 源码只提供 `PreAttributeChange` / `PreAttributeBaseChange` 回调机制，没有自动规定 MaxHealth 变化时如何同步 Health；需要项目侧在合适回调或外部系统中处理，这是开发实践推断；源码路径：`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Public/AttributeSet.h:211`、`:225`。

## Instant GE 和 Duration GE 对 AttributeSet 回调的触发理解错误

- Instant/periodic execute 路径会进入 `InternalExecuteMod`，触发 `PreGameplayEffectExecute` / `PostGameplayEffectExecute` 并修改 base value；源码路径：`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Private/GameplayEffect.cpp:3907`、`:3930`、`:3946`。
- Duration/Infinite 的普通持续 modifier 通常注册到 `FAggregator`，更新 current value；源码注释明确类似 5 秒 +10 移速 buff 的应用不会触发 `PreGameplayEffectExecute`；源码路径：`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Private/GameplayEffect.cpp:4347`、`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Public/AttributeSet.h:193`。

## PostGameplayEffectExecute 中写了只应该服务端执行的逻辑但没有判断 Authority

- `PostGameplayEffectExecute` 会在 GE execute 路径调用；普通 GE 应用受 ASC 权限/预测逻辑影响，但完整预测/客户端回滚路径本轮未展开，未确认；源码路径：`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Private/AbilitySystemComponent.cpp:798`、`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Private/GameplayEffect.cpp:3946`。
- 开发实践推断：死亡、发奖励、生成 Actor 等权威逻辑需要显式考虑 Authority，不要只因为在 AttributeSet 回调中就假设一定只跑服务端。

## UI 直接 Tick 读取属性而不是监听 AttributeChangeDelegate

- ASC 提供 `GetGameplayAttributeValueChangeDelegate`，ActiveGE 容器在属性 current value 更新后广播 `FOnGameplayAttributeValueChange`；开发实践推断，UI 应绑定 delegate 而不是 Tick 轮询；源码路径：`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Public/AbilitySystemComponent.h:534`、`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Private/GameplayEffect.cpp:3790`。

## AttributeSet 中写了过多业务逻辑，导致和 Character / Ability / Effect 耦合过重

- 源码注释明确 `PreAttributeChange` 用于 clamp，不用于触发“damage applied”这类 gameplay events；测试类关于死亡处理也说明可以在 AttributeSet 或 Actor 层，取舍依项目；源码路径：`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Public/AttributeSet.h:211`、`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Private/AbilitySystemTestAttributeSet.cpp:112`。
- 开发实践推断：AttributeSet 更适合做属性约束、meta attribute 转换和轻量通知；复杂战斗流程应放到 Ability、GE/Execution、ASC/Character 或项目事件系统。
