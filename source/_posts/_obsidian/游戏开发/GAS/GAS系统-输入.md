---
aliases:
  - GAS Input
tags:
  - 数据结构
  - GAS
  - 
title: GAS系统-输入
date: 2025-08-01
author: jimmyzou
toc_number: true
decscription:
---

GAS提供了一个可扩展的输入绑定机制，支持了包括输入绑定、输入同步、Confirm/Cancel等机制。默认实现中只提供了基础框架，并没有默认的接入流程，所以需要一定的定制开发才能完全使用。

---

### **1. 关联GA与InputID**

通过在`GiveAbility` 时，给 `FGameplayAbilitySpec` 指定 `InputID` 来实现。在UE的实例代码中，`UGameplayAbilitySet` 中包含了这个信息，其中的 `FGameplayAbilityBindInfo` 中包含了一个GA类与一个InputID。

```cpp
USTRUCT()  
struct FGameplayAbilityBindInfo  
{ 
    GENERATED_USTRUCT_BODY()  
     
    UPROPERTY(EditAnywhere, Category = BindInfo)  
    TEnumAsByte<EGameplayAbilityInputBinds::Type> Command = EGameplayAbilityInputBinds::Ability1;
      
    UPROPERTY(EditAnywhere, Category = BindInfo)  
    TSubclassOf<UGameplayAbility> GameplayAbilityClass;  
};
```

在 `GiveAbilities` 函数中，InputID会作为构造函数参数保存在 `FGameplayAbilitySpec` 中，这里就完成了GA与InputID的绑定。

```cpp
void UGameplayAbilitySet::GiveAbilities(UAbilitySystemComponent* ASC) const  
{  
    for (const FGameplayAbilityBindInfo& BindInfo : Abilities)  
    {
        if (BindInfo.GameplayAbilityClass)  
        {
            ASC->GiveAbility(FGameplayAbilitySpec(
                BindInfo.GameplayAbilityClass,
                1,
                (int32)BindInfo.Command)
            );  
        }    
    }
}
```

这里InputID是int类型是为了方便策划在蓝图中定义自己的枚举来标识输入，蓝图枚举可以直接作为int参数传递到C++中。

在实际生产实现中，我们可以自己实现一套关联机制，只要最终我们能够将一个InputID保存在关联的 `FGameplayAbilitySpec` 中即可。

---

### **2. `AbilitySystemComponent` 与输入组件绑定**

官方示例的实现在 `UAbilitySystemComponent::BindAbilityActivationToInputComponent` 中。这一步是让 `UAbilitySystemComponent` 监听所有依赖的输入事件，并以InputID调用 `UAbilitySystemComponent::AbilityLocalInputPressed` 与 `UAbilitySystemComponent::AbilityLocalInputReleased`。

比较特别的是 `GenericConfirmInputID` 与 `GenericCancelInputID`。这两个成员是用于需要二次确认的GA，比如释放时触发范围框选，需再次确认释放或者取消。这两个成员变量对应确认与取消的InputID。

同样，实际生产过程中我们可以自己重写这个机制。因为这个示例实现使用的是旧的输入系统，而现在大多使用[[增强输入系统]]。自己实现这个机制只需要保证：

- 在输入按下和抬起时，调用 `UAbilitySystemComponent::AbilityLocalInputPressed` 和 `UAbilitySystemComponent::AbilityLocalInputReleased` 并将InputID作为参数。

- Confirm 和 Cancel 按钮按下时，调用 `UAbilitySystemComponent::LocalInputConfirm` 和 `UAbilitySystemComponent::LocalInputCancel`。

- 或者设置 `GenericConfirmInputID` 与 `GenericCancelInputID`。这种方式和上面的绑定选一种即可。 

---

### **3. 输入触发逻辑**

收到按下输入后：

- 首先判断 `GenericConfirmInputID` 或 `GenericCancelInputID` 是否有设置回调，如果有则只触发 Confirm 或 Cancel。

- 否则遍历所有InputID相同的 `FGameplayAbilitySpec`，如果其已经在运行则调用 `InputPressed`，并且发送 ReplicatedEvent - `EAbilityGenericReplicatedEvent::InputPressed`。

- 如果GA没有在运行，则调用 `TryActivateAbility`

收到抬起输入则只会触发 `InputReleased`，并且发送 ReplicatedEvent - `EAbilityGenericReplicatedEvent::InputReleased`。

---

### **4. 输入事件的同步**

输入事件的同步分两个部分：

- **ServerSetInputPressed**，在Ability的 `bReplicateInputDirectly` 设置的情况下，客户端输入按下后会通过RPC调用服务器的 `AbilitySpecInputPressed`。

- 通过[[GAS系统-事件]]系统通知到服务器。注意这个同步并不是自动完成的，客户端收到输入事件后，只会调用客户端的事件回调，如果这个事件需要同步到服务器，则需要客户端自己在回调中通过 `ServerSetReplicatedEvent` 将事件同步到服务器。


