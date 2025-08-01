---
aliases:
tags:
  - 虚幻引擎

title: Gameplay Task
date: 2025-08-01
author: jimmyzou
toc_number: true
decscription: 
---

GameplayTask是UE中用于封装可重用游戏功能的基础类，开发者可以通过继承UGameplayTask基类并重写成员函数来实现自定义任务逻辑。该系统主要应用于两个场景：[[GAS系统]]和[[行为树|行为树]]。

---
### **1. 核心概念**

GameplayTask机制的核心特点包括：

-  **基于资源的互斥机制**：两个资源需求交集非空的任务不能同时执行，它们之间形成互斥关系

- **基于优先级的打断机制**：不同优先级的互斥任务中，高优先级任务可以暂停低优先级任务，并在资源释放后恢复低优先级任务

- **异步执行**：任务可以在后台异步执行，不阻塞主线程

GameplayTask的核心类包括：

-  `UGameplayTask`：任务本身，包含业务逻辑实现

-  `UK2Node_LatentGameplayTaskCall`：GameplayTask的蓝图节点

-  `IGameplayTaskOwnerInterface`：GameplayTask所有者的抽象

-  `UGameplayTaskComponent`：管理Actor拥有的所有GameplayTask实例，处理互斥和优先级逻辑

-  `UGameplayTaskResource`：GameplayTask使用的“资源”概念的基类

---

### **2. 关键实现**