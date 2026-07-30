- **@property** (**nonatomic**, **strong**) UIImageView *icon;

- [**self**.icon mas_makeConstraints:^(MASConstraintMaker *make) {}];

```objectivec
- setMyValue:(id) foo;
```

> id形态表示参数foo可以是任何类的实例

![](https://developer.apple.com/library/archive/documentation/Cocoa/Conceptual/ProgrammingWithObjectiveC/Art/buttoninheritance.png)

`id`类型定义了一个通用对象指针。在声明变量时可以使用`id`，但会丢失对象的编译时信息。
