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
- CPP 增加了可以指明底层数据结构的特性。
- Foundation 框架定义了 `NS_ENUM`、`NS_OPTIONS` 宏，兼容了 C 和 CPP 不同的编译模式：
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
- 第三方库尽可能自修改名称，加上 3 个字母的前缀。

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

### 21、理解错误模型

- 抛出异常要考虑释放资源的问题，所以一般抛出异常的场景是严重错误，宁愿终止程序，不考虑释放。
- 一般的错误，返回 nil / 0，或使用 `NSError`
    ```objc
    - (BOOL)doSomething:(NSError**)error
    {
        if (error occur) {
            if (error) {
                *error = ...
            }
            return NO;
        } else {
            return YES;
        }
    }
    ```
    使用枚举表示错误码。

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

### 27、「class-continuation 分类」隐藏实现细节

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


