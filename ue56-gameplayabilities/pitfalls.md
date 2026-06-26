# 常见坑：GAS Debug / Log / 调试排错（第十六轮）

## 以为 Ability 激活失败一定会有明显日志

- `TryActivateAbility` 对 invalid handle/Ability 会 warning，但 pending remove、ActorInfo 无效、simulated proxy 等路径会直接返回 false 或少量日志；源码路径：`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Private/AbilitySystemComponent_Abilities.cpp:1585`、`:1593`、`:1607`、`:1618`。
- `CanActivateAbility` 的 cooldown/cost/tag/input/Blueprint 失败细节受 `FScopedCanActivateAbilityLogEnabler` 控制，不一定在普通日志级别直接可见；源码路径：`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Private/AbilitySystemComponent_Abilities.cpp:1794`；`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Private/Abilities/GameplayAbility.cpp:475`、`:485`、`:495`、`:505`。
- 排查时应结合 `AbilityFailedCallbacks`、failure tags、`showdebug abilitysystem` 和 `LogAbilitySystem` Verbose/VeryVerbose；源码路径：`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Public/AbilitySystemComponent.h:549`；`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Private/AbilitySystemComponent_Abilities.cpp:2531`。

## 把 `ABILITY_VLOG` 当作可用新接口

- 当前 `ABILITY_VLOG` 宏带有 deprecated 的 `static_assert`，新排错不应依赖它；源码路径：`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Public/AbilitySystemLog.h:29`、`:44`。
- VisualLogger 相关应查 `UE_VLOG` / `UE_VLOG_UELOG` / `GrabDebugSnapshot` / `ABILITY_VLOG_ATTRIBUTE_GRAPH` 等实际调用点；源码路径：`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Public/AbilitySystemLog.h:54`；`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Private/AbilitySystemComponent.cpp:1905`。

## 以为 `showdebug abilitysystem` 会自动选择正确 ASC

- GameplayAbilities 侧只确认注册了 `UAbilitySystemComponent::OnShowDebugInfo` 到 HUD debug 链，目标选择还受 HUD debug target 与 `ShouldUseDebugTargetFromHud` 影响；源码路径：`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Private/GameplayAbilitiesModule.cpp:73`、`:84`；`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Private/AbilitySystemComponent.cpp:2329`、`:2355`。
- 如果 ASC 放在 PlayerState 而 Avatar 是 Character，开发实践推断：必须确认 debug target 最终能找到正确 ASC，否则看到的可能是空数据或错误对象。

## 忽略 GameplayDebugger 的 server/local 对比能力

- `FGameplayDebuggerCategory_Abilities` 会收集并绘制 tags、abilities、effects、attributes，并能比较 server/local 数据；源码路径：`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Private/GameplayDebuggerCategory_Abilities.cpp:146`、`:391`、`:504`、`:688`。
- 常见误区是只看客户端 UI 或只看服务端日志，导致 replication mode、Minimal tags/cues、Attribute RepNotify 的问题被漏掉；开发实践推断，源码依据为 GameplayDebugger 明确保存本地/服务端数据并绘制差异。

## 用 IgnoreCooldowns / IgnoreCosts 调试后忘记恢复

- `AbilitySystem.IgnoreCooldowns` 和 `AbilitySystem.IgnoreCosts` 是 cheat CVar，会让 `CheckCooldown` / `CheckCost` 逻辑被绕过；源码路径：`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Private/AbilitySystemGlobals.cpp:40`、`:41`；`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Public/AbilitySystemGlobals.h:161`、`:164`。
- 调试 Cost/Cooldown GE 配置时如果这些 CVar 仍开启，会掩盖 Cooldown GrantedTag、Cost Modifier、SetByCaller 缺失等配置错误；开发实践推断。

## 用 GlobalAbilityScale 当正式玩法规则

- `AbilitySystem.GlobalAbilityScale` 源码说明用于 testing/iteration，never for shipping；源码路径：`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Private/AbilitySystemGlobals.cpp:43`。
- 它会被 WaitDelay、PlayMontageAndWait、RootMotion AbilityTask 等非 shipping scale 入口使用；源码路径：`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Private/Abilities/Tasks/AbilityTask_WaitDelay.cpp:19`；`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Private/Abilities/Tasks/AbilityTask_PlayMontageAndWait.cpp:115`。

## Cue 不播放时只查资产，不查全局 CVar

- `AbilitySystem.DisableGameplayCues` 会让 `ShouldSuppressGameplayCues` 抑制 cue；`AbilitySystem.GameplayCue.RunOnDedicatedServer` 也影响 dedicated server cue 运行；源码路径：`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Private/GameplayCueManager.cpp:44`、`:50`、`:179`。
- `AbilitySystem.DisplayGameplayCues`、`AbilitySystem.LogGameplayCueActorSpawning` 和 `AbilitySystem.GameplayCueFailLoads` 是 Cue 排错入口；源码路径：`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Private/GameplayCueManager.cpp:38`、`:41`；`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Private/GameplayCueSet.cpp:47`。

## TargetData / GenericReplicatedEvent 没有 Consume

- `ConsumeClientReplicatedTargetData` 和 `ConsumeGenericReplicatedEvent` 用于清理 replicated data/event 缓存；源码路径：`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Private/AbilitySystemComponent_Abilities.cpp:3838`、`:3849`。
- 未消费旧数据可能导致后续 delegate 立即命中旧事件或重复触发；这是开发实践推断，源码依据是 `CallOrAddReplicatedDelegate` 会在已有缓存事件时直接调用 delegate；源码路径：`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Private/AbilitySystemComponent_Abilities.cpp:4066`。

## RPC Batch 打开后误判调用顺序

- `AbilitySystem.ServerRPCBatching.Log` 会打印 batch 中 TryActivateAbility、TargetData、EndAbility 等调用；源码路径：`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Private/AbilitySystemComponent_Abilities.cpp:4098`、`:4134`、`:4150`、`:4170`。
- 常见误区是只看单个 RPC 日志而忽略 batch flush 后的顺序；开发实践推断，排查时应同时打开 batch log。

## 把 Basic HUD / CheatManager 命令当作运行时业务接口

- `AbilitySystemDebugHUD` 和 `AbilitySystemCheatManagerExtension` 都是调试/cheat 辅助入口；源码路径：`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Private/AbilitySystemDebugHUD.cpp:607`；`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Private/AbilitySystemCheatManagerExtension.cpp:579`、`:673`。
- 开发实践推断：项目运行时代码不应依赖 CheatManager debug command 完成授予、激活、应用 GE 等正式逻辑。

## 误以为本轮确认了所有 CVar

- `ABILITY_LOG_SCOPE`、`AbilitySystemCVars.h`、`AbilitySystem.Log` 同名 CVar 本轮在指定 GameplayAbilities 源码范围内未确认。
- `showdebug abilitysystem` 的控制台命令解析属于 UE 通用 showdebug/HUD 系统，本轮只确认 GameplayAbilities 侧注册和 ASC 输出链；源码路径：`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Private/GameplayAbilitiesModule.cpp:73`、`:84`。

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

# 常见坑：GameplayAbilitiesEditor / Blueprint / K2 工具链（第十轮）

## AbilityTask 蓝图节点创建了但没有正确激活

- `UK2Node_LatentAbilityCall` 的运行时激活依赖 GameplayTasksEditor 基类展开出的 `UGameplayTask::ReadyForActivation` 调用；源码路径：`Engine/Source/Editor/GameplayTasksEditor/Private/K2Node_LatentGameplayTaskCall.cpp:53`、`:899`、`:900`。
- 如果自定义节点或手写 C++ 创建 Task 后没有走 `ReadyForActivation`，不会进入 Task 的正常 activate 流程；这是开发实践推断，源码依据是基类 K2 展开显式生成 activation 调用；源码路径：`Engine/Source/Editor/GameplayTasksEditor/Private/K2Node_LatentGameplayTaskCall.cpp:563`、`:899`。

## 自定义 AbilityTask 静态工厂函数元数据不符合 K2 节点预期

- `UK2Node_LatentAbilityCall` 通过 `RegisterClassFactoryActions<UAbilityTask>` 发现 AbilityTask 工厂函数，并在校验中读取 factory function metadata；源码路径：`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilitiesEditor/Private/K2Node_LatentAbilityCall.cpp:77`、`:90`。
- `HideThen`、`DefaultToSelf`、`BlueprintInternalUseOnly` 等元数据会影响节点 pin 和可用性；源码路径：`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilitiesEditor/Private/K2Node_LatentAbilityCall.cpp:14`、`:99`、`Engine/Source/Editor/GameplayTasksEditor/Private/K2Node_LatentGameplayTaskCall.cpp:593`。

## GameplayAbility 蓝图基类选错

- `UGameplayAbilitiesBlueprintFactory` 默认 parent 是 `UGameplayAbility`，并在创建时拒绝非 `UGameplayAbility` 子类；源码路径：`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilitiesEditor/Private/GameplayAbilitiesBlueprintFactory.cpp:270`、`:271`、`:291`。
- 如果通过其他路径创建了普通 Blueprint 或错误 parent class，就不会获得 Gameplay Ability Graph、默认 `K2_ActivateAbility` / `K2_OnEndAbility` 事件等工厂初始化；源码路径：`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilitiesEditor/Private/GameplayAbilitiesBlueprintFactory.cpp:310`、`:325`、`:326`。

## GameplayCueNotify 类型选错，导致持续 Cue 无法正确移除

- Cue 编辑器创建 notify 时默认提供 `UGameplayCueNotify_Static` 与 `AGameplayCueNotify_Actor`；Static 是 UObject，Actor notify 可以持有实例状态和生命周期；源码路径：`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilitiesEditor/Private/SGameplayCueEditor.cpp:1404`、`:1405`、`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Public/GameplayCueNotify_Static.h:19`、`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Public/GameplayCueNotify_Actor.h:20`。
- 开发实践推断：持续循环表现应优先选择能处理 active/remove 生命周期的 notify 形态；否则 Add/Remove 类 cue 容易残留表现；运行时事件类型依据见 `EGameplayCueEvent`；源码路径：`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Public/GameplayCue_Types.h:24`。

## GameplayCue tag 和 Notify 资产没有正确绑定

- `UK2Node_GameplayCueEvent` 从 `GameplayCue` 根 tag 下生成事件节点，但具体 tag 到 notify 的资产映射依赖 editor cue set；源码路径：`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilitiesEditor/Private/K2Node_GameplayCueEvent.cpp:84`、`:87`、`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilitiesEditor/Private/GameplayCueTagDetails.cpp:172`。
- `GameplayCueTagDetails` 会列出当前 tag 关联 notify，并提供 Add New；如果没有正确创建或扫描到 notify，运行时 manager 可能找不到 cue；源码路径：`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilitiesEditor/Private/GameplayCueTagDetails.cpp:94`、`:130`、`:145`、`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Private/GameplayCueManager.cpp:982`。

## GameplayEffect Modifier Magnitude 配置错误但编辑器 UI 没能阻止

- `FGameplayEffectModifierMagnitudeDetails` 只根据 magnitude 类型显示对应编辑行；它不等同于运行时数值合法性校验；源码路径：`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilitiesEditor/Private/GameplayEffectModifierMagnitudeDetails.cpp:28`、`:47`、`:54`、`:61`、`:68`。
- SetByCaller、AttributeBased、CustomCalculation 的运行时语义仍由 GE spec、attribute capture 和 calculation 处理；源码路径：`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Public/GameplayEffect.h:1082`、`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Private/GameplayEffect.cpp:1540`。

## Attribute 选择了错误 AttributeSet 中的属性

- Attribute picker 扫描 `UAttributeSet` 派生类及属性，并可被 `HideInDetailsView` 过滤；源码路径：`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilitiesEditor/Private/AttributeDetails.cpp:268`、`:274`、`:296`。
- 编辑器能帮助选择 `FGameplayAttribute`，但不会保证目标 ASC 运行时一定拥有对应 AttributeSet；运行时 ASC 是否有 set 仍由 `HasAttributeSetForAttribute` 等接口判断；源码路径：`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Private/AttributeDetails.cpp:494`、`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Public/AbilitySystemComponent.h:473`。

## GameplayTagRequirements 配置方向错误

- 请求中的 `FGameplayTagRequirementsDetails` 未确认存在；本轮只确认运行时 `FGameplayTagRequirements` 类型与编辑器对 GameplayTag property 的 customization；源码路径：`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Public/GameplayEffectTypes.h:1426`、`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilitiesEditor/Private/GameplayAbilitiesEditorModule.cpp:175`。
- 开发实践推断：Required / Blocked、Source / Target、Application / Ongoing / Removal 方向配反时，details 面板不一定能阻止运行时逻辑失败；源码依据是 GE 运行时检查这些 tag requirement；源码路径：`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Private/GameplayEffect.cpp:1295`、`:1324`。

## 以为 Details 面板校验等同于运行时校验

- `FGameplayEffectDetails` 主要根据 DurationPolicy/Period 隐藏或刷新 UI 行；源码路径：`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilitiesEditor/Private/GameplayEffectDetails.cpp:32`、`:48`、`:54`、`:78`。
- Details customization 影响编辑体验，不替代 `ApplyGameplayEffectSpecToSelf`、CanApply、Immunity、TagRequirements 等运行时检查；源码路径：`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Private/AbilitySystemComponent.cpp:778`、`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Private/GameplayEffect.cpp:1295`。

## 编辑器模块代码被误用到运行时模块

- `GameplayAbilitiesEditor.Build.cs` 依赖 `BlueprintGraph`、`KismetCompiler`、`GraphEditor`、`PropertyEditor`、`Sequencer` 等 Editor 模块；源码路径：`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilitiesEditor/GameplayAbilitiesEditor.Build.cs:25`、`:28`、`:30`、`:31`、`:40`、`:41`。
- 开发实践推断：运行时模块引用这些 editor-only 类型会造成打包或非编辑器构建失败；源码依据是这些依赖只在 `GameplayAbilitiesEditor` 模块声明，运行时 `GameplayAbilities` 模块不应反向依赖 editor 模块；源码路径：`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilitiesEditor/GameplayAbilitiesEditor.Build.cs:20`。

## 在 Runtime Build 中引用 GameplayAbilitiesEditor 类型导致打包失败

- `FGameplayAbilitiesEditorModule`、`UGameplayAbilitiesBlueprintFactory`、`UK2Node_LatentAbilityCall`、`UK2Node_GameplayCueEvent` 都位于 `GameplayAbilitiesEditor` 模块；源码路径：`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilitiesEditor/Private/GameplayAbilitiesEditorModule.cpp:61`、`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilitiesEditor/Public/GameplayAbilitiesBlueprintFactory.h:13`、`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilitiesEditor/Public/K2Node_LatentAbilityCall.h:16`、`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilitiesEditor/Public/K2Node_GameplayCueEvent.h:14`。
- 运行时可加载的数据资产是 `UGameplayAbilityBlueprint`、`UGameplayEffect`、GameplayCueNotify 等 runtime 类型；factory、asset actions、details customization 只是编辑器创建/编辑入口；源码路径：`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Public/GameplayAbilityBlueprint.h:18`、`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Public/GameplayEffect.h:2411`。

## K2 节点或 Editor 工具分析时误改 Engine 源码

- 本 Skill 的工作边界要求只读分析 UE 源码，只更新 `.codex/skills/ue56-gameplayabilities/` 下文档；源码路径记录于 `SKILL.md` 的工作边界。
- 如果需要修复项目蓝图节点或 editor 扩展，开发实践推断应在项目/插件侧实现并引用运行时接口，避免直接修改 `Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilitiesEditor`；源码依据是 editor module 已有清晰模块边界；源码路径：`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilitiesEditor/GameplayAbilitiesEditor.Build.cs:20`。

# 常见坑：GameplayAbilityTargetActor / TargetData / Targeting 体系（第十一轮）

## WaitTargetData 创建了 TargetActor 但没有正确 Confirm / Cancel

- `UserConfirmed` 模式会在 `FinalizeTargetActor` 中绑定 Confirm / Cancel 输入，`Instant` 会立即确认，`Custom` / `CustomMulti` 不会自动绑定；源码路径：`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Private/Abilities/Tasks/AbilityTask_WaitTargetData.cpp:167`、`:171`、`:174`。
- ASC 的 confirm/cancel 输入最终广播 `GenericLocalConfirmCallbacks` / `GenericLocalCancelCallbacks`，如果输入未绑定或 TargetActor 没绑定，Task 会一直等；源码路径：`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Private/AbilitySystemComponent_Abilities.cpp:2931`、`:2935`、`:2938`、`:2942`。

## 客户端 TargetData 生成了但服务端没有收到

- 预测客户端 only 在 `OnTargetDataReadyCallback` 里发送 TargetData；发送前会创建 `FScopedPredictionWindow`；源码路径：`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Private/Abilities/Tasks/AbilityTask_WaitTargetData.cpp:272`、`:280`、`:283`、`:288`。
- 服务端必须按相同 AbilitySpecHandle + ActivationPredictionKey 绑定 `AbilityTargetDataSetDelegate`，否则只能依赖缓存补触发；源码路径：`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Private/Abilities/Tasks/AbilityTask_WaitTargetData.cpp:207`、`:208`、`:211`、`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Private/AbilitySystemComponent_Abilities.cpp:4028`。

## TargetData RPC 没有正确 PredictionKey

- `AbilityTargetDataMap` 的 key 是 `FGameplayAbilitySpecHandleAndPredictionKey`，只比较 AbilityHandle 和 PredictionKey 当前值；源码路径：`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Public/Abilities/GameplayAbilityTypes.h:500`、`:510`、`:516`。
- `CallServerSetReplicatedTargetData` 在 batch scope 内发现 prediction key invalid 会 warning；源码路径：`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Private/AbilitySystemComponent_Abilities.cpp:4234`、`:4237`。

## 服务端收到 TargetData 后没有 Consume，导致重复触发

- WaitTargetData 的服务端 replicated callback 会先调用 `ConsumeClientReplicatedTargetData`；源码路径：`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Private/Abilities/Tasks/AbilityTask_WaitTargetData.cpp:222`、`:228`。
- `ConsumeClientReplicatedTargetData` 会 clear TargetData 并重置 confirmed/cancelled 标记；如果自定义流程不消费，后续 `CallReplicatedTargetDataDelegatesIfSet` 可能再次广播缓存；源码路径：`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Private/AbilitySystemComponent_Abilities.cpp:3838`、`:3843`、`:3844`、`:4028`。

## CustomMulti 模式下误以为任务会自动结束

- WaitTargetData 在 replicated 和 local ready 回调里都只在 `ConfirmationType != CustomMulti` 时 `EndTask`；源码路径：`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Private/Abilities/Tasks/AbilityTask_WaitTargetData.cpp:255`、`:302`。
- 开发实践推断：CustomMulti 必须由 Ability 或外部逻辑显式结束 Task，否则 Ability 可能长期 active。

## TargetActor 持有对象引用但 EndTask 后没有清理

- WaitTargetData `OnDestroy` 会 destroy TargetActor；TargetActor `EndPlay` 会解绑 GenericLocalConfirm / Cancel callbacks，源码注释说明这些绑定会抑制其他 ability 使用相同按键；源码路径：`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Private/Abilities/Tasks/AbilityTask_WaitTargetData.cpp:358`、`:362`、`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Private/Abilities/GameplayAbilityTargetActor.cpp:26`、`:30`、`:40`、`:41`。
- 自定义 TargetActor 若额外绑定外部 delegate、timer、UI、reticle，需要在 EndPlay / Task OnDestroy 中清理；这是开发实践推断，源码依据是内置类主动清理 confirm/cancel 与 reticle；源码路径：`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Private/Abilities/GameplayAbilityTargetActor_Trace.cpp:27`、`:31`。

## TargetDataHandle 为空仍继续应用 GE

- `FGameplayAbilityTargetDataHandle` 可以为空，`Num` / `IsValid` 是检查入口；源码路径：`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Public/Abilities/GameplayAbilityTargetTypes.h:220`、`:226`。
- 蓝图库提供 `GetDataCountFromTargetData` 与 `TargetDataHasActor/HitResult/Origin/EndPoint`，读取前应先检查；源码路径：`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Public/AbilitySystemBlueprintLibrary.h:230`、`:275`、`:279`、`:287`、`:295`。

## 从 TargetData 读取 Actor / HitResult 前没有检查类型

- `GetHitResultFromTargetData` 在没有 HitResult 时返回默认 `FHitResult`，`GetActorsFromTargetData` index 无效时返回空数组；源码路径：`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Private/AbilitySystemBlueprintLibrary.cpp:532`、`:548`、`:618`、`:633`。
- `ActorArray`、`SingleTargetHit`、`LocationInfo` 支持的数据不同，应先用 `TargetDataHas*` 判断；源码路径：`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Public/Abilities/GameplayAbilityTargetTypes.h:448`、`:550`、`:380`。

## HitResult 没写入 EffectContext，导致 GameplayCue 没有正确位置

- `AddTargetDataToContext` 只有当 TargetData `HasHitResult()` 且 context 还没有 HitResult 时才写入；源码路径：`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Private/GameplayAbilityTargetTypes.cpp:65`、`:67`。
- `FGameplayEffectContext::AddHitResult` 会把 HitResult 保存到 context，并在没有 world origin 时用 `TraceStart` 设置 origin；CueParameters 会保存 EffectContext；源码路径：`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Private/GameplayEffectTypes.cpp:221`、`:230`、`:233`、`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Private/AbilitySystemGlobals.cpp:420`、`:425`。

## Trace 起点 / 终点基于客户端相机，服务端校验不足

- Trace 起点来自 `StartLocation`，终点由 PlayerController view point + MaxRange 计算；源码路径：`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Private/Abilities/GameplayAbilityTargetActor_SingleLineTrace.cpp:30`、`:32`、`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Private/Abilities/GameplayAbilityTargetActor_Trace.cpp:90`、`:95`、`:98`、`:131`。
- 服务端 RPC validate 只检查 TargetData item 是否有效，不检查距离、视线、遮挡或阵营；这些需要项目级服务端校验，开发实践推断；源码路径：`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Private/AbilitySystemComponent_Abilities.cpp:3971`、`:3974`。

## Radius TargetActor 直接信任客户端范围结果

- `AGameplayAbilityTargetActor_Radius` 构造时设置 `ShouldProduceTargetDataOnServer = true`，说明默认倾向服务端生成范围结果；源码路径：`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Private/Abilities/GameplayAbilityTargetActor_Radius.cpp:20`、`:25`。
- 开发实践推断：范围目标选择不要直接信任客户端 ActorArray，服务端应重算 overlap 或至少二次过滤；源码依据是 WaitTargetData 支持服务端生成 TargetData 的路径；源码路径：`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Private/Abilities/Tasks/AbilityTask_WaitTargetData.cpp:285`、`:290`。

## TargetActor 中写了过多伤害 / 结算逻辑

- TargetData 基类职责是产生和消费目标数据；GE 应用由 `ApplyGameplayEffectSpec` 和 ASC 完成；源码路径：`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Public/Abilities/GameplayAbilityTargetTypes.h:46`、`:60`、`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Private/GameplayAbilityTargetTypes.cpp:21`、`:47`。
- 开发实践推断：TargetActor 应偏目标选择和轻量校验，伤害/治疗/资源消耗应交给 Ability + GameplayEffect，避免选目标层和结算层耦合。

## VisualizeTargeting 被误用成真正选目标逻辑

- `UAbilityTask_VisualizeTargeting` 只有 `TimeElapsed` delegate，没有 `ValidData` / `Cancelled` TargetData 输出；源码路径：`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Public/Abilities/Tasks/AbilityTask_VisualizeTargeting.h:21`、`:26`、`:30`。
- 它的用途是生成/使用 TargetActor 做 visualization，时间到后广播并 EndTask；源码路径：`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Private/Abilities/Tasks/AbilityTask_VisualizeTargeting.cpp:143`、`:153`、`:168`、`:174`。

## RPC batch 打开后误判 TargetData 调用顺序

- `CallServerSetReplicatedTargetData` 在 batch scope 内可能只写入 `ExistingBatchData->TargetData`；没有 batch 或 batch 未 started 时才直接调用 `ServerSetReplicatedTargetData`；源码路径：`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Private/AbilitySystemComponent_Abilities.cpp:4217`、`:4223`、`:4230`、`:4241`、`:4245`。
- 调试时看到客户端调用了 TargetData 发送函数，不代表独立 RPC 已立即发出；这是源码批处理路径确认的行为；源码路径：`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Private/AbilitySystemComponent_Abilities.cpp:4222`。

# 常见坑：GameplayTag / ResponseTable / Ability Tag 条件体系（第十二轮）

## 把 GameplayCueTag 当作状态 tag 使用

- GameplayCue tag 用于 CueSet/Notify 路由，`FGameplayCueParameters` 保存 `MatchedTagName` / `OriginalTag`，而角色状态查询来自 ASC `GameplayTagCountContainer`；源码路径：`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Public/GameplayCue_Types.h:281`、`:284`、`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Public/AbilitySystemComponent.h:597`。
- 开发实践推断：状态判断用 `State.*` / `Status.*` owned tags，表现路由用 `GameplayCue.*`，不要把 Cue 是否播放当作 gameplay state。

## LooseGameplayTag 以为会自动复制

- `AddLooseGameplayTag` 注释明确 loose tags 不复制，需要调用方自行保证客户端/服务端一致；源码路径：`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Public/AbilitySystemComponent.h:649`、`:651`、`:652`。
- 需要复制的手动状态应考虑 `AddReplicatedLooseGameplayTag` 或通过 GE Granted Tags 表达；源码路径：`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Public/AbilitySystemComponent.h:690`、`:694`、`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Private/GameplayEffect.cpp:4373`。

## ReplicatedLooseTag 和 GE GrantedTag 混用导致状态不一致

- ReplicatedLooseTags 和 GE GrantedTags 都会最终影响 ASC tag count，但来源与生命周期不同：ReplicatedLooseTags 由手动 API 控制，GE GrantedTags 随 ActiveGE 添加/移除；源码路径：`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Public/AbilitySystemComponent.h:1966`、`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Private/GameplayEffect.cpp:4373`、`:4670`。
- 开发实践推断：同一语义状态不要同时用 replicated loose tag 和 GE granted tag 表达，除非明确设计了叠加 count 和移除顺序。

## AbilityTags / ActivationOwnedTags / ActivationRequiredTags 混淆

- `AbilityTags` 表示 Ability 自身分类；`ActivationOwnedTags` 是 Ability 激活期间加到 ASC 的状态；`ActivationRequiredTags` 是激活前必须存在的 ASC owned tags；源码路径：`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Public/Abilities/GameplayAbility.h:496`、`:765`、`:769`。
- `PreActivate` 添加 `ActivationOwnedTags`，`DoesAbilitySatisfyTagRequirements` 检查 `ActivationRequiredTags`；源码路径：`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Private/Abilities/GameplayAbility.cpp:963`、`:386`。

## ActivationBlockedTags 配错导致 Ability 永远不能激活

- `DoesAbilitySatisfyTagRequirements` 会检查 ASC owned tags 是否命中 `ActivationBlockedTags`，命中就失败并写入 blocked failure tag；源码路径：`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Private/Abilities/GameplayAbility.cpp:375`、`:333`。
- 常见错误是把 Ability 分类 tag 或永久状态 tag 配进 blocked 条件，导致角色长期拥有该 tag 时 Ability 永远失败；这是开发实践推断。

## CancelAbilitiesWithTag 配错导致误取消其他技能

- Ability 激活时 `ApplyAbilityBlockAndCancelTags` 会把 `CancelAbilitiesWithTag` 传给 `CancelAbilities`，排除 requesting ability；源码路径：`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Private/Abilities/GameplayAbility.cpp:985`、`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Private/AbilitySystemComponent_Abilities.cpp:1421`、`:1423`。
- 开发实践推断：Cancel tag 应使用足够具体的 AbilityTags，例如取消 `Ability.Channeling`，避免用过宽的 `Ability` 根 tag。

## BlockAbilitiesWithTag 配错导致技能互相锁死

- 激活时 `BlockAbilitiesWithTags` 增加 `BlockedAbilityTags` count，结束时 `UnBlockAbilitiesWithTags` 减少 count；源码路径：`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Private/AbilitySystemComponent_Abilities.cpp:1412`、`:1418`、`:1433`、`:1438`。
- 如果 Ability 没正确 `EndAbility`，block tag 不会被移除；源码路径：`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Private/Abilities/GameplayAbility.cpp:868`。

## SourceRequiredTags / TargetRequiredTags 方向理解反了

- `DoesAbilitySatisfyTagRequirements` 只在调用方传入 `SourceTags` / `TargetTags` 时检查 source/target required/blocked tags；源码路径：`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Private/Abilities/GameplayAbility.cpp:376`、`:380`、`:387`、`:391`。
- 开发实践推断：自身状态优先用 ActivationRequired/Blocked；目标状态来自 TargetData/事件/调用方填入的 TargetTags，否则 TargetRequiredTags 不会替你自动查目标 ASC。

## GE Asset Tags 和 Granted Tags 混淆

- Asset Tags 描述 GE 自身，Granted Tags 是 GE 应用后授予目标的状态；源码路径：`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Public/GameplayEffect.h:2144`、`:2147`。
- `UGameplayEffect::GetOwnedGameplayTags` 的 legacy 行为曾经与名称不一致，源码中有 ensure/warning 提醒应使用 `GetGrantedTags` 获取授予目标的 tags；源码路径：`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Private/GameplayEffect.cpp:218`、`:228`、`:235`、`:241`。

## Dynamic Asset Tags 和 Dynamic Granted Tags 加错位置

- `AddGrantedTag(s)` 写 `Spec->DynamicGrantedTags`，会作为运行时授予目标的 tags；`AddAssetTag(s)` 调 `AddDynamicAssetTag` / `AppendDynamicAssetTags`，会加入 Spec asset/source spec tags；源码路径：`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Private/AbilitySystemBlueprintLibrary.cpp:1039`、`:1054`、`:1069`、`:1084`、`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Private/GameplayEffect.cpp:1875`。
- 常见错误是把状态 tag 加到 Dynamic Asset Tags，结果目标 ASC 没有该状态；或者把查询用 tag 加到 Dynamic Granted Tags，导致移除/查询条件不匹配；这是开发实践推断。

## GameplayTagResponseTable 响应 GE 又授予触发 tag，造成循环

- ResponseTable 监听 ASC tag event，响应时对同一 ASC `ApplyGameplayEffectToSelf`；响应 GE 的 GrantedTags 又会更新同一 ASC tag map；源码路径：`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Private/GameplayTagResponseTable.cpp:66`、`:181`、`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Private/GameplayEffect.cpp:4373`。
- 开发实践推断：Response GE 不应授予 positive/negative 触发 tag 本身，建议把触发 tag 与响应结果 tag 分层命名。

## WaitGameplayTag 监听错 ASC

- WaitGameplayTag base 从任务绑定的 ASC 注册 `RegisterGameplayTagEvent`，销毁时解绑；源码路径：`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Private/Abilities/Tasks/AbilityTask_WaitGameplayTagBase.cpp:17`、`:31`。
- 开发实践推断：监听目标状态时确认 Task 绑定的是目标 ASC 还是自身 ASC；否则 tag 确实变化了但任务不触发。

## UI 直接轮询 tag，而不是监听 tag event

- ASC 提供 `RegisterGameplayTagEvent` 与蓝图 tag changed wrapper；`GetGameplayTagCount` 注释说明客户端可读但可能不是服务器最新值；源码路径：`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Public/AbilitySystemComponent.h:743`、`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Public/AbilitySystemBlueprintLibrary.h:99`、`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Private/AbilitySystemComponent.cpp:690`、`:692`。
- 开发实践推断：UI 应监听 tag event 初始化/更新状态，而不是 Tick 轮询。

## MinimalReplication 模式下误以为所有 granted tags 都来自 ActiveGE 复制

- Minimal replication mode 下 GEs 不复制完整数据，但它们授予的 tags 会通过 `MinimalReplicationTags` 复制；源码路径：`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Public/AbilitySystemComponent.h:719`、`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Private/GameplayEffect.cpp:4383`、`:4384`。
- 调试客户端状态时，不要只看 ActiveGE 列表是否完整，还要看 ASC owned tags / MinimalReplicationTags；这是开发实践推断。

## Tag 命名没有分层，导致配置难维护

- 源码通过 UPROPERTY meta 区分 AbilityTagCategory、OwnedTagsCategory、SourceTagsCategory、TargetTagsCategory、GameplayCue 等用途；源码路径：`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Public/Abilities/GameplayAbility.h:498`、`:758`、`:766`、`:778`、`:786`、`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Public/Abilities/GameplayAbility.h:686`。
- 命名分层建议属于开发实践推断：建议区分 `Ability.*`、`State.*`、`Status.*`、`Cooldown.*`、`GameplayCue.*`，避免一个 tag 同时承担行为分类、状态判断和表现路由。
---

# 常见坑：GameplayEffectComponent / GEComponents（第十三轮）

## 旧版 GE 字段和 GEComponents 混用导致配置理解错误

- UE5.6 中 `ChanceToApplyToTarget_DEPRECATED`、`ApplicationRequirements_DEPRECATED`、`ConditionalGameplayEffects`、旧 tag containers、immunity/remove query 等字段已经标记 deprecated，并由转换函数迁移到组件；源码路径：`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Public/GameplayEffect.h:2253`、`:2258`、`:2262`、`:2310`、`:2350`、`:2359`、`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Private/GameplayEffect.cpp:437`、`:448`。
- 新配置优先看 `GEComponents` 数组；旧字段仍可能为兼容读取被回填，不能把它们当作新项目主配置入口；源码路径：`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Public/GameplayEffect.h:2421`、`:2422`、`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Private/GameplayEffect.cpp:831`、`:851`。

## Asset Tags 和 Target Granted Tags 混淆

- Asset Tags 描述 GE 自身，不授予目标；Granted Tags 会在 ActiveGE 添加时进入目标 ASC tag count，移除时减少；源码路径：`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Public/GameplayEffect.h:2143`、`:2147`、`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Private/GameplayEffect.cpp:4373`、`:4670`。
- 新配置分别对应 `UAssetTagsGameplayEffectComponent` 与 `UTargetTagsGameplayEffectComponent`；源码路径：`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Public/GameplayEffectComponents/AssetTagsGameplayEffectComponent.h:13`、`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Public/GameplayEffectComponents/TargetTagsGameplayEffectComponent.h:13`。

## Instant GE 配置 OngoingTagRequirements / GrantedTags / GrantAbilities

- `UTargetTagRequirementsGameplayEffectComponent::IsDataValid` 对 Instant GE 配 OngoingTagRequirements 报 invalid；源码路径：`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Private/GameplayEffectComponents/TargetTagRequirementsGameplayEffectComponent.cpp:165`、`:172`。
- `UTargetTagsGameplayEffectComponent`、`UBlockAbilityTagsGameplayEffectComponent`、`UAbilitiesGameplayEffectComponent` 都会对 Instant GE 上的持续型配置报 invalid；源码路径：`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Private/GameplayEffectComponents/TargetTagsGameplayEffectComponent.cpp:42`、`:50`、`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Private/GameplayEffectComponents/BlockAbilityTagsGameplayEffectComponent.cpp:41`、`:49`、`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Private/GameplayEffectComponents/AbilitiesGameplayEffectComponent.cpp:175`、`:180`。

## TargetTagRequirements 方向理解错误

- TargetTagRequirements 组件读取目标 ASC owned tags 来检查 Application/Removal/Ongoing requirements；Application 不满足或 Removal 已满足会阻止应用，Ongoing 不满足会 inhibit 已添加 ActiveGE；源码路径：`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Private/GameplayEffectComponents/TargetTagRequirementsGameplayEffectComponent.cpp:21`、`:24`、`:29`、`:143`。
- 开发实践推断：如果你想检查施法者状态，应确认这些 tag 是否通过 Spec source tags / Ability 条件表达，而不是误放到 TargetTagRequirements；源码依据是该组件使用 `ActiveGEContainer.Owner->GetOwnedGameplayTags` 获取目标 ASC tags；源码路径：`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Private/GameplayEffectComponents/TargetTagRequirementsGameplayEffectComponent.cpp:21`、`:132`。

## BlockAbilityTags GE 移除后没有按预期解除阻塞

- BlockAbilityTags 通过 ActiveGE 添加时 `Owner->BlockAbilitiesWithTags(GetBlockedAbilityTags())`、移除时 `UnBlockAbilitiesWithTags` 维护计数；源码路径：`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Private/GameplayEffect.cpp:4377`、`:4674`、`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Private/AbilitySystemComponent_Abilities.cpp:1433`、`:1438`。
- 如果同一 blocked tag 有多个来源，移除一个 GE 只减少一次 count；开发实践推断：排查时看是否还有其他 Ability/GE 也在阻塞同一 tag。源码依据：ASC `BlockedAbilityTags` 使用 `UpdateTagCount`；源码路径：`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Private/AbilitySystemComponent_Abilities.cpp:1435`、`:1440`。

## Immunity Query 写错导致 GE 永远无法应用

- Immunity 组件会把 `AllowGameplayEffectApplication` 注册到 ASC `GameplayEffectApplicationQueries`，incoming GE 匹配任一 `ImmunityQueries` 时返回 false 阻止应用；源码路径：`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Private/GameplayEffectComponents/ImmunityGameplayEffectComponent.cpp:24`、`:66`、`:70`。
- `FGameplayEffectQuery` 可以按 owning/effect/source tags、attribute、source、definition、自定义 delegate 匹配；写得过宽会屏蔽过多 GE；源码路径：`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Public/GameplayEffect.h:1460`、`:1464`、`:1468`、`:1476`、`:1480`、`:1484`。

## RemoveOther Query 写太宽导致清掉不该清的 GE

- RemoveOther 组件应用成功后在 authority 上遍历 `RemoveGameplayEffectQueries` 并调用 `RemoveActiveEffects`；源码路径：`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Private/GameplayEffectComponents/RemoveOtherGameplayEffectComponent.cpp:16`、`:18`、`:39`。
- 组件会设置 `IgnoreHandles` 避免移除自身，但不会替项目保护其他语义上不该移除的 GE；源码路径：`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Private/GameplayEffectComponents/RemoveOtherGameplayEffectComponent.cpp:23`、`:40`、`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Public/GameplayEffect.h:1488`。

## AdditionalEffects 触发递归或重复应用

- AdditionalEffects 在 `OnGameplayEffectApplied` 中继续调用 `ApplyGameplayEffectSpecToSelf`，完成时也会在 OnEffectRemoved 中应用 OnComplete GE；源码路径：`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Private/GameplayEffectComponents/AdditionalEffectsGameplayEffectComponent.cpp:74`、`:111`。
- 开发实践推断：如果附加 GE 又配置回触发原 GE，或完成 GE 又触发同一链路，可能形成递归/重复应用；源码依据是 AdditionalEffects 没有全局递归保护，只按配置创建和应用 spec；源码路径：`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Private/GameplayEffectComponents/AdditionalEffectsGameplayEffectComponent.cpp:34`、`:74`。

## GrantedAbilities 依赖 ActiveGE 生命周期但 GE 被提前移除

- Abilities 组件在 ActiveGE 添加时绑定 OnEffectRemoved / OnInhibitionChanged，移除时按 RemovalPolicy 清理 Ability；源码路径：`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Private/GameplayEffectComponents/AbilitiesGameplayEffectComponent.cpp:147`、`:151`、`:152`、`:158`。
- RemovalPolicy 为 `CancelAbilityImmediately` 会 `ClearAbility`，`RemoveAbilityOnEnd` 会 `SetRemoveAbilityOnEnd`，提前移除 ActiveGE 会提前触发这些策略；源码路径：`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Private/GameplayEffectComponents/AbilitiesGameplayEffectComponent.cpp:124`、`:128`、`:133`。

## ChanceToApply 在预测客户端和服务器结果不一致

- ChanceToApply 用 `FMath::FRand()` 和 scalable float 结果决定是否阻止应用；源码路径：`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Private/GameplayEffectComponents/ChanceToApplyGameplayEffectComponent.cpp:24`、`:27`。
- 开发实践推断：LocalPredicted GE 如果客户端和服务器都执行随机判定，可能出现客户端预测成功但服务器失败，或反之；源码依据是客户端有 prediction key 时也会进入 `ApplyGameplayEffectSpecToSelf`，而 periodic 之外的 GE 可以预测；源码路径：`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Private/AbilitySystemComponent.cpp:812`、`:862`。

## CustomCanApply 写入有副作用逻辑

- CustomCanApply 在应用前检查阶段调用 requirement CDO 的 `CanApplyGameplayEffect`，任一 false 就阻止应用；源码路径：`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Private/GameplayEffectComponents/CustomCanApplyGameplayEffectComponent.cpp:13`、`:18`、`:21`。
- 开发实践推断：`CanApply` 类逻辑应保持只读/无副作用；源码依据是同一检查可能发生在预测、服务端或重复尝试应用时，并且组件本身是 GE 资产单例；源码路径：`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Public/GameplayEffectComponent.h:23`、`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Private/AbilitySystemComponent.cpp:844`。

## Response GE 又授予触发 ResponseTable 的 tag，造成循环

- ResponseTable 会监听 ASC tag count 并对同一 ASC 应用/移除响应 GE；GE granted tags 又会更新同一 ASC tag map；源码路径：`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Private/GameplayTagResponseTable.cpp:66`、`:181`、`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Private/GameplayEffect.cpp:4373`。
- 开发实践推断：Response GE 不应再授予 positive/negative 触发 tag 本身，建议把触发 tag 和响应结果 tag 分层命名。

## GEComponents 写成复杂战斗系统核心

- GEComponents 是 GE 资产配置子对象，源码注释明确只有一个 component 供所有 applied instances 共享，不应存 per-execution runtime state；源码路径：`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Public/GameplayEffectComponent.h:23`、`:24`。
- 开发实践推断：复杂结算优先放 ExecutionCalculation / Ability / 项目战斗系统，GEComponent 更适合声明式配置和轻量 hook；源码依据是 GE 到组件的调用点较少，注释要求实现者仔细阅读 GE flow 并注册回调；源码路径：`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Public/GameplayEffectComponent.h:19`、`:20`。

## 以为 GEComponents 本身会复制运行时状态

- GEComponents 本身是 GE 内的 instanced 配置对象，不是每个 ActiveGE 的复制状态；ActiveGE、MinimalReplicationTags、GameplayCue、Attribute 等结果才走对应复制路径；源码路径：`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Public/GameplayEffect.h:2422`、`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Public/GameplayEffectComponent.h:23`、`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Public/AbilitySystemComponent.h:719`。
---

# 常见坑：ExecutionCalculation / MMC / Attribute Capture（第十四轮）

## 把 MMC 当成能输出多个 Attribute 修改

- `UGameplayModMagnitudeCalculation::CalculateBaseMagnitude` 只返回一个 float；它接入的是 `FGameplayEffectModifierMagnitude` 的 CustomCalculationClass 分支；源码路径：`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Public/GameplayModMagnitudeCalculation.h:29`、`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Private/GameplayEffect.cpp:1066`。
- 多属性输出应看 `FGameplayEffectCustomExecutionOutput::OutputModifiers`，这是 ExecutionCalculation 路径；源码路径：`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Public/GameplayEffectExecutionCalculation.h:242`、`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Private/GameplayEffect.cpp:3146`。

## 把 ExecutionCalculation 当成持续 buff 的普通 modifier

- Execution 在 `ExecuteActiveEffectsFrom` 中执行，输出 modifiers 会走 `InternalExecuteMod` 直接修改 base；源码路径：`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Private/GameplayEffect.cpp:3065`、`:3136`、`:3155`、`:3907`。
- 持续属性修正通常由 Duration/Infinite GE 的普通 modifiers 加入 aggregator；源码路径：`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Private/GameplayEffect.cpp:4347`、`:4350`。这是开发实践推断。

## Attribute Capture Source / Target 方向写反

- 捕获定义通过 `AttributeSource` 区分 Source 和 Target；Source 捕获来自 `EffectContext.GetInstigatorAbilitySystemComponent()`，Target 捕获来自目标 ASC；源码路径：`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Public/GameplayEffectAttributeCaptureDefinition.h:43`、`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Private/GameplayEffect.cpp:1764`、`:1748`。
- Source/Target 写反会导致 `FindCaptureSpecByDefinition` 找不到或读到错误 ASC 的属性；源码路径：`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Private/GameplayEffect.cpp:2500`。

## Snapshot / Non-Snapshot 理解错误

- Snapshot 捕获会 `TakeSnapshotOf` 当前 aggregator；Non-Snapshot 保存 aggregator 引用；源码路径：`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Private/GameplayEffect.cpp:3756`、`:3760`。
- 只有 Non-Snapshot capture 会注册 linked aggregator callback 并在依赖变化时参与重算；源码路径：`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Private/GameplayEffect.cpp:2389`、`:2395`、`:3368`。

## SetByCaller key 写错或忘记设置

- SetByCaller 有 Name 和 GameplayTag 两套 key；setter 分别写 `SetByCallerNameMagnitudes` / `SetByCallerTagMagnitudes`；源码路径：`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Public/GameplayEffect.h:1243`、`:1244`、`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Private/GameplayEffect.cpp:2200`、`:2208`。
- 读取缺失时会返回默认值，并在 `WarnIfNotFound` 为 true 时记录 error；源码路径：`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Private/GameplayEffect.cpp:2216`、`:2238`。

## AttributeBased Modifier 捕获了错误 Attribute

- AttributeBased magnitude 从 `BackingAttribute` 对应的 capture spec 取值，并要求 spec 已有有效 captured attribute；源码路径：`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Private/GameplayEffect.cpp:968`、`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Public/GameplayEffect.h:164`。
- `AttemptCalculateMagnitude` 会先通过 `CanCalculateMagnitude` 检查 capture defs；失败时 magnitude 变 0 并可能记录 warning；源码路径：`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Private/GameplayEffect.cpp:1125`、`:1136`、`:1955`。

## ExecutionCalculation 中直接修改 Actor 状态或播放表现

- ExecutionCalculation 的源码接入点是数值执行：读取 params、输出 modifiers、手动标记 cues/conditional/stack；源码路径：`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Public/GameplayEffectExecutionCalculation.h:304`、`:229`、`:220`、`:223`。
- 开发实践推断：Actor 状态流转、表现播放、权限校验应放 Ability、GameplayCue 或项目系统；ExecutionCalculation 中写副作用会放大预测与重放风险。源码依据：预测 GE 可在客户端先执行；源码路径：`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Private/GameplayEffect.cpp:2924`。

## ExecutionCalculation 中写随机逻辑导致预测不一致

- `PredictivelyExecuteEffectSpec` 会在有效 prediction key 下执行普通 modifiers 和 executions；源码路径：`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Private/GameplayEffect.cpp:2924`、`:3001`。
- 开发实践推断：随机、时间和外部状态应由服务端权威决定，或提前写入 SetByCaller 并保证客户端/服务端使用相同输入。

## MMC 中读取外部非确定性状态

- MMC 的外部依赖注册默认不允许非权威客户端；若开启且同时依赖 attribute capture，会触发 ensure；源码路径：`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Public/GameplayModMagnitudeCalculation.h:52`、`:60`、`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Private/GameplayModMagnitudeCalculation.cpp:24`。
- 开发实践推断：MMC 适合确定性数值函数，不适合读取会在客户端/服务端不同步的外部 mutable 状态。

## DOT / HOT 周期执行和 Duration modifier 混淆

- Instant/Periodic execute 路径会直接执行 modifier 并修改 base；Duration/Infinite 普通 modifier 会进入 aggregator 影响 current value；源码路径：`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Private/GameplayEffect.cpp:3065`、`:3907`、`:4347`。
- 常见错误是把一次性伤害做成持续 aggregator modifier，或把持续移速 buff 做成 periodic execution；这是开发实践推断。

## Damage meta attribute 和 ExecutionCalculation 输出职责混淆

- ExecutionCalculation 可以输出任意 `FGameplayModifierEvaluatedData`，AttributeSet 的 `PostGameplayEffectExecute` 会在执行后收到 callback data；源码路径：`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Private/GameplayEffectExecutionCalculation.cpp:361`、`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Private/GameplayEffect.cpp:3946`。
- Damage 是否作为 meta attribute 不是 GAS 固定类型，是项目实践；本轮源码未发现固定 `Damage` attribute 类型，未确认。

## Aggregator op 顺序理解错误

- 聚合公式是 `((BaseValue + AddBase) * MultiplyAdditive / DivideAdditive * MultiplyCompound) + AddFinal`，Override 优先返回 override magnitude；源码路径：`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Public/GameplayEffectTypes.h:116`、`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Private/GameplayEffectAggregator.cpp:78`、`:98`。
- 旧名 `Additive/Multiplicitive/Division` 仍作为隐藏兼容枚举存在；源码路径：`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Public/GameplayEffectTypes.h:143`、`:144`、`:145`。

## Override modifier 和 Additive modifier 叠加结果不符合预期

- `FAggregatorModChannel::EvaluateWithBase` 先检查 qualified Override，命中就直接返回 override magnitude，不再继续加算其他 op；源码路径：`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Private/GameplayEffectAggregator.cpp:76`、`:78`。
- 开发实践推断：如果需要“覆盖后再加成”，应拆分设计或确认 evaluation channel 顺序，而不是期待同一 channel 中 Override 后继续加 Additive。

## Captured tags 没更新导致计算结果不符合预期

- AttributeBased 和 aggregator qualification 使用 captured source/target tags 或 tag filters；源码路径：`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Private/GameplayEffect.cpp:983`、`:984`、`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Private/GameplayEffectAggregator.cpp:28`。
- `CapturedSourceTags` / `CapturedTargetTags` 标记 `NotReplicated`，source tags 在 spec source capture 时 recapture，target tags 在 execute/apply 路径刷新；源码路径：`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Public/GameplayEffect.h:1200`、`:1204`、`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Private/GameplayEffect.cpp:1776`、`:3081`。

## 编辑器配置了 CustomCalculationClass 但没有正确 capture 需要的属性

- `FGameplayEffectModifierMagnitude::GetAttributeCaptureDefinitions` 会从 MMC CDO 的 `GetAttributeCaptureDefinitions` 收集捕获需求；源码路径：`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Private/GameplayEffect.cpp:1217`、`:1230`。
- Editor details 会根据 execution CDO 的 valid captures 显示 scoped modifiers，但运行时仍以 capture 是否有效为准；源码路径：`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilitiesEditor/Private/GameplayEffectExecutionDefinitionDetails.cpp:91`、`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Private/GameplayEffect.cpp:1125`。

---

# 常见坑：GAS Tests / 官方测试用例反推实践（第十五轮）

## 把引擎测试类当作正式业务模板照搬

- `AAbilitySystemTestPawn` 是测试 Pawn：构造 replicated ASC，并在 `PostInitializeComponents` 调 `InitStats`；源码路径：`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Private/AbilitySystemTestPawn.cpp:14`、`:15`、`:25`。
- 它没有在自身初始化里处理完整项目常见的 Owner/Avatar/PlayerState/输入/网络架构；`InitAbilityActorInfo` 在 PredictionKey 测试里手动调用；源码路径：`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Private/Tests/PredictionKeyTests.cpp:465`。
- 开发实践推断：TestPawn 可借鉴为自动化测试最小宿主，不应直接作为业务 Character 模板。

## TestAttributeSet 的 Damage 写法和项目战斗系统边界混淆

- `Damage` 注释明确不是 persistent attribute；源码路径：`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Public/AbilitySystemTestAttributeSet.h:36`、`:37`、`:38`。
- `PostGameplayEffectExecute` 中把 Damage 转成 `Health -= Damage` 并清零；源码路径：`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Private/AbilitySystemTestAttributeSet.cpp:100`、`:109`、`:110`。
- 开发实践推断：Damage meta attribute 是机制示例，不等于完整伤害、死亡、抗性、表现系统都应写在 AttributeSet。

## 测试中直接构造 GE 的方式和项目资产配置混淆

- `GameplayEffectTests.cpp` 用 `NewObject<UGameplayEffect>` 与 `AddModifier` helper 直接写 `Effect->Modifiers`；源码路径：`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Private/Tests/GameplayEffectTests.cpp:126`、`:750`、`:753`。
- 开发实践推断：这适合机制单测；正式项目通常通过 GE 资产、Spec、SetByCaller、DynamicTags 等运行时数据组合，不应把测试 helper 当项目配置流程。

## 只验证 CurrentValue，不验证 BaseValue

- Aggregator 测试明确验证 Mana current 变化，并验证 BaseValue 不变；源码路径：`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Private/Tests/GameplayEffectTests.cpp:183`、`:216`、`:240`。
- 常见错误是只看 UI 当前值，不检查 base/current 差异，导致 Duration/Infinite GE 与 Instant/Periodic GE 的效果判断混乱。

## 只验证服务端属性，不验证客户端复制结果

- `UAbilitySystemTestAttributeSet::GetLifetimeReplicatedProps` 中调用 `DISABLE_ALL_CLASS_REPLICATED_PROPERTIES`，`DOREPLIFETIME` 代码块被注释；源码路径：`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Private/AbilitySystemTestAttributeSet.cpp:119`、`:121`、`:124`。
- 因此当前 AttributeSet 测试不能证明项目侧 Attribute RepNotify、`GAMEPLAYATTRIBUTE_REPNOTIFY` 或 UI delegate 的客户端复制链正确，未确认。

## 忽略 tag count 的叠加语义

- Tag count 测试显示设置子 tag 后父 tag count 也会体现，两个子 tag 叠加后父 count 为 3；源码路径：`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Private/Tests/GameplayTagCountContainerTests.cpp:21`、`:27`、`:37`、`:40`。
- 常见错误是把 tag 当 bool，而不是 count，导致多来源 GE/LooseTag 移除一个来源后误以为状态应消失。

## 忽略 ActiveGE 移除后的清理

- Mana buff 测试在移除 ActiveGE 后断言 Mana 回到原值；源码路径：`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Private/Tests/GameplayEffectTests.cpp:179`、`:188`、`:191`。
- Stacking/Aggregator 测试也保存 handles 并逐个移除后检查恢复；源码路径：`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Private/Tests/GameplayEffectTests.cpp:265`、`:278`。

## 忽略 PredictionKey 的 catch-up / reject

- PredictionKey 测试绑定 reject、caught-up、reject-or-caught-up delegate；源码路径：`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Private/Tests/PredictionKeyTests.cpp:111`、`:112`、`:113`。
- `FScopedDiscardPredictions` 测试覆盖 SilentlyDrop、Accept、Reject；源码路径：`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Private/Tests/PredictionKeyTests.cpp:486`、`:500`、`:512`。

## 只看单测通过，不检查网络场景

- 当前 Tests 目录有 PredictionKey 单测，但未覆盖完整 Ability 激活 RPC、TargetData RPC、ActiveGE replication、Attribute RepNotify、ReplicationProxy、Iris；源码扫描路径：`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Private/Tests`。
- 开发实践推断：多人项目需要额外做 listen server / dedicated server / autonomous proxy / simulated proxy 场景测试。

## 自动化测试里没有覆盖 Ability End / Cancel 的全部业务路径

- ASC 测试覆盖 `CancelAbilityHandle` 和 ended callback；源码路径：`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Private/Tests/AbilitySystemComponentTests.cpp:166`、`:167`、`:168`。
- 但 Cost/Cooldown、AbilityTask、Montage、TargetData、网络预测失败后的 End/Cancel 清理没有在当前测试里完整覆盖，未确认。

## 没有区分 Instant / Duration / Infinite / Periodic GE

- 测试分别构造 Instant damage、Infinite mana buff、Periodic damage、HasDuration stacking；源码路径：`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Private/Tests/GameplayEffectTests.cpp:126`、`:166`、`:281`、`:330`。
- 常见错误是用一个 GE 测试结论外推所有 DurationPolicy。

## 没有验证 GameplayCue Remove

- Direct API 测试显式 `AddGameplayCue` 后 `RemoveGameplayCue` 并断言 `OnRemove` 与 tag count；源码路径：`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Private/Tests/GameplayEffectTests.cpp:443`、`:458`、`:461`、`:466`。
- 只测 `ExecuteGameplayCue` 不能证明持续 Cue 的移除逻辑正确；源码路径：`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Private/Tests/GameplayEffectTests.cpp:527`、`:529`。

## 没有验证 AttributeSet 回调顺序

- 当前 TestAttributeSet 只实际启用 `PostGameplayEffectExecute` 的 Damage 处理；`PreGameplayEffectExecute` 主体在 `#if 0` 中，实际返回 true；源码路径：`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Private/AbilitySystemTestAttributeSet.cpp:29`、`:31`、`:89`、`:92`。
- 因此 Pre/Post callback 的完整顺序和复杂 clamp/damage/death 流程仍需项目侧测试，未确认。

## 没有验证 SetByCaller 缺失情况

- SetByCaller stacking duration 测试设置 `FSetByCallerFloat` 的 DataName，并在 spec 上调用 `SetSetByCallerMagnitude`；源码路径：`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Private/Tests/GameplayEffectTests.cpp:366`、`:373`、`:382`、`:394`。
- 缺失 SetByCaller key 的错误/默认值路径没有在这个测试中覆盖，未确认。

## 没有验证 TargetData 校验失败路径

- 当前扫描未发现 `GameplayAbilityTargetingTests.cpp`，TargetData / TargetActor / WaitTargetData / TargetData RPC 未被 `Private/Tests` 覆盖，未确认。
- 开发实践推断：项目中凡是客户端 TargetData 驱动伤害或 GE 应用，都应补服务端校验失败、取消、consume、防重复触发测试。
