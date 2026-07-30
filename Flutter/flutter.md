#### Widget

- StatelessWidget

- StatefulWidget --> State.setState()

#### Dart 语法

**变量和运算符：**

- var、num、list、String、const、final、late、Object、dynamic

- `?` 可空修饰符

- `??` 、`??=`空合运算符

- `..`、`..?` 级联运算符

- `as`

- `is`、`is!`

- `?` `?[]` `!.`

**注释**

- `//`：单行

- `/* */`：多行

- `///`：文档注释

**Dart支持以下10种数据类型：**

1. Numbers，`num`、`int`、`double`，数字类型
2. Strings，`String`，字符串
3. Booleans，`bool`，布尔
4. Lists，`List`，有序列表
5. Sets，`Set`，无序集合，**两个集合间通过 `difference`、`union` 、`intersection` 方法可以分别计算差集、并集、交集。计算的结果也是一个集合**
6. Maps，`Map`，映射
7. Records，`Record`，记录类型，Dart中的元组概念（Dart3.0引入）
8. null，`Null`，空类型
9. Symbols，`Symbol`，标识符，类似JS中的Symbol
10. Runes，符文类型，处理Unicode符号

**Dart中的异步：**
*Dart是单线程语言*，异步的核心是：

- Event Loop 概念

- async/await

- Future

- Steam
