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

# 常见坑：AbilityTask / GameplayTask 异步流程（第六轮）

## AbilityTask 创建后忘记调用 ReadyForActivation

- `NewAbilityTask` 只创建对象并调用 `InitTask`，不会自动执行派生 `Activate`；真正进入激活流程的是 `UGameplayTask::ReadyForActivation`；源码路径：`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Public/Abilities/Tasks/AbilityTask.h:130`、`Engine/Source/Runtime/GameplayTasks/Private/GameplayTask.cpp:58`。
- 开发实践推断：C++ 中手动创建 Task 后忘记 `ReadyForActivation`，Task delegate 不会被触发；蓝图节点由 `K2Node_LatentAbilityCall` 封装调用细节，本轮未展开；源码路径：`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Public/Abilities/Tasks/AbilityTask.h:23`。

## Task 绑定 delegate 后没有在 OnDestroy / EndTask 中解绑

- 源码注释要求 AbilityTask override `OnDestroy` 并 unregister callbacks，且 `UGameplayTask::OnDestroy` 注释要求最后调用 `Super::OnDestroy`；源码路径：`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Public/Abilities/Tasks/AbilityTask.h:43`、`Engine/Source/Runtime/GameplayTasks/Classes/GameplayTask.h:291`。
- 内置 Task 示例会在 `OnDestroy` 中移除 ASC/Ability/Anim/Attribute delegates；源码路径：`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Private/Abilities/Tasks/AbilityTask_WaitGameplayEvent.cpp:79`、`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Private/Abilities/Tasks/AbilityTask_PlayMontageAndWait.cpp:204`、`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Private/Abilities/Tasks/AbilityTask_WaitAttributeChange.cpp:119`。

## Ability 结束后 Task 仍然持有对象引用

- Ability `EndAbility` 会对 `ActiveTasks` 调用 `TaskOwnerEnded`，`UAbilityTask::OnDestroy` 会清空 `Ability` 引用；源码路径：`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Private/Abilities/GameplayAbility.cpp:819`、`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Private/Abilities/Tasks/AbilityTask.cpp:113`。
- 开发实践推断：自定义 Task 如果在外部 delegate/timer/TargetActor 中遗留引用，就可能在 Ability 结束后回调失效对象；应按内置 Task 模式在 `OnDestroy` 解绑。

## NonInstanced Ability 中使用需要状态的 AbilityTask

- NonInstanced Ability 在 CDO 上执行，源码注释明确不能有状态；AbilityTask 又依赖 per-activation `ActiveTasks` 与 Ability/ASC 指针，这是不匹配的开发实践推断；源码路径：`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Public/Abilities/GameplayAbilityTypes.h:36`、`:45`、`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Public/Abilities/GameplayAbility.h:820`。

## InstancedPerExecution Ability 中依赖后续输入事件导致不可靠

- ASC 输入路径对 `InstancedPerExecution` 发出 warning：输入交互只可能作用于最新生成实例，因此输入事件不可靠；源码路径：`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Private/AbilitySystemComponent_Abilities.cpp:2799`、`:2835`。
- 开发实践推断：需要等待按下/释放、confirm/cancel、多帧 TargetData 的 Ability 更适合 `InstancedPerActor` 或明确管理实例生命周期。

## Task 广播 delegate 后忘记结束，导致 Ability 一直 active

- `UGameplayTask::EndTask` 才会进入 `OnDestroy` 和 deactivated 流程；Ability `EndAbility` 也只会清理仍在 `ActiveTasks` 中的 Task；源码路径：`Engine/Source/Runtime/GameplayTasks/Private/GameplayTask.cpp:167`、`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Private/Abilities/GameplayAbility.cpp:819`。
- 内置一次性 Task 通常广播后 `EndTask`，例如 `WaitInputPress`、`WaitDelay`、`WaitGameplayEffectRemoved`；源码路径：`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Private/Abilities/Tasks/AbilityTask_WaitInputPress.cpp:45`、`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Private/Abilities/Tasks/AbilityTask_WaitDelay.cpp:43`、`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Private/Abilities/Tasks/AbilityTask_WaitGameplayEffectRemoved.cpp:72`。

## Montage Task 中 Ability 取消但 Montage 没停

- `PlayMontageAndWait` 在 `OnDestroy` 中只有当 `AbilityEnded && bStopWhenAbilityEnds` 时才调用 `StopPlayingMontage`，而显式 `ExternalCancel` 会广播 `OnCancelled` 并走 `Super::ExternalCancel`；源码路径：`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Private/Abilities/Tasks/AbilityTask_PlayMontageAndWait.cpp:195`、`:204`、`:215`。
- `StopPlayingMontage` 还要求 ASC 当前 animating ability 和当前 montage 匹配，才会 `CurrentMontageStop`；源码路径：`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Private/Abilities/Tasks/AbilityTask_PlayMontageAndWait.cpp:247`、`:259`。

## TargetData Task 中客户端确认了但服务端没收到

- 客户端确认 TargetData 时通过 `CallServerSetReplicatedTargetData` 发送，服务端存入 `AbilityTargetDataMap` 并广播 `TargetSetDelegate`；源码路径：`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Private/Abilities/Tasks/AbilityTask_WaitTargetData.cpp:288`、`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Private/AbilitySystemComponent_Abilities.cpp:3945`。
- ASC 在 `CallServerSetReplicatedTargetData` 中会对无效 prediction key 打 warning；prediction key、AbilityHandle 或服务端监听 delegate 不匹配都会让 Task 等不到数据，这是基于源码路径的开发实践推断；源码路径：`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Private/AbilitySystemComponent_Abilities.cpp:4217`、`:4237`、`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Private/Abilities/Tasks/AbilityTask_WaitTargetData.cpp:211`。

## WaitGameplayEvent tag 配错导致事件不触发

- `WaitGameplayEvent` 默认 `OnlyMatchExact=true` 时只绑定 `GenericGameplayEventCallbacks.FindOrAdd(Tag)`；非精确匹配才走 `AddGameplayEventTagContainerDelegate`；源码路径：`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Private/Abilities/Tasks/AbilityTask_WaitGameplayEvent.cpp:29`。
- ASC `HandleGameplayEvent` 对 exact callback 用传入 `EventTag` 查找，对 container delegate 用 `EventTag.MatchesAny(SearchPair.Key)`；源码路径：`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Private/AbilitySystemComponent_Abilities.cpp:2559`、`:2570`。

## WaitAttributeChange 监听了错误 ASC 或错误 Attribute

- `WaitAttributeChange` 使用 `OptionalExternalOwner` 时会改为监听外部 Actor 的 ASC，否则监听 owning Ability 的 ASC；源码路径：`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Private/Abilities/Tasks/AbilityTask_WaitAttributeChange.cpp:18`、`:114`。
- 监听本身绑定 `GetGameplayAttributeValueChangeDelegate(Attribute)`，Attribute 选错或 ASC 选错都不会收到预期变化；源码路径：`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Private/Abilities/Tasks/AbilityTask_WaitAttributeChange.cpp:49`、`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Public/AbilitySystemComponent.h:534`。

## 预测场景下重复广播 delegate

- AbilityTask 基类只检查 Ability 是否 active 后再广播，不会自动替自定义 Task 去重；源码路径：`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Private/Abilities/Tasks/AbilityTask.cpp:190`。
- 预测文档说明客户端和服务端可能在不同时间执行同一逻辑，输入/TargetData 等 Task 通过 prediction window 与 replicated event/target data 协调；自定义 Task 的重复广播/回滚策略本轮未完整展开，未确认；源码路径：`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Public/GameplayPrediction.h:72`、`:192`。

## 在 AbilityTask 中写过多业务逻辑，导致复用困难

- 源码定义 AbilityTask 是 small、self-contained operation，模式是 start-and-wait；源码路径：`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Public/Abilities/Tasks/AbilityTask.h:20`。
- 开发实践推断：Task 更适合封装异步等待、delegate 转发、预测/RPC 小段接入；复杂伤害、死亡、冷却、GE 配置、UI 状态机不应塞进 Task。

# 常见坑：GameplayCue 体系（第七轮）

## GameplayCueTag 没有和 Notify 正确绑定

- CueSet 通过 `GameplayCueDataMap` 查 cue tag，找不到或映射为 `INDEX_NONE` 时直接返回 false；源码路径：`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Private/GameplayCueSet.cpp:78`、`:80`、`:90`。
- Manager 扫描资产时要求资产注册表里的 `GameplayCueName` 能转换成有效 GameplayTag，否则会记录 warning 并不加入全局集合；源码路径：`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Private/GameplayCueManager.cpp:944`、`:974`、`:999`。

## GameplayEffect 配了 GameplayCue 但 DurationPolicy 理解错误

- Instant/periodic execute 触发 `Executed`；Duration/Infinite 添加时触发 `OnActive`/`WhileActive`，移除时触发 `Removed`；源码路径：`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Public/GameplayEffectTypes.h:963`、`:966`、`:969`、`:972`、`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Private/GameplayEffect.cpp:3205`、`:4434`、`:4734`。
- 把持续特效配在只能响应 `Executed` 的路径上，通常不会得到正确的移除回调，这是开发实践推断；源码依据：`Executed` 注释用于 instant effects 或 periodic ticks；源码路径：`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Public/GameplayEffectTypes.h:969`。

## ExecuteGameplayCue 用来做持续特效导致无法正确移除

- `ExecuteGameplayCue` 只生成 `Executed` 事件，不进入 `ActiveGameplayCues` 持续容器；源码路径：`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Private/AbilitySystemComponent.cpp:1300`、`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Private/GameplayCueManager.cpp:1417`。
- 持续 cue 应使用 `AddGameplayCue`/`RemoveGameplayCue` 或 Duration/Infinite GE 生命周期；源码路径：`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Private/AbilitySystemComponent.cpp:1312`、`:1401`、`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Private/GameplayEffect.cpp:4411`、`:4714`。

## AddGameplayCue 后忘记 RemoveGameplayCue

- `AddGameplayCue` 会向 `FActiveGameplayCueContainer` 添加 cue 并增加 tag count，`RemoveCue` 才会减少 tag count 并触发 `Removed`；源码路径：`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Private/GameplayCueInterface.cpp:305`、`:327`、`:330`、`:375`。
- Ability 直接 Add cue 时可用 `bRemoveOnAbilityEnd` 自动加入 `TrackedGameplayCues`，Ability 结束会清理；源码路径：`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Private/Abilities/GameplayAbility.cpp:1721`、`:1738`、`:852`。

## GameplayCueNotify_Actor 持有状态但没有正确清理

- Actor notify 回收时会清事件标记、owner delegate、latent action、timer、owner、隐藏和 detach；自定义状态也需要按同样思路清理是开发实践推断；源码路径：`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Private/GameplayCueNotify_Actor.cpp:389`、`:397`、`:431`、`:437`、`:441`。
- `GameplayCueFinishedCallback` 会通知 Manager 回收/销毁；不结束的 Actor notify 会占用实例或残留状态；源码路径：`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Private/GameplayCueNotify_Actor.cpp:361`、`:381`、`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Private/GameplayCueManager.cpp:565`。

## Looping Cue 没有处理 Removed

- `AGameplayCueNotify_Looping::OnRemove_Implementation` 会调用 `RemoveLoopingEffects`，后者停止 LoopingEffects；源码路径：`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Private/GameplayCueNotify_Looping.cpp:108`、`:133`、`:142`。
- 如果持续效果只处理 `OnActive/WhileActive` 而不处理 `Removed`，looping particles/audio 可能不停止，这是开发实践推断；源码依据：LoopingEffects 的停止只在 `RemoveLoopingEffects` 路径；源码路径：`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Private/GameplayCueNotify_Looping.cpp:39`、`:110`。

## Static Cue 中试图保存实例状态

- Static notify 通过 `LoadedGameplayCueClass->GetDefaultObject(false)` 的 CDO 处理事件，不为每次 cue 生成实例；源码路径：`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Private/GameplayCueSet.cpp:306`。
- 在 Static/Burst Notify 中保存每次触发状态会被共享，这是开发实践推断；需要状态或 latent 时使用 Actor/BurstLatent/Looping；源码路径：`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Public/GameplayCueNotify_Burst.h:16`、`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Public/GameplayCueNotify_BurstLatent.h:16`。

## GameplayCue 在预测场景下重复播放

- ASC 的 NetMulticast cue implementation 会跳过 local client prediction key，ActiveGE 复制也会检查本地预测 effect 以避免重复触发 cue；源码路径：`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Private/AbilitySystemComponent.cpp:1438`、`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Private/GameplayEffect.cpp:2745`、`:2751`。
- 完整预测失败后的 GameplayCue 回滚/重放规则本轮未展开，未确认；源码确认存在 `OnPredictiveGameplayCueCatchup` 处理预测 catchup；源码路径：`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Private/AbilitySystemComponent.cpp:1797`。

## Minimal replication mode 下误以为 ActiveGE 会完整复制 Cue

- Minimal replication 下 GE cue 会走 `AddGameplayCue_MinimalReplication` / `RemoveGameplayCue_MinimalReplication`，因为 ActiveGE 不按 Full 模式完整复制给所有客户端；源码路径：`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Private/GameplayEffect.cpp:4418`、`:4424`、`:4720`、`:4724`。
- `MinimalReplicationGameplayCues` 复制条件为 `COND_SkipOwner`，且容器在 Full replication mode 下不复制；源码路径：`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Private/AbilitySystemComponent.cpp:1655`、`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Private/GameplayCueInterface.cpp:452`。

## Cue 参数里 Source / Instigator / EffectCauser 理解错误

- `FGameplayCueParameters::GetInstigator/GetEffectCauser/GetSourceObject` 优先读显式字段，缺失时回退到 `EffectContext`；源码路径：`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Private/GameplayEffectTypes.cpp:1284`、`:1295`、`:1306`。
- ASC 默认参数只把 OwnerActor 填入 `Instigator`，AvatarActor 填入 `EffectCauser`；源码路径：`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Private/AbilitySystemComponent.cpp:1209`。

## GameplayCue 写了过多业务逻辑，导致表现层和战斗逻辑耦合

- GE 上的 `GameplayCues` 注释描述的是 sounds、particle effects 等非模拟反应，cue RPC 也是 unreliable；源码路径：`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Public/GameplayEffect.h:2297`、`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Public/AbilitySystemComponent.h:880`。
- 开发实践推断：GameplayCue 应主要承载表现，不要把伤害结算、死亡、奖励、库存等权威业务放在 cue notify 中；dedicated server 还可能 suppress cosmetic cue；源码路径：`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Private/GameplayCueManager.cpp:177`。

## 客户端没有加载 Cue 资产导致不播放

- CueSet 发现 `LoadedGameplayCueClass` 为空时会尝试 resolve，仍为空则交给 Manager 的 missing cue 处理；源码路径：`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Private/GameplayCueSet.cpp:279`、`:289`、`:296`。
- Manager 默认不同步加载缺失 cue、默认异步加载并把事件排队；如果加载失败会 warning，事件不会立即播放；源码路径：`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Private/GameplayCueManager.cpp:321`、`:326`、`:352`、`:378`。

## 父子 GameplayTag 匹配和精确匹配理解错误

- CueSet 会为没有精确 notify 的子 tag 指向父 tag notify；notify 的 `IsOverride=false` 时还会继续调用父级 data；源码路径：`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Private/GameplayCueSet.cpp:386`、`:397`、`:312`、`:343`。
- 因此 `GameplayCue.Damage.Fire` 可能触发父级 `GameplayCue.Damage` notify；这不是精确匹配 bug，而是 CueSet 加速 map 与 override 语义；源码路径：`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Private/GameplayCueSet.cpp:401`。

# 常见坑：GAS 网络预测 / RPC / Replication / Serialization（第八轮）

## LocalPredicted Ability 没有正确处理预测失败

- 服务端拒绝客户端预测激活时会调用 `ClientActivateAbilityFailed`，客户端会广播 reject delegate、设置 activation rejected，并对实例 Ability 调用 `K2_EndAbility`；源码路径：`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Private/AbilitySystemComponent_Abilities.cpp:2251`、`:2253`、`:2288`、`:2296`。
- 业务中如果把表现或状态改动放在 AbilityTask / Cue / GE 外部路径，需要自己处理失败后的清理；这是开发实践推断。源码只确认 GAS 通过 PredictionKey delegate 和失败 RPC 提供回调点；源码路径：`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Public/GameplayPrediction.h:88`、`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Private/GameplayPrediction.cpp:595`。

## 跨帧 AbilityTask 继续使用旧 PredictionKey

- 源码注释明确 latent / timer / AbilityTask 跨帧后需要新的 `FScopedPredictionWindow`，并以 WaitInputRelease 为例；源码路径：`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Public/GameplayPrediction.h:198`。
- 旧 key 可能在 scope 结束后不再适合新的预测动作，导致 TargetData、GenericEvent 或 GE 预测被拒绝；源码路径：`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Private/AbilitySystemComponent_Abilities.cpp:4234`、`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Private/GameplayEffect.cpp:2931`。

## TargetData 客户端生成了但服务端没有收到

- 客户端应通过 `CallServerSetReplicatedTargetData` 发送 TargetData；服务端 `ServerSetReplicatedTargetData` 会缓存数据并广播 `TargetSetDelegate`；源码路径：`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Private/Abilities/Tasks/AbilityTask_WaitTargetData.cpp:288`、`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Private/AbilitySystemComponent_Abilities.cpp:3945`、`:3968`。
- 自定义 TargetData struct 缺少 native `NetSerialize` 会导致序列化失败；源码路径：`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Private/GameplayAbilityTargetTypes.cpp:230`、`:240`。

## TargetData 校验失败后没有正确 Cancel

- `ServerSetReplicatedTargetData_Validate` 会拒绝 invalid TargetData item；源码路径：`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Private/AbilitySystemComponent_Abilities.cpp:3971`。
- 取消路径应走 `ServerSetReplicatedTargetDataCancelled`，该路径会设置 cancelled 状态并广播 `TargetCancelledDelegate`；源码路径：`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Private/AbilitySystemComponent_Abilities.cpp:3985`、`:3990`、`:3995`。
- 具体 TargetActor 校验失败后如何转成 Cancel，本轮未完整展开，未确认。

## GenericReplicatedEvent 没有被 Consume 导致重复触发

- `InvokeReplicatedEvent` 会把 `bTriggered` 设为 true，delegate 未绑定时会缓存；源码路径：`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Private/AbilitySystemComponent_Abilities.cpp:3886`、`:3890`。
- `ConsumeGenericReplicatedEvent` 负责把 `bTriggered` 清回 false；WaitInputPress/Release 收到事件后会调用它；源码路径：`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Private/AbilitySystemComponent_Abilities.cpp:3849`、`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Private/Abilities/Tasks/AbilityTask_WaitInputPress.cpp:37`、`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Private/Abilities/Tasks/AbilityTask_WaitInputRelease.cpp:37`。

## ServerOnly Ability 在客户端直接执行表现逻辑

- `InternalTryActivateAbility` 会阻止客户端按 ServerOnly / ServerInitiated 策略直接激活不该本地执行的 Ability；源码路径：`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Private/AbilitySystemComponent_Abilities.cpp:1761`。
- 开发实践推断：客户端表现应由 GameplayCue、Montage replication、或服务端确认后的客户端路径驱动，不应把权威 ServerOnly Ability 当 LocalPredicted Ability 使用；源码依据：`ClientActivateAbilitySucceed` / `ClientActivateAbilityFailed` 区分确认与拒绝；源码路径：`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Private/AbilitySystemComponent_Abilities.cpp:2339`、`:2251`。

## Mixed / Minimal ReplicationMode 理解错误

- `EGameplayEffectReplicationMode` 明确定义 Minimal、Mixed、Full 三种 ActiveGE 复制模式；源码路径：`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Public/AbilitySystemComponent.h:81`。
- `FActiveGameplayEffectsContainer::GetReplicationCondition` 中 Minimal 不复制 ActiveGE，Mixed 在 authority 上 owner-only，Full 给所有连接；源码路径：`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Private/GameplayEffect.cpp:4875`、`:4882`、`:4887`、`:4899`。

## 以为 ActiveGameplayEffect 在所有客户端都会完整复制

- ASC 的 `ActiveGameplayEffects` 使用动态复制条件，具体条件由 ReplicationMode 决定；源码路径：`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Private/AbilitySystemComponent.cpp:1633`、`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Private/GameplayEffect.cpp:4875`。
- Minimal/Mixed 场景下其他客户端更多依赖 `MinimalReplicationTags` 和 `MinimalReplicationGameplayCues`；源码路径：`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Public/AbilitySystemComponent.h:1909`、`:1887`、`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Private/AbilitySystemComponent.cpp:1655`。

## AttributeSet 复制了但 Attribute RepNotify 没正确写

- ASC 可复制 AttributeSet subobject，但具体 Attribute 属性仍需要 UE 通用复制声明和 RepNotify；源码路径：`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Private/AbilitySystemComponent.cpp:1710`、`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Public/AttributeSet.h:404`。
- `GAMEPLAYATTRIBUTE_REPNOTIFY` 会调用 `SetBaseAttributeValueFromReplication`，否则 aggregator / attribute change delegate 可能收不到正确更新；源码路径：`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Public/AttributeSet.h:404`、`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Private/GameplayEffect.cpp:3561`。

## 预测属性没有使用 REPNOTIFY_Always

- 预测文档明确建议 predicted attributes 使用 `REPNOTIFY_Always`，否则客户端预测值与服务端复制值相同或回滚时可能不触发 RepNotify；源码路径：`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Public/GameplayPrediction.h:127`、`:138`。
- `GAMEPLAYATTRIBUTE_REPNOTIFY` 与 `SetBaseAttributeValueFromReplication` 是让预测属性复制后触发 delegate 的 GAS 接入点；源码路径：`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Public/AttributeSet.h:404`、`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Private/GameplayEffect.cpp:3606`。

## GameplayCue 在预测和服务端确认时重复播放

- ASC 的 NetMulticast cue implementation 会在本地 prediction key 已经预测过时跳过，避免重复执行；源码路径：`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Private/AbilitySystemComponent.cpp:1438`。
- ActiveGE 复制到客户端时也会检查是否已有本地 predicted effect，以避免重复触发 cue；源码路径：`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Private/GameplayEffect.cpp:2745`、`:2751`。
- 自定义 Cue / Task 里手动播放表现时仍可能重复，这是开发实践推断；完整预测失败后的 cue 回滚本轮未展开，未确认。

## predicted GE 回滚后表现没有清理

- predicted instant GE 会临时作为 infinite ActiveGE，并通过 prediction key 的 reject/caught-up delegate 触发客户端移除；源码路径：`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Private/AbilitySystemComponent.cpp:862`、`:910`、`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Private/GameplayEffect.cpp:4264`。
- 如果表现没有绑定到 GE / Cue / AbilityTask 的结束路径，reject 后可能残留；这是开发实践推断，源码确认 GAS 提供了 predicted ActiveGE 移除入口；源码路径：`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Private/GameplayEffect.cpp:4546`、`:4553`。

## ASC 放在 PlayerState / Character 时 Owner / Avatar / NetOwner 配置错误

- ASC 区分 `OwnerActor` 与 `AvatarActor`，并在 `InitAbilityActorInfo` 中初始化；源码路径：`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Public/AbilitySystemComponent.h:1546`、`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Private/AbilitySystemComponent_Abilities.cpp:141`。
- ASC 可通过 `IAbilitySystemReplicationProxyInterface` 使用 AvatarActor 等对象作为 replication proxy；源码路径：`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Private/AbilitySystemComponent.cpp:1786`、`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Public/AbilitySystemReplicationProxyInterface.h:21`。
- PlayerState / Character 的具体项目取舍不是源码直接规定，未确认。开发实践推断：无论放哪里，都要保证 Owner / Avatar / NetOwner 与 locally controlled 判断一致。

## 自定义 AbilityTask 中发送 RPC 没带正确 PredictionKey

- TargetData 与 GenericReplicatedEvent 都按 AbilitySpecHandle + PredictionKey 建 cache；key 不匹配会导致服务端 Task 收不到或收到错误数据；源码路径：`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Public/Abilities/GameplayAbilityTypes.h:500`、`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Public/AbilitySystemComponent.h:1699`。
- 需要跨帧发送 RPC 的 Task 应考虑新的 `FScopedPredictionWindow`，这是源码注释明确的预测窗口使用场景；源码路径：`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Public/GameplayPrediction.h:198`。

## RPC batch 打开后调试时误判调用顺序

- RPC batch 只批处理 `CallServerTryActivateAbility`、`CallServerSetReplicatedTargetData`、`CallServerEndAbility`，在 `ServerAbilityRPCBatch_Internal` 中统一展开执行；源码路径：`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Public/AbilitySystemComponent.h:1328`、`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Private/AbilitySystemComponent_Abilities.cpp:4132`。
- `CallServerTryActivateAbility` / `CallServerSetReplicatedTargetData` / `CallServerEndAbility` 在有 batch 时只写入 batch 数据，不一定立即发出独立 RPC；源码路径：`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Private/AbilitySystemComponent_Abilities.cpp:4192`、`:4217`、`:4250`。

# 常见坑：GAS 全局入口与蓝图辅助 API（第九轮）

## Actor 没实现 IAbilitySystemInterface，导致 BlueprintLibrary 找不到 ASC

- `UAbilitySystemBlueprintLibrary::GetAbilitySystemComponent` 只是调用 `UAbilitySystemGlobals::GetAbilitySystemComponentFromActor`；Globals 先查 `IAbilitySystemInterface`，再 fallback 到 `FindComponentByClass<UAbilitySystemComponent>()`，都失败就返回 null；源码路径：`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Private/AbilitySystemBlueprintLibrary.cpp:77`、`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Private/AbilitySystemGlobals.cpp:233`、`:240`、`:247`、`:254`。
- 排查方向：确认传入 Actor 非空、实现了 C++ `IAbilitySystemInterface` 或身上直接挂了 ASC；接口不能直接在蓝图实现；源码路径：`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Public/AbilitySystemInterface.h:15`、`:26`。

## ASC 放在 PlayerState，但 Character 返回了错误 ASC

- `IAbilitySystemInterface` 注释明确 ASC 可以存在于另一个 Actor，例如 Pawn 使用 PlayerState 的 component；如果 Character/Pawn 接口返回自己的空 ASC 或错误 ASC，蓝图库会信任该返回值；源码路径：`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Public/AbilitySystemInterface.h:25`、`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Private/AbilitySystemGlobals.cpp:240`、`:243`。
- 开发实践推断：PlayerState 持有 ASC 时，Character/Pawn 对外暴露 ASC 的接口应转发到 PlayerState 的 ASC，并与 `InitAbilityActorInfo` 的 Owner/Avatar 模型保持一致；源码依据：`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Public/AbilitySystemComponent.h:1546`、`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Private/AbilitySystemComponent_Abilities.cpp:141`。

## OwnerActor / AvatarActor 初始化正确，但接口返回对象错误

- ASC 内部可以正确区分 OwnerActor / AvatarActor，但外部蓝图库查 ASC 的结果完全取决于传入 Actor 的 interface 或组件 fallback；源码路径：`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Public/AbilitySystemComponent.h:1546`、`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Private/AbilitySystemGlobals.cpp:240`、`:247`。
- 排查方向：不要只检查 `InitAbilityActorInfo`，还要检查外界传入 Character、Pawn、PlayerState、Controller 时 `GetAbilitySystemComponent` 返回的是否同一个权威 ASC；这是开发实践推断，源码依据是接口查找优先级固定；源码路径：`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Private/AbilitySystemGlobals.cpp:240`、`:243`。

## EffectContext 中 Instigator / EffectCauser / SourceObject 理解错误

- ASC `MakeEffectContext` 会把 OwnerActor / AvatarActor 写成 Instigator / EffectCauser；`FGameplayEffectContext::AddInstigator` 还会缓存 InstigatorASC；源码路径：`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Private/AbilitySystemComponent.cpp:470`、`:477`、`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Private/GameplayEffectTypes.cpp:177`、`:187`。
- GameplayCueParameters 的 `GetInstigator`、`GetEffectCauser`、`GetSourceObject` 会优先读显式字段，缺失时 fallback 到 EffectContext；Cue 或 Execution 中读到的对象和你手动填的参数可能不一致；源码路径：`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Private/GameplayEffectTypes.cpp:1284`、`:1295`、`:1306`。

## Blueprint 创建 Spec 后忘记设置 SetByCaller

- 蓝图库只有调用 `AssignSetByCallerMagnitude` 或 `AssignTagSetByCallerMagnitude` 时才写入 Spec 的 SetByCaller magnitude；源码路径：`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Private/AbilitySystemBlueprintLibrary.cpp:985`、`:1004`。
- 如果 GE 的 magnitude 依赖 SetByCaller，但应用前没有写入，数值异常或 warning 的完整行为在 GE 侧；本轮确认 invalid SpecHandle 会 warning；源码路径：`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Private/AbilitySystemBlueprintLibrary.cpp:998`、`:1013`、`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Public/GameplayEffect.h:1082`、`:1085`。

## Dynamic Tags 加到了错误位置，导致 GE / Cue / tag requirement 不生效

- `AddGrantedTag(s)` 修改 `Spec->DynamicGrantedTags`，`AddAssetTag(s)` 调用 `Spec->AddDynamicAssetTag` / `AppendDynamicAssetTags`；这两类 tag 语义不同，不能互换；源码路径：`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Private/AbilitySystemBlueprintLibrary.cpp:1039`、`:1054`、`:1069`、`:1084`。
- Dynamic Granted Tags 会随 ActiveGE 更新 tag map；Dynamic Asset Tags 会进入 Spec 的 dynamic asset tags 聚合；源码路径：`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Private/GameplayEffect.cpp:2176`、`:4374`、`:4671`。

## TargetDataHandle 为空但仍应用 GE

- 蓝图库提供 `GetDataCountFromTargetData`、`TargetDataHasActor`、`TargetDataHasHitResult`、`TargetDataHasOrigin`、`TargetDataHasEndPoint`；应用 GE 或读取目标前应先检查；源码路径：`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Public/AbilitySystemBlueprintLibrary.h:230`、`:275`、`:279`、`:287`、`:295`。
- `FGameplayAbilityTargetDataHandle` 是可为空的 handle，底层有 `Clear()`；蓝图库未确认提供直接 Clear 包装；源码路径：`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Public/Abilities/GameplayAbilityTargetTypes.h:193`、`:215`。

## 从 TargetData 取 Actor / HitResult 时没有检查有效性

- `GetActorsFromTargetData`、`GetHitResultFromTargetData`、`GetTargetDataOrigin`、`GetTargetDataEndPoint` 都依赖 index 和数据类型；源码提供 `TargetDataHas*` 检查入口；源码路径：`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Private/AbilitySystemBlueprintLibrary.cpp:532`、`:592`、`:605`、`:618`、`:636`、`:651`、`:678`、`:691`。
- 开发实践推断：蓝图里不要直接从 index 0 读取并假设一定有 Actor/HitResult，尤其是 TargetData 可能来自客户端、过滤器或取消路径；源码依据是 TargetData 网络路径按 handle 缓存和复制，而非强制目标类型；源码路径：`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Private/AbilitySystemComponent_Abilities.cpp:3945`。

## GameplayCueParameters 来源混乱，导致 Cue 里拿不到正确 HitResult

- `FGameplayCueParameters` 既可手动构造，也可从 EffectContext / GE Spec / RPC Spec 初始化；不同来源会影响 Location、Normal、Instigator、EffectCauser、SourceObject、HitResult；源码路径：`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Private/AbilitySystemBlueprintLibrary.cpp:939`、`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Private/AbilitySystemGlobals.cpp:378`、`:387`、`:420`。
- 蓝图库 `MakeGameplayCueParameters` 能显式填入很多字段，但 Cue 参数 getter 仍可能 fallback 到 EffectContext；源码路径：`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Public/AbilitySystemBlueprintLibrary.h:411`、`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Private/GameplayEffectTypes.cpp:1284`、`:1295`、`:1306`。

## AbilitySystemGlobals 配置没有加载，导致 Cue 或 failure tag 行为异常

- GameplayAbilities 模块首次创建 Globals 时会读取 `UGameplayAbilitiesDeveloperSettings::AbilitySystemGlobalsClassName`，创建单例并调用 `InitGlobalData`；源码路径：`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Private/GameplayAbilitiesModule.cpp:31`、`:36`、`:38`。
- `InitGlobalData` 初始化 cue manager、global tags、TargetData/EffectContext cache；如果项目配置错误，Cue 路径、failure tag、序列化 cache 等全局行为都可能受影响；源码路径：`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Private/AbilitySystemGlobals.cpp:64`、`:82`、`:84`、`:87`。
- DeveloperSettings 中的 GameplayCueNotifyPaths、ActivateFail* tags、PredictTargetGameplayEffects、ReplicateActivationOwnedTags 等都是全局行为入口；源码路径：`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Public/GameplayAbilitiesDeveloperSettings.h:61`、`:69`、`:76`、`:80`、`:100`。

## 蓝图辅助函数绕过了项目自己的权限 / 服务端校验逻辑

- `UAbilitySystemBlueprintLibrary` 的工具函数大多是转发或句柄数据加工，例如发送 GameplayEvent、修改 SetByCaller、添加 loose tags、构造 TargetData；源码路径：`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Private/AbilitySystemBlueprintLibrary.cpp:82`、`:985`、`:1310`、`:374`。
- 开发实践推断：这些蓝图 API 不替项目做“是否允许应用 GE / 是否允许改 tag / 是否允许信任客户端 TargetData”的业务校验，权威逻辑仍应放在 ASC/Ability/GE/服务端路径中；源码依据是蓝图库函数没有项目级权限分支；源码路径：`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Private/AbilitySystemBlueprintLibrary.cpp:82`、`:1310`。
