# Scope-based Resource Management
* **基于作用域的资源管理（Scope-based Resource Management）** 在 v6.5 中引入，其原理是利用 GCC 和 Clang 的 `type my_var __attribute__((__cleanup__(cleanup_func)))` 的语法，在变量离开作用域时自动调用其清理函数（类似 C++ 中的“析构函数”）的功能，实现内存的自动释放或者锁的自动释放，从而显著减少了函数中 `goto` 的使用。

## 内核中的 cleanup 功能

## 从 cleanup 功能到 class

* 如果有定义
```c
DEFINE_GUARD(mutex, struct mutex *, mutex_lock(_T), mutex_unlock(_T)); //定义了基于作用域的 class mutex
guard(mutex)(&uclamp_mutex);                                           //使用 class mutex 创建一个匿名实例
```

* 对于为锁准备的宏定义 `DEFINE_GUARD()/guard()`
```c
#define __DEFINE_CLASS_IS_CONDITIONAL(_name, _is_cond)  \
static __maybe_unused const bool class_##_name##_is_conditional = _is_cond

#define DEFINE_GUARD(_name, _type, _lock, _unlock) \
    __DEFINE_CLASS_IS_CONDITIONAL(_name, false); \
    DEFINE_CLASS(_name, _type, if (_T) { _unlock; }, ({ _lock; _T; }), _type _T); \
    static inline void * class_##_name##_lock_ptr(class_##_name##_t *_T) \
    { return (void *)(__force unsigned long)*_T; }

#define guard(_name) \
    CLASS(_name, __UNIQUE_ID(guard))
```

* 第一次宏展开为
```c
static __maybe_unused const bool class_mutex_is_conditional = false;
DEFINE_CLASS(mutex, struct mutex *, if (_T) { mutex_unlock(_T); }, ({ mutex_lock(_T); _T; }), struct mutex * _T); \
static inline void * class_mutex_lock_ptr(class_mutex_t *_T) \
{ return (void *)(__force unsigned long)* _T; }

CLASS(mutex, __UNIQUE_ID(guard))(&uclamp_mutex);
```

* 对于通用的 `DEFINE_CLASS()/CLASS()` 的宏定义
```c
#define DEFINE_CLASS(_name, _type, _exit, _init, _init_args...)     \
typedef _type class_##_name##_t;                    \
static inline void class_##_name##_destructor(_type *p)         \
{ _type _T = *p; _exit; }                       \
static inline _type class_##_name##_constructor(_init_args)     \
{ _type t = _init; return t; }

#define CLASS(_name, var)                       \
    class_##_name##_t var __cleanup(class_##_name##_destructor) =   \
        class_##_name##_constructor

#define __cleanup(func)         __attribute__((__cleanup__(func)))

#define ___PASTE(a,b) a##b
#define __PASTE(a,b) ___PASTE(a,b)
#define __UNIQUE_ID(prefix) __PASTE(__PASTE(__UNIQUE_ID_, prefix), __COUNTER__)
```

* 第二次宏展开并格式化之后为
```c
static __maybe_unused const bool class_mutex_is_conditional = false;

typedef struct mutex * class_mutex_t;

static inline void class_mutex_destructor(struct mutex * *p)
{
    struct mutex * _T = *p;
    if (_T) {
        mutex_unlock(_T);
    };
}
static inline struct mutex * class_mutex_constructor(struct mutex * _T)
{
    struct mutex * t = ({ mutex_lock(_T); _T; });
    return t;
}
static inline void * class_mutex_lock_ptr(class_mutex_t *_T)
{
    return (void *)(__force unsigned long)* _T;
}

class_mutex_t __UNIQUE_ID_guard__COUNTER__ __attribute__((__cleanup__(class_mutex_destructor))) =
    class_mutex_constructor(&uclamp_mutex);
```
* 最后展开看到的变量名 `__UNIQUE_ID_guard__COUNTER__` 其实不是这样的，`__COUNTER__` 是 [GCC 预定义宏](https://gcc.gnu.org/onlinedocs/cpp/Common-Predefined-Macros.html)，每次展开时自动递增的一个整型数（从 0 开始），与 `##` 结合可以组装成一个唯一的变量名，详细解释如下：
> `__COUNTER__`
>
> This macro expands to sequential integral values starting from `0`. In conjunction with the `##` operator, this provides a convenient means to generate unique identifiers. Care must be taken to ensure that `__COUNTER__` is not expanded prior to inclusion of precompiled headers which use it. Otherwise, the precompiled headers will not be used.

* 前面的 `__UNIQUE_ID_` 不是 GCC 内建宏或扩展，而是为了防止变量名冲突而人为添加的前缀，主要还是靠后面的 `__COUNTER__` 的宏扩展。
  * `__UNIQUE_ID(prefix)` 定义最早引入于 [Re_ [PATCH] utilize _Static_assert() for BUILD_BUG_ON() when the compiler supports it - Rusty Russell](https://lore.kernel.org/all/87ehk6443a.fsf@rustcorp.com.au/)
  * commit 6f33d58794ef4cef4b2c706029810f9688bd3026 ("__UNIQUE_ID()")

### 限定范围的锁资源管理
* `scoped_guard (name, args...) { }` 与 `CLASS(name, scope)(args)` 类似，不同之处在于变量（具有明确的名称“`scope`”）在 `for` 循环中声明，以便其范围绑定到下一个（复合）语句。
* 比如对于宏的使用 `scoped_guard(raw_spinlock, &pctrl->lock)`
  * drivers/pinctrl/intel/pinctrl-intel.c
```c
static irqreturn_t intel_gpio_irq(int irq, void *data)
{
...
        scoped_guard(raw_spinlock, &pctrl->lock) {
            pending = readl(is);
            enabled = readl(reg);
        }
...
}
```
* `scoped_guard()` 的宏定义如下：
```c
#define __guard_ptr(_name) class_##_name##_lock_ptr
#define __is_cond_ptr(_name) class_##_name##_is_conditional

/*
 * Helper macro for scoped_guard().
 *
 * Note that the "!__is_cond_ptr(_name)" part of the condition ensures that
 * compiler would be sure that for the unconditional locks the body of the
 * loop (caller-provided code glued to the else clause) could not be skipped.
 * It is needed because the other part - "__guard_ptr(_name)(&scope)" - is too
 * hard to deduce (even if could be proven true for unconditional locks).
 */
#define __scoped_guard(_name, _label, args...)              \
    for (CLASS(_name, scope)(args);                 \
         __guard_ptr(_name)(&scope) || !__is_cond_ptr(_name);   \
         ({ goto _label; }))                    \
        if (0) {                        \
_label:                                 \
            break;                      \
        } else

#define scoped_guard(_name, args...)    \
    __scoped_guard(_name, __UNIQUE_ID(label), args)
```
* 该宏展开并格式化后如下：
```c
static irqreturn_t intel_gpio_irq(int irq, void *data)
{
...
    for (class_raw_spinlock_t scope __attribute__((__cleanup__(class_raw_spinlock_destructor))) = class_raw_spinlock_constructor(&pctrl->lock); 
         class_raw_spinlock_lock_ptr(&scope) || !class_raw_spinlock_is_conditional;
         ({ goto __UNIQUE_ID_label522; }))
        if (0) {
__UNIQUE_ID_label522:
            break; //自动解锁
        } else {
            pending = readl(is);
            enabled = readl(reg);
        }  //自动解锁
...
}
```
* 自旋锁 `class_raw_spinlock_t scope` 的作用域范围在 `for` 循环里，`for` 循环初始化时加锁，循环以任意一种形式结束时解锁。
* 其中用到的 `CLASS(raw_spinlock)` 即 `class_raw_spinlock_t` 在 include/linux/spinlock.h 中定义
```c
DEFINE_LOCK_GUARD_1(raw_spinlock, raw_spinlock_t,
            raw_spin_lock(_T->lock),
            raw_spin_unlock(_T->lock))
```
* 这个宏定义在 include/linux/cleanup.h 中定义
```c
#define __DEFINE_UNLOCK_GUARD(_name, _type, _unlock, ...)       \
typedef struct {                            \
    _type *lock;                            \
    __VA_ARGS__;                            \
} class_##_name##_t;                            \
                                    \
static inline void class_##_name##_destructor(class_##_name##_t *_T)    \
{                                   \
    if (_T->lock) { _unlock; }                  \
}                                   \
                                    \
static inline void *class_##_name##_lock_ptr(class_##_name##_t *_T) \
{                                   \
    return (void *)(__force unsigned long)_T->lock;         \
}


#define __DEFINE_LOCK_GUARD_1(_name, _type, _lock)          \
static inline class_##_name##_t class_##_name##_constructor(_type *l)   \
{                                   \
    class_##_name##_t _t = { .lock = l }, *_T = &_t;        \
    _lock;                              \
    return _t;                          \
}

#define DEFINE_LOCK_GUARD_1(_name, _type, _lock, _unlock, ...)      \
__DEFINE_CLASS_IS_CONDITIONAL(_name, false);                \
__DEFINE_UNLOCK_GUARD(_name, _type, _unlock, __VA_ARGS__)       \
__DEFINE_LOCK_GUARD_1(_name, _type, _lock)
```
* 展开该宏并格式化之后
```c
static __attribute__((__unused__)) const bool class_raw_spinlock_is_conditional = false;
typedef struct {
    raw_spinlock_t *lock;
    /*__VA_ARGS__*/;
} class_raw_spinlock_t;

static inline __attribute__((__gnu_inline__)) __attribute__((__unused__)) __attribute__((no_instrument_function))
void class_raw_spinlock_destructor(class_raw_spinlock_t *_T)
{
    if (_T->lock) {
        _raw_spin_unlock(_T->lock);
    }
}

static inline __attribute__((__gnu_inline__)) __attribute__((__unused__)) __attribute__((no_instrument_function))
void *class_raw_spinlock_lock_ptr(class_raw_spinlock_t *_T)
{
    return (void *)( unsigned long)_T->lock;
}

static inline __attribute__((__gnu_inline__)) __attribute__((__unused__)) __attribute__((no_instrument_function))
class_raw_spinlock_t class_raw_spinlock_constructor(raw_spinlock_t *l)
{
    class_raw_spinlock_t _t = { .lock = l }, *_T = &_t;
    _raw_spin_lock(_T->lock);
    return _t;
}
```

#### 带尝试版本的变体
* `CLASS(raw_spinlock)` 还有带尝试版本的变体 `raw_spinlock_try`
```c
DEFINE_LOCK_GUARD_1_COND(raw_spinlock, _try, raw_spin_trylock(_T->lock))
```
* 它用以下宏：
```c
#define typeof_member(T, m) typeof(((T*)0)->m)

#define raw_spin_trylock(lock)  __cond_lock(lock, _raw_spin_trylock(lock))

#define EXTEND_CLASS(_name, ext, _init, _init_args...)          \
typedef class_##_name##_t class_##_name##ext##_t;           \
static inline void class_##_name##ext##_destructor(class_##_name##_t *p)\
{ class_##_name##_destructor(p); }                  \
static inline class_##_name##_t class_##_name##ext##_constructor(_init_args) \
{ class_##_name##_t t = _init; return t; }


#define DEFINE_LOCK_GUARD_1_COND(_name, _ext, _condlock)        \
    __DEFINE_CLASS_IS_CONDITIONAL(_name##_ext, true);       \
    EXTEND_CLASS(_name, _ext,                   \
             ({ class_##_name##_t _t = { .lock = l }, *_T = &_t;\
                if (_T->lock && !(_condlock)) _T->lock = NULL;  \
            _t; }),                     \
             typeof_member(class_##_name##_t, lock) l)      \
    static inline void * class_##_name##_ext##_lock_ptr(class_##_name##_t *_T) \
    { return class_##_name##_lock_ptr(_T); }
```
* 该宏展开并格式化后如下：
```c
static __attribute__((__unused__)) const bool class_raw_spinlock_try_is_conditional = true;
typedef class_raw_spinlock_t class_raw_spinlock_try_t;

static inline __attribute__((__gnu_inline__)) __attribute__((__unused__)) __attribute__((no_instrument_function))
void class_raw_spinlock_try_destructor(class_raw_spinlock_t *p)
{
    class_raw_spinlock_destructor(p);
}

static inline __attribute__((__gnu_inline__)) __attribute__((__unused__)) __attribute__((no_instrument_function))
class_raw_spinlock_t class_raw_spinlock_try_constructor(typeof(((class_raw_spinlock_t*)0)->lock) l)
{
    class_raw_spinlock_t t = ({
        class_raw_spinlock_t _t = { .lock = l }, *_T = &_t;
        if (_T->lock && !((_raw_spin_trylock(_T->lock))))
            _T->lock = ((void *)0); //注意这里，尝试加锁失败会把 .lock 与置零，作为标志
        _t;
    });
    return t;
}

static inline __attribute__((__gnu_inline__)) __attribute__((__unused__)) __attribute__((no_instrument_function))
void * class_raw_spinlock_try_lock_ptr(class_raw_spinlock_t *_T)
{
    return class_raw_spinlock_lock_ptr(_T);
}
```
* 对应地，使用的地方也应该是带尝试版本的 `scoped_guard(raw_spinlock_try) { do_something; }`，然而，我并没有找到类似使用的地方，只能伪造一个并展开如下：
```c
for (class_raw_spinlock_try_t scope __attribute__((__cleanup__(class_raw_spinlock_try_destructor)))
        = class_raw_spinlock_try_constructor(&pctrl->lock); //只前尝试加锁失败会导致 .lock 为零，赋给 scope.lock
     class_raw_spinlock_try_lock_ptr(&scope) //尝试加锁失败会导致 scope.lock = 0，该条件为 false，走后面的“或”
        || !class_raw_spinlock_try_is_conditional; //class raw_spinlock_try 的 class_raw_spinlock_try_is_conditional 是 true，最终整个表达式为 false
     ({ goto __UNIQUE_ID_label522; }))
    if (0) {
__UNIQUE_ID_label522: //加锁成功，完成一次循环后，从 for 循环的第三个条件跳转到这里
        break;        //随后结束循环，离开作用域会自动释放锁
    } else {
        do_something; //尝试拿锁成功才会走到这里
    }
```
* 可见尝试加锁失败时并不会走进循环体
* 类似的 `class` 还有用宏 `DEFINE_GUARD_COND()` 定义的 `mutex_try` 和 `mutex_intr`
```c
DEFINE_GUARD_COND(mutex, _try, mutex_trylock(_T))
DEFINE_GUARD_COND(mutex, _intr, mutex_lock_interruptible(_T) == 0)
```

#### 加锁失败返回错误的条件版本

* 此外，在带尝试版本的基础上，还有加锁失败返回错误的版本 `scoped_cond_guard()`，它使用的 `class lock` **必须** 是带条件版本的，比如 `raw_spinlock_try`、`mutex_intr`、`iio_claim_direct_try`，否则会编译出错
* 其宏定义为：
```c
# define __compiletime_assert(condition, msg, prefix, suffix)       \
    do {                                \
        /*                          \
         * __noreturn is needed to give the compiler enough \
         * information to avoid certain possibly-uninitialized  \
         * warnings (regardless of the build failing).      \
         */                         \
        __noreturn extern void prefix ## suffix(void)       \
            __compiletime_error(msg);           \
        if (!(condition))                   \
            prefix ## suffix();             \
    } while (0)

#define _compiletime_assert(condition, msg, prefix, suffix) \
    __compiletime_assert(condition, msg, prefix, suffix)

#define compiletime_assert(condition, msg) \
    _compiletime_assert(condition, msg, __compiletime_assert_, __COUNTER__)

#define BUILD_BUG_ON_MSG(cond, msg) compiletime_assert(!(cond), msg)

#define BUILD_BUG_ON(condition) \
    BUILD_BUG_ON_MSG(condition, "BUILD_BUG_ON failed: " #condition)

#define __scoped_cond_guard(_name, _fail, _label, args...)      \
    for (CLASS(_name, scope)(args); true; ({ goto _label; }))   \
        if (!__guard_ptr(_name)(&scope)) {          \
            BUILD_BUG_ON(!__is_cond_ptr(_name));        \
            _fail;                      \
_label:                                 \
            break;                      \
        } else

#define scoped_cond_guard(_name, _fail, args...)    \
    __scoped_cond_guard(_name, _fail, __UNIQUE_ID(label), args)
```
* `scoped_cond_guard()` 用的地方不多，我这里伪造了一个 `scoped_cond_guard(raw_spinlock, return -EINTR, &pctrl->lock) { do_something; }` 宏，其展开的结果看看逻辑就好了，实际上并没有这个宏，而且编译到这个地方的时候会出错
```c
for (class_raw_spinlock_t scope __attribute__((__cleanup__(class_raw_spinlock_destructor)))
        = class_raw_spinlock_constructor(&pctrl->lock);
     true; //都会走进循环体
     ({ goto __UNIQUE_ID_label522; }))
    if (!class_raw_spinlock_lock_ptr(&scope)) { //pctrl->lock 的值为 NULL 时会走进这个条件
        do {
            __attribute__((__noreturn__)) extern void __compiletime_assert_523(void)
                __attribute__((__error__("BUILD_BUG_ON failed: " "!__is_cond_ptr(raw_spinlock)")));
            if (!(!(!class_raw_spinlock_is_conditional))) //由于使用的是不带尝试的版本的 class raw_spinlock，此条件为 true 导致出错
                __compiletime_assert_523();
        } while (0);
        return -4;
__UNIQUE_ID_label522:
        break;
    } else {
        do_something;
    }
```
* 改用  `scoped_cond_guard(raw_spinlock_try, return -EINTR, &pctrl->lock) { do_something; }` 宏就没问题了，展开为：
```c
for (class_raw_spinlock_try_t scope __attribute__((__cleanup__(class_raw_spinlock_try_destructor)))
        = class_raw_spinlock_try_constructor(&pctrl->lock); //只前尝试加锁失败会导致 .lock 为零，赋给 scope.lock
    true; //都会走进循环体
    ({ goto __UNIQUE_ID_label522; }))
    if (!class_raw_spinlock_try_lock_ptr(&scope)) {//尝试加锁失败会导致 scope.lock = 0，整个条件为 true，走进其管辖的语句
        do {
            __attribute__((__noreturn__)) extern void __compiletime_assert_523(void)
                __attribute__((__error__("BUILD_BUG_ON failed: " "!__is_cond_ptr(raw_spinlock_try)")));
            if (!(!(!class_raw_spinlock_try_is_conditional))) //由于使用的是带尝试的版本的 class raw_spinlock_try，整个条件为 false，不会出错
                __compiletime_assert_523();
        } while (0);
        return -4;    //尝试拿锁失败，返回约定好的错误码
__UNIQUE_ID_label522: //加锁成功，完成一次循环后，从 for 循环的第三个条件跳转到这里
        break;        //随后结束循环，离开作用域会自动释放锁
    } else {
        do_something; //尝试拿锁成功才会走到这里
    }
```

## References
- [Scope-based resource management for the kernel [LWN.net]](https://lwn.net/Articles/934679/)
- [[PATCH v3 00/57] Scope-based Resource Management](https://lore.kernel.org/all/20230612090713.652690195@infradead.org/)
- [Scope-based Cleanup Helpers — The Linux Kernel documentation](https://docs.kernel.org/next/core-api/cleanup.html)
- [Linux Kernel Development - Automatic Cleanup 1_2 – Cycling and Coding Around the World](http://hackerbikepacker.com/kernel-auto-cleanup-1)
- [Linux Kernel Development - Automatic Cleanup 2_2 – Cycling and Coding Around the World](https://hackerbikepacker.com/kernel-auto-cleanup-2)
- [GCC - Common Predefined Macros (The C Preprocessor)](https://gcc.gnu.org/onlinedocs/cpp/Common-Predefined-Macros.html)
