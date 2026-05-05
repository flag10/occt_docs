# Standard_Transient / Standard_Type / Standard_Persistent 深入解析

> **一句话**：`Standard_Transient` 是 OCCT 所有对象的根基，通过内嵌原子计数器实现零额外开销的侵入式引用计数；`Standard_Type` 则构建了一套完整的全局类型注册体系，让每个 `Standard_Transient` 子类都能在运行时被精确识别和动态转换——两者共同构成了 OCCT 智能指针 `handle<T>` 和 RTTI 的底层基石。

## 目录

1. [Standard_Transient — 瞬态对象根基类](#1-standard_transient--瞬态对象根基类)
2. [Standard_Type — 运行时类型描述符](#2-standard_type--运行时类型描述符)
3. [Handle 智能指针体系演变](#3-handle-智能指针体系演变)
4. [Standard_Persistent — 持久化对象根基类](#4-standard_persistent--持久化对象根基类)
5. [总结对比](#5-总结对比)
6. [One More Thing — 为什么 OCCT 不使用 std::shared_ptr](#6-one-more-thing--为什么-occt-不使用-stdshared_ptr)

---

## 1. Standard_Transient — 瞬态对象根基类

### 1.1 定位

`Standard_Transient` 是 OCCT 整个瞬态（堆分配、运行时）对象层次的**抽象基类根**，约 953 个类直接或间接继承自它。它提供两项核心基础设施：

| 能力 | 说明 |
|------|------|
| **侵入式引用计数** | `std::atomic_int myRefCount_`，供 `handle<T>` 智能指针管理生命周期 |
| **运行时类型信息（RTTI）** | `DynamicType()` / `IsInstance()` / `IsKind()`，基于 `Standard_Type` 全局注册表 |

**源文件位置：**
- `src/FoundationClasses/TKernel/Standard/Standard_Transient.hxx` — 类声明（145 行）
- `src/FoundationClasses/TKernel/Standard/Standard_Transient.cxx` — RTTI 方法实现（76 行）

### 1.2 引用计数机制

```cpp
class Standard_Transient {
private:
    std::atomic_int myRefCount_;   // 原子计数器

public:
    // 构造函数将计数器初始化为 0
    Standard_Transient() : myRefCount_(0) {}

    // 拷贝构造同样重置为 0 —— 新对象从零开始计数
    Standard_Transient(const Standard_Transient&) : myRefCount_(0) {}

    // 赋值操作符为 no-op —— 不复制引用计数
    Standard_Transient& operator=(const Standard_Transient&) { return *this; }

    // 递增：使用 relaxed 内存序，因为只需要原子性，无需同步
    void IncrementRefCounter() noexcept {
        myRefCount_.fetch_add(1, std::memory_order_relaxed);
    }

    // 递减：使用 release 内存序确保对象写操作对删除者可见
    // 计数归零时加 acquire fence，确保所有操作完成后再 delete
    int DecrementRefCounter() noexcept {
        const int aResult = myRefCount_.fetch_sub(1, std::memory_order_release) - 1;
        if (aResult == 0) {
            std::atomic_thread_fence(std::memory_order_acquire);
        }
        return aResult;
    }

    // 虚拟析构函数
    virtual ~Standard_Transient() = default;

    // 删除方法 —— 子类可重载以使用自定义分配器
    virtual void Delete() const { delete this; }
};
```

**设计要点：**

- **计数器命名**：`myRefCount_` 使用下划线前缀，旨在减少与派生类成员名冲突的概率（OCCT 代码库历史中多数成员为驼峰命名）。
- **拷贝语义**：拷贝构造函数将计数器重置为 0，赋值操作为无操作。这意味着从已有对象拷贝构造出的新对象拥有独立的引用计数，不会被意外共享生命周期。
- **内存序优化**：递增用 `relaxed`（仅需原子性），递减用 `release` + 条件 `acquire`（仅在真正需要销毁对象时才强制同步），避免了每次递减都使用 `acq_rel` 的高开销。
- **栈保护**：`This()` 方法在引用计数为 0 时抛出异常，防止对栈对象或尚未完全构造/已析构的对象创建 handle。

### 1.3 RTTI 机制

#### 1.3.1 宏体系

每个继承自 `Standard_Transient` 的类通过以下宏之一声明 RTTI 支持：

```cpp
// 内联版本 —— 所有代码在头文件中，适用于小型继承树
#define DEFINE_STANDARD_RTTI_INLINE(Class, Base)

// 分离编译版本 —— 声明在头文件，实现在 .cxx
#define DEFINE_STANDARD_RTTIEXT(Class, Base)
#define IMPLEMENT_STANDARD_RTTIEXT(Class, Base)
```

这些宏展开后生成以下关键成员：

```cpp
// 以 DEFINE_STANDARD_RTTIEXT(Geom_Surface, Standard_Transient) 为例：
class Geom_Surface : public Standard_Transient {
public:
    typedef Standard_Transient base_type;                        // 基类类型别名

    static constexpr const char* get_type_name() {               // 类名字符串
        static_assert(std::is_base_of<Standard_Transient, Geom_Surface>::value
                      && !std::is_same<Standard_Transient, Geom_Surface>::value);
        return "Geom_Surface";
    }

    static const occ::handle<Standard_Type>& get_type_descriptor();   // 类型描述符获取
    virtual const occ::handle<Standard_Type>& DynamicType() const override;
};
```

`IMPLEMENT_STANDARD_RTTIEXT` 实现：

```cpp
const occ::handle<Standard_Type>& Geom_Surface::get_type_descriptor() {
    static const occ::handle<Standard_Type> THE_TYPE_INSTANCE =
        Standard_Type::Register(
            typeid(Geom_Surface),                         // C++ 原生 type_info
            get_type_name(),                              // 可读类名
            sizeof(Geom_Surface),                         // 对象大小
            Geom_Surface::base_type::get_type_descriptor() // 父类描述符
        );
    return THE_TYPE_INSTANCE;
}

const occ::handle<Standard_Type>& Geom_Surface::DynamicType() const {
    return STANDARD_TYPE(Geom_Surface);  // 即 Geom_Surface::get_type_descriptor()
}
```

#### 1.3.2 类型注册表

`Standard_Type::Register()` 维护一个**全局注册表**：

```cpp
// 类型：NCollection_DataMap<const char*, Standard_Type*>
// 键：type_info::name() 的深拷贝，值：Standard_Type 描述符指针
static registry_type& GetRegistry();
```

**注册流程：**

1. 首次调用 `Class::get_type_descriptor()` 时，静态局部变量触发 `Standard_Type::Register()`
2. `Register()` 加互斥锁检查注册表，若该 `type_info::name()` 已存在则直接返回
3. 若不存在，在一块连续内存中分配 `Standard_Type` 对象 + 两个字符串的深拷贝（`type_info::name()` 和用户提供的类名）
4. 将描述符插入 `DataMap`，返回指针

**销毁流程：**

- 由于 `Standard_Type` 也被 `handle<Standard_Type>` 管理（`get_type_descriptor()` 返回 `static const handle<Standard_Type>`），类型描述符与程序同生命周期
- 若 `Standard_Type` 被析构（程序退出时），其析构函数从注册表中移除自身项

#### 1.3.3 类型查询 API

```cpp
// 判断 this 是否为 AType 的精确实例（非派生类）
bool IsInstance(const handle<Standard_Type>& AType) const {
    return (AType == DynamicType());  // 指针比较，因为每种类型只有一个描述符
}

// 判断 this 是否属于 AType 或其派生类
bool IsKind(const handle<Standard_Type>& aType) const {
    return DynamicType()->SubType(aType);  // 沿父类链向上遍历
}
```

`SubType()` 实现（`Standard_Type.cxx:36`）：从当前类型沿父类链向上遍历，逐级比较描述符指针，直至匹配或到达根节点。**不支持多重继承。**

#### 1.3.4 现代 C++ 接口（OCCT 8.0+）

```cpp
// 旧写法：
if (aCurve->IsKind(STANDARD_TYPE(Geom_Circle))) { ... }

// 新写法（occ 命名空间，定义于 Standard_Type.hxx:218-264）：
if (occ::is_kind<Geom_Circle>(aCurve)) { ... }          // 类型及派生
if (occ::is_instance<Geom_Circle>(aCurve)) { ... }       // 精确类型
auto aCircle = occ::down_cast<Geom_Circle>(aCurve);      // 安全下行转换
```

---

## 2. Standard_Type — 运行时类型描述符

### 2.1 定位

`Standard_Type` 是 OCCT RTTI 体系的核心中枢。它本身也继承自 `Standard_Transient`，作为**类型描述符对象**被全局注册表管理。每个 `Standard_Transient` 的派生类都拥有一个与之关联的唯一 `Standard_Type` 实例。

**源文件位置：**
- `src/FoundationClasses/TKernel/Standard/Standard_Type.hxx` — 类声明 + 宏定义 + 现代模板函数（268 行）
- `src/FoundationClasses/TKernel/Standard/Standard_Type.cxx` — 注册表实现 + `SubType()` 查询算法（160 行）

### 2.2 类结构

```cpp
class Standard_Type : public Standard_Transient
{
public:
    // ===== 属性查询 =====
    const char* SystemName() const { return mySystemName; }  // type_info::name() 字符串
    const char* Name()       const { return myName; }        // 用户定义类名 (例: "Geom_Circle")
    size_t      Size()       const { return mySize; }        // 对象大小 (字节, sizeof(T))

    // 父类描述符
    const occ::handle<Standard_Type>& Parent() const { return myParent; }

    // ===== 类型判断 =====

    // 判断当前类型是否为 theOther 或 theOther 的子类（沿父链向上查）
    bool SubType(const occ::handle<Standard_Type>& theOther) const;
    bool SubType(const char* const theOther) const;

    // 打印描述符信息
    void Print(Standard_OStream& theStream) const;

    // ===== 静态工厂 =====

    // 模板函数：获取类 T 的类型描述符
    template <class T>
    static const occ::handle<Standard_Type>& Instance()
    {
        return T::get_type_descriptor();
    }

    // 注册（或查找）一个类型描述符，返回新创建或已存在的描述符
    static Standard_Type* Register(const std::type_info&             theInfo,
                                   const char*                       theName,
                                   size_t                            theSize,
                                   const occ::handle<Standard_Type>& theParent);

    // 析构函数：从注册表中移除自身
    ~Standard_Type() override;

private:
    const char*                mySystemName;  //!< 编译器产出的 type_info::name()
    const char*                myName;        //!< 用户定义类名字符串
    size_t                     mySize;        //!< sizeof(Class)
    occ::handle<Standard_Type> myParent;      //!< 父类类型描述符（构造父链）
};
```

### 2.3 全局注册表实现

类型描述符的全部生命周期由**全局注册表**管理，注册表位于 `Standard_Type.cxx` 的匿名命名空间中：

```cpp
// 哈希函数 —— 对 C 字符串做散列
struct typeNameHasher {
    size_t operator()(const char* const theType) const noexcept {
        return opencascade::hashBytes(theType, static_cast<int>(strlen(theType)));
    }
    bool operator()(const char* cosnt t1, const char* const t2) const noexcept {
        return strcmp(t1, t2) == 0;
    }
};

// 注册表类型：NCollection_DataMap<type_info::name(), Standard_Type*>
using registry_type = NCollection_DataMap<const char*, Standard_Type*, typeNameHasher>;

// 函数内静态变量 —— 首次访问时构造，程序退出时最后销毁
registry_type& GetRegistry() {
    static registry_type theRegistry(2048, NCollection_BaseAllocator::CommonBaseAllocator());
    return theRegistry;
}

// 初始化种子 —— 确保 Standard_Transient 的描述符最先被注册
occ::handle<Standard_Type> theType = STANDARD_TYPE(Standard_Transient);
```

**注册流程（`Standard_Type::Register()`）：**

```cpp
Standard_Type* Standard_Type::Register(
    const std::type_info&             theInfo,       // C++ 原生 RTTI
    const char*                       theName,       // 用户类名
    size_t                            theSize,       // sizeof(T)
    const occ::handle<Standard_Type>& theParent)     // 父类描述符
{
    static std::mutex           aMutex;              // 线程安全的单次注册
    std::lock_guard<std::mutex> aLock(aMutex);

    registry_type& aRegistry = GetRegistry();
    Standard_Type* aType;

    // 步骤 1：如果已注册过（例如跨 DLL 边界重复调用），直接返回已有实例
    if (aRegistry.Find(theInfo.name(), aType))
    {
        return aType;
    }

    // 步骤 2：计算两块字符串的长度
    const size_t anInfoNameLen = strlen(theInfo.name()) + 1;
    const size_t aNameLen      = strlen(theName) + 1;

    // 步骤 3：一次 malloc 分配 Standard_Type + 两个字符串的连续内存
    //   内存布局: [Standard_Type | type_info::name() 深拷贝 | 用户类名 深拷贝 ]
    char* aMemoryBlock =
        static_cast<char*>(Standard::AllocateOptimal(sizeof(Standard_Type) + anInfoNameLen + aNameLen));

    char* anInfoNameCopy = aMemoryBlock + sizeof(Standard_Type);
    char* aNameCopy      = anInfoNameCopy + anInfoNameLen;
    strncpy(anInfoNameCopy, theInfo.name(), anInfoNameLen);
    strncpy(aNameCopy, theName, aNameLen);

    // 步骤 4：在已分配内存上 placement new 构造 Standard_Type
    aType = new (aMemoryBlock) Standard_Type(anInfoNameCopy, aNameCopy, theSize, theParent);

    // 步骤 5：以 type_info::name() 为键插入注册表
    aRegistry.Bind(anInfoNameCopy, aType);
    return aType;
}
```

**注册时机图：**

```
程序启动
  │
  ├── 静态初始化：GetRegistry() 创建空 DataMap (容量 2048)
  │   └── STANDARD_TYPE(Standard_Transient) 触发 Standard_Transient 描述符注册（根节点）
  │
  ├── 首次使用某类型时：例如 Geom_Circle::get_type_descriptor()
  │   │  static handle<Standard_Type> 触发 Register()
  │   │  ├── 递归触发父类 Geom_Surface::get_type_descriptor()
  │   │  │   ├── 递归触发 Standard_Transient::get_type_descriptor() → 已存在，返回
  │   │  │   └── 注册 Geom_Surface，parent = Standard_Transient
  │   │  └── 注册 Geom_Circle，parent = Geom_Surface
  │   └── 返回 static handle（此后每次调用都返回同一实例）
  │
  └── 程序退出时：static handle 析构 → Standard_Type 析构 → 从注册表 UnBind
```

**析构流程：**

```cpp
Standard_Type::~Standard_Type()
{
    registry_type& aRegistry = GetRegistry();
    Standard_ASSERT(aRegistry.UnBind(mySystemName),
                    "Standard_Type::~Standard_Type() cannot find itself in registry",
                    Standard_VOID_RETURN);
}
```

当程序退出时，各编译单元中的 `static handle<Standard_Type>` 析构，触发 `Standard_Type` 的 `Delete()`，最终调用析构函数从全局注册表中移除自身。断言确保注册表状态一致。

### 2.4 SubType() 子类型判定算法

`SubType()` 是整个 RTTI 查询的核心，负责判断当前类型是否"是（或继承自）"目标类型：

```cpp
// 按指针判断（通过 handle）
bool Standard_Type::SubType(const occ::handle<Standard_Type>& theOther) const
{
    if (theOther.IsNull())
        return false;

    const Standard_Type* aTypeIter = this;

    // 沿父类链向上遍历
    while (aTypeIter && theOther->mySize <= aTypeIter->mySize)
    {
        if (theOther.get() == aTypeIter)   // 指针比较 —— 每类只有一个描述符实例
            return true;
        aTypeIter = aTypeIter->Parent().get();  // 获取父类描述符
    }
    return false;
}

// 按字符串名判断
bool Standard_Type::SubType(const char* const theName) const
{
    if (!theName)
        return false;

    const Standard_Type* aTypeIter = this;
    while (aTypeIter)
    {
        if (IsEqual(theName, aTypeIter->Name()))
            return true;
        aTypeIter = aTypeIter->Parent().get();
    }
    return false;
}
```

**算法特点：**

| 特性 | 说明 |
|------|------|
| **指针比较** | 每种类型只有一个 `Standard_Type` 实例 → 指针相等即类型匹配，O(1) 逐级判定 |
| **size 剪枝** | `theOther->mySize <= aTypeIter->mySize`：子类的 sizeof 必然不小于父类，不满足则跳过，加速遍历 |
| **单继承** | 每个 `Standard_Type` 只有一个 `myParent`，形成严格的线性链 |
| **最坏复杂度** | O(继承深度)，OCCT 建模类层次通常 < 10 层 |

**查询示意图：**

```
                         [X] 表示类型为 X 的对象

Geom_Circle 继承链:   Standard_Transient  →  Geom_Curve  →  Geom_Conic  →  Geom_Circle
                         mySize=8              mySize=24      mySize=32      mySize=48

对于 Geom_Circle 对象调用 IsKind(STANDARD_TYPE(Geom_Curve)):
  SubType(Geom_Curve):
    Iteration 1: aTypeIter=Geom_Circle(48)  ≠ Geom_Curve(24)  → 48 ≥ 24, 上溯父类
    Iteration 2: aTypeIter=Geom_Conic(32)   ≠ Geom_Curve(24)  → 32 ≥ 24, 上溯父类
    Iteration 3: aTypeIter=Geom_Curve(24)   =  Geom_Curve(24)  → 匹配, 返回 true ✓
```

### 2.5 宏体系与 Standard_Type 的衔接

回顾 `DEFINE_STANDARD_RTTIEXT` 宏展开后与 `Standard_Type` 的交互：

```
DEFINE_STANDARD_RTTIEXT(Geom_Circle, Geom_Conic)
  │
  ├── static get_type_descriptor()
  │     ├── 内部 static handle<Standard_Type> THE_TYPE_INSTANCE
  │     ├── 首次调用时 → Standard_Type::Register(typeid(Geom_Circle), "Geom_Circle",
  │     │                                          sizeof(Geom_Circle),
  │     │                                          Geom_Conic::get_type_descriptor())
  │     │   ├── Register() 检查注册表 → 未注册则分配 + placement new Standard_Type
  │     │   ├── 插入 DataMap <type_info::name(), Standard_Type*>
  │     │   └── 返回 Standard_Type* → 赋值给 handle<Standard_Type> 管理生命周期
  │     └── 后续调用直接返回 static handle（线程安全的静态局部变量）
  │
  ├── virtual DynamicType() const override
  │     └── 返回 get_type_descriptor() → 虚函数分发，派生类返回各自描述符
  │
  └── static_assert(std::is_base_of<Geom_Conic, Geom_Circle>::value)
        └── 编译期验证继承关系，多重继承或错误基类会编译失败
```

**两个宏变体的区别：**

| 宏 | 场景 | get_type_descriptor() | DynamicType() |
|----|------|----------------------|---------------|
| `DEFINE_STANDARD_RTTI_INLINE` | 小型类/仅头文件 | `inline static` 在头文件 | `inline virtual` 在头文件 |
| `DEFINE_STANDARD_RTTIEXT` + `IMPLEMENT_STANDARD_RTTIEXT` | 常规类 | 声明在头文件，定义在 .cxx | 声明在头文件，定义在 .cxx |

### 2.6 现代 C++ 模板接口

`Standard_Type.hxx` 底部（第 211-266 行）定义了 `occ` 命名空间下的模板辅助函数，提供比旧式宏更类型安全的语法：

```cpp
namespace occ {

// 判断对象是否为 T 类型或其派生类型（IsKind 的现代版本）
template <class T>
inline bool is_kind(const Standard_Transient* theObject) {
    return theObject != nullptr && theObject->IsKind(T::get_type_descriptor());
}

template <class T, class TBase>
inline bool is_kind(const handle<TBase>& theObject) {
    return !theObject.IsNull() && theObject->IsKind(T::get_type_descriptor());
}

// 判断对象是否恰好为 T 类型（IsInstance 的现代版本）
template <class T>
inline bool is_instance(const Standard_Transient* theObject) {
    return theObject != nullptr && theObject->IsInstance(T::get_type_descriptor());
}

template <class T, class TBase>
inline bool is_instance(const handle<TBase>& theObject) {
    return !theObject.IsNull() && theObject->IsInstance(T::get_type_descriptor());
}

// 获取类型 T 的描述符（等价于 STANDARD_TYPE(T)）
template <class T>
inline const handle<Standard_Type>& type_of() {
    return T::get_type_descriptor();
}

} // namespace occ
```

**新旧写法对比：**

```cpp
// 旧式宏写法（兼容，沿用至今）
if (aCurve->IsKind(STANDARD_TYPE(Geom_Circle))) { ... }
Handle(Geom_Circle) aCircle = Handle(Geom_Circle)::DownCast(aCurve);

// 现代模板写法（OCCT 8.0+，推荐）
if (occ::is_kind<Geom_Circle>(aCurve)) { ... }
auto aCircle = occ::down_cast<Geom_Circle>(aCurve);
```

### 2.7 设计要点总结

1. **全局唯一性**：每种类型只有**一个** `Standard_Type` 实例，注册表用 `type_info::name()` 做键保证跨编译单元去重。即使同一个类在多个 `.dll` / `.so` 中各自有 `static handle`，`Register()` 中的互斥锁确保仅首次调用创建实例，后续返回同一指针。

2. **连续内存分配**：`Standard_Type` 对象和两个字符串（`mySystemName`、`myName`）在一次 `AllocateOptimal` 中分配，减少 `malloc` 调用次数并提高缓存局部性。需用 placement new 构造对象，析构后整块释放。

3. **静态初始化顺序安全**：通过 `GetRegistry()` 的函数内静态变量模式（Meyers' Singleton）保证注册表在首次访问前构造。`theType` 全局变量作为初始化种子，确保 `Standard_Transient`（所有类的根）的描述符最先注册。

4. **线程安全**：`Register()` 内部使用 `std::mutex`，但日常的 `get_type_descriptor()` 调用无锁（返回 `static` 变量引用）。多个线程同时首次访问同一类型时，只有一个线程执行注册，其余在 `static` 初始化处阻塞等待。

5. **单继承限制**：整个 RTTI 体系只支持单根继承链。`DEFINE_STANDARD_RTTIEXT` 宏中的 `static_assert(std::is_base_of<Base, Class>::value)` 在编译期即拒绝多重继承的类。`SubType()` 的父链遍历也天然不处理多父类情形。

---

## 3. Handle 智能指针体系演变

### 3.1 旧宏体系：`Handle(Class)`

OCCT 早期版本（<7.0）使用预处理器宏定义智能指针类型：

```cpp
#define Handle(Class) opencascade::handle<Class>
```

用法：

```cpp
Handle(Geom_Circle) aCircle = new Geom_Circle(anAxis, aRadius);
Handle(Standard_Transient) anObj = aCircle;  // 向上转型
```

每个类还需要配套的宏来生成 `Handle_XXX` 兼容别名：

```cpp
// 为瞬态类生成 Handle_XXX typedef (已废弃)
#define DEFINE_STANDARD_HANDLE(C1, C2)
//    → 生成: typedef Handle(C1) Handle_C1;  // 标记为 deprecated

// 为持久类生成 Handle_XXX typedef (已废弃)
#define DEFINE_STANDARD_PHANDLE(C1, C2)
//    → 生成: typedef Handle(C1) Handle_C1;  // 标记为 deprecated
```

`Handle(Class)` 宏至今仍在 OCCT 8.0 中被大量使用以保持向后兼容，核心定义位于 `Standard_Handle.hxx:448`：

```cpp
#define Handle(Class) opencascade::handle<Class>
```

兼容宏文件的现状：`Standard_DefineHandle.hxx`（40 行）中所有旧宏（`IMPLEMENT_STANDARD_HANDLE`、`IMPLEMENT_STANDARD_RTTI` 等）已被**定义为空**，仅做占位用途以保证旧的包含文件不会报错。

### 3.2 现代模板：`occ::handle<T>`

#### 3.2.1 完整模板定义

`opencascade::handle<T>` 是一个**侵入式智能指针**，灵感来自 `boost::intrusive_ptr`。完整定义在 `Standard_Handle.hxx:52-401`。

**核心存储：**

```cpp
template <class T>
class handle {
private:
    Standard_Transient* entity;   // 仅存储一个指向基类的裸指针！
    // ...
};
```

关键点：模板参数 `T` 仅用于编译期类型安全，**运行时只存储一个 `Standard_Transient*` 指针**。所有 handle 实例都是相同大小（一个指针），无论指针指向什么类型。

#### 3.2.2 生命周期管理

```cpp
// 构造函数 —— 从原始指针获取所有权
handle(const T* thePtr) : entity(const_cast<T*>(thePtr)) {
    BeginScope();  // 调用 entity->IncrementRefCounter()
}

// 拷贝构造函数
handle(const handle& theHandle) : entity(theHandle.entity) {
    BeginScope();
}

// 移动构造函数 —— 转移所有权，不修改引用计数
handle(handle&& theHandle) noexcept : entity(theHandle.entity) {
    theHandle.entity = nullptr;
}

// 析构函数
~handle() {
    EndScope();  // 调用 DecrementRefCounter()，若归零则 Delete()
}
```

**Assign（赋值）核心逻辑：**

```cpp
void Assign(Standard_Transient* thePtr) {
    if (thePtr == entity) return;   // 自赋值检查
    EndScope();                      // 释放旧对象
    entity = thePtr;                // 指向新对象
    BeginScope();                   // 增加新对象的引用计数
}

void BeginScope() {
    if (entity != nullptr)
        entity->IncrementRefCounter();    // relaxed atomic increment
}

void EndScope() {
    if (entity != nullptr && entity->DecrementRefCounter() == 0)
        entity->Delete();                 // delete this
    entity = nullptr;
}
```

#### 3.2.3 类型转换

**向上转型（派生→基类）**：依据 `OCCT_HANDLE_NOCAST` 宏有两种策略：

- **默认（`reinterpret_cast` 方案）**：提供 `operator const handle<Base>&()` 隐式类型转换操作符，通过 `reinterpret_cast` 将 `handle<Derived>*` 转换为 `handle<Base>*`。得益于所有 handle 大小相同且内部仅存 `Standard_Transient*`，这是安全的。
- **`OCCT_HANDLE_NOCAST` 方案**：禁用隐式转换操作符，改为提供泛型拷贝构造函数和赋值操作符，使用 SFINAE + `std::enable_if` 确保只能从派生类型构造。

**向下转型（基类→派生类）**：

```cpp
// 经由 dynamic_cast 实现安全转换
template <class T2>
static handle DownCast(const handle<T2>& theObject) {
    return handle(dynamic_cast<T*>(const_cast<T2*>(theObject.get())));
}
```

现代写法：

```cpp
auto aLine = occ::down_cast<Geom_Line>(aCurve);   // 安全下行
```

#### 3.2.4 类型别名层

```cpp
namespace occ {
    template <class T>
    using handle = opencascade::handle<T>;     // 现代别名 (推荐)
}

#define Handle(Class) opencascade::handle<Class>  // 旧宏 (兼容)
```

三者等价：

```cpp
occ::handle<Geom_Circle>         a1;  // 推荐：现代 C++ 写法
opencascade::handle<Geom_Circle> a2;  // 完整限定名
Handle(Geom_Circle)              a3;  // 旧写法，向后兼容
```

#### 3.2.5 std::hash 支持

允许 handle 直接用于 `std::unordered_map` 等容器：

```cpp
namespace std {
    template <class TheTransientType>
    struct hash<Handle(TheTransientType)> {
        size_t operator()(const Handle(TheTransientType)& theHandle) const noexcept {
            return static_cast<size_t>(reinterpret_cast<std::uintptr_t>(theHandle.get()));
        }
    };
}
```

### 3.3 演进总结

| 时期 | Handle 实现 | 特点 |
|------|------------|------|
| OCCT < 7.0 | `Handle_Class` 单独生成的类 | 每类一对 .hxx / .cxx 文件，手动写实现 |
| OCCT 7.0 ~ 7.9 | `opencascade::handle<T>` | 统一模板，`Handle(Class)` 宏作为别名 |
| OCCT 8.0+ | `occ::handle<T>`（推荐） | `occ` 命名空间别名，`Handle(Class)` 标记 deprecated |

---

## 4. Standard_Persistent — 持久化对象根基类

### 4.1 定位与原始用途

`Standard_Persistent` 是 OCCT 旧版面向对象数据库（OODB）存储机制的持久对象根基类。

**源文件位置：**
- `src/FoundationClasses/TKernel/Standard/Standard_Persistent.hxx` — 类声明（44 行）
- `src/FoundationClasses/TKernel/Standard/Standard_Persistent.cxx` — 仅 17 行（只含一行 `IMPLEMENT_STANDARD_RTTIEXT` 宏）

**类定义：**

```cpp
//! Root of "persistent" classes, a legacy support of
//! object oriented databases, now outdated.
class Standard_Persistent : public Standard_Transient
{
public:
    DEFINE_STANDARD_ALLOC
    DEFINE_STANDARD_RTTIEXT(Standard_Persistent, Standard_Transient)

    Standard_Persistent() : _typenum(0), _refnum(0) {}

    int& TypeNum() { return _typenum; }

private:
    int _typenum;   // 类型编号 —— 存储时标识对象类型
    int _refnum;    // 引用编号 —— 存储时建立对象间的引用关系

    friend class Storage_Schema;  // 允许 Storage_Schema 直接访问 _refnum
};
```

**两个关键整数字段：**

| 字段 | 用途 |
|------|------|
| `_typenum` | 类型编号。通过 `TypeNum()` 公开读写。存储/检索时标识对象的运行时类型。 |
| `_refnum` | 引用编号。私有成员，仅通过友元 `Storage_Schema` 访问。存储时唯一标识每个对象实例，并用于序列化对象间的指针引用关系。 |

### 4.2 Storage 子系统集成

`Standard_Persistent` 与 `Storage_Schema` 紧密协作，构成 OCCT 的**二进制持久化框架**：

```
Standard_Transient
  ├── Standard_Persistent          ← 持久对象根
  │     └── PCDM_Document          ← OCAF 文档基类（唯一的直接子类）
  └── Storage_Schema               ← 存储/检索算法引擎
        └── (无子类，直接实例化)
```

**Storage_Schema 如何使用 Standard_Persistent 的字段：**

```cpp
// 写入持久对象头部（类型 + 引用编号）
void WritePersistentObjectHeader(const occ::handle<Standard_Persistent>& sp,
                                 const occ::handle<Storage_BaseDriver>& theDriver) {
    theDriver->WritePersistentObjectHeader(sp->_refnum, sp->_typenum);
}

// 写入对象引用（写 _refnum 或 0 表示空引用）
void WritePersistentReference(const occ::handle<Standard_Persistent>& sp,
                              const occ::handle<Storage_BaseDriver>& theDriver) {
    theDriver->PutReference(sp.IsNull() ? 0 : sp->_refnum);
}
```

**其他依赖 `handle<Standard_Persistent>` 的核心组件：**

| 组件 | 文件 | 角色 |
|------|------|------|
| `Storage_Root` | `Storage_Root.hxx:93` | 持有 `handle<Standard_Persistent> myObject`，封装作为存储根的对象 |
| `Storage_CallBack` | `Storage_CallBack.hxx:31-40` | 抽象回调接口，所有方法均接受 `handle<Standard_Persistent>` 参数 |
| `Storage_DefaultCallBack` | `Storage_DefaultCallBack.cxx:25-43` | 默认回调实现，"未知类型"的回退处理 |
| `Storage_Data` | `Storage_Data.hxx:165-175` | 存储数据集，`AddRoot()` 接受 `handle<Standard_Persistent>` |
| `Storage_InternalData` | `Storage_InternalData.hxx:36-53` | 内部数据管理，持有 `HArray1<handle<Standard_Persistent>>` 数组 |
| `FSD_BinaryFile` | `FSD_BinaryFile.cxx:688-713` | 二进制文件驱动，使用 `handle<Standard_Persistent>` 数组 |
| `Storage_Macros.hxx` | 全文 | 存储框架宏，大量使用 `handle<Standard_Persistent>` 做参数类型 |

### 4.3 历史演变

通过 Git 历史可见：

```
14d4e91171  Coding - Global Refactoring OCCT as a part of 8.0.0 (#955)
5647b46a34  Configuration - Reorganize repository structure #450
```

`Standard_Persistent` 自 OCCT 最初版本即存在（Matra Datavision 时期），是早期 OCAF（Open CASCADE Application Framework）使用面向对象二进制数据库（如 ObjectStore）时期的产物。随着 OCAF 逐步迁移到基于 XML 和二进制文件的存储方式，`Standard_Persistent` 的作用被大幅削弱。

### 4.4 当前状态：名义存活，实质废弃

#### 继承链极端萎缩

在整个 OCCT 8.0 代码库中，`Standard_Persistent` 仅有一个直接子类：

```
Standard_Transient
  └── Standard_Persistent                (2 个私有字段: _typenum, _refnum)
        └── PCDM_Document   [叶子节点]   (空类，仅有 RTTI 宏)
```

`PCDM_Document` 本身是一个**完全空的类**：

```cpp
class PCDM_Document : public Standard_Persistent
{
public:
    DEFINE_STANDARD_RTTIEXT(PCDM_Document, Standard_Persistent)
};
```

没有任何额外成员或方法。且 `PCDM_Document` **没有任何子类**。

#### Storage 框架仍在使用

`Storage_Schema` 仍在 **4 处**被直接实例化：

| 文件 | 行号 |
|------|------|
| `src/ApplicationFramework/TKCDF/PCDM/PCDM_StorageDriver.cxx` | 41 |
| `src/ApplicationFramework/TKCDF/PCDM/PCDM_ReadWriter_1.cxx` | 276 |
| `src/ApplicationFramework/TKCDF/PCDM/PCDM_ReadWriter_1.cxx` | 402 |
| `src/ApplicationFramework/TKCDF/PCDM/PCDM_ReadWriter_1.cxx` | 453 |

`handle<Standard_Persistent>` 仍在 Storage 模块的 **42 处**被作为参数/返回/成员类型使用。

#### 评价

`Standard_Persistent` 处于一种矛盾的状态：

- **代码注释**明确标明："legacy support of object oriented databases, now outdated"
- **运行时**仍被 Storage 框架通过 `handle<Standard_Persistent>` 泛型操作序列化对象
- **继承树**已退化到仅有一根独苗（`PCDM_Document`），且该子类为空
- 没有新的类需要继承它，它几乎纯粹作为 Storage 框架的类型标记存在

可以说 `Standard_Persistent` 是**运行时存活、架构上已废弃**的典型，其功能已由 OCAF 的 XML/二进制存储方案替代，但作为底层 Storage 框架的类型系统标记不得不继续保留。

---

## 5. 总结对比

| 维度 | Standard_Transient | Standard_Type | Standard_Persistent |
|------|-------------------|---------------|---------------------|
| **定位** | 瞬态对象根（运行时） | RTTI 类型描述符中枢 | 持久对象根（存储/检索） |
| **继承关系** | 独立基类 | 继承自 Standard_Transient | 继承自 Standard_Transient |
| **核心能力** | 引用计数 + RTTI 接口 | 类型注册表 + 父链查询 + SubType | 父类能力 + 类型/引用编号 |
| **子类数量** | ~953 个类 | 与 Standard_Transient 子类 1:1 | 1 个（PCDM_Document） |
| **活跃度** | 核心基础设施 | 核心基础设施 | 名义存活，实质废弃 |
| **关键接口** | IncrementRefCounter / DynamicType | Register / SubType / Instance | TypeNum / _refnum (友元访问) |
| **文件大小** | 145 + 76 行 | 268 + 160 行 | 44 + 17 行 |

**关键设计理念：**

- **侵入式引用计数**：引用计数器内嵌在 `Standard_Transient` 对象自身中，`handle<T>` 存储的是裸指针，无需额外堆分配。这使得 `sizeof(handle<T>) == sizeof(void*)`，在传递和存储 handle 时极为高效。OCCT 整个建模引擎（Geom、TopoDS、BRep）大量使用 handle 传递共享对象，这一设计决定了 OCCT 的内存管理范式。

- **单继承 RTTI**：OCCT 的 RTTI 系统通过编译期 `static_assert(std::is_base_of)` 保证类型的正确性，运行时 `SubType()` 沿父类链遍历，支持类型查询和动态转换，但**不支持多重继承**——这在 OCCT 的建模类层次中是足够且刻意简化的选择。

- **全局注册表 + 懒初始化**：`Standard_Type::Register()` 维护的全局 DataMap 确保每种类型只有唯一的描述符实例。通过函数内静态变量 + 互斥锁实现线程安全的懒初始化，日常查询（`get_type_descriptor()`）完全无锁。连续内存分配（描述符与字符串名字一次 malloc）减少碎片并提高缓存友好性。

- **持久化退化为标记**：`Standard_Persistent` 的两个整数字段 `_typenum` 和 `_refnum` 曾是 OODB 存储时代与数据库引擎交互的关键，但随着 OCAF 转向文件格式存储，这些字段的实际语义已经稀释，类本身仅作为类型标记继续存在。

---

## 6. One More Thing — 为什么 OCCT 不使用 std::shared_ptr

### 6.1 核心问题：为何不用 std::shared_ptr？

更深层的问题是：既然 C++11 有了 `std::shared_ptr`，OCCT 为何坚持侵入式方案？

**二者存储结构的根本差异：**

```
std::shared_ptr<T> (非侵入式):              OCCT handle<T> (侵入式):

  shared_ptr<T>                            handle<T>
  ┌──────────────┐                         ┌────────────────┐
  │ T* 对象指针   │ ─────────────────→ T对象 │ Standard_Transient* │ ──→ T对象
  │ 控制块指针    │ ──→ ┌─────────────┐     └────────────────┘      ┌──────────────┐
  └──────────────┘     │ 强引用计数=2  │                           │ myRefCount_=2 │ ← 在对象内部
                       │ 弱引用计数=1  │                           │ ...T的成员... │
                       │ 删除器/分配器  │                           └──────────────┘
                       └─────────────┘
  sizeof = 16 字节 (x64)                   sizeof = 8 字节 (x64)
```

引用计数的存储位置不同，这决定了以下四个方面的优劣。

### 6.2 理由一：内存开销

OCCT 的 `TopoDS_Shape` 是 handle 大户——一个复杂模型可能同时存在百万级 handle。

```
一百万 handle 的内存占用：
  OCCT handle<T>:          100万 ×  8 字节 =  8 MB
  std::shared_ptr<T>:      100万 × 16 字节 = 16 MB
  差额：                                     8 MB 常驻内存
```

每个 handle 省 8 字节。OCCT 中每个几何对象（点、边、面、体）都由 handle 持有，大量算法临时持有 handle 进行计算。在内存敏感的大型模型处理场景（百万面片），这是一个显著的常数因子差异。此外，handle 越窄，CPU 缓存能容纳的就越多，间接提升计算吞吐。

### 6.3 理由二：裸指针与 handle 混用的安全性（最关键）

这是最具技术深度的原因。考虑这个场景：

```cpp
// ===== ⚠️ std::shared_ptr 方案 =====
shared_ptr<Geom_Circle> aCircle = make_shared<Geom_Circle>(...);
Geom_Curve* pCurve = aCircle.get();        // 裸指针（无计数）
shared_ptr<Geom_Curve> bCurve(pCurve);     // 💀 创建了第二个控制块！
// 控制块 1 (aCircle): 强引用=1, 弱引用=0
// 控制块 2 (bCurve): 强引用=1, 弱引用=0
aCircle.reset();  // 控制块 1 的强引用→0, delete Geom_Circle 对象
bCurve->Method(); // 对已释放内存调用 → 未定义行为/崩溃

// 正确的 shared_ptr 做法需要 enable_shared_from_this:
class Geom_Circle : public enable_shared_from_this<Geom_Circle> { ... };
shared_ptr<Geom_Curve> bCurve = aCircle->shared_from_this(); // ✓ 但前提是 aCircle 存在

// ===== ✓ OCCT handle<T> 方案 =====
Handle(Geom_Circle) aCircle = new Geom_Circle(...);
Standard_Transient* pCurve = aCircle.get();   // 裸指针
Handle(Geom_Curve) bCurve(pCurve);            // ✓ 直接工作！同享 myRefCount_
// 因为计数器在对象内部，无论通过哪个 handle 创建，都操作同一个 myRefCount_
// enable_shared_from_this 的等价物天然内建于 Standard_Transient
```

OCCT 的建模算法频繁需要在裸指针和 handle 之间转换。侵入式设计让"裸指针→handle"的转换天然安全——因为计数器始终在对象体内，任何一个 handle 的构造都能找到它。而 `shared_ptr` 裸指针转 handle 是经典的 UB 陷阱，需要 `enable_shared_from_this` 且仅对已被 `shared_ptr` 持有的对象有效。

**OCCT 中的典型场景：**

```cpp
// 算法内部使用裸指针，避免 refcount 操作开销
void SomeAlgorithm(const Standard_Transient* pObj) {
    // ... 复杂计算 ...
    // 需要延长生命周期时，安全地转为 handle
    Handle(Standard_Transient) aHandle(pObj);  // ✓ 直接工作
}

// 如果用 shared_ptr，上述写法会导致 double delete
```

### 6.4 理由三：零额外堆分配（构造函数代价）

```cpp
// 三种创建方式的 malloc 次数：
new Geom_Circle(...);                // 1 次 malloc：对象 + myRefCount_（内置）
make_shared<Geom_Circle>(...);       // 1 次 malloc：对象 + 控制块（make_shared 合并分配）
shared_ptr<Geom_Circle>(new ...);    // 2 次 malloc：对象 + 控制块（分离）
```

OCCT 常规用法 `new T` 即为 1 次分配；`shared_ptr` 只有 `make_shared` 才能做到同等效果。然而 `make_shared` 有析构时机问题——弱引用存在时对象内存不能释放。对于几何建模这种高频创建/销毁对象的场景，每次减少一次堆分配有实际性能意义。

### 6.5 理由四：历史与现实——OCCT 比 C++11 早了约 15 年

OCCT 的 `Standard_Transient` 源自 1990 年代 Matra Datavision 时期，当时还没有 `std::shared_ptr`（C++11 于 2011 年标准落地）。当 C++11 出现时，OCCT 的侵入式系统已经稳定运行近 20 年，约 953 个类依赖它。

更重要的是，即使站在今天的视角重新评估：对于 OCCT 这种**高性能几何内核、裸指针和 handle 密集交织**的场景，侵入式方案仍是更优解：

| 维度 | OCCT 侵入式 handle | std::shared_ptr |
|------|-------------------|-----------------|
| `sizeof(smart_ptr)` | 8 字节 | 16 字节 |
| 裸指针 → 智能指针 | 天然安全 | 需 `enable_shared_from_this` |
| 每次 `new T` 的 malloc | 1 次 | 2 次（或 `make_shared`） |
| 控制块分配 | 无（内嵌对象） | 额外分配 |
| `weak_ptr` 支持 | 无（设计选择） | 有 |
| 多重继承类型转换 | 支持（`reinterpret_cast` 策略） | 有限制 |

### 6.6 这个模式叫什么？

这和 COM 的 `IUnknown::AddRef()/Release()` + `CComPtr`、`boost::intrusive_ptr` 是同一模式——**侵入式引用计数 + 外部 RAII 壳**。

```
框架       | 被管理对象            | RAII 壳        | 对标
───────────┼───────────────────────┼────────────────┼────────
OCCT       | Standard_Transient    | handle<T>      | 本文主题
COM/WinRT  | IUnknown              | CComPtr        | Windows 组件模型
Boost      | intrusive_ptr_add_ref | intrusive_ptr  | Boost 侵入式指针
Linux      | kref (struct kref)    | 手动 get/put   | Linux 内核对象
```

核心原则：**对象提供计数原语，智能指针提供 RAII 语义——职责分离，各司其职。**

---

> **下一篇文章预告 — OCCT 中 gp 与 Geom 两套类型的设计原理**
