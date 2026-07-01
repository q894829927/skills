# GAS 最终模板化规则与生成前检查清单

本文件是 `ue56-gameplayabilities` Skill 的收束入口：只写规则、结构和检查清单，不生成完整业务代码。需要源码细节时回到对应专题文档复核。

## 1. 生成 GAS 代码前必须确认

- ASC 放在哪里：Character、PlayerState 或其他 Actor；先确认 OwnerActor / AvatarActor 分离模型，见 `core-classes.md`。
- OwnerActor / AvatarActor 是谁：必须规划 `InitAbilityActorInfo` 调用时机，见 `pitfalls.md`。
- AttributeSet 有哪些：确认属性、meta attribute、复制需求和 UI 监听需求，见 `attributes.md`。
- Ability 是否需要预测：确认 `NetExecutionPolicy`、PredictionKey、TargetData/RPC、Cue 重复播放风险，见 `networking-prediction.md`。
- 是否需要 Cost / Cooldown：优先使用 GE，并确认 Cooldown Granted Tag，见 `gameplay-effects.md`。
- 是否需要 TargetData：确认目标来自 Actor、HitResult、Location 还是 TargetActor，见 `targeting-targetdata.md`。
- 是否需要 GameplayCue：确认一次性表现还是持续表现，见 `gameplay-cues.md`。
- 是否需要 GEComponents：确认是否用声明式组件处理 tags、免疫、移除其他 GE、额外 GE、授予 Ability，见 `gameplay-effect-components.md`。
- 是否需要 ExecutionCalculation / MMC：确认是“算一个数”还是“输出多个属性修改”，见 `calculations-captures.md`。
- 是否需要网络复制：确认 ActiveGE、Attribute、Cue、Tag、TargetData、ReplicationMode，见 `networking-prediction.md`。
- 是否需要 UI 监听 Attribute / Tag：优先 delegate，不建议 Tick 轮询，见 `attributes.md` 和 `gameplay-tags-response.md`。

## 2. AttributeSet 生成规则

- 属性声明：属性值优先用 `FGameplayAttributeData`，配合 GAS 属性访问宏；`ATTRIBUTE_ACCESSORS` 若出现，按项目侧封装处理，见 `attributes.md`。
- RepNotify：AttributeSet 子对象复制和具体 Attribute 复制不是一回事；需要 `ReplicatedUsing`、`DOREPLIFETIME` 和 `GAMEPLAYATTRIBUTE_REPNOTIFY`，见 `attributes.md`。
- Base / Current：Instant / Periodic GE 通常修改 BaseValue；Duration / Infinite GE 通常经 Aggregator 影响 CurrentValue，见 `attributes.md` 和 `gameplay-effects.md`。
- Meta Attribute：Damage 这类 meta attribute 是开发实践，不是 GAS 固定类型；用后通常清零，见 `attributes.md`。
- Clamp 边界：CurrentValue clamp 看 `PreAttributeChange`；Instant/Periodic GE 结算看 `PreGameplayEffectExecute` / `PostGameplayEffectExecute`，见 `attributes.md`。
- 常见坑：AttributeSet 没加入 ASC、属性复制没写全、直接 Set 绕过 GAS、MaxHealth 改变后 Health 未同步 clamp，见 `pitfalls.md`。

## 3. GameplayEffect 生成规则

- Instant / Duration / Infinite：一次性伤害/治疗/扣资源用 Instant；限时 buff/debuff/cooldown 用 Duration；永久被动/装备属性用 Infinite，见 `gameplay-effects.md`。
- Modifier / Execution / MMC：固定值、等级曲线和单属性修正用 Modifier；单个自定义 magnitude 用 MMC；复杂多属性结算用 ExecutionCalculation，见 `calculations-captures.md`。
- Cost / Cooldown GE：Cost 通常 Instant 修改资源；Cooldown 通常 Duration + Granted Cooldown Tag，见 `gameplay-effects.md`。
- SetByCaller：运行时数值必须在 Spec 应用前设置，Name/GameplayTag key 不要混用，见 `globals-blueprint-library.md` 和 `calculations-captures.md`。
- GEComponents：只保存 GE 配置，不保存 per-target/per-application 状态，见 `gameplay-effect-components.md`。
- GameplayCue：GE 可配置 GameplayCues；Instant 更适合 Execute，Duration/Infinite 更适合 Add/Remove 生命周期，见 `gameplay-cues.md`。
- Tag requirements：Asset Tags、Granted Tags、Application/Ongoing/Removal requirements、Modifier source/target tags 要分清，见 `gameplay-tags-response.md`。
- 常见坑：CDO 当运行时状态、Cooldown 没 Granted Tag、SetByCaller 缺失、Duration/Instant 语义混淆、AdditionalEffects 递归，见 `pitfalls.md`。

## 4. GameplayAbility 生成规则

- InstancingPolicy：需要状态、AbilityTask、latent 流程时优先实例化策略；避免 NonInstanced 保存成员状态，见 `core-classes.md`。
- NetExecutionPolicy：本地纯表现用 LocalOnly；需要客户端预测用 LocalPredicted；权威结算用 ServerOnly/ServerInitiated，见 `networking-prediction.md`。
- CommitAbility：ASC 不会自动替业务 Ability 调用 Commit；Ability 实现必须主动调用并处理失败，见 `call-flows.md`。
- Cost / Cooldown：`CommitAbility` 成功后才真正 ApplyCooldown / ApplyCost；失败后通常应 EndAbility，见 `gameplay-effects.md`。
- AbilityTask：创建后调用 `ReadyForActivation`；绑定 delegate 后在 `OnDestroy` / `EndTask` 清理，见 `ability-tasks.md`。
- End / Cancel：Ability 激活后必须有稳定结束路径；取消时清理 task、tag、cue、montage/targeting 等资源，见 `pitfalls.md`。
- Prediction：跨帧逻辑需要新的 prediction window 或明确服务端确认；不要假设本地预测一定成功，见 `networking-prediction.md`。
- 常见坑：忘记 Commit、Commit 失败不 End、EndAbility 不调用、AbilityTask 悬挂、LocalPredicted 不处理 reject，见 `pitfalls.md`。

## 5. AbilityTask / TargetData 生成规则

- WaitTargetData：适合玩家瞄准、点击、确认/取消目标；简单 Actor/HitResult/Location 可直接用 BlueprintLibrary 创建 TargetData，见 `targeting-targetdata.md`。
- TargetActor：适合封装持续目标选择、Trace、范围、放置预览；不要承载伤害结算核心逻辑，见 `targeting-targetdata.md`。
- 服务端校验：客户端 TargetData 不能直接信任；服务端应校验 Actor、HitResult、距离、阵营、可见性等项目规则，开发实践推断，见 `targeting-targetdata.md`。
- GenericReplicatedEvent：适合输入 press/release、confirm/cancel、network sync；使用后注意 consume，见 `ability-tasks.md` 和 `networking-prediction.md`。
- Task 清理：Task 广播后是否 EndTask 要明确；`OnDestroy` 中解绑 ASC delegate、montage delegate、tag/attribute delegate、target actor 引用，见 `ability-tasks.md`。
- 常见坑：TargetData 服务端没收到、PredictionKey 不匹配、CustomMulti 误以为自动结束、TargetActor 没清理、RPC batch 下误判顺序，见 `pitfalls.md`。

## 6. GameplayCue 生成规则

- Static / Actor Notify：一次性无状态表现优先 Static；需要持续状态、循环特效或生命周期管理用 Actor，见 `gameplay-cues.md`。
- Execute / Add / Remove：瞬时表现用 Execute；持续表现用 Add/Remove 成对；不要用 Execute 做需要移除的持续特效，见 `gameplay-cues.md`。
- GE 触发 Cue：通过 GE GameplayCues 字段可让效果生命周期驱动 Cue；注意 Instant 与 Duration/Infinite 事件语义不同，见 `gameplay-cues.md`。
- 参数来源：Cue 参数可来自 EffectContext、HitResult、TargetData、Source/Target Tags；需要命中点时确保 HitResult 进入 Context，见 `gameplay-cues.md` 和 `targeting-targetdata.md`。
- 网络与重复播放：预测、本地播放、服务端确认、NetMulticast 可能造成重复表现；用 GameplayCue 只做表现，状态判断看 ASC tags，见 `networking-prediction.md`。
- 常见坑：Cue tag 未绑定 Notify、Actor Notify 没清理、Looping Cue 没处理 Removed、Static Cue 保存状态、客户端没加载 Cue 资产，见 `pitfalls.md`。

## 7. Damage / Healing 推荐流程

```text
Ability / TargetData
  -> HitResult / Target ASC
  -> EffectContext 写入 Instigator / Causer / SourceObject / HitResult
  -> GE Spec 设置 SetByCaller 或 Attribute Capture
  -> Modifier / MMC / ExecutionCalculation 计算数值
  -> Damage meta attribute 或直接 Health Modifier
  -> AttributeSet::PostGameplayEffectExecute 处理 Damage -> Health / Clamp
  -> Attribute change delegate 通知 UI
  -> GameplayCue 用 Context / HitResult 播放表现
```

- 简单治疗可用 Modifier；复杂治疗/伤害抗性/暴击/护盾可用 ExecutionCalculation，见 `calculations-captures.md`。
- Damage meta attribute 是项目实践，不是 GAS 固定类型；用后清零，见 `attributes.md`。
- 死亡、掉落、任务、表现等大型业务不要塞满 AttributeSet；开发实践推断，见 `attributes.md`。

## 8. UI 监听规则

- Attribute：用 `GetGameplayAttributeValueChangeDelegate` 监听，避免 Tick 轮询，见 `attributes.md`。
- Tag：用 `RegisterGameplayTagEvent` 或 AbilityTask/Blueprint wrapper 监听 tag count 变化，见 `gameplay-tags-response.md`。
- GameplayCue：只作为表现，不作为状态源；状态应来自 ASC tag / Attribute / ActiveGE，见 `gameplay-cues.md`。
- 网络延迟：客户端 UI 可能先看到预测值，再被服务端纠正；预测属性和 RepNotify 规则见 `networking-prediction.md`。

## 9. 网络预测检查清单

- LocalPredicted Ability：确认调用端、本地预测、服务端确认/拒绝路径。
- PredictionKey：跨帧、AbilityTask、TargetData、GenericReplicatedEvent 不要复用过期 key。
- `FScopedPredictionWindow`：需要预测写操作时确认是否在有效窗口内。
- TargetData RPC：确认 `CallServerSetReplicatedTargetData`、服务端 delegate、consume。
- GenericReplicatedEvent：确认 press/release/confirm/cancel 是否 consume。
- Server confirm / reject：确认 `ClientActivateAbilitySucceed` / `ClientActivateAbilityFailed` 后表现和任务清理。
- GameplayCue 重复播放：确认预测 Cue 与服务端 Cue 是否去重或可接受。
- Attribute 服务器纠正：确认 RepNotify / delegate / UI 能处理回滚。
- ActiveGE replication mode：Full / Mixed / Minimal 的可见数据不同，见 `networking-prediction.md`。

## 10. 排错入口

| 问题 | 优先查文档 | 源码入口 |
|---|---|---|
| 技能不能激活 | `debugging-logging.md`、`call-flows.md`、`pitfalls.md` | `Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Private/AbilitySystemComponent_Abilities.cpp:1583`、`:1683` |
| GE 没生效 | `gameplay-effects.md`、`gameplay-effect-components.md`、`debugging-logging.md` | `Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Private/AbilitySystemComponent.cpp:798`；`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Private/GameplayEffect.cpp:881` |
| Attribute 没变化 | `attributes.md`、`calculations-captures.md`、`debugging-logging.md` | `Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Private/GameplayEffect.cpp:3907`、`:3946` |
| Cue 不播放 | `gameplay-cues.md`、`debugging-logging.md` | `Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Private/GameplayCueManager.cpp:169`、`:217` |
| TargetData 没到服务端 | `targeting-targetdata.md`、`networking-prediction.md`、`debugging-logging.md` | `Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Private/AbilitySystemComponent_Abilities.cpp:3945`、`:3838` |
| Tag 没同步 | `gameplay-tags-response.md`、`networking-prediction.md` | `Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Public/AbilitySystemComponent.h:649`、`:690`、`:719` |
| 预测失败 | `networking-prediction.md`、`debugging-logging.md` | `Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Private/AbilitySystemComponent_Abilities.cpp:2251`；`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Private/GameplayPrediction.cpp:10` |
| 性能异常 | `quick-reference.md` 的 Stats / Profiling 收束、`debugging-logging.md` | `Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Public/AbilitySystemStats.h:8`、`:44` |

## 11. 未来可选扩展

- `STAT_AggregatorEvaluate` 已声明和定义，但本轮未确认实际 `SCOPE_CYCLE_COUNTER` 使用点；源码路径：`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Public/AbilitySystemStats.h:30`、`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Private/AbilitySystemStats.cpp:25`。
- 本轮未确认 GameplayAbilities 侧存在 `DECLARE_MEMORY_STAT` 或 `DECLARE_DWORD_COUNTER_STAT` 使用点；确认的是 `DECLARE_DWORD_ACCUMULATOR_STAT_EXTERN` 的 AbilityTask Count；源码路径：`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Public/AbilitySystemStats.h:44`。
- 本轮未确认 TargetData、PredictionKey、GenericReplicatedEvent 有独立专用 stat；已确认相关 RPC/调试入口见 `debugging-logging.md` 和 `networking-prediction.md`。
- 真机多人压力、Cue 资产异步加载宽路径、复杂 GEComponents 递归、项目侧 AbilityTask 泄漏等性能问题需要项目场景压测，本 Skill 不再按轮次展开。
