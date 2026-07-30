#### Objective-C:C的超集

扩展名：

- .h:头文件
- .m：源代码文件
- .mm:源代码文件，可以包含Objective-C和C、C++代码。

在Objective-C，类别与消息的关系比较松散，调用方法视为对对象发送消息，所有方法都被视为对消息的回应。所有消息处理直到运行时（runtime）才会动态决定，并交由类别自行决定如何处理收到的消息。也就是说，一个类别不保证一定会回应收到的消息，如果类别收到了一个无法处理的消息，程序只会抛出异常，不会出错或崩溃。

消息传递

```cpp
obj.method(argument);
```

```objectivec
[obj method:argument];
```

#### 数据类型

- 基本数据类型：int、float、char...

- 复合数据类型：数组、结构体、枚举...

- 指针

- 内存管理

#### 类

类规格说明包含了定义（interface)与实现（implementation）两部分。定义包含了类声明和实例变量的定义，以及类相关的方法。实现部分包含了类方法的实际代码。类声明由`@interface`编译选项开始，由`@end`编译选项结束

**公有属性和行为在@interface声明内部定义。如果声明只读属性，可以在@property后面加（readonly）**

![](https://www.runoob.com/wp-content/uploads/2015/08/2011021920525849.jpg)

```objc
@interface MyObject : NSObject{
    @property (readonly) int memberVar1;    //实体变量
    int menberVar2;
}

+(return_type) class_method;    //类方法

-(return_type) instance_method1;    //实例方法
-(return_type) instance_method2:(int) p1;
-(return_type) instance_method3:(int) p1 andPar:(int) p2;
@end
```

- 方法前面的`+`号代表类方法，`-`号代表实例方法。
- `@interface`：定义的类实例变量默认权限为`protected`
- `@Implementation`：定义的类实例变量默认权限为`private`
- 方法名一般以小写字母开头，使用驼峰命名法

#### 创建对象

创建对象需要通过`alloc`以及`init`   两个消息，`alloc`的作用是分配内存，`init`则是初始化对象。

```objectivec
MyObject * my = [[MyObject alloc] init];
```

#### 属性

基本的定义使用`@property`编译选项

```objc
@interface Person:NSObject{
    @public
        NSString *name;
    @private
        int age;
}

@property(copy) NSString *name;
@property(readonly) int age;

-(id) initWithAge:(int)age;
@end
```

属性的访问方法由`@synthesize`关键字来实现，它由属性的声明自动的产生一对访问方法。也可以选择`@dynamic`关键字表明访问方法由程序员手工提供

```objc
@implementation Person
@synthesize name;
@dynamic age;

-(id) intWithAge:(int)initAge{
    age = initAge;
    return self;
}

-(int) age{
    return 29;
}
@end
```

#### 快速枚举【看不懂】

#### 协议（Protocol）

类似于Java中的接口。

有两种定义协议的方式：

- 由编译器保证的“正式协议”

- 为特定目的设定的“非正式协议”

**一个类必须实现它声明匹配的协议中的所有方法，OC2.0允许标记协议中某些方法是可选的**

协议的定义

```objectivec
@protocol Locking
- (void)lock;
- (void)unlock;
@end
```

采纳协议：

```objectivec
@interface SomeClass :SomeSuperClass<Locking>

@end
@implementation SomeClass
- (void)lock{

}
- (void)unlock{

}
@end
```

-----

#### 动态类型

一个对象接收到消息之后，通常有三种处理消息的可能手段：

1. 回应该消息并运行方法

2. 无法回应，则转发消息给其他对象（可能类似于子类转发消息给父类）

3. 以上均无，就要处理无法回应抛出的例外

只要进行三者其一，该消息就算完成任务而被丢弃。

**若对“nil“（空指针对象）发送消息，该消息通常会被忽略，取决于编译器选项可能会抛出例外**

不同参数形态：

```objectivec
//id形态表示参数"foo"可以是任何类的实例。
- setMyValue:(id) foo;
//id<aProtocol>表示"foo"可以是任何类的实例，但必须采纳"aProtocol"协议。
- setMyValue:(id <aProtocol>) foo;
//该声明表示"foo"必须是"NSNumber"的实例。
- setMyValue:(NsNumber*) foo;
```

----

#### 转发

-----

#### 类别

在运行时，分类中的方法与类原有的方法并无区别，其代码可以访问包括私有类成员变量在内的所有成员变量。

分类中声明了与原有类中相同的方法，分类中的方法会被调用。**这个特性可以用于修正原有代码中的错误，更可以从根本上改变程序中原有类的行为。**

----

- NSString：不可变

- NSMutableString：可变

----

### 对象发送和接收消息

基本语法：

```objectivec
[someObject doSomething];
//someObject是消息的接收者，doSomething是将在该接收者上调用的方法。
//someObject将接收doSomething消息
```

发送一个Objective-C消息在概念上非常类似于调用一个C函数。

----

Objective-C 编译器的自动引用计数（ARC）功能处理对象的回收。

----

`self`是指“收到此消息的对象”，它是一个指针，用于调用<mark>当前接收对象</mark>的属性和方法。

`super`用于调用父类的属性和方法

-----

### 对象是动态创建的

`NSObject`根类提供类一个类方法`alloc`，用于为对象分配内存，并清除对象属性分配的内存，将其设置为0。避免内存中包含之前存储内容的垃圾数据的问题，但不足以完全初始化一个对象。

`alloc`和`init` 方法一般结合使用。

```objectivec
@interface NSObject
+ (id)new;
+ (id)alloc;
- (id)init;
@endobjectivec
```

```objectivec
XYZObject *object = [XYZObject new];
//等同于
XYZObject *object = [[XYZObject alloc] init];
```

因为对象的类是在运行时确定的，所以在创建或处理实例时，给变量赋什么类型都没有关系。

C 中的标准相等运算符 `==` 用于测试两个变量值是否相等，当处理对象时，== 运算符用于测试两个独立的指针是否指向同一个对象。如果需要测试两个对象是否表示相同的数据，你需要调用 `isEqual`方法，调用`compare`比较大小。

- 基本类型变量初始化时最好赋初值，会清空之前的栈垃圾；

- 对象类型变量初始化时没必要赋空值，因为会自动赋值`nil`；

在 Objective-C 中向 `nil` 发送消息是完全允许的。如果你确实向 `nil` 发送消息，显然什么也不会发生。

向`nil`发送消息，根据不同的消息有不同的返回值：

- `nil`：对象类型

- 0：基本数值类型

- `NO`：`BOOL`

检验一个对象是否为空：

```objectivec
if(someObj != nil){
    //TODO
}
//或者
if(someObj){
    //TODO
}
```

----

`@property`声明的属性会自动生成`getter`和`setter`方法，在`@property`后面添加`(readonly)`将不生成`setter`方法。

属性值可以使用点语法进行访问：

```objectivec
NSString *firstName = somePerson.firstName;
//等同于
NSString *firstName = [somePerson firstName];

somePerson.lastName = @"Johnny";
//等同于
[somePerson setLastName : @"Johnny"];
```

下划线前缀表明你正在访问实例变量而不是局部变量

一个 `init` 方法应该在执行自己的初始化之前，将结果赋值给调用超类初始化方法得到的 `self` 。超类可能无法正确初始化对象并返回 `nil` ，因此你应该始终检查 `self` 不是 `nil` 再执行自己的初始化。



**延迟初始化**



**懒加载**



**默认情况下，一个 Objective-C 属性是原子性的，可以使用`(nonatomic)`来指定属性不是原子性的**



### 避免强引用循环：如果一组对象通过强关系形成一个循环，即使组外没有强引用，它们也会彼此保持活跃状态。使用弱引用`(weak)`来声明属性，使用`__weak`来声明弱引用对象

因为弱引用不会使对象保持存活，所以在引用对象被释放的同时，引用仍然可能在使用中。为了避免指向现在已被释放对象原先占用的内存的危险悬空指针，当对象被释放时，弱引用会自动被设置为 `nil` 。

**不安全引用**

与弱引用的区别：被回收后会得到一个悬空指针，指向已经被释放对象的内存地址。

```objectivec
//属性
@property (unsafe_unretained) NSObject *unsafeProperty;
//变量
NSObject * __unsafe_unretained unsafeReference;
```



----

### 定制现有类

#### 类别：向现有类中添加方法

```objectivec
@interface ClassName (CategoryName)

@end
```

一旦您声明了一个类别并实现了其中的方法，就可以从该类的任何实例中使用这些方法，就好像它们是原始类接口的一部分

类别可以用来声明实例方法或类方法，但通常不适合声明额外的属性。在类别接口中包含属性声明是有效的语法，但在类别中无法声明额外的实例变量。这意味着编译器不会合成任何实例变量，也不会合成任何属性访问方法。你可以在类别的实现中编写自己的访问方法，但除非原始类已经存储了该值，否则你无法跟踪该属性的值。

**避免类别方法名称冲突：** 如果一个类别中声明的方法名称与原类中的方法名称相同，或者与同一类（或其超类）的另一个类别中的方法名称相同，运行时将使用哪个方法实现的行为是未定义的。

#### 类扩展：向现有类添加属性
