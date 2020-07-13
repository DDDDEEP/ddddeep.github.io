---
layout: post
title:      "阅读「Effective Objc」"
subtitle:   ""
date:       2020-07-07 00:00:00
author:     "DEEP"
header-img: "img/home-bg.jpg"
catalog: true
tags:
    -   读书笔记
    -   iOS
---

写给自己的话：

- 不要浪费时间复读知识，读书笔记的目的是可以以后快速回忆知识，详细的回忆还是要回归原书。
- 不要让写笔记的时间占用看更多书的时间，

## 一、 Objc

### 1、了解 Objc 起源

- Objc 是 C 的超集，运行时才检查对象类型，消息发送，具体运行代码由运行时决定。

### 2、类头文件尽量少引入其它头文件

- 尽量使用 `@class` 向前声明，避免循环引用。
- 协议难免要 `#import`，单独的协议尽量放在单一头文件再引入；委托协议则尽量在实现文件中，通过扩展声明并实现。
- （额外的，尽量不要在头文件声明类使用协议；`@protocol` 也不行，会找不到定义）

### 3、多用字面量语法

- 字符串、数值、数组、字典、下标、取键值。
- 字面量不能包含 `nil`。
- 创建的对象都是不可变的。

### 4、多用类型常量，少用 `#define`

- `static const NSTimeInterval kAnimationDuration = 0.3` 不可修改且有类型信息、只在当前编译单元生效、编译器不会创建「外部符号」。
- 全局变量，只需头文件内 `extern` 即可使用，编译器不用看定义，因为它知道链接成二进制后肯定能找到；只能定义一次。
    ```objc
    // header，用类名作前缀
    extern NSString *const EOCLoginManagerDidLoginString;

    // imp
    NSString *const EOCLoginManagerDidLoginString = @"string";
    ```

### 5、用枚举类

- C 风格：
    ```objc
    enum EOCConnectionState {
        EOCConnectionStateDisconnected,
        EOCConnectionStateConnected,
    };
    typedef enum EOCConnectionState EOCConnectionState; // 不然使用的时候要显式 enum
    ```
- C++ 增加了可以指明底层数据结构的特性。
- Foundation 框架定义了 `NS_ENUM`、`NS_OPTIONS` 宏，兼容了 C 和 C++ 不同的编译模式：
    ```objc
    typedef NS_ENUM(NSUInteger, EOCCnectionState) {
        EOCConnectionStateDisconnected,
        EOCConnectionStateConnected,
    };
    typedef NS_OPTIONS(NSUInteger, EOCPermittedDirection) {
        EOCPermittedDirectionUp   = 1 << 0,
        EOCPermittedDirectionDown = 1 << 1,
    };
    ```
- `switch` 使用枚举不能有 `default`。

## 二、对象、消息、运行期

### 6、理解属性

- 对象布局在编译期已经固定，访问实例变量时会替换为偏移码；这样如果修改了类定义，则需重新编译。
  
  Objc 的方案是在运行期才查找偏移值，并通过存取方法抽象查找偏移的过程。
- `@synthesize` 指定实例变量名字，`@dynamic` 指定不要自动合成方法。
- `@property (nonatomic, readonly, assign, getter=isON) BOOL on;`。
- iOS 的 `atomic` 开销大；另外其只保证原子性，若单线程连续读同一值的同时，其它线程写值，这种情况也不能保证线程安全。

### 7、对象内部尽量直接访问实例变量

- 优点：不经过方法派发效率更高。
- 缺点：绕过内存语义，copy 失效；不触发 KVO；不利于断点监测。
- `init` 及 `dealloc` 方法应直接读写实例变量；其它情况，折中方案是只在写方法用设置方法，但有特例：
  
  - 子类无法直接访问超类的实例变量。
  - 惰性初始化变量。

### 8、理解「对象等同性」（`isEqual`)

- `NSObject` 的两个方法：`isEqual`、`hash`，`hash` 返回值相等是 `isEqual` 相等的必要条件，默认实现一般只有指针相等才判断 `YES`。
- 自定义实现首先判断 `self == object`；然后实现 `hash`。

    collection 会使用 `hash` 返回值作索引，尽量减少碰撞，经典实现是用类属性异或起来，计算速度快。

    ```objc
    - (BOOL)isEqualToPerson:(EOCPerson*)otherPerson {
        if (self == otherPerson) return YES;
        ...
    }
    - (BOOL)isEqual:(id)object {
        if ([self class] == [object class]) {
            return [self isisEqualToPerson:(EOCPerson*)object];
        } else {
            return [super isEqual:object];
        }
    }
    ```
- 不要修改加入 collection 后的对象的内容，否则会出现 set 包含相同对象的问题。

### 9、用「类族模式（工厂模式？）」隐藏实现细节

- 可以将实现细节隐藏在一套简单的公共接口后面：
    ```objc
    @implementation EOCEmployee

    + (EOCEmployee*)employeeWithType:(EOCEmployeeType)type {
        switch (type) {
            case EOCEmployeeTypeDeveloper:
                return [EOCEmployeeDeveloper new];
                break;
            case EOCEmployeeTypDesigner:
                return [EOCEmployeeTypDesigner new];
                break;
        }
    }
    ```
    可以避免使用 if else。
- `NSArray` 和 `NSMutableArray` 也是用类族，因此 `class` 方法无意义，应用 `isKindOfClass`。

### 10、关联对象存放自定义数据（无法继承子类时使用）

- 一般需要在对象存放信息时，可以从对象的类继承一个子类；
  
  但有时候类的实例是某种机制创建出来的，无法继承子类，此时可以用关联对象。
- 一般 `UIAlertView` 这么做：

    ```objc
    - (void) askUserAQuestion 
    {
       ...
    }

    // protocol method
    - (void)alertView:(UIAlertView*)alertView
        clickedButtonAtIndex:(NSInteger)buttonIndex
    {
        if (buttonIndex == 0) {
            [self doCancel];
        } else {
            [self doContinue];
        }
    }
    ```
    这样方法分散，可以用 block 这么做，集中逻辑：
    ```objc
    - (void) askUserAQuestion
    {
       ...
       void (^block)(NSInteger) = ^(NSInteger buttonIndex) {
           if else...
       }
       objc_setAssociatedObject(...);
    }

    // protocol method
    - (void)alertView:(UIAlertView*)alertView
        clickedButtonAtIndex:(NSInteger)buttonIndex
    {
        void (^block)(NSInteger) = objc_getAssociatedObject(...);
        block(buttonIndex);
    }
    ```
    
- 很难调试，还有保留环问题，尽量别滥用，优先继承。

### 11、理解 `objc_msgSend`

- `void objc_msgSend(id self, SEL _cmd, ...)`，还有边界情况的 `objc_msgSend_stret`、`objc_msgSend_fpret`、`objc_msgSendSuper`。

    一旦找到调用的方法实现，则会「跳转过去」，真正的方法实现可视作 C 函数：
    
    `<return_type> Class_selector(id self, SEL _cmd, ...)`，其与 send 函数很像，可以进行「尾调用优化」。

- 「优化」是如果函数最后操作是调用另外一个函数，那么编译器会会生产跳转到另一函数的指令码，同时不会推入新的「栈帧」，这样可以减缓「栈溢出」。
- 传递消息过程：快速映射表、类方法列表、超类方法列表，找不到就消息转发。

### 12、理解「消息转发」机制

- **动态方法解析**：`+ (BOOL)resolveInstanceMethod:(SEL)selector`，返回能否新增一个实例方法处理 SEL，或 `resolveClassMethod:`。

    一般配合 `class_addMethod`、`@dynamic`。

- **备援接收者**：`- (id)forwardingTargetForSelector:(SEL)selector`，不能则返回 nil，该方法无法修改消息内容。
- **完整消息转发**：`- (void)forwardInvocation:(NSInvocation*)invocation`，一般同时会修改消息内容。

    如果不应由本类处理，则调用超类同名方法；`NSObject` 的同名方法实现会调用 `doesNotRecognizeSelector:`，抛出异常。

### 13、「方法调配技术」（hook）

- `id (*IMP)(id, SEL, ...)`
    ```objc
    Method originalMethod = class_getInstanceMethod([NSString class], @selector(lowercaseString));
    Method swappedMethod = class_getInstanceMethod([NSString class], @selector(uppercaseString));
    method_exchangeImplementations(originalMethod, swappedMethod);

    @interface NSString(EOCMyAdditions)
    - (NSString*)eoc_myLowercaseString;
    @end
    ```

### 14、理解「类对象」

- 各种 Runtime。
- `isMemeberOfClass`、`isKindOfClass`；类对象是单例的。

## 三、接口与 API 设计

### 15、用前缀避免命名空间冲突

- 实现文件的纯 C 函数、全局变量也是顶级符号。
- 用 3 个字母作所有类和纯 C 函数的前缀。
- 第三方库尽可能自修改名称，加上 3 个字母的前缀。（EOCABCClass）

### 16、提供「全能初始方法」

- 应该实现一个全能初始方法，所有其它初始方法都调用它。
- 对于子类继承，如果超类的全能方法不适用，则应该覆写它；可抛出异常。

    因为父类的实例变量只能通过父类方法初始化。
- `NSCoder` 也是如此：`initWithCoder`。

### 17、实现 `description` 方法

- 实现：
    ```objc
    (NSString*)description
    {
        return [NSStringstringWithFormat:@"<%@: %p, \"%@ %@\">", [self class], self, _firstName];
    }
    ```

- 还有 `debugDescription`

### 18、尽量使用不可变对象

- readonly 还是声明一下内存语义，方便以后修改。
- 当某属性仅可内部修改，可以在扩展将 readonly 改为 readwrite。
- 不要公开可变的 collection，而应该声明不可变属性，然后内部用可变 collection 提供相关方法：
  ```objc
  // header
  @property (copy, readonly) NSString *firstName;
  @property (strong, readonly) NSSet *friends;

  // imp
  @interface EOCPerson ()
  @property (copy, readwrite) NSString *firstName;
  @end

  @implementation EOCPerson
  {
      // 相当于 private
      NSMutableSet *_internalFriends;
  }
  - (NSSet*)friends
  {
      return [_internalFriends copy];
  }
  ```

### 19、使用清晰的命名方式

- 创建新返回值的方法，首个词应是返回值的类型或修饰。
- 参数类型名词放在参数前。
- 如果在对象上执行操作，则应包含动词。
- 不要用 str 等的简称。
- Boolean 属性加 is 前缀；Boolean 返回值方法用 has 或 is 前缀。
- get 前缀用于「输出参数」保存返回值的方法。
- 类与协议命名通顺。

### 20、为私有方法名加前缀

- 苹果官方一般用 _ 作前缀，我们一般用 p_。

### 21、理解错误模型（常用错误处理）

- 抛出异常要考虑释放资源的问题，所以一般抛出异常的场景是严重错误，宁愿终止程序，不考虑释放。
- 一般的错误，返回 nil / 0，或使用 `NSError`
    ```objc
    - (BOOL)doSomething:(NSError**)error
    {
        if (error occur) {
            if (error) { // 很重要！一定要判非空才解指针赋值
                *error = ...
            }
            return NO;
        } else {
            return YES;
        }
    }
    ```
    使用枚举表示错误码。
- 额外的：block 里面不要直接使用 `NSError** error` 进行 `*error = xxx`，应使用 `_block` 临时变量，跳出 block 后再赋值。

    [stackoverflow](https://stackoverflow.com/questions/45923106/call-method-that-assigns-nserror-in-block-crashes)
    
### 22、理解 NSCopying 协议

- `NSCopying`、`NSMutableCopying`，实现 `copyWithZone:`、`mutableCopyWithZone`。
- 特别注意，`copy` 只应返回不可变对象，`mutableCopy` 只应返回可变对象。
- `NSCopying` 并无指明浅拷贝、深拷贝，需要查询文档，可以考虑自定义 `deepCopy`，或 init 中增加 copyItems 参数（参考 `initWithSet:`）

## 四、协议与分类

### 23、通过委托与数据源协议进行对象间通信

- 委托对象（大对象？），其持有被委托对象（小对象？），`被委托对象.delegate = 委托对象`，`delegate` 属性是 `weak` 或 `unsafe_unretained`。
- 如果是委托协议，则一般在扩展声明即可；委托协议的方法一般是 `@optional`，配合 `respondsToSelector`。
- 委托协议命名：以被委托对象为基准；
  
  委托方法命名：应传入被委托对象参数，以识别不同的被委托对象。
- 信息流向：数据源 -> 委托对象 -> 被委托对象，一般数据源和被委托对象是同一个。
- `@optional` 的委托方法多次调用时，使用位段优化 `respondsToSelector`：

    ```objc
    @interface EOCNetworkFetcher ()
    {
        struct
        {
            unsigned int didReceiveData : 1;
            unsigned int didError : 1;
        } _delegateFlags
    }

    @imp
    // set 方法
    (void)setDelegate:(id<EOCNetworkFetcher>delegate)
    {
        _delegate = delegate;
        _delegateFlags.didReceiveData = [delegate respondsToSelector:@selector(networkFetcher:didReceiveData:)];
        ...
    }
    @end
    ```

### 24、将类的实现代码分散于便于管理的数个分类中

- 类的方法太多，拆成分类管理：
- 将私有方法放入命名 Private 的分类，隐藏实现细节；不对外公开这个头文件。
- `-[EOCPerson(Friendship) addFriend:]` 符号断点。

### 25、第三方分类名称（与方法）加前缀

- 多个分类添加重名方法，以最后一个加载的为准，难调试
    ```objc
    @interface NSString (EOC_HTTP)
    - (NSString*)eoc_urlEncodeString;
    @end
    ```

### 26、分类不要声明属性

- 编译器不能为分类中的属性合成实例变量；
    可以 `@dynamic`，也可以用关联对象，还可以用 readonly 属性。

### 27、「class-continuation 分类」隐藏实现细节（扩展）

- 可以声明属性（作为私有，不公开，但其实硬用运行期方法也能调出），与直接在 imp 声明实例变量相似。
- 兼容 C++ 头文件：
    ```objc
    // "EOCClass.h"
    // 这样不好，引入了该头文件后，文件会以 objc++ 编译（作为 .mm 文件）
    #include "SomeCppClass.h"
    @interface EOCClass : NSObject
    {
    @private
        SomeCppClass _cppCLass;
    }

    // 这样也不行，class 是 C++ 关键字
    class SomeCppClass;
        SomeCppClass *_cppClass;
    ```
    把它弄进扩展就很好。
- 扩展可将 readonly 改为 readwrite；

    将 p_ 系列私有方法在扩展声明是可选的，是很清晰的。

- 协议在头文件的类上前置声明无意义（无法找到定义），协议可以拖到扩展声明。

### 28、通过协议提供匿名对象

- 使用 id 修饰协议对象，隐藏实现细节。
- `@protocol` 前置声明适用于实例变量上：
    ```objc
    @protocol EOCDatabaseConnection;

    @interface
    - (id<EOCDatabaseConnection>)xxx;
    ```
## 五、内存管理

### 29、理解引用计数

- MRC 的老生常谈。

### 30、以 ARC 简化引用计数

- ARC 老生常谈。

### 31、在 `dealloc` 方法中只释放引用并解除监听

- `dealloc` 在应用程序强制终止时可能不会执行，因此自定义的 `close` 方法应自行另处调用，可以：
    ```objc
    - (void)close
    {
        _close = YES;
    }

    - (void)dealloc
    {
        if (!_closed) {
            ERROR; // 若 dealloc 是还没调用自定义 close，则应是严重的错误。
            [self close]; // 可选，其实 dealloc 里不应该调用其它方法:
        }
    }
    ```
- 不要在 `dealloc` 调用其它方法，因为此时某些对象可能已经被摧毁，引起错误。

    另外调用 `dealloc` 的线程是执行「final release」的线程，如果执行一些需特定线程的方法则可能出错。
- 不要调用属性存取方法，因为其可能被覆写或被 KVO，不安全。

### 32、编写「异常安全代码」时留意内存管理问题

- 在 Objc 中，原则上只有严重错误需要终止才应该抛出异常。
    ```objc
    // MRC
    EOCSomeClass *object;
    @try {
        object = [[EOCSomeClass alloc] init];
        [object doSomething];
    }
    @catch (...) {
        do Something;
    }
    @finally {
        [object release] // 维护太麻烦了
    }
    ```
    在 ARC 下，类似上面的代码，不会自动添加内存管理代码。

    因为需要加入大量样板代码，这些代码影响运行期的性能。

    可以使用 `-fobjc-arc-exceptions` 强制开启该功能。
- Objc++ 可以开启，因为 C++ 执行的代码与 ARC 附加代码类似，性能损失不大。

    而且 C++ 经常使用异常。

### 33、弱引用避免保留环

- weak 老生常谈。

### 34、以「自动释放池块」降低内存峰值

- autoreleasepool 老生常谈，可以 for 语句里面优化临时变量。

### 35、「僵尸对象」调试内存管理问题

- `export NSZombieEnabled="YES"`，僵尸对象会把释放后的类改变：

    `EOCClass` -> `_NSZombie_EOCCLass`

    新类是运行期生成的，其从 `_NSZombie_` 的模板类复制出来：

    ```objc
    ......
    ```
    
    运行期系统把 `dealloc` hook 成上面的代码，其修改对象的 `isa`，指向僵尸类。

    僵尸类的名字即体现其是僵尸，又体现它原来是哪个类。

### 36、不要用 `retainCount`

- 该方法只返回某个时间点的计数值，但其没有体现是否又自动释放操作。
- 该方法不会返回 0。
- 对于 `NSString`、`NSNumber`，因为使用了「标签指针」的优化概念，所以对应返回值没有意义；另外浮点数对象不会进行优化。
- 单例对象的保留、释放操作是「空操作」，但不同单例的计数也可能不一样，无意义。

## 六、块与大中枢派发

### 37、理解「块」

- block 的老生常谈。
- 提了一下 if else 的块赋值，因为块是对象，在 if else 里面的块默认是栈上的，要 `copy`。

### 38、为常用块类型创建 `typedef`

- 使用 `typedef` 除了舒服，当需要增加参数时，在 `typedef` 上增加，其它要改的地方就会显示出来。
- 可以为同一个 block 签名定义多个 `typedef`，维护性更好。

### 39、用 handler 块降低代码分散度

- block 与委托模式对比：代码更紧凑，

    另外如果有多种处理方法，委托模式要 if else 区分被委托对象；block 则更灵活。

- 设计的方法可加多个不同阶段的处理 block。处理 block 还带 `NSError *error` 参数。
    
    对比使用同一个 block，更清晰简洁；但使用同一个 block 可以有更多错误信息，处理更精细的错误。
- 基于 handler 设计 API，好处之一是可以传入 `NSOpertaionQueue` 参数，指定执行的 GCD 队列。

### 40、用块引用所属对象不要出现保留环

- 看下面：

    ```objc
    @implementation EOCNetworkFetcher
    - (void) startWithHandler:(Handler)completion
    {
        self.handler = completion;
    }

    - (void)p_requestCompleted
    {
        if (_handler) {
            _handler(_data);
        }
    }

    // 例 1，block -> EOCClass -> fetcher -> block
    @implementation EOCClass {
        EOCNetworkFetcher *_fetcher;
        NSData *_data;
    }

    - (void)test
    {
        _fetcher = [EOCNewworkFetcer new];
        [_fetcher startWithHandler:^(NSData *data) {
            NSLog(_fetcher.url);
            _data = data;
        }]；
    }

    // 例 2，执行 handler 后置 _fetcher 为空，但一直不执行则泄漏
    - (void)test
    {
        _fetcher = [EOCNewworkFetcer new];
        [_fetcher startWithHandler:^(NSData *data) {
            NSLog(_fetcher.url);
            _data = data;
            _fetcher = nil;
        }]；
    }

    // 例 3，self 不持有 fetcher，但 block -> fetcher -> block
    - (void)test
    {
        EOCNewworkFetcer *fetcher = [EOCNewworkFetcer new];
        [fetcher startWithHandler:^(NSData *data) {
            NSLog(fetcher.url);
            _data = data;
        }]；
    }

    // 例 4，执行 block 后置 nil
      - (void)p_requestCompleted
    {
        if (_handler) {
            _handler(_data);
        }
        _handler = nil;
    }
    ```

### 41、多用 GCD 队列，少用同步锁

- `@synchronized()` 和 `NSLock` 容易死锁，性能也较低。
- 异步派发需要拷贝块，所以执行时间比同步块慢。
- 存取方法话题，并行队列比串行队列快，但读写操作混在一起随意执行，使用 barrier:
    ```objc
    _syncQueue = dispatch_get_global_queue(DISPATCH_QUEUE_PRIORITY_DEFAULT, 0);

    (NSString*)someString
    {
        _block NSString *localSomeString;
        dispatch_sync(_syncQueue, ^{
            localSomeString = _someString;f
        });
        return localSomeString;
    }

    // 加入 barrier 后，剩余的读先执行完，再写，再执行之后的读
    (void)setSomeString:(NSString*)someString
    {
        // 改成 sync 更快
        dispatch_barrier_async(_syncQueue, ^{
            _someString = someString;
        });
    }
    ```

### 42、多用 GCD，少用 `performSelector`

- 原因一，内存泄漏：
    ```objc
    SEL selector;
    if () {
        // ret 应该由 ARC 释放
        selector = @selector(newObject);
    } else if () {
        // ret 应该由 ARC 释放
        selector = @selector(copy);
    } else {
        // 是否 ARC 都不应释放
        selector = @selector(someProperty);
    }
    id ret = [object performSelector:selector];
    ```
    编译器不知道方法名和返回值，所以 ARC 无法判断是否释放返回值，ARC 最终不会添加释放操作。
- 原因二，返回值只能是 id 或 对象类型，如果返回整数等其它值需要复杂的转换。
- 原因三，传参不方便，只有一个 `withObject:` ，如果传字典则由性能和维护问题。
- 替代， `performSelector` 的特点是指定线程并有延时相关参数，但用 GCD 一样可行。

### 43、掌握 GCD 及操作队列的使用时机（对比 `NSOperation`）

- GCD 是纯 C 的 API，但队列是 Objc 对象；`NSOperationQueue` 是完整的 Objc 对象，性能差距不大，优势有：

    可以取消操作；
    
    可以直接指定操作的依赖关系，以及优先级；

    可以通过 KVO 监测相关属性。

### 44、通过 dispatch group 机制，根据系统资源状况执行任务

- 把任务归入 group 中，可同步或异步执行任务组，获得通知。
- GCD 具体调度几个线程取决于系统。

### 45、`dispatch_once` 执行只需运行一次的线程安全代码

- 下面：
    ```objc
    + (id)sharedInstance
    {
        static EOCClass *sharedIntance = nil;
        static dispatch_once_t onceToken;
        dispatch_once(&onceToken, ^{
            sharedInstance = [[self alloc] init];
        });
        return sharedInstance;
    }
    ```
### 46、不要使用 `dispatch_get_current_queue`

- 如果要解决 sync 自己的执行队列的死锁，可能这么做：

    ```objc
    (NSString*)someString
    {
        __block NSString *localSomeString;
        dispatch_block_t accessorBlock = ^{
            localSomeString = _someString;
        };
        if (dispatch_get_current_queue() == _syncQueue) {
            accessorBlock();
        } else {
            dispatch_sync(_syncQueue, accessorBlock);
        }
        return localSomeString;
    }
    ```
    
    但其实有死锁风险，如：

    ```objc
    dispatch_sync(queueA, ^{
        dispatch_sync(queueB, ^{
            if (dispatch_get_current_queue() == queueA) {
                block();
            } else {
                // 实际返回 queueB
                dispatch_sync(queueA, accessorBlock);
            }
        });
    });
    ```
    
    上面的问题解决的根本是，`syncQueue` 只应该用来同步属性的访问，而不应该还能在其里调用 `someString`。（即不可重入方法）

- 队列有层级关系，最高队列是「全局并发队列」。（树上只有最近祖先串行结点有意义）
    「目标队列」即「父结点」。
    
    容易坑的地方是：有些 API 可指定执行队列参数，但实际执行的队列是最近祖先串行队列，`dispatch_get_current_queue` 返回的也是它。

    可以使用 `set_specific` API 设定队列的关联数据，其获取 API 会自下向上寻找数据，找到最近的、有数据的父结点队列。

## 七、系统框架

### 47、熟悉系统框架

- Core Foundation 是 C 框架，Foundation 框架许多功能都能找到对应 API。

### 48、多用块枚举（迭代），少用 for

- `NSEnumberator` 迭代器。
- for in 包装了迭代器，实现了 `NSFastEnumberation` 协议的对象都可以迭代。
- `enumberate` 方法通过 block 遍历，可传入选项掩码。

### 49、对自定义管理语义的 collection 使用无缝桥接

### 50、构建缓存使用 `NSCache` 而非 `NSDictionary`

- `NSCache` 在系统资源将耗尽时，可以自动删减缓存。（最久未使用）

    其不会自动拷贝键，且是线程安全的。
    
    配合 `NSPurgeableData` 更好，会自动删减。

### 51、精简 `load` 与 `initialize`

- `load` 方法在应用启动时自动执行（不要手动执行），执行时运行时系统很「脆弱」，里面不要使用其它类。（不知道其它类是否已经 `load`）
    
    `load` 不遵循继承，如果类没实现，那么其超类实现都不会调用。

    同时执行 `load` 是阻塞的，不要进行锁操作。

- `initialize` 是惰性的，在程序首次使用类前调用。

    运行期此时是正常状态，而为了保证线程安全，会保证同时只有执行该方法的线程运行，其它所有线程阻塞。

    执行是符合继承的，如果类未实现则调用父类实现。
    
    所以一般：
    
    ```objc
    // BaseClass
    + (void)initalize
    {
        if (self == [BaseClass class]) {
            NSLog(@"%@ initizlized", self);
        }
    }
    ```
    
    同样的，不要执行繁琐任务，执行 `initialize` 的线程是不可控制的，甚至 UI 线程也会执行它。

    此外，代码不可依赖于调用该方法的时间点，未来可能会更新运行时系统。

    最后，代码若使用了未初始化的其它类，那么系统又会迫使它初始化；甚至会出现 A、B 类互相使用，而 A 类数据未初始化好的问题。

- `initialize` 的特殊用法：在里面初始化无法在编译期初始化的全局变量。

    ```objc
    static NSMutalbeArray *kSomeObjects;

    @implementation EOCClass
    + (void)initialize
    {
        if (self == [EOCClass class]) {
            kSomeObjects = [NSMutalbeArray new];
        }
    }
    ```

### 52、`NSTimer` 会保留其目标对象

- 类成员变量（计时器）-> 计时器-> 类本身 `self`，即使弄一个 stop 方法最后设置 `nil`，但也不能保证该方法一定能调用到。

    ```objc
    (void)dealloc
    {
        [_pollTimer invalidate]; // 如果循环引用则不会调用 dealloc
    }
    (void)stopPolling
    {
        [_pollTimer invalidate];
        _pollTimer = nil;
    }
    ```

    解决方法：让计时器支持块。（现在已经支持了，不用自己加了）

    ```objc
    @interface NSTimer(EOCBlockSupport)

    + (NSTimer*)eoc_scheduleTimerWithTimeInterval:(NSTimeInterval)interval
                                            block:(void(^)())block
                                        repeats:(BOOL)repeats;
                                        
    @end

    @implementation NSTimer(EOCBlockSupport)

    + (NSTimer*)eoc_scheduleTimerWithTimeInterval:(NSTimeInterval)interval block:(void (^)())block repeats:(BOOL)repeats
    {
        return [self scheduledTimerWithTimeInterval:interval
                                            target:self
                                        selector:@selector(eoc_blockInvoke:)
                                        userInfo:[block copy]    repeats:repeats];
    }

    + (void)eoc_blockInvoke:(NSTimer*)timer
    {
        void (^block)() = timer.userInfo;
        if (block) {
            block();
        }
    }
    @end
    ```

    这时类对象和计时器互相引用，但没所谓。

    然后实际 block 使用 `__weak EOCClass *weakSelf = self;` 和 `EOCClass *strongSelf = weakSelf;`。

    这样在 `self == nil` 后忘记调用 `invalidate` ，再次触发 block 也不会崩溃。

- 为什么不直接在 target 参数使用 strong weak self？因为这样做的话 ARC 会直接转为强引用，weak 无意义。
    