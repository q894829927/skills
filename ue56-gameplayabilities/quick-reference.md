# UE 5.6 GAS 速查表

> 适用范围：基于当前 `ue56-gameplayabilities` Skill 知识库整理，用于快速判断“什么时候用哪个 GAS 类 / 函数 / 配置”。默认只面向项目侧开发，不修改 `Engine/` 源码。

## 0. 一句话总览

```text
ASC = 能力系统总管
GA  = 技能流程
GE  = 效果结果
AttributeSet = 属性数据
GameplayTag = 状态/分类标签
GameplayCue = 表现特效
AbilityTask = 异步等待/持续流程
```

最常见主线：

```text
GiveAbility
    ↓
TryActivateAbility / TryActivateAbilitiesByTag
    ↓
CanActivateAbility
    ↓
ActivateAbility
    ↓
CommitAbility
    ↓
Apply GameplayEffect
    ↓
AttributeSet 回调 / GameplayCue 表现
    ↓
EndAbility
```

---

## 1. 类职责速查

| 类 / 结构 | 中文理解 | 主要职责 | 常见使用位置 |
|---|---|---|---|
| `UAbilitySystemComponent` | ASC，总管 | 授予/激活 Ability，应用/移除 GE，管理 AttributeSet、Tag、Cue、复制与预测 | Character / PlayerState / Actor 组件 |
| `UGameplayAbility` | GA，技能 | 技能能否释放、释放流程、输入、蒙太奇、投射物、目标选择、结束 | 技能 C++ 类或蓝图 Ability |
| `FGameplayAbilitySpec` | 已授予技能记录 | 保存 Ability CDO、等级、InputID、Handle、实例、动态 Tag | ASC 的 `ActivatableAbilities` 内部 |
| `UGameplayEffect` | GE，效果配置 | 修改属性、授予 Tag、触发 Cue、持续时间、周期、堆叠、Cost/Cooldown | GE 蓝图资产 |
| `FGameplayEffectSpec` | 一次 GE 应用数据 | 保存 GE Def、Level、Context、SetByCaller、捕获属性、动态 Tag | Ability/ASC 应用 GE 前创建 |
| `FActiveGameplayEffect` | 已生效 GE 实例 | 保存持续/无限 GE 的运行时状态、Handle、Timer、Stack、PredictionKey | ASC 的 ActiveGE 容器内部 |
| `UAttributeSet` | 属性集合 | 定义 Health、Mana、Damage 等属性，处理属性回调 | ASC 管理的子对象 |
| `FGameplayAttributeData` | 属性值 | 保存 `BaseValue` 与 `CurrentValue` | AttributeSet 成员 |
| `FGameplayAttribute` | 属性句柄 | 定位 AttributeSet 中具体属性 | GE Modifier / ASC 查询 |
| `UGameplayCueManager` | Cue 管理器 | 路由 GameplayCue，播放一次性或持续表现 | ASC / GE 触发 |
| `UAbilityTask` | Ability 异步任务 | 等待输入、事件、目标数据、蒙太奇、Tag、属性变化等 | Ability 内部 |

---

## 2. 什么时候用 GA，什么时候用 GE

| 需求 | 用 GA 还是 GE | 说明 |
|---|---|---|
| 玩家按键释放火球 | GA | 技能流程、生成投射物、播放动作 |
| 火球造成伤害 | GE | Instant GE 修改 Damage/Health |
| 火球消耗蓝量 | GE | Cost GE，通常 Instant，Mana `AddBase -20` |
| 火球进入冷却 | GE | Cooldown GE，通常 HasDuration + Granted Cooldown Tag |
| 加速 5 秒 | GE | Duration GE，注册 Aggregator，影响 CurrentValue |
| 眩晕 2 秒 | GE | Duration GE + Granted `State.Stun` Tag |
| 引导施法、等待目标选择 | GA + AbilityTask | GA 管流程，AbilityTask 等待事件/目标 |
| 命中特效、音效 | GameplayCue | 表现层，不负责核心数值 |

记忆：

```text
GA 负责“怎么做”
GE 负责“改什么”
Cue 负责“表现什么”
AttributeSet 负责“存什么”
ASC 负责“管起来”
```

---

## 3. ASC 常用函数速查

### 3.1 Ability 授予 / 激活 / 取消

| 你想做什么 | 函数 | 常见位置 | 注意点 |
|---|---|---|---|
| 给角色技能 | `GiveAbility` | 服务端初始化、升级、装备 | authority-only，通常服务端调用 |
| 移除某个技能 | `ClearAbility` | 卸下装备、移除状态 | 服务端为主 |
| 清空技能 | `ClearAllAbilities` | 重置角色 | 谨慎使用 |
| 按 Handle 激活技能 | `TryActivateAbility` | 输入绑定、业务代码 | 需要已有 `FGameplayAbilitySpecHandle` |
| 按 Tag 激活技能 | `TryActivateAbilitiesByTag` | 输入系统 | 推荐用 Ability Tag 做解耦 |
| 取消某个技能 | `CancelAbility` | 状态打断 | 按 Ability CDO 取消 |
| 按 Tag 批量取消技能 | `CancelAbilities` | 沉默、眩晕、死亡 | 可按 WithTags/WithoutTags 筛选 |
| 查找技能 Spec | `FindAbilitySpecFromHandle` | 高级业务逻辑 | 修改 Spec 后记得 Dirty，不建议随便改内部数组 |

### 3.2 GameplayEffect 应用 / 查询 / 移除

| 你想做什么 | 函数 | 常见位置 | 注意点 |
|---|---|---|---|
| 创建一次 GE Spec | `MakeOutgoingSpec` | ASC 侧 | 带 Level 和 Context |
| Ability 创建 GE Spec | `MakeOutgoingGameplayEffectSpec` | Ability 内部 | 会带 Ability 上下文 |
| 给自己应用 GE | `ApplyGameplayEffectToSelf` | Buff、Cost、Cooldown、回血 | 可直接用 GE Class |
| 给目标应用 GE | `ApplyGameplayEffectToTarget` | 伤害、治疗、Debuff | 需要目标 ASC |
| 给自己应用已有 Spec | `ApplyGameplayEffectSpecToSelf` | SetByCaller 场景 | 动态数值推荐走 Spec |
| 给目标应用已有 Spec | `ApplyGameplayEffectSpecToTarget` | 动态伤害、治疗 | 火球命中常用 |
| 移除 Active GE | `RemoveActiveGameplayEffect` | 取消 Buff、解除 Debuff | 需要 ActiveGE Handle |
| 查询 Active GE | `GetActiveGameplayEffect` | UI、状态查询 | 返回 const 指针，不应直接改 |

### 3.3 Attribute / Tag / Cue

| 你想做什么 | 函数 | 常见位置 | 注意点 |
|---|---|---|---|
| 读当前属性值 | `GetNumericAttribute` | UI、业务判断 | 返回 CurrentValue |
| 设置 BaseValue | `SetNumericAttributeBase` | 初始化/特殊业务 | 一般更推荐用 GE 修改 |
| 监听属性变化 | `GetGameplayAttributeValueChangeDelegate` | UI 血条/蓝条 | 推荐，不要每帧轮询 |
| 查询是否有 Tag | `HasMatchingGameplayTag` | 状态判断 | 如是否眩晕、是否冷却 |
| 查询全部 Tags | `HasAllMatchingGameplayTags` | 复合条件 | 需要全部满足 |
| 查询任一 Tag | `HasAnyMatchingGameplayTags` | 任一状态阻止 | 常用于阻止释放 |
| 添加本地 Loose Tag | `AddLooseGameplayTag` | 临时本地状态 | 源码注释：不复制 |
| 移除 Loose Tag | `RemoveLooseGameplayTag` | 清理状态 | 网络同步需自己处理 |
| 一次性 Cue | `ExecuteGameplayCue` | 命中爆炸、治疗瞬间 | 瞬时表现 |
| 添加持续 Cue | `AddGameplayCue` | 燃烧、护盾光环 | 需要移除 |
| 移除持续 Cue | `RemoveGameplayCue` | Buff 结束 | 与 Add 配对 |

---

## 4. GA 生命周期速查

| 函数 | 什么时候发生 | 你通常做什么 | 注意点 |
|---|---|---|---|
| `CanActivateAbility` | ASC 激活前检查 | 额外释放条件，如必须有目标 | 默认已检查 Cost/Cooldown/Tag 等，不要无脑重写 |
| `ShouldAbilityRespondToEvent` | GameplayEvent 触发 Ability 时 | 判断是否响应事件 | 仅事件触发相关 |
| `CallActivateAbility` | ASC 调进 Ability | 内部先 `PreActivate` 再 `ActivateAbility` | 通常不直接重写 |
| `PreActivate` | Ability 正式激活前 | 内部设置状态、Tag、ActiveCount | 通常不重写 |
| `ActivateAbility` | 技能主体开始 | 播蒙太奇、生成投射物、启动 AbilityTask、调用 Commit | 最常重写 |
| `CommitAbility` | 技能真正“成交” | 检查并应用 Cost/Cooldown | 必须由 Ability 主动调用 |
| `CommitCheck` | Commit 内部检查 | 最后检查 Cost/Cooldown | 激活后资源可能变化，所以要重查 |
| `CommitExecute` | Commit 成功后 | ApplyCooldown + ApplyCost | 默认先 Cooldown 后 Cost |
| `EndAbility` | 技能结束 | 清理任务、状态、结束逻辑 | Activate 之后一定要能结束 |
| `CancelAbility` | 技能被取消 | 打断施法、清理表现 | 常用于眩晕/死亡/沉默 |

常见 Ability 写法：

```cpp
void UGA_Fireball::ActivateAbility(
    const FGameplayAbilitySpecHandle Handle,
    const FGameplayAbilityActorInfo* ActorInfo,
    const FGameplayAbilityActivationInfo ActivationInfo,
    const FGameplayEventData* TriggerEventData)
{
    if (!CommitAbility(Handle, ActorInfo, ActivationInfo))
    {
        EndAbility(Handle, ActorInfo, ActivationInfo, true, true);
        return;
    }

    // 技能主体：生成投射物、播放蒙太奇、等待目标等

    EndAbility(Handle, ActorInfo, ActivationInfo, false, false);
}
```

---

## 5. GE 类型速查

| GE 类型 | DurationPolicy | 常见用途 | 是否进入 ActiveGE | 对 Attribute 的影响 |
|---|---|---|---|---|
| Instant | `Instant` | 伤害、治疗、扣蓝、一次性加经验 | 通常不长期存在 | 通常执行后修改 BaseValue |
| Duration | `HasDuration` | 限时 Buff、Debuff、Cooldown、Stun | 会进入 ActiveGE | 通常通过 Aggregator 影响 CurrentValue；有 Period 时周期执行 |
| Infinite | `Infinite` | 被动光环、装备属性、永久状态 | 会进入 ActiveGE | 通常通过 Aggregator 影响 CurrentValue |
| Periodic | `HasDuration`/`Infinite` + `Period` | 中毒、燃烧、周期回血 | 会进入 ActiveGE 并按周期执行 | 周期触发时可像 Instant 一样执行修改 BaseValue |

---

## 6. Modifier 速查

| 配置项 | 说明 | 常见例子 |
|---|---|---|
| Attribute | 要修改哪个属性 | `Health`、`Mana`、`Damage`、`MoveSpeed` |
| ModifierOp | 怎么改 | AddBase、AddFinal、Override 等 |
| Magnitude | 改多少 | 固定值、曲线、属性计算、SetByCaller |
| Source Tags | 来源需要什么 Tag | 施法者有 `Buff.PowerUp` 时增强 |
| Target Tags | 目标需要什么 Tag | 只对 `State.Burning` 目标生效 |

Magnitude 来源：

| 来源 | 适合场景 |
|---|---|
| `ScalableFloat` | 固定值或随等级曲线变化 |
| `AttributeBased` | 根据攻击力、法强、防御等计算 |
| `CustomCalculationClass` / MMC | 一个 Modifier 的自定义计算 |
| `SetByCaller` | 运行时动态传值，如本次火球伤害 |
| `ExecutionCalculation` | 多属性捕获、护甲、暴击、抗性等复杂结算 |

---

## 7. AttributeSet 回调速查

| 回调 | 什么时候触发 | 适合做什么 | 不适合做什么 |
|---|---|---|---|
| `PreAttributeChange` | CurrentValue 改之前 | Clamp 当前值，如 Health ≤ MaxHealth | 复杂战斗事件 |
| `PostAttributeChange` | CurrentValue 改之后 | 轻量通知、派生数据 | 大量 gameplay 逻辑 |
| `PreAttributeBaseChange` | BaseValue 改之前 | Clamp 基础值 | 复杂事件回调 |
| `PostAttributeBaseChange` | BaseValue 改之后 | 记录基础值变化 | 再次修改入参 |
| `PreGameplayEffectExecute` | Instant/Periodic GE 执行前 | 最终校验、减免、拒绝本次执行 | 普通 Duration Buff 回调 |
| `PostGameplayEffectExecute` | Instant/Periodic GE 执行后 | Damage -> Health、清空 Damage、死亡前轻量处理 | 大型表现/销毁流程 |
| `OnAttributeAggregatorCreated` | 创建属性 Aggregator 时 | 配置 Aggregator 评估元数据 | Clamp 具体数值 |

Damage 常见写法：

```cpp
void UMyAttributeSet::PostGameplayEffectExecute(const FGameplayEffectModCallbackData& Data)
{
    if (Data.EvaluatedData.Attribute == GetDamageAttribute())
    {
        const float LocalDamage = GetDamage();
        SetDamage(0.f);

        if (LocalDamage > 0.f)
        {
            SetHealth(FMath::Clamp(GetHealth() - LocalDamage, 0.f, GetMaxHealth()));
        }
    }
}
```

---

## 8. Cost / Cooldown 速查

| 类型 | 本质 | 推荐配置 | 检查时机 | 应用时机 |
|---|---|---|---|---|
| Cost | Instant GE | `Mana AddBase -20` / `Stamina AddBase -30` | `CanActivateAbility` 和 `CommitCheck` | `CommitAbility -> ApplyCost` |
| Cooldown | Duration GE + Granted Tag | `Duration=3` + `GrantedTags=Cooldown.Fireball` | `CanActivateAbility` 和 `CommitCheck` | `CommitAbility -> ApplyCooldown` |

记忆：

```text
Cost 看属性够不够
Cooldown 看 Tag 有没有
CommitAbility 真正扣资源和加冷却
```

---

## 9. GameplayCue 速查

| 需求 | 推荐 Cue 调用 | 例子 |
|---|---|---|
| 一次性爆炸、命中、治疗闪光 | `ExecuteGameplayCue` | `GameplayCue.Fireball.Impact` |
| 持续燃烧、护盾、加速光环 | `AddGameplayCue` | `GameplayCue.State.Burning` |
| 移除持续表现 | `RemoveGameplayCue` | Buff 结束移除光环 |
| GE 自带表现 | GE 的 `GameplayCues` 数组 | 伤害 GE 自动触发命中 Cue |

Cue 只做表现，不建议在 Cue 里处理核心伤害/属性逻辑。

---

## 10. 常见玩法模板速查

### 10.1 火球术

```text
GA_Fireball
    ActivateAbility
    CommitAbility
        GE_Cost_Fireball: Mana -20
        GE_Cooldown_Fireball: Granted Cooldown.Fireball 3s
    Spawn FireballProjectile
    Projectile Hit
        GE_Damage_Fireball: Damage SetByCaller
        AttributeSet: Damage -> Health
        GameplayCue.Fireball.Impact
    EndAbility
```

### 10.2 治疗术

```text
GA_Heal
    CommitAbility
    MakeOutgoingSpec(GE_Heal)
    SetByCaller(Data.Heal, HealAmount) 可选
    ApplyGameplayEffectSpecToTarget
    AttributeSet: Health Clamp 到 MaxHealth
    EndAbility
```

### 10.3 加速 Buff

```text
GE_Buff_Speed
    DurationPolicy = HasDuration
    Duration = 5
    Modifier: MoveSpeed AddFinal +300
    Optional GrantedTags: Buff.Speed
    Optional GameplayCue: GameplayCue.Buff.Speed
```

### 10.4 中毒 DoT

```text
GE_Debuff_Poison
    DurationPolicy = HasDuration
    Duration = 6
    Period = 1
    Modifier/Execution: Damage +10 per tick
    GrantedTags: Debuff.Poison
    GameplayCue: GameplayCue.Debuff.Poison
```

### 10.5 眩晕

```text
GE_Debuff_Stun
    DurationPolicy = HasDuration
    Duration = 2
    GrantedTags: State.Stun

GA 里可用 ActivationBlockedTags / CancelAbilitiesWithTag 阻止或取消技能。
```

---

## 11. 常见问题速查

| 问题 | 优先检查 |
|---|---|
| Ability 放不出来 | ASC 是否初始化 `InitAbilityActorInfo`；是否 GiveAbility；是否有 Owner/Avatar；是否被 Tag 阻止；Cost/Cooldown 是否失败 |
| Commit 失败 | Mana/Stamina 是否足够；Cooldown Tag 是否存在；SetByCaller 是否设置；Cost GE Modifier 是否有效 |
| 伤害不生效 | 目标是否有 ASC；目标是否有 AttributeSet；GE Modifier Attribute 是否正确；SetByCaller Tag 是否一致 |
| Buff 不回退 | GE 是否 Duration/Infinite；是否正确移除 ActiveGE；Modifier 是否注册到 Aggregator |
| Cooldown 不生效 | Cooldown GE 是否 HasDuration；是否配置 Granted Cooldown Tag；Ability 的 CooldownTags 是否能取到 |
| UI 不更新 | Attribute 是否 ReplicatedUsing；是否调用 `GAMEPLAYATTRIBUTE_REPNOTIFY`；UI 是否绑定 ASC delegate |
| Cue 不播放 | GameplayCue Tag 是否注册；Cue Notify 路径是否被扫描；GE/Cue 调用是否在正确 ASC 上 |
| 客户端预测异常 | NetExecutionPolicy、PredictionKey、周期 GE 客户端预测限制、ASC 复制模式 |

---

## 12. 学习顺序建议

```text
1. ASC：GiveAbility / TryActivateAbility / ApplyGameplayEffect
2. GA：ActivateAbility / CommitAbility / EndAbility
3. GE：Instant / Duration / Infinite / Modifier / SetByCaller
4. AttributeSet：Health / Mana / Damage / 回调 / 复制
5. GameplayCue：Execute / Add / Remove
6. AbilityTask：WaitInput / WaitGameplayEvent / WaitTargetData / WaitMontage
7. Prediction / Replication：LocalPredicted、PredictionKey、TargetData 复制
```

先不要背所有 API。每做一个技能，只记它那条链路需要的函数。

---

## 13. TargetData / TargetActor 速查

一句话记忆：

```text
TargetActor = 选目标的临时 Actor
TargetData = 选出来的结果
WaitTargetData = Ability 里等待目标结果并处理复制的 AbilityTask
```

| 需求 | 推荐入口 | 注意点 |
|---|---|---|
| 等玩家瞄准 / 点击地面 / 确认目标 | `UAbilityTask_WaitTargetData` | 会创建或接收 TargetActor，输出 `ValidData` / `Cancelled`；源码路径：`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Public/Abilities/Tasks/AbilityTask_WaitTargetData.h:16`、`:28`、`:31` |
| 只做瞄准预览 | `UAbilityTask_VisualizeTargeting` | 只广播 `TimeElapsed`，不返回 TargetData；源码路径：`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Public/Abilities/Tasks/AbilityTask_VisualizeTargeting.h:21`、`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Private/Abilities/Tasks/AbilityTask_VisualizeTargeting.cpp:168` |
| 已经有 Actor 目标 | `AbilityTargetDataFromActor` / `AbilityTargetDataFromActorArray` | 对应 `FGameplayAbilityTargetData_ActorArray`；源码路径：`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Private/AbilitySystemBlueprintLibrary.cpp:393`、`:401` |
| 已经有 HitResult | `AbilityTargetDataFromHitResult` | 对应 `FGameplayAbilityTargetData_SingleTargetHit`，可把 HitResult 写入 EffectContext；源码路径：`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Private/AbilitySystemBlueprintLibrary.cpp:515`、`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Private/GameplayAbilityTargetTypes.cpp:65` |
| 只有位置 / 方向 | `AbilityTargetDataFromLocations` | 对应 `FGameplayAbilityTargetData_LocationInfo`；源码路径：`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Private/AbilitySystemBlueprintLibrary.cpp:380` |
| 直线瞄准 | `AGameplayAbilityTargetActor_SingleLineTrace` | 起点来自 `StartLocation`，终点来自 PlayerController aim；源码路径：`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Private/Abilities/GameplayAbilityTargetActor_SingleLineTrace.cpp:30`、`:32` |
| 地面 AOE / 放置点 | `AGameplayAbilityTargetActor_GroundTrace` | 确认条件由 `bLastTraceWasGood` 决定；源码路径：`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Private/Abilities/GameplayAbilityTargetActor_GroundTrace.cpp:143`、`:202` |
| 范围内 Actor | `AGameplayAbilityTargetActor_Radius` | 默认 `ShouldProduceTargetDataOnServer = true`，更适合服务端生成范围结果；源码路径：`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Private/Abilities/GameplayAbilityTargetActor_Radius.cpp:20`、`:25` |

TargetData 类型选择：

| 类型 | 适合什么 | 源码路径 |
|---|---|---|
| `FGameplayAbilityTargetData_ActorArray` | 一个或多个目标 Actor | `Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Public/Abilities/GameplayAbilityTargetTypes.h:448` |
| `FGameplayAbilityTargetData_SingleTargetHit` | 命中点、命中 Actor、物理材质、TraceStart/Location | `Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Public/Abilities/GameplayAbilityTargetTypes.h:550` |
| `FGameplayAbilityTargetData_LocationInfo` | 纯 source / target 位置或 transform | `Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Public/Abilities/GameplayAbilityTargetTypes.h:380` |

TargetData 网络必查：

| 问题 | 优先检查 |
|---|---|
| 客户端数据没到服务端 | 是否走到 `CallServerSetReplicatedTargetData`；服务端是否绑定 `AbilityTargetDataSetDelegate`；源码路径：`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Private/Abilities/Tasks/AbilityTask_WaitTargetData.cpp:288`、`:211` |
| 服务端收到但重复触发 | 是否调用 `ConsumeClientReplicatedTargetData` 清缓存；源码路径：`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Private/AbilitySystemComponent_Abilities.cpp:3838`、`:3843` |
| PredictionKey 不匹配 | `AbilityTargetDataMap` key 是 AbilitySpecHandle + PredictionKey；源码路径：`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Public/Abilities/GameplayAbilityTypes.h:500`、`:516` |
| 自定义 TargetData 复制失败 | 派生 struct 必须有 native NetSerialize，否则 handle NetSerialize 会 Fatal；源码路径：`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Private/GameplayAbilityTargetTypes.cpp:230`、`:240` |
| 客户端 trace 可作弊 | 服务端 RPC validate 只检查 TargetData item 有效，项目级距离/视线/阵营要自己校验；源码路径：`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Private/AbilitySystemComponent_Abilities.cpp:3971`、`:3974` |

TargetData 应用 GE / Cue 推荐链：

```text
WaitTargetData.ValidData
    ↓
检查 TargetDataHasActor / HitResult / Origin / EndPoint
    ↓
MakeOutgoingSpec + SetByCaller
    ↓
ApplyGameplayEffectSpecToTarget 或 TargetData::ApplyGameplayEffectSpec
    ↓
AddTargetDataToContext 写入 HitResult / Origin
    ↓
GameplayCueParameters 通过 EffectContext 获得命中上下文
```

源码路径：`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Private/AbilitySystemBlueprintLibrary.cpp:592`、`:605`、`:636`、`:678`、`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Private/GameplayAbilityTargetTypes.cpp:21`、`:45`、`:67`、`:72`、`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Private/AbilitySystemGlobals.cpp:420`、`:425`。

---

## 14. GameplayTag / ResponseTable 速查

一句话记忆：

```text
ASC TagCount = 状态账本
Ability Tags = 技能分类与激活条件
GE Granted Tags = 效果授予的状态
ResponseTable = TagCount 变化自动转成响应 GE
```

| 需求 | 推荐入口 | 注意点 |
|---|---|---|
| 判断角色是否有状态 | `ASC::HasMatchingGameplayTag` / `GetGameplayTagCount` | 客户端读取可能受网络延迟影响；源码路径：`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Public/AbilitySystemComponent.h:576`、`:686`、`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Private/AbilitySystemComponent.cpp:690` |
| 监听状态变化 | `RegisterGameplayTagEvent` / 蓝图 wrapper / WaitGameplayTag task | `NewOrRemoved` 只看 0/非0，`AnyCountChange` 看任意 count 变化；源码路径：`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Public/AbilitySystemComponent.h:743`、`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Public/AbilitySystemBlueprintLibrary.h:99`、`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Public/Abilities/Tasks/AbilityTask_WaitGameplayTagBase.h:15` |
| Ability 激活期间给自己加状态 | `ActivationOwnedTags` | `PreActivate` 添加，`EndAbility` 移除；源码路径：`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Public/Abilities/GameplayAbility.h:765`、`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Private/Abilities/GameplayAbility.cpp:963`、`:837` |
| Ability 释放前要求/禁止状态 | `ActivationRequiredTags` / `ActivationBlockedTags` | 在 `DoesAbilitySatisfyTagRequirements` 中检查 ASC owned tags；源码路径：`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Public/Abilities/GameplayAbility.h:769`、`:773`、`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Private/Abilities/GameplayAbility.cpp:386` |
| Ability 互斥/打断 | `CancelAbilitiesWithTag` / `BlockAbilitiesWithTag` | 激活时 cancel/block，结束时 unblock；源码路径：`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Public/Abilities/GameplayAbility.h:757`、`:761`、`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Private/AbilitySystemComponent_Abilities.cpp:1410` |
| Buff/Debuff/Cooldown 授予状态 | GE Granted Tags | UE5.6 建议通过 `UTargetTagsGameplayEffectComponent` 配置；源码路径：`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Public/GameplayEffectComponents/TargetTagsGameplayEffectComponent.h:13`、`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Public/GameplayEffect.h:2147` |
| 不想创建 GE 的手动状态 | Loose Tags | 默认不复制；源码路径：`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Public/AbilitySystemComponent.h:649` |
| 手动状态需要复制 | Replicated Loose Tags | 会覆盖 simulated proxies 上本地 tag count；源码路径：`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Public/AbilitySystemComponent.h:690`、`:694` |
| Minimal 模式下需要同步 GE granted tags | `MinimalReplicationTags` | 由 GE/Ability 内部维护，通常不手动操作；源码路径：`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Public/AbilitySystemComponent.h:719`、`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Private/GameplayEffect.cpp:4383` |
| Tag count 自动驱动 GE 响应 | `UGameplayTagReponseTable` | 注意源码类名拼写是 `Reponse`；源码路径：`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Public/GameplayTagResponseTable.h:62` |
| 表现事件 | GameplayCueTag | 开发实践推断：`GameplayCue.*` 只做表现路由，不要当状态 tag；源码依据：`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Public/GameplayCueSet.h:18`、`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Public/AbilitySystemComponent.h:597` |

Tag 语义选择：

| Tag 类型 | 推荐命名（开发实践推断） | 用途 |
|---|---|---|
| 行为分类 | `Ability.Attack.Melee` | AbilityTags，供 cancel/block/query |
| 角色状态 | `State.Stunned` / `State.Casting` | ASC owned tags、ActivationOwnedTags、GE Granted Tags |
| 效果状态 | `Status.Haste` / `Status.Slow` | GE Granted Tags 或 ResponseTable 触发 tag |
| 冷却 | `Cooldown.Fireball` | Cooldown GE Granted Tags |
| 表现 | `GameplayCue.Fireball.Impact` | Cue 路由 |

ResponseTable 必查：

| 问题 | 优先检查 |
|---|---|
| 表没生效 | `GameplayTagResponseTableName` 是否配置，ASC 是否调用过 `InitAbilityActorInfo`；源码路径：`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Public/GameplayAbilitiesDeveloperSettings.h:104`、`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Private/AbilitySystemComponent_Abilities.cpp:186` |
| 响应强度不对 | Positive/Negative 是否抵消，`SoftCountCap` 是否截断，ResponseTable 用的是 `GetAggregatedStackCount`；源码路径：`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Private/GameplayTagResponseTable.cpp:111`、`:112`、`:142`、`:143` |
| 响应 GE 没移除 | TotalCount 是否归零或反向；handles 是否有效；源码路径：`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Private/GameplayTagResponseTable.cpp:120`、`:132`、`:152` |
| 响应循环 | 响应 GE 是否又授予触发 tag；开发实践推断，源码依据：`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Private/GameplayTagResponseTable.cpp:66`、`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Private/GameplayEffect.cpp:4373` |
---

## 15. GameplayEffectComponent / GEComponents 速查

一句话记忆：

```text
GEComponents = UE5.6 中 GameplayEffect 的模块化配置层
它们改的是 GE 行为配置，不是每次应用的运行时状态对象
```

| 需求 | 优先看哪个组件 | 注意点 |
|---|---|---|
| 给目标授予 Buff / Debuff / Cooldown 状态 tag | `UTargetTagsGameplayEffectComponent` | 写入 `CachedGrantedTags`，ActiveGE 添加时进入目标 ASC tag count；源码路径：`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Public/GameplayEffectComponents/TargetTagsGameplayEffectComponent.h:13`、`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Private/GameplayEffect.cpp:4373` |
| 给 GE 自身打查询/分类 tag | `UAssetTagsGameplayEffectComponent` | Asset Tags 不授予目标；源码路径：`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Public/GameplayEffectComponents/AssetTagsGameplayEffectComponent.h:13`、`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Public/GameplayEffect.h:2144` |
| 应用前要求目标有/没有某些 tag，或应用后按 tag inhibit/remove | `UTargetTagRequirementsGameplayEffectComponent` | Application 阻止应用，Ongoing inhibit，Removal remove；源码路径：`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Public/GameplayEffectComponents/TargetTagRequirementsGameplayEffectComponent.h:15`、`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Private/GameplayEffectComponents/TargetTagRequirementsGameplayEffectComponent.cpp:24`、`:143` |
| 某个 GE 生效期间阻塞一类 Ability | `UBlockAbilityTagsGameplayEffectComponent` | ActiveGE 添加时 block，移除时 unblock；源码路径：`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Public/GameplayEffectComponents/BlockAbilityTagsGameplayEffectComponent.h:13`、`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Private/GameplayEffect.cpp:4377`、`:4674` |
| 让一个状态免疫某类 incoming GE | `UImmunityGameplayEffectComponent` | ActiveGE 添加后注册 ASC application query；源码路径：`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Public/GameplayEffectComponents/ImmunityGameplayEffectComponent.h:16`、`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Private/GameplayEffectComponents/ImmunityGameplayEffectComponent.cpp:24`、`:66` |
| 新 GE 应用成功后清掉旧 GE | `URemoveOtherGameplayEffectComponent` | authority-only，按 `FGameplayEffectQuery` 移除；源码路径：`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Public/GameplayEffectComponents/RemoveOtherGameplayEffectComponent.h:15`、`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Private/GameplayEffectComponents/RemoveOtherGameplayEffectComponent.cpp:18`、`:39` |
| GE 应用或结束时连带应用其他 GE | `UAdditionalEffectsGameplayEffectComponent` | 应用时 conditional，结束时 OnComplete；小心递归；源码路径：`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Public/GameplayEffectComponents/AdditionalEffectsGameplayEffectComponent.h:13`、`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Private/GameplayEffectComponents/AdditionalEffectsGameplayEffectComponent.cpp:43`、`:111` |
| ActiveGE 存在期间临时授予 Ability | `UAbilitiesGameplayEffectComponent` | authority-only，依赖 ActiveGE 生命周期和 RemovalPolicy；源码路径：`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Public/GameplayEffectComponents/AbilitiesGameplayEffectComponent.h:38`、`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Private/GameplayEffectComponents/AbilitiesGameplayEffectComponent.cpp:44`、`:124` |
| GE 有概率应用 | `UChanceToApplyGameplayEffectComponent` | 概率来自 `FScalableFloat`，按 GE level 计算；预测场景小心客户端/服务器不一致；源码路径：`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Public/GameplayEffectComponents/ChanceToApplyGameplayEffectComponent.h:14`、`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Private/GameplayEffectComponents/ChanceToApplyGameplayEffectComponent.cpp:24`、`:27` |
| GE 有自定义应用条件 | `UCustomCanApplyGameplayEffectComponent` | 调 `UGameplayEffectCustomApplicationRequirement::CanApplyGameplayEffect`，建议保持无副作用；源码路径：`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Public/GameplayEffectComponents/CustomCanApplyGameplayEffectComponent.h:14`、`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Private/GameplayEffectComponents/CustomCanApplyGameplayEffectComponent.cpp:18` |

旧项目迁移必查：

| 旧字段/旧概念 | UE5.6 新入口 |
|---|---|
| `InheritableGameplayEffectTags` | `UAssetTagsGameplayEffectComponent`；源码路径：`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Private/GameplayEffect.cpp:545`、`:554` |
| `InheritableOwnedTagsContainer` | `UTargetTagsGameplayEffectComponent`；源码路径：`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Private/GameplayEffect.cpp:627`、`:636` |
| `Application/Ongoing/RemovalTagRequirements` | `UTargetTagRequirementsGameplayEffectComponent`；源码路径：`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Private/GameplayEffect.cpp:757`、`:783` |
| `GrantedApplicationImmunity*` | `UImmunityGameplayEffectComponent`；源码路径：`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Private/GameplayEffect.cpp:648`、`:703` |
| `RemoveGameplayEffectsWithTags` / `RemoveGameplayEffectQuery` | `URemoveOtherGameplayEffectComponent`；源码路径：`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Private/GameplayEffect.cpp:716`、`:744` |
| `ConditionalGameplayEffects` / Expiration effects | `UAdditionalEffectsGameplayEffectComponent`；源码路径：`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Private/GameplayEffect.cpp:566`、`:592` |
| `GrantedAbilities` | `UAbilitiesGameplayEffectComponent`；源码路径：`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Private/GameplayEffect.cpp:464`、`:491` |

配置归属判断：

```text
GEComponent：声明式改变 GE 应用/激活/移除行为
Ability：输入、流程、目标选择、动画、异步等待
ExecutionCalculation：复杂多属性结算
AttributeSet：属性承载和属性回调
项目系统：权限、阵营、复杂规则、跨系统状态机
```

开发实践推断：GEComponent 是 GE 资产单例子对象，不要在里面保存 per-target/per-application 状态；源码依据：`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Public/GameplayEffectComponent.h:23`、`:24`。
---

## 16. GAS 数值计算速查

一句话记忆：

```text
ModifierMagnitude = 算一个数
MMC = 自定义算一个数
ExecutionCalculation = 复杂结算并输出一个或多个属性修改
Attribute Capture = 把 Source/Target 属性带进计算
Aggregator = 持续效果的当前值计算器
```

| 玩法需求 | 推荐入口 | 注意点 |
|---|---|---|
| 固定数值 buff/debuff | ScalableFloat Modifier | `FScalableFloat` 按 `Value * Curve[Level]` 求值；源码路径：`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Public/ScalableFloat.h:12`、`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Private/GameplayEffect.cpp:1147` |
| 随技能/GE level 成长 | ScalableFloat 或 AttributeBased 的 coefficient/pre/post | GE spec level 会传给 scalable float；源码路径：`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Private/GameplayEffect.cpp:1009`、`:1076` |
| 运行时传入数值 | SetByCaller | 先写 spec 的 Name/Tag magnitude，再应用 GE；缺失时可 log error 并返回默认值；源码路径：`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Private/GameplayEffect.cpp:2200`、`:2208`、`:2216`、`:2238` |
| 依赖一个属性算一个 modifier | AttributeBased Modifier | 需要正确配置 source/target capture 和 snapshot；源码路径：`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Private/GameplayEffect.cpp:966`、`:983` |
| 依赖多个属性但只返回一个数 | MMC | MMC `CalculateBaseMagnitude` 返回单个 float；源码路径：`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Public/GameplayModMagnitudeCalculation.h:29` |
| 依赖多个属性并输出多个属性修改 | ExecutionCalculation | 输出 `TArray<FGameplayModifierEvaluatedData>`；源码路径：`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Public/GameplayEffectExecutionCalculation.h:242`、`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Private/GameplayEffect.cpp:3146` |
| Damage -> Health | ExecutionCalculation + AttributeSet 回调 | 开发实践推断：复杂伤害结算放 Execution，最终属性变化进入 `Pre/PostGameplayEffectExecute`；源码路径：`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Private/GameplayEffect.cpp:3136`、`:3907`、`:3946` |
| 治疗 | 简单治疗用 Modifier，复杂治疗用 Execution | 两者最终都可进入 `InternalExecuteMod`；源码路径：`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Private/GameplayEffect.cpp:3114`、`:3155` |
| DOT / HOT | Periodic GE execute 路径 | Periodic/Instant 执行 modifiers 会直接改 base，并触发 AttributeSet GE execute 回调；源码路径：`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Private/GameplayEffect.cpp:3065`、`:3907` |
| 冷却缩减、移速、攻速等持续属性 | Duration/Infinite GE + Aggregator Modifier | ActiveGE modifier 写入 attribute aggregator，dirty 后重算 current value；源码路径：`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Private/GameplayEffect.cpp:4347`、`:4350`、`:3263` |
| 暴击、随机、时间相关 | 服务端决定后 SetByCaller，或服务端权威 Execution | 开发实践推断：预测路径会客户端先执行，非确定性逻辑容易与服务器不一致；源码路径：`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Private/GameplayEffect.cpp:2924`、`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Private/AbilitySystemComponent.cpp:862` |
| Snapshot 攻击力 | Source Attribute Capture + Snapshot | Snapshot 会复制当前 aggregator；源码路径：`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Private/GameplayEffect.cpp:3756` |
| 实时防御力 | Target Attribute Capture + Non-Snapshot | Non-Snapshot 保存 aggregator 引用并可注册依赖重算；源码路径：`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Private/GameplayEffect.cpp:3760`、`:4196` |

排查清单：

| 问题 | 优先检查 |
|---|---|
| MMC 算出来是 0 | 是否重写 `CalculateBaseMagnitude`，是否声明/捕获了需要的 attributes；源码路径：`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Private/GameplayModMagnitudeCalculation.cpp:14`、`:30` |
| SetByCaller 为 0 或报错 | Spec 是否调用 `SetSetByCallerMagnitude`，key 是 Name 还是 GameplayTag；源码路径：`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Private/GameplayEffect.cpp:2200`、`:2208`、`:2216` |
| AttributeBased 不更新 | Capture 是否是 Snapshot；Non-Snapshot 才会 linked aggregator callback；源码路径：`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Private/GameplayEffect.cpp:2389`、`:2411` |
| Duration buff 数值叠加不对 | 查 `EGameplayModOp` 公式和 evaluation channel；源码路径：`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Public/GameplayEffectTypes.h:116`、`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Private/GameplayEffectAggregator.cpp:76` |
| Execution 没触发 Cue 或 conditional GE | 是否调用 `MarkGameplayCuesHandledManually` / `MarkConditionalGameplayEffectsToTrigger`；源码路径：`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Private/GameplayEffectExecutionCalculation.cpp:346`、`:351` |
| 预测下数值闪回 | predicted GE / predictive mods 会先本地执行再由服务器纠正；源码路径：`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Private/AbilitySystemComponent.cpp:862`、`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Private/GameplayEffect.cpp:4265` |

---

## 17. GAS 测试与官方示例速查

详细专题见 `tests-practices.md`。这些测试用于验证引擎机制，不是业务模板。

| 需求 | 优先看 | 结论 |
|---|---|---|
| 最小测试 Pawn | `Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Public/AbilitySystemTestPawn.h:17`、`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Private/AbilitySystemTestPawn.cpp:14`、`:25` | TestPawn 持有 replicated ASC，实现 `IAbilitySystemInterface`，并在 `PostInitializeComponents` 调 `InitStats`。 |
| 最小测试 AttributeSet | `Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Public/AbilitySystemTestAttributeSet.h:13`、`:30`、`:36`、`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Private/AbilitySystemTestAttributeSet.cpp:92` | `Mana` 用 `FGameplayAttributeData` 验证 Base/Current；`Damage` 是非持久 meta attribute 示例。 |
| GE 应用机制 | `Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Private/Tests/GameplayEffectTests.cpp:126`、`:166`、`:281`、`:330`、`:357` | 覆盖 Instant、Infinite、Periodic、StackLimit、SetByCaller duration。 |
| Attribute / Aggregator | `Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Private/Tests/GameplayEffectTests.cpp:194`、`:222`、`:240`、`:259` | 验证 modifier op 聚合、Mana CurrentValue 改变、BaseValue 不变。 |
| Ability 激活 | `Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Private/Tests/AbilitySystemComponentTests.cpp:132`、`:157`、`:166`、`:172`、`:196` | 覆盖 GiveAbility、TryActivateAbility、CancelAbilityHandle、activation inhibited failure。 |
| GameplayCue | `Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Private/Tests/GameplayEffectTests.cpp:417`、`:537`、`:679`、`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Private/Tests/GameplayCueTests.h:16` | 覆盖 direct API、GE API、replication mode 下 cue 路径；资产扫描/异步加载未覆盖。 |
| GameplayTag count | `Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Private/Tests/GameplayTagCountContainerTests.cpp:16`、`:21`、`:38`、`:55` | 验证显式 tag count 与父 tag count 语义。 |
| PredictionKey | `Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Private/Tests/PredictionKeyTests.cpp:15`、`:62`、`:83`、`:88`、`:432`、`:476` | 验证 dependent key、catch-up/reject delegate、`FScopedPredictionWindow`。 |
| 当前未覆盖 | `Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Private/Tests` | TargetData、AbilityTask 生命周期、GEComponents、ExecutionCalculation/MMC/Attribute Capture、真实网络复制、Iris、ResponseTable 均未在当前 tests 中完整覆盖。 |

开发实践推断：写项目侧自动化测试时，可以借鉴 `GameplayEffectTests.cpp` 的 transient GE 机制测试、`PredictionKeyTests.cpp` 的 prediction key wrapper、`GameplayTagCountContainerTests.cpp` 的 tag count 断言；不要照搬 `AbilitySystemTestAttributeSet` 的 `mutable`、禁用复制或 `#if 0` 战斗逻辑，源码路径：`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Public/AbilitySystemTestAttributeSet.h:15`、`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Private/AbilitySystemTestAttributeSet.cpp:31`、`:121`。
