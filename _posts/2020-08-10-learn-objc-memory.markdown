---
layout: post
title:      "「知识梳理」Objc-内存"
subtitle:   ""
date:       2020-08-30 00:00:00
author:     "DEEP"
header-img: "img/home-bg.jpg"
catalog: true
tags:
    -   知识梳理
    -   iOS
---

各种书籍与博文层出不穷，其中书籍往往比较系统，而源码与底层相关的知识一般分散于各种博文。

这里主要做博文的碎片知识整理。

## 逆向

### 基础

- [砸壳流程](https://hello-david.github.io/archives/82a2b295.html)
- p12 是什么？ipa 包的结构？（是 ipa 整个被私钥 L 签名，Provisioning Profile = [Entitlements + 证书] 被公钥 A 签名）
- 简述证书和打包流程，以及重签名流程。
- [Mach-O 文件](https://github.com/Desgard/iOS-Source-Probe/blob/master/C/mach-o/Mach-O%20%E6%96%87%E4%BB%B6%E6%A0%BC%E5%BC%8F%E6%8E%A2%E7%B4%A2.md)
- Segment 和 Section？TEXT 和 DATA 段是什么？
- [fishhook 原理](https://github.com/Desgard/iOS-Source-Probe/blob/master/C/fishhook/%E5%B7%A7%E7%94%A8%E7%AC%A6%E5%8F%B7%E8%A1%A8%20-%20%E6%8E%A2%E6%B1%82%20fishhook%20%E5%8E%9F%E7%90%86%EF%BC%88%E4%B8%80%EF%BC%89.md)

## Runtime

### 类

#### `isa_t`

- [isa 结构](https://github.com/draveness/analyze/blob/master/contents/objc/%E4%BB%8E%20NSObject%20%E7%9A%84%E5%88%9D%E5%A7%8B%E5%8C%96%E4%BA%86%E8%A7%A3%20isa.md)
- isa 和 objc_object 结构？isa 有几种类型？（isa 是一个 64 位 union，objc_object 是包含它的 struct；类型有 Tagged、nonpointer、pointer）（也有给 watchos 32 位用的 isa，包含 indexcls）
- nonpointer 怎么优化存储指针地址的位数？（64 位 CPU，每次访问内存读 8 字节数据，指针为对齐，起始地址必是 8 的倍数）（利用了类地址都以 8 字节对齐，然后 ARM64 64 位指针实际只用了 33 位，只需存储 30 位；另外系统强制规定，实例对象地址都以 16 字节对齐）
- isa 和 superclass 的那张图画一下？（子元类 isa-> 根元类-> 根元类，根元类 superclass-> 根类-> nil）
- 引用计数是怎么优化的？（Tagged，内容存储在指针里，如 NSNumber，而 NSString 会被 clang 尽可能弄成 clang，计数无意义；nonpointer，extra_rc 存计数，不够则标记并开额外的 SideTable）（额外的 SideTable 和普通 isa，地址作为 key 找到计数值）
- [Tagged 源码](https://www.jianshu.com/p/3176e30c040b) ｜ [Tagged 字符串](https://swift.gg/2018/10/08/tagged-pointer-strings/)
- 

#### `objc_category`

- [category 原理](https://tech.meituan.com/2015/03/03/diveintocategory.html)
- 为什么 category 不能添加实例变量？（运行期决议，对象内存布局已确定）
- category 的编译流程。（初始化结构 category_t，包含方法、属性、协议列表，用类和分类拼接命名 static 变量，将所有分类串成离了表，加载入 DATA 段）
- dyld 加载 category 流程。（读取 DATA 段，插入对应类，新方法插入类方法列表前面，所以不是覆盖；分类 load 顺序和编译顺序相同）

#### `objc_class`

- [class 结构](https://github.com/draveness/analyze/blob/master/contents/objc/%E6%B7%B1%E5%85%A5%E8%A7%A3%E6%9E%90%20ObjC%20%E4%B8%AD%E6%96%B9%E6%B3%95%E7%9A%84%E7%BB%93%E6%9E%84.md)
- objc_class 结构？（类在编译期就确定内存的位置，包含 isa 8、superclass 8、cache_t 16、class_data_bits_t 8，共 40）
- bits 的结构？（状态位 + class_rw_t，class_rw_t 包含 class_ro_t 和方法、协议、属性列表，class_ro_t 还包括 ivar 列表）
- class_rw_t、class_ro_t 的初始化过程？（编译期 ro 作为 class_data_bits_t.data，realizeClass 后才新加空 rw 切换，methodizeClass 再填充 rw）
- extension 是编译期决议，它与实现文件一起形成完整的类，跟随类的生亡，必须有类源码才能加。
- [成员变量实质](http://quotation.github.io/objc/2015/05/21/objc-runtime-ivar-access.html)
- Non Fragile ivars 机制有什么意义？（编译后的库内存布局已定，若 NSObject 以后增加变量，则库的内存布局无效）（ivar_t 有 offset 记录偏移值，实际取的时候再移动偏移）
- LLVM 是如何优化变量寻址的？ivar.offset 为什么是 int 指针？（`*((&obj->isa.cls->data()->ro->ivars->first)[N]->offset)` ，太慢了，因此使用了全局变量记录每个类对应的每个变量的偏移）（指针指向全局变量，编译期就能确定）
- 如何动态添加成员变量？（只能在创建类期间添加，不然，会影响已经创建的类实例的工作）
- [消息发送+缓存](https://tech.meituan.com/2015/08/12/deep-understanding-object-c-of-method-caching.html) ｜ [源码分析](https://github.com/Desgard/iOS-Source-Probe/blob/master/Objective-C/Runtime/objc_msgSend%E6%B6%88%E6%81%AF%E4%BC%A0%E9%80%92%E5%AD%A6%E4%B9%A0%E7%AC%94%E8%AE%B0%20-%20%E5%AF%B9%E8%B1%A1%E6%96%B9%E6%B3%95%E6%B6%88%E6%81%AF%E4%BC%A0%E9%80%92%E6%B5%81%E7%A8%8B.md) ｜ [最详细](http://yulingtianxia.com/blog/2016/06/15/Objective-C-Message-Sending-and-Forwarding/)
- 为什么方法是列表，不是散列表？（配合分类机制，有顺序的，且节省空间）
- 方法缓存在哪里？在父类找到的方法会存到子类吗？缓存大小的机制？（类的 cache_t，会，奇数次满会清空并扩大）
- [还可以怎么优化消息发送？](http://www.mulle-kybernetik.com/artikel/Optimization/opti-3-imp-deluxe.html)（使用 CF 函数；使用函数指针调用，包括 objc_msgSend 的调用；如果是不同类的相同方法调用，则可以自己做多个缓存 IMP 列表，用 isa 作 key）
- [给 objc_msgSend 画个图？](https://github.com/yulingtianxia/yulingtianxia.github.io/blob/master/resources/MessageForward/%E6%B6%88%E6%81%AF%E5%8F%91%E9%80%81%E4%B8%8E%E8%BD%AC%E5%8F%91%E8%B7%AF%E5%BE%84%E6%B5%81%E7%A8%8B%E5%9B%BE.jpg)
- 为什么 objc_msgSend 要用汇编实现？（1、不能定义一个 C 函数，可以有可变的参数的同时，调用任意的 C 函数指针，因为函数指针类型是无法预先全部定义出来；

  2、可以免去局部变量拷贝的操作，即免去 prologue 和 epilogue 机制，参数会直接被存放在寄存器中，当找到 IMP 时，可以直接使用。）
- 查找：（编译器决定用 objc_msgSendSuper，objc_msgSend_stret 等）（汇编内找 cache -> lookupImpOrForward 不找 cache -> 【cahce ｜本类方法列表｜父类】 -> 是否动态解析过 -> 去解析，回到 lookupImpOrForward）
- 转发：（还是没有就返回 IMP _objc_msgForward_impcache） -> （决定用 _objc_msgForward 等） -> （调用 handler，默认实现是 crash + log，__CFInitialize 时又换掉默认实现） -> （forwarding，methodSignture 未实现或返回空时，doesNotRecoginze）
- 三个转发过程：动态方法 resolveMethod、简单转发的 forwarding target、自定义转发 methodSignature + Invocation。
- msgSend 前置汇编实现已经查找了 cache，lookUpImpOrForward 传入 cache == NO。

### 各种函数

#### 基础轮子

- [比较详细](http://www.cocoachina.com/articles/29199#cocoachina12)
- LoadExclusive & ClearExclusive：原子读（arm 实现是 ldxr、clrex 指令加解锁），StoreExclusive：`__c11_atomic_compare_exchange_weak((_Atomic(uintptr_t) *)dst, &oldvalue, value`，dst == oldvalue，则 dst = value；否则 oldvalue = dst，返回 false。
- addc、subc：`__builtin_addcl(lhs, rhs, carryin, carryout);`
- slowpath、fastpath：__builtin_expect，汇编指令优化，都是判真，但 slow 表示假的概率大。
```c
#define SIDE_TABLE_WEAKLY_REFERENCED (1UL << 0)  // 是否有过 weak
#define SIDE_TABLE_DEALLOCATING      (1UL << 1)  // 是否销毁中
#define SIDE_TABLE_RC_ONE            (1UL << 2)  // 第一个计数位
#define SIDE_TABLE_RC_PINNED         (1UL <<(WORD_BITS-1)) // 最高位是溢出位

#define SIDE_TABLE_RC_SHIFT 2
#define SIDE_TABLE_FLAG_MASK (SIDE_TABLE_RC_ONE-1)

#define RC_HALF (1ULL << 7)       // extra_rc 半满，x64 是 8 位
````
- __sync_synch：调用后，禁止操作内存运算符，要进行析构了。
- `overrelease_error`：过度 release，检查释放位已设置时调用。

#### SideTable

- [结构描述](https://blog.csdn.net/u013378438/article/details/82790332) 
- SideTables() 全局获取表，大小是 StripeCount（8 iPhone 其它 64），重载 operator[this]，通过 indexForPointer 获取对应 SideTable。
- 每个 SideTable 有一个 spinlock_t 非公平自旋锁，减小锁粒度；SideTable 有两个 hash 表：`typedef objc::DenseMap<DisguisedPtr<objc_object>（存储类型）, size_t（value）, true(value 0 时是否自动释放)> RefcountMap;` 和 `weak_table_t`。
- 轮子描述：
- `sidetable_lock`：根据 this 锁对应表。
- `sidetable_getExtraRC_nolock`：根据 this 获取对应计数。
- `sidetable_retainCount`：只有指针类型 isa 才会调，流程和上面一样，但返回结果加了个 1。
- `sidetable_retain`：根据 this 找到并写，加 1。
- `sidetable_tryRetain`：只有 `_objc_loadWeak` 通过 -> `rootTryRetain` -> `rootRetain(true, false)` 调到这里，说明 tryRetain 是 true 时，已经 lock 了 SideTable；流程是，尝试找到并加 1，如果找不到则设置为 1，如果释放中则返回 false。
- `sidetable_addExtraRC_nolock`：只有 extrarc 第一次满或 release 失败恢复时才调，让 SideTable 的引用计数加上参数值。
- 关于 `SIDE_TABLE_RC_PINNED`：已经溢出时，sidetable 的 retain 和 release 都不更改引用计数了，不再释放对象。

#### retain、release

- `rootRetain(bool tryRetain, bool handleOverflow)`：第一次非 weak 是 (false, false)；其实除开加解锁的代码，就两部分：若指针 isa，则 tryRetain 或 retain，否则处理 extrarc。
- `rootRetain_overflow(bool tryRetain, bool handleOverflow)`：rootRetain(tryRetain, true)；extrarc 溢出时调到这，然后重来（感觉没必要这么写），最后把 extrarc 拆成两半，一半给 SideTable。
- `rootRelease(bool tryRetain, bool handleOverflow)`：反操作，extrarc 减到借位时，borrow = sidetable_subExtraRC_nolock(RC_HALF)，如果大于 0 则继续。否则检查 dealloc 位，没问题则设置并 objc_msgSend。
- 可见优化 isa， SideTable 加和减都是用 RC_HALF 为单位的，平时都是 extrarc ++/--

#### weak_table

- [比较详细，数据结构+源码](https://juejin.cn/post/6844904023938564109)
- 小总结：referent 即 id，referrer 即 id *，referrers 即 哈希表或 inner 小数组；table 对多个 entry，entry 对一个 id 对多个 id *。
- 无论 weak_table 还是 entry 内的 weak_referrer_t *，对于删除碰撞位后，都不会设置哨兵，因为遍历时都是整个遍历完的。
- `DisguisedPtr`：封装的指针，方便地址和 unsigned long 互转；`weak_referrer_t`：指向 DisguisedPtr<objc_object *> 的指针，相当于 id *。
- `weak_enrty_t`：一个类似 id 的 referent，和一个 32 字节的两种形式的 union：inline 则为 4 个 weak_referrer_t；outline 为 1 个哈希表指针 weak_referrer_t *，1 位记录位和 63 位记录元素个数，还有两个指针：哈希 mask 和最大冲突数。
- 因为指针后 3 位都是 000，然后 1 位记录位可以区别出 inline 还是 outline。（函数 `entry->out_of_line()`）
- `weak_table_t`：就是典型的 hash，for weak_enrty_t，记录个数和最大长度 mask。
- [weak_系列方法源码](https://blog.csdn.net/u013378438/article/details/82790332)
- `weak_entry_for_referent(weak_table_t *, objc_object *)`：找到对应 entry，哈希 `hash_pointer` + 线性探测。
- `weak_entry_insert(weak_table_t *, weak_entry_t *)`：就普通的哈希插入 + 碰撞线行探测。
- `weak_entry_remove(weak_table_t *, weak_entry_t *)`：单纯地 remove 参数 entry。
- `weak_resize(weak_table_t *, size_t)`：创新大小的表，然后将旧的一个个 insert 进去。
- `weak_compact_maybe`：大于 1000 且 1/16 满时，缩小表大小为 1/8，也是通过 resize。
- `weak_grow_maybe`：3/4 满时，乘 2 倍，一开始是 64 个。
- `append_referrer`：就是维护哈希表，如果是 inner 满了则转成 outer 的哈希表，满了就 grow_refs_xxxx。
- `remove_referrer`：一样。
- `grow_refs_and_insert`：同 weak_groe_maybe，一开始是 8 个。
- 

#### weak 方法
- `weak_register_no_lock`：先判断是否释放中，分有无自定义 RR 两种情况；如果释放中则看参数 crashIfDeallocating 是否 crash；最后找找到 referrent 对应 entry，没有就创建，把 referrer 弄进去。
- `weak_unregister_no_lock`：简单地找到对应 entry，然后删掉里面对应地 referrer_id，如果 entry 空了则也顺便删掉。
- `storeWeak<DontHaveOld, DoHaveNew, DoCrashIfDeallocating> (location, (objc_object*)newObj)`：先按情况锁 old 和 new 表，如果有 New 且无元类则初始化（+initialize）；然后如果有 Old 则 unregister；然后如果有 New 则 register。


### ARC

#### 修饰符和编译期代码

- [理解 ARC](https://xietao3.com/2019/05/ARC/)
- `objc_storeStrong`：release 左参数旧值，retain 右参数新值并设置。
- `objc_initWeak`：DontHaveOld, DoHaveNew，DoCrashIfDeallocating。
- `objc_destroyWeak`：DoHaveOld, DontHaveNew，DontCrash。
- `clearDeallocating_slow`：dealloc 会调这个，清空对应的 SideTable 中的 refcnt、weak_table_entry。
- `objc_loadWeakRetained`：尝试强引用 weak 对应的对象并返回，无 customRR，则尝试 tryRetain，否则调自定义函数。
- ARC 如何自动加 storeStrong：
``` objc
// ARC 自动加代码示例
// void strongFunction() {
//     id obj = [NSObject new];
//     __strong id obj1 = obj;
void strongFunction() {
    id obj = obj_msgSend(NSObject, @selector(new));
    id obj1 = objc_retain(obj)
    objc_storeStrong(&obj, null); (id *location, id obj) {
        id prev = *location;
        if (obj == prev) {
            return;
        }
        objc_retain(obj);
        *location = obj;
        objc_release(prev);
    }
    objc_storeStrong(&obj1, null);
}

// void weakFunction() {
//     __weak id obj = [NSObject new];
// }
void weakFunction() {
  	id temp = objc_msgSend(NSObject, @selector(new));
    (id * obj)
  	objc_initWeak(obj, temp); (id *location, id newObj) {
        if (! newObj) {
            *location = nil;
            return nil;
        }
        return storeWeak <DontHaveOld, DoHaveNew, DoCrashIfDeallocating>
            (location, (objc_object*)newObj);
    }
  	objc_release(temp);
  	objc_destroyWeak(&obj);
}

// void weak1Function() {
//     id obj = [NSObject new];
//     __weak id obj1 = obj;
// }
void weak1Function() {
	id obj = objc_msgSend(NSObject, @selector(new));
    (id * obj1)
	objc_initWeak(obj1, obj);
	objc_destroyWeak(&obj1);
	objc_storeStrong(obj, null);
}

// void weak2Function() {
//     id obj = [NSObject new];
//     __weak id obj1 = obj;
//     NSLog(@"%@", obj1);
// }
void weak2Function() {
	id obj = objc_msgSend(NSObject, @selector(new));
    (id * obj1)
	objc_initWeak(obj1, obj);
    // 留意，LLVM 8.0 后，不再将 weak 的临时变量 autorelease，而是后面直接 release。
	id temp = objc_loadWeakRetained(obj1);
	NSLog(@"%@", temp);
	objc_release(temp);
	objc_destroyWeak(obj1);
	objc_storeStrong(obj, null);
}
```

#### Autorelease

- [Autorelease 源码](https://github.com/draveness/analyze/blob/master/contents/objc/%E8%87%AA%E5%8A%A8%E9%87%8A%E6%94%BE%E6%B1%A0%E7%9A%84%E5%89%8D%E4%B8%96%E4%BB%8A%E7%94%9F.md)
- 描述自动释放池原理？什么时候释放？（双向链表 AutoreleasePoolPage，hotPage 是当前活跃的 Page，每个 Page 是 4096 字节，和线程一一对应，哨兵是 nil）（每个 runloop 迭代中都加入了自动释放池 Push 和 Pop）
- `autorelease`：
- `objc_autoreleasePoolPush`：
- `objc_autoreleasePoolPop`：
- `__autoreleasing`：id * 的都被编译器隐式用其修饰，使对应对象已加入 Pool 中，将 `NSError *` 传入参数 `NSError **` 时，会自动修饰：
- 修饰符即调用 `objc_retainAutoreleaseAndReturn`，是 retain 再 autorelease，调用的是 `objc_retainAutorelease`。
- `objc_autoreleaseReturnValue` 和 `objc_retainAutoreleaseReturnValue`。
-  __

``` objc
// NSError *__autoreleasing error; 
// ￼ if (! [data writeToFile: filename options: NSDataWritingAtomic error:&error]) { 
// 　　NSLog(@"Error: %@", error); 
// }
{
    NSError *error; 
    NSError *__autoreleasing tempError = error; // 编译器添加 
    if (! [data writeToFile: filename options: NSDataWritingAtomic error:&tempError]) 
    ￼{ 
    　　error = tempError; // 编译器添加 
    　　NSLog(@"Error: %@", error); 
    }
}

// === 注意！！！有些方法会隐式用 @autoreleasepool ===
- (void)loopThroughDictionary:(NSDictionary *)dict error:(NSError **)error {
　　NSError * __block tempError; // 加 __ block 保证可以在 Block 内被修改。
　　[dict enumerateKeysAndObjectsUsingBlock:^(id key, id obj, BOOL *stop) { 
    // @autoreleasepool 隐式添加
       // 如果不用 __block，新创建的 NSError 出去就被 Pool 释放
　　　　if (there is some error) { 
　　　　　　*tempError = [NSError errorWithDomain:@"MyError" ￼ code: 1 userInfo: nil];
　　　　} ￼ 
　　}] 
　　if (error != nil) { 
　　　　*error = tempError; 
　　} ￼
}

```

``` objc
// 手动写 @autoreleasepool 时，实际代码
// @autoreleasepool {
//     __autoreleasing NSObject *a = [NSObject new];
// }
{
    void *token = objc_autoreleasePoolPush(); {
        autoreleaseFast(POOL_SENTINEL); {
            AutoreleasePoolPage *hotPage = hotPage();
            if (hotPage && ! hotPage-> full()) {
                return hotPage-> add(obj);
            } else if (hotPage) {
                return autoreleaseFullPage(obj, hotPage);
            } else {
                return autoreleaseNoPage(obj);
            }
        }
    }
    
    NSObject *a = [NSObject new];
    [a autorelease]; {
        if (isTaggedPointer()) return (id)this;
        if (prepareOptimizedReturn(ReturnAtPlus1)) return (id)this;
        autoreleaseFast(this)
    }
    
    objc_autoreleasePoolPop(token); (void * token) {
       // 地址是 4096 的整数倍，0x1000
        AutoreleasePoolPage *page = pageForPointer(token);
        id *stop = (id *)token;

        page-> releaseUntil(stop); (id *stop) {
            while (this-> next != stop) {
                AutoreleasePoolPage *hotPage = hotPage();

                while (hotPage-> empty()) {
                    hotPage = hotPage-> parent;
                    setHotPage(page);
                }

                hotPage-> unprotect();
                id obj = *--hotPage-> next;
                memset((void *)hotPage-> next, SCRIBBLE, sizeof(* hotPage-> next));
                hotPage-> protect();

                if (obj != POOL_SENTINEL) {
                    objc_release(obj);
                }
            }
            setHotPage(this);
        }

        if (page-> child) {
            if (page-> lessThanHalfFull()) {
                page-> child-> kill();
            } else if (page-> child-> child) {
                // 如果半满，则保留子 Page，宏观上优化效率
                page-> child-> child-> kill();
            }
        }
    }
}
```
- [autorelease 优化](http://blog.sunnyxx.com/2014/10/15/behind-autorelease/)
- 描述 Pool 与 ARC 相关的优化机制？（MRC 时代，alloc 等开头的返回方法谁创建谁释放，其它返回方法则认为已被注册到 Pool 中）（ARC 为适配它，会分别对应位置加 release 和 autorelease）
  
  （对如 array 的调用时，有外层和内层，内层返回时已 alloc、autorelease，返回的是持有它的临时变量，但临时变量在外层也被 retian、release，优化的就是内层的 autorelease 和 外层的 retain）
- `objc_autoreleaseReturnValue` 和 `objc_retainAutoreleaseReturnValue`


``` objc
// 优化的 Autorelease 示例
// id retArray() {
//     return [[NSMutableArray alloc] init];
// }
id retArray() {
    id obj = objc_msgSend(objc_msgSend(NSMutableArray, @selector(alloc)), @selector(init));
    objc_autoreleaseReturnValue(obj); (id obj) {
        if (prepareOptimizedReturn(ReturnAtPlus1)) return obj; (ReturnDisposition disposition) {
            // TLS 线程局部存储的封装
            ASSERT(getReturnDisposition() == ReturnAtPlus0);

            // __builtin_return_address 获取 caller 的下一个指令的地址
            if (callerAcceptsOptimizedReturn(__builtin_return_address(0))) {
                // callerAcceptsOptimizedReturn 尝试匹配下一个指令是 movq xxx
                // 并判断后面有无调用 objc_retainAutoreleasedReturnValue
                if (disposition) setReturnDisposition(disposition);
                return true;
            }

            return false;
        }
        return objc_autorelease(obj);
    }
    return obj;
}

// void noRetButUseArray() {
//     // array 可替换成 retArray()
//     __strong id obj = [NSMutableArray array];
// }
void noRetButUseArray() {
    id obj = objc_msgSend(objc_msgSend(NSMutableArray, @selector(array)));
    objc_retainAutoreleaseReturnValue(obj); (id obj) {
        if (acceptOptimizedReturn() == ReturnAtPlus1) return obj; {
            ReturnDisposition disposition = getReturnDisposition();
            setReturnDisposition(ReturnAtPlus0);
            return disposition;
        }
        return objc_retain(obj);
    }
    [obj release];
}
```

## Block
- 

### 杂项

- [JSCore](https://tech.meituan.com/2018/08/23/deep-understanding-of-jscore.html)
  - JS 运行机制。
  - 桥接原理。
  - JSPatch 桥接原理。


- [frames 和 bounds](https://www.dazhuanlan.com/2019/10/02/5d947f179c59c/?__cf_chl_jschl_tk__=818bf6884f3a78dc581681ed6ac386e99e3e708d-1600597908-0-AbMICLnXkCrcXmM85-QworPijg9cQB8ffRJ771jqCSA7yZnV-zUz6rEvmphMUyl6QFPAG94PIPtWQqpI9_JJiiDWhZnx0PFvp9stVJDgU50FnzZr8eMOSMx9Q6mWUiSbPD4a5bd4ZjsC81tCOFBBfL6cV6pHfXCrGz7zn3CYO61NKUTOc3BhWhgZs2iVbvn6LWKu64sqAQnR_Qc-3QliMrS2RcsBD8Z2C1QzdROG8ok_ZnBU4SKq4DNgjOp3XUIyRH2O02ia0A-pyVIvQSnxInKNJSGVrXHQiztrajPOngx0oDe14NdWZZq_Pm5ZJTiBog)