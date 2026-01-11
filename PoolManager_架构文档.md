# Pool Manager 项目架构文档

## 目录
1. [项目概述](#项目概述)
2. [架构设计](#架构设计)
3. [核心数据结构](#核心数据结构)
4. [核心流程与调用堆栈](#核心流程与调用堆栈)
5. [内存池实现机制](#内存池实现机制)
6. [工厂模式实现](#工厂模式实现)
7. [关键设计模式](#关键设计模式)

---

## 项目概述

Pool Manager 是一个 Unreal Engine 插件，用于实现对象池管理，避免频繁创建和销毁对象带来的性能开销。该系统采用**工厂模式**和**懒加载**策略，支持多种对象类型（UObject、Actor、UserWidget等）。

### 核心特性
- **按类组织**：每个对象类型拥有独立的池容器
- **懒加载**：池在首次使用时创建，而非预分配
- **异步生成**：支持优先级队列，每帧限制生成数量以避免卡顿
- **句柄系统**：使用 `FPoolObjectHandle` 进行间接引用，支持异步操作
- **工厂模式**：不同类型对象使用专门的工厂处理创建/销毁逻辑

---

## 架构设计

### 整体架构图

```
┌─────────────────────────────────────────────────────────────┐
│                    UPoolManagerSubsystem                      
│                    (WorldSubsystem)                          
│                                                              
│  ┌──────────────────────────────────────────────────────┐   
│  │  PoolsInternal: TArray<FPoolContainer>                  
│  │  ┌──────────────────────────────────────────────┐      
│  │  │  FPoolContainer (按类组织)                         
│  │  │  ├─ ObjectClass: UClass*                            
│  │  │  ├─ Factory: UPoolFactory_UObject*                  
│  │  │  └─ PoolObjects: TArray<FPoolObjectData>             
│  │  │      └─ FPoolObjectData                             
│  │  │          ├─ PoolObject: UObject*                   
│  │  │          ├─ Handle: FPoolObjectHandle               
│  │  │          └─ bIsActive: bool                          
│  │  └──────────────────────────────────────────────┘      
│  └──────────────────────────────────────────────────────┘   
│                                                               
│  ┌──────────────────────────────────────────────────────┐   
│  │  AllFactoriesInternal:                                   
│  │  TMap<UClass*, UPoolFactory_UObject*>                    
│  │  ├─ UObject → UPoolFactory_UObject                       
│  │  ├─ AActor → UPoolFactory_Actor                         
│  │  └─ UUserWidget → UPoolFactory_UserWidget               
│  └──────────────────────────────────────────────────────┘   
└─────────────────────────────────────────────────────────────┘
                              │
                              │ 使用
                              ▼
┌─────────────────────────────────────────────────────────────┐
│                    UPoolFactory_UObject                       
│                    (基类工厂)                                 
│                                                               
│  SpawnQueueInternal: TArray<FSpawnRequest>                   
│  ├─ RequestSpawn() → 入队                                    
│  ├─ OnNextTickProcessSpawn() → 每帧处理                      
│  └─ ProcessRequestNow() → 立即生成                           
│                                                               
│  ┌──────────────────────────────────────────────────────┐   
│  │  派生类:                                                  
│  │  ├─ UPoolFactory_Actor                                  
│  │  │   └─ SpawnNow: World->SpawnActor()                    
│  │  ├─ UPoolFactory_UserWidget                             
│  │  │   └─ SpawnNow: CreateWidget()                         
│  │  └─ UPoolFactory_UObject (基类)                          
│  │      └─ SpawnNow: NewObject()                            
│  └──────────────────────────────────────────────────────┘   
└─────────────────────────────────────────────────────────────┘
```

### 模块关系图

```
┌─────────────────┐
│ PoolManagerTypes │  ← 核心数据结构定义
│  (Types.h/cpp)  │
└────────┬────────┘
         │
         ├─────────────────────────────────────┐
         │                                     │
         ▼                                     ▼
┌────────────────────┐              ┌─────────────────────┐
│PoolManagerSubsystem│              │ PoolFactory_UObject 
│  (Subsystem.h/cpp) │              │   (Factory.h/cpp)   
│                    │              │                     
│ - 管理所有池         │              │ - 对象创建/销毁     
│ - 提供对外接口       │◄─────────────┤ - 状态管理          
│ - 懒加载池容器       │  使用工厂      │ - 生成队列处理      
└────────────────────┘              └─────────────────────┘
         │                                     │
         │                                     │ 继承
         │                                     ▼
         │                            ┌─────────────────────┐
         │                            │ PoolFactory_Actor   │
         │                            │ PoolFactory_Widget  │
         │                            └─────────────────────┘
         │
         ▼
┌─────────────────────┐
│ PoolManagerSettings │  ← 项目配置
│   (Settings.h/cpp)  │
└─────────────────────┘
```

---

## 核心数据结构

### 1. FPoolObjectHandle - 对象句柄

**作用**：提供对象的间接引用，支持异步操作。即使对象尚未生成，也能立即获得句柄。

```cpp
struct FPoolObjectHandle
{
    const UClass* ObjectClass;  // 对象类型
    FGuid Hash;                 // 唯一标识符（用于查找和比较）
};
```

**关键特性**：
- 使用 `FGuid` 作为唯一标识，支持哈希查找
- 在对象生成前即可获得，实现异步操作
- 支持通过句柄查找和操作对象

### 2. FPoolObjectData - 池对象数据

**作用**：存储池中每个对象的完整信息。

```cpp
struct FPoolObjectData
{
    TObjectPtr<UObject> PoolObject;  // 实际对象指针
    FPoolObjectHandle Handle;        // 关联的句柄
    bool bIsActive;                  // 是否处于激活状态
};
```

**状态管理**：
- `bIsActive = false`：对象在池中，处于非激活状态（Inactive）
- `bIsActive = true`：对象已被取出使用（Active）

### 3. FPoolContainer - 池容器

**作用**：按对象类型组织池，每个类型一个容器。

```cpp
struct FPoolContainer
{
    TObjectPtr<const UClass> ObjectClass;           // 该池管理的对象类型
    TObjectPtr<UPoolFactory_UObject> Factory;       // 对应的工厂
    TArray<FPoolObjectData> PoolObjects;           // 该类型的所有对象
};
```

**设计要点**：
- 一个 `FPoolContainer` 对应一种对象类型
- 通过 `ObjectClass` 进行查找和匹配
- 每个容器关联一个工厂，负责该类型对象的创建/销毁

### 4. FSpawnRequest - 生成请求

**作用**：封装对象生成所需的所有信息。

```cpp
struct FSpawnRequest
{
    FPoolObjectHandle Handle;              // 对象句柄（自动生成）
    FTransform Transform;                 // 生成位置/变换
    ESpawnRequestPriority Priority;        // 优先级
    FSpawnCallbacks Callbacks;             // 回调函数
};
```

**优先级系统**：
- `Critical`：立即处理，不进入队列
- `High`：插入队列头部
- `Medium`：插入队列中高优先级之后
- `Normal`：添加到队列末尾

---

## 核心流程与调用堆栈

### 流程1：从池中取出对象（TakeFromPool）

#### 完整调用堆栈

```
用户代码
  │
  ├─> UPoolManagerSubsystem::TakeFromPool(UClass*)
  │   │
  │   └─> TakeFromPoolOrNull(UClass*, FTransform)
  │       │
  │       ├─> FindPool(UClass*)  // 查找或创建池
  │       │   └─> PoolsInternal.FindByKey(ObjectClass)
  │       │
  │       ├─> 遍历 PoolObjects，查找 IsFree() 的对象
  │       │
  │       ├─> Pool->GetFactoryChecked().OnTakeFromPool(Object, Transform)
  │       │   │
  │       │   └─> [可选] IPoolObjectCallback::OnTakeFromPool()
  │       │
  │       └─> SetObjectStateInPool(EPoolObjectState::Active, Object, Pool)
  │           │
  │           ├─> PoolObject->bIsActive = true
  │           └─> Factory->OnChangedStateInPool(Active, Object)
  │               │
  │               └─> [可选] IPoolObjectCallback::OnChangedStateInPool()
  │
  └─> 返回 FPoolObjectHandle
```

#### 关键代码路径

```cpp
// 1. 查找池
FPoolContainer* Pool = FindPool(ObjectClass);
if (!Pool) return nullptr;  // 池不存在，返回null

// 2. 查找空闲对象
for (const FPoolObjectData& DataIt : Pool->PoolObjects)
{
    if (DataIt.IsFree())  // bIsActive == false && IsValid()
    {
        FoundData = &DataIt;
        break;
    }
}

// 3. 激活对象
Pool->GetFactoryChecked().OnTakeFromPool(&Object, Transform);
SetObjectStateInPool(EPoolObjectState::Active, Object, *Pool);
```

### 流程2：创建新对象（CreateNewObjectInPool）

#### 完整调用堆栈

```
用户代码
  │
  ├─> UPoolManagerSubsystem::TakeFromPool(UClass*)
  │   │
  │   └─> CreateNewObjectInPool(FSpawnRequest)
  │       │
  │       ├─> FindPoolOrAdd(ObjectClass)  // 懒加载：池不存在则创建
  │       │   │
  │       │   ├─> FindPool(ObjectClass)
  │       │   │   └─> PoolsInternal.FindByKey(ObjectClass)
  │       │   │
  │       │   └─> [池不存在] PoolsInternal.AddDefaulted_GetRef()
  │       │       ├─> Pool.ObjectClass = ObjectClass
  │       │       └─> Pool.Factory = FindPoolFactoryChecked(ObjectClass)
  │       │
  │       ├─> Request.Handle = FPoolObjectHandle::NewHandle(Class)
  │       │   └─> Handle.Hash = FGuid::NewGuid()
  │       │
  │       ├─> 设置回调：Request.Callbacks.OnPreRegistered
  │       │   └─> [Lambda] RegisterObjectInPool(ObjectData)
  │       │
  │       └─> Pool->GetFactoryChecked().RequestSpawn(Request)
  │           │
  │           └─> UPoolFactory_UObject::RequestSpawn_Implementation()
  │               │
  │               ├─> [Critical优先级] ProcessRequestNow(Request)
  │               │   │
  │               │   └─> [立即生成，不进入队列]
  │               │
  │               ├─> [其他优先级] 插入 SpawnQueueInternal
  │               │   └─> 根据优先级插入队列
  │               │
  │               └─> [队列为空时] SetTimerForNextTick(OnNextTickProcessSpawn)
  │
  └─> 返回 FPoolObjectHandle（对象可能尚未生成）
```

#### 异步生成流程

```
下一帧 Tick
  │
  └─> UPoolFactory_UObject::OnNextTickProcessSpawn()
      │
      ├─> 获取配置：GetSpawnObjectsPerFrame()  // 每帧生成数量限制
      │
      ├─> 循环处理（最多 ObjectsPerFrame 个）
      │   │
      │   ├─> DequeueSpawnRequest(OutRequest)
      │   │   └─> SpawnQueueInternal.RemoveAt(0)
      │   │
      │   └─> ProcessRequestNow(OutRequest)
      │       │
      │       ├─> SpawnNow(Request)
      │       │   │
      │       │   ├─> [UObject] NewObject<UObject>()
      │       │   ├─> [Actor] World->SpawnActor()
      │       │   └─> [Widget] CreateWidget()
      │       │
      │       ├─> OnPreRegistered(Request, ObjectData)
      │       │   │
      │       │   └─> Request.Callbacks.OnPreRegistered(ObjectData)
      │       │       │
      │       │       └─> [Lambda] RegisterObjectInPool(ObjectData)
      │       │           │
      │       │           ├─> FindPoolOrAdd(ObjectClass)
      │       │           ├─> Pool.PoolObjects.Emplace(ObjectData)
      │       │           └─> SetObjectStateInPool(State, Object, Pool)
      │       │
      │       └─> OnPostSpawned(Request, ObjectData)
      │           │
      │           ├─> Request.Callbacks.OnPostSpawned(ObjectData)
      │           └─> [可选] IPoolObjectCallback::OnTakeFromPool()
      │
      └─> [队列未空] SetTimerForNextTick(OnNextTickProcessSpawn)
          └─> 下一帧继续处理
```

### 流程3：归还对象到池（ReturnToPool）

#### 完整调用堆栈

```
用户代码
  │
  ├─> UPoolManagerSubsystem::ReturnToPool(UObject*)
  │   │
  │   ├─> FindPoolOrAdd(Object->GetClass())
  │   │
  │   ├─> Pool->GetFactoryChecked().OnReturnToPool(Object)
  │   │   │
  │   │   └─> [可选] IPoolObjectCallback::OnReturnToPool()
  │   │
  │   └─> SetObjectStateInPool(EPoolObjectState::Inactive, Object, Pool)
  │       │
  │       ├─> PoolObject->bIsActive = false
  │       └─> Factory->OnChangedStateInPool(Inactive, Object)
  │           │
  │           └─> [Actor] SetActorHiddenInGame(true)
  │               │        SetActorEnableCollision(false)
  │               │        SetActorTickEnabled(false)
  │               │        SetActorLocation(MaxPos)  // 移到远处
```

---

## 内存池实现机制

### 1. 懒加载策略

**核心思想**：池容器在首次使用时创建，而非预分配。

```cpp
FPoolContainer& UPoolManagerSubsystem::FindPoolOrAdd(const UClass* ObjectClass)
{
    // 1. 先查找
    if (FPoolContainer* Pool = FindPool(ObjectClass))
    {
        return *Pool;
    }
    
    // 2. 不存在则创建
    FPoolContainer& Pool = PoolsInternal.AddDefaulted_GetRef();
    Pool.ObjectClass = ObjectClass;
    Pool.Factory = FindPoolFactoryChecked(ObjectClass);  // 查找或创建工厂
    return Pool;
}
```

**优势**：
- 节省内存：只创建实际使用的池
- 动态扩展：按需创建，无需预配置
- 简化管理：自动处理工厂关联

### 2. 对象状态管理

**状态转换图**：

```
┌─────────┐
│  None   │  ← 对象未在池中
└─────────┘
     │
     │ RegisterObjectInPool()
     ▼
┌──────────┐
│ Inactive │  ← 对象在池中，空闲可用
└──────────┘
     │
     │ TakeFromPool()
     ▼
┌────────┐
│ Active │  ← 对象被取出使用
└────────┘
     │
     │ ReturnToPool()
     ▼
┌──────────┐
│ Inactive │
└──────────┘
```

**实现细节**：

```cpp
void UPoolManagerSubsystem::SetObjectStateInPool(
    EPoolObjectState NewState, 
    UObject& InObject, 
    FPoolContainer& InPool)
{
    FPoolObjectData* PoolObject = InPool.FindInPool(InObject);
    PoolObject->bIsActive = (NewState == EPoolObjectState::Active);
    
    // 通知工厂进行状态相关操作
    InPool.GetFactoryChecked().OnChangedStateInPool(NewState, &InObject);
}
```

### 3. 异步生成队列

**队列处理机制**：

```cpp
void UPoolFactory_UObject::OnNextTickProcessSpawn_Implementation()
{
    // 1. 获取每帧生成限制
    int32 ObjectsPerFrame = UPoolManagerSettings::Get().GetSpawnObjectsPerFrame();
    
    // 2. 处理一批请求（避免单帧卡顿）
    const int32 NumToSpawn = FMath::Min(ObjectsPerFrame, SpawnQueueInternal.Num());
    for (int32 Index = 0; Index < NumToSpawn; ++Index)
    {
        FSpawnRequest OutRequest;
        if (DequeueSpawnRequest(OutRequest))
        {
            ProcessRequestNow(OutRequest);
        }
    }
    
    // 3. 队列未空，下一帧继续
    if (!SpawnQueueInternal.IsEmpty())
    {
        World->GetTimerManager().SetTimerForNextTick(
            this, &ThisClass::OnNextTickProcessSpawn);
    }
}
```

**设计要点**：
- **分帧处理**：避免单帧生成大量对象造成卡顿
- **优先级队列**：高优先级请求优先处理
- **Critical 立即处理**：紧急请求不进入队列，立即生成

### 4. 句柄系统

**句柄的作用**：
1. **异步引用**：对象生成前即可获得引用
2. **唯一标识**：通过 `FGuid` 唯一标识对象
3. **查找优化**：支持哈希查找，O(1) 复杂度

```cpp
FPoolObjectHandle FPoolObjectHandle::NewHandle(const UClass* InObjectClass)
{
    FPoolObjectHandle Handle;
    Handle.ObjectClass = InObjectClass;
    Handle.Hash = FGuid::NewGuid();  // 生成唯一ID
    return Handle;
}
```

**查找机制**：

```cpp
FPoolObjectData* FPoolContainer::FindInPool(const FPoolObjectHandle& Handle)
{
    // 使用 FindByKey，内部使用 operator== 比较 Handle
    return PoolObjects.FindByKey(Handle);
}

// Handle 的相等比较
bool operator==(const FPoolObjectHandle& A, const FPoolObjectHandle& B)
{
    return A.Hash == B.Hash;  // 通过 Hash 比较
}
```

---

## 工厂模式实现

### 工厂层次结构

```
UPoolFactory_UObject (基类)
  │
  ├─> UPoolFactory_Actor
  │   └─> 处理 Actor 的创建/销毁/状态管理
  │
  ├─> UPoolFactory_UserWidget
  │   └─> 处理 Widget 的创建/销毁/状态管理
  │
  └─> [可扩展] 自定义工厂
      └─> 继承并实现特定类型的逻辑
```

### 工厂查找机制

**类层次遍历**：

```cpp
UPoolFactory_UObject* UPoolManagerSubsystem::FindPoolFactoryChecked(
    const UClass* ObjectClass) const
{
    const UClass* CurrentClass = ObjectClass;
    
    // 向上遍历类层次，查找最近的工厂
    while (CurrentClass != nullptr)
    {
        if (const TObjectPtr<UPoolFactory_UObject>* FoundFactory = 
            AllFactoriesInternal.Find(CurrentClass))
        {
            return *FoundFactory;
        }
        CurrentClass = CurrentClass->GetSuperClass();
    }
    
    checkf(false, TEXT("Factory not found!"));
    return nullptr;
}
```

**示例**：
- 请求 `AProjectile`（继承自 `AActor`）
- 查找顺序：`AProjectile` → `AActor` → `UObject`
- 找到 `AActor` 对应的 `UPoolFactory_Actor`，使用该工厂

### 工厂职责划分

#### 1. 对象创建（SpawnNow）

```cpp
// 基类：UObject
UObject* UPoolFactory_UObject::SpawnNow_Implementation(const FSpawnRequest& Request)
{
    return NewObject<UObject>(GetOuter(), Request.GetClassChecked());
}

// Actor 工厂
UObject* UPoolFactory_Actor::SpawnNow_Implementation(const FSpawnRequest& Request)
{
    return World->SpawnActor(
        Request.GetClassChecked<AActor>(), 
        &Request.Transform, 
        SpawnParameters);
}

// Widget 工厂
UObject* UPoolFactory_UserWidget::SpawnNow_Implementation(const FSpawnRequest& Request)
{
    return CreateWidget(GetWorld(), Request.GetClassChecked<UUserWidget>());
}
```

#### 2. 对象销毁（Destroy）

```cpp
// 基类
void UPoolFactory_UObject::Destroy_Implementation(UObject* Object)
{
    Object->ConditionalBeginDestroy();
}

// Actor 工厂
void UPoolFactory_Actor::Destroy_Implementation(UObject* Object)
{
    CastChecked<AActor>(Object)->Destroy();
}

// Widget 工厂
void UPoolFactory_UserWidget::Destroy_Implementation(UObject* Object)
{
    CastChecked<UUserWidget>(Object)->RemoveFromParent();
    Object->ConditionalBeginDestroy();
}
```

#### 3. 状态管理（OnChangedStateInPool）

```cpp
// Actor 工厂：管理可见性、碰撞、Tick
void UPoolFactory_Actor::OnChangedStateInPool_Implementation(
    EPoolObjectState NewState, 
    UObject* InObject)
{
    AActor* Actor = CastChecked<AActor>(InObject);
    const bool bActivate = (NewState == EPoolObjectState::Active);
    
    Actor->SetActorHiddenInGame(!bActivate);
    Actor->SetActorEnableCollision(bActivate);
    Actor->SetActorTickEnabled(bActivate);
    
    if (!bActivate)
    {
        Actor->SetActorLocation(MaxPos);  // 移到远处
    }
}

// Widget 工厂：管理可见性
void UPoolFactory_UserWidget::OnChangedStateInPool_Implementation(
    EPoolObjectState NewState, 
    UObject* InObject)
{
    UUserWidget* Widget = CastChecked<UUserWidget>(InObject);
    const bool bActivate = (NewState == EPoolObjectState::Active);
    
    Widget->SetVisibility(
        bActivate ? ESlateVisibility::Visible : ESlateVisibility::Collapsed);
}
```

---

## 关键设计模式

### 1. 单例模式（Subsystem）

`UPoolManagerSubsystem` 继承自 `UWorldSubsystem`，由引擎自动管理生命周期。

```cpp
// 获取单例
UPoolManagerSubsystem& PoolManager = UPoolManagerSubsystem::Get();
```

### 2. 工厂模式

不同类型的对象使用专门的工厂处理，实现关注点分离。

### 3. 策略模式

通过工厂的虚函数重写，实现不同对象类型的创建/销毁策略。

### 4. 观察者模式

通过 `IPoolObjectCallback` 接口，对象可以接收池管理事件。

```cpp
// 对象实现接口
class AProjectile : public AActor, public IPoolObjectCallback
{
    virtual void OnTakeFromPool_Implementation(
        bool bIsNewSpawned, 
        const FTransform& Transform) override
    {
        // 对象被取出时的处理
    }
    
    virtual void OnReturnToPool_Implementation() override
    {
        // 对象归还时的处理
    }
};
```

### 5. 懒加载模式

池容器和对象按需创建，避免不必要的内存占用。

---

## 性能优化要点

### 1. 分帧生成
- 每帧限制生成数量（`SpawnObjectsPerFrame`）
- 避免单帧卡顿

### 2. 优先级队列
- Critical 立即处理
- High/Medium 优先处理
- Normal 正常排队

### 3. 对象复用
- 优先从池中取出空闲对象
- 减少对象创建/销毁开销

### 4. 状态管理优化
- Actor 移到远处而非销毁
- 禁用碰撞、Tick 等，减少性能开销

---

## 扩展指南

### 添加新的对象类型支持

1. **创建工厂类**：
```cpp
UCLASS()
class UPoolFactory_MyType : public UPoolFactory_UObject
{
    GENERATED_BODY()
    
    virtual const UClass* GetObjectClass_Implementation() const override
    {
        return UMyType::StaticClass();
    }
    
    virtual UObject* SpawnNow_Implementation(const FSpawnRequest& Request) override
    {
        // 实现创建逻辑
    }
    
    virtual void Destroy_Implementation(UObject* Object) override
    {
        // 实现销毁逻辑
    }
};
```

2. **注册工厂**：
   - 在项目设置中添加工厂类到 `PoolFactories` 列表

3. **使用**：
```cpp
FPoolObjectHandle Handle = UPoolManagerSubsystem::Get()
    .TakeFromPool<UMyType>();
```

---

## Pool Manager vs C++ 传统内存池

### 核心区别对比

| 维度 | C++ 传统内存池 | UE Pool Manager |
|------|---------------|-----------------|
| **管理对象** | 原始内存块（字节数组） | 完整的游戏对象（UObject及其派生类） |
| **分配粒度** | 字节级别 | 对象级别 |
| **生命周期** | 只负责内存分配/释放 | 管理对象的完整生命周期 |
| **对象状态** | 已分配/未分配（二进制） | None/Inactive/Active（多状态） |
| **构造/析构** | 不负责，由调用者处理 | 负责对象的创建和销毁 |
| **复杂度** | 相对简单（主要是内存管理） | 复杂（对象管理+状态管理+异步处理） |
| **使用场景** | 优化频繁的小对象分配 | 优化游戏对象的创建/销毁 |

### 详细对比分析

#### 1. 管理目标不同

**C++ 传统内存池**：
```cpp
// 典型的内存池实现
class MemoryPool {
    void* Allocate(size_t size);  // 分配原始内存
    void Deallocate(void* ptr);   // 释放内存
};

// 使用示例
MemoryPool pool;
int* ptr = (int*)pool.Allocate(sizeof(int));  // 只分配内存
*ptr = 42;
pool.Deallocate(ptr);  // 只释放内存
```

**UE Pool Manager**：
```cpp
// 管理完整的游戏对象
FPoolObjectHandle Handle = UPoolManagerSubsystem::Get()
    .TakeFromPool<AProjectile>();  // 获取完整的 Actor 对象

// 对象已经构造完成，可以直接使用
// 归还时，对象被停用而非销毁
PoolManager.ReturnToPool(Handle);
```

#### 2. 生命周期管理

**C++ 内存池**：
- ✅ 只负责内存的分配和释放
- ❌ 不负责对象的构造和析构
- ❌ 不关心对象的状态

**UE Pool Manager**：
- ✅ 负责对象的创建（通过工厂）
- ✅ 负责对象的销毁（通过工厂）
- ✅ 管理对象的激活/停用状态
- ✅ 处理对象的可见性、碰撞、Tick 等属性

#### 3. 状态管理复杂度

**C++ 内存池**：
```
内存块状态：
┌─────────┐      Allocate()      ┌─────────┐
│ 未分配   │ ──────────────────> │ 已分配   │
└─────────┘                      └─────────┘
     ▲                                 │
     │                                 │
     └──────── Deallocate() ──────────┘
```

**UE Pool Manager**：
```
对象状态：
┌─────────┐
│  None   │  ← 对象未在池中
└─────────┘
     │
     │ RegisterObjectInPool()
     ▼
┌──────────┐
│ Inactive │  ← 在池中，隐藏，禁用碰撞/Tick
└──────────┘
     │
     │ TakeFromPool()
     ▼
┌────────┐
│ Active │  ← 正在使用，可见，启用碰撞/Tick
└────────┘
     │
     │ ReturnToPool()
     ▼
┌──────────┐
│ Inactive │
└──────────┘
```

#### 4. 对象创建方式

**C++ 内存池**：
```cpp
// 内存池只分配内存，需要手动构造
class ObjectPool {
    void* Allocate() { /* 分配内存 */ }
};

// 使用
void* mem = pool.Allocate();
MyClass* obj = new(mem) MyClass();  // Placement new
// ... 使用对象
obj->~MyClass();  // 手动析构
pool.Deallocate(mem);
```

**UE Pool Manager**：
```cpp
// Pool Manager 负责完整的创建流程
FPoolObjectHandle Handle = PoolManager.TakeFromPool<AProjectile>();

// 对象可能：
// 1. 从池中取出（已存在，只需激活）
// 2. 异步创建（下一帧生成）
// 3. 立即创建（Critical 优先级）

// 归还时自动处理停用
PoolManager.ReturnToPool(Handle);
// 对象被隐藏、移到远处、禁用碰撞等，但未销毁
```

#### 5. 性能优化重点

**C++ 内存池**：
- ✅ 减少 `malloc/free` 系统调用
- ✅ 减少内存碎片
- ✅ 提高小对象分配速度
- ✅ 预分配大块内存，减少分配次数

**UE Pool Manager**：
- ✅ 避免对象构造/析构开销（复用对象）
- ✅ 避免 GC 压力（对象不销毁）
- ✅ 分帧生成，避免单帧卡顿
- ✅ 状态切换而非销毁重建

#### 6. 使用场景

**C++ 内存池**：
```cpp
// 适合场景：
// 1. 频繁分配的小对象（如链表节点）
struct Node {
    int data;
    Node* next;
};

// 2. 固定大小的对象
class SmallObject {
    char buffer[64];
};

// 3. 临时缓冲区
char* tempBuffer = (char*)pool.Allocate(1024);
```

**UE Pool Manager**：
```cpp
// 适合场景：
// 1. 游戏对象（Actor）
FPoolObjectHandle BulletHandle = PoolManager.TakeFromPool<ABullet>();

// 2. UI 元素（Widget）
FPoolObjectHandle WidgetHandle = PoolManager.TakeFromPool<UHealthBar>();

// 3. 特效对象
FPoolObjectHandle EffectHandle = PoolManager.TakeFromPool<AExplosionEffect>();
```

### 为什么 UE 需要 Pool Manager 而不是传统内存池？

#### 1. UE 对象的复杂性

UE 对象（特别是 Actor）的创建/销毁涉及：
- 构造函数的执行
- 组件的创建和注册
- 世界场景的注册
- 网络复制的初始化
- GC 系统的注册
- 渲染资源的分配

这些操作的开销远大于简单的内存分配。

#### 2. GC 系统的限制

UE 使用垃圾回收系统，频繁创建/销毁对象会导致：
- GC 压力增大
- 内存碎片增加
- 性能波动

Pool Manager 通过复用对象，避免这些问题。

#### 3. 游戏对象的特殊性

游戏对象需要：
- **状态管理**：激活/停用时的不同行为
- **可见性控制**：隐藏时不应渲染
- **碰撞管理**：停用时不应参与物理
- **Tick 控制**：停用时不应更新

这些需求超出了传统内存池的范畴。

### 类比理解

**C++ 内存池** ≈ **停车场**
- 只提供停车位（内存）
- 不关心车里是什么（对象类型）
- 不关心车的状态（对象状态）

**UE Pool Manager** ≈ **租车公司**
- 提供完整的车辆（游戏对象）
- 管理车辆的状态（激活/停用）
- 负责车辆的维护（状态切换）
- 车辆归还后可以再次出租（对象复用）

### 总结

| 特性 | C++ 内存池 | UE Pool Manager |
|------|-----------|----------------|
| **本质** | 内存分配器 | 对象生命周期管理器 |
| **优化目标** | 减少内存分配开销 | 减少对象创建/销毁开销 |
| **管理范围** | 内存块 | 完整对象 + 状态 |
| **适用对象** | 任意类型 | UE 对象系统（UObject） |
| **复杂度** | 低 | 高 |
| **使用难度** | 需要手动管理构造/析构 | 自动管理，使用简单 |

**结论**：UE Pool Manager 是一个**对象池管理器**，而不仅仅是内存池。它管理的是完整的游戏对象及其生命周期，而不仅仅是内存分配。

---

## 总结

Pool Manager 通过以下核心机制实现高效的对象池管理：

1. **懒加载池容器**：按需创建，节省内存
2. **工厂模式**：不同类型对象使用专门工厂
3. **异步生成队列**：分帧处理，避免卡顿
4. **句柄系统**：支持异步引用和快速查找
5. **状态管理**：通过工厂统一管理对象状态

该系统设计灵活、可扩展，适合在需要频繁创建/销毁对象的场景中使用。

**与 C++ 内存池的区别**：Pool Manager 是**对象池管理器**，管理完整的游戏对象及其生命周期，而不仅仅是内存分配。它更适合 UE 的复杂对象系统和 GC 环境。

