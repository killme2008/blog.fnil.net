---
layout: post
title: "Swift 编程之闭包（翻译）"
date: 2014-06-10 01:56:56 +0800
comments: true
categories: Swift,翻译
---

‌‌最近Apple新出的 [Swift](https://developer.apple.com/swift/) 语言非常火，我们公司也在陆续组织翻译[《The swift programming language》](https://itunes.apple.com/us/book/the-swift-programming-language/id881256329?mt=11) 这本官方电子书，为公司博客增加一些人气。这篇博客就是我负责的其中之一的产物。在公司博客放了一份，这里也放一份。



闭包
--------

**闭包(Closures)**是可以在你的代码里传递和使用的，自包含的功能代码块。Swift 里的闭包跟 C 和 Objective-C 里的 block 类似，也就是其他语言里的所谓的 lambda 。


闭包可以从定义的上下文(Context)里捕获和存储常量或者变量的引用。这被称为“闭合”（*closing* over）了这些常量和变量，这也是“闭包”名称的由来。Swift 帮你处理了所有捕获相关的内存管理。

**注意**

```
不用担心你不理解“捕获”这个概念。会在下文里详细解释。
```

在[函数](https://www.zybuluo.com/ghosert/note/16165)一章中介绍的全局和嵌套函数，其实是闭包的特殊形式，闭包表现为三种形式：

- 拥有一个函数名，并且不捕获任何值的闭包称为全局函数。
- 拥有一个函数名，并且从外部函数捕获值的闭包，称为嵌套函数。
- 使用轻量级语法编写，并且没有命名的闭包表达式，可以从周围上下文中捕获值


Swift 的闭包表达式语法拥有一个干净、清晰风格，针对大多数应用场景里做了优化，倾向于简明、整洁的语法：

- 根据上下文信息，为参数和返回值的自动做类型推断。
- 单一闭包表达式的隐式返回。
- 参数的速记法。
- 拖尾闭包（Trailing Closure）语法。

下面我们开始详细介绍。

<!-- more -->

### 闭包表达式 

在[嵌套函数](https://www.zybuluo.com/ghosert/note/16165#nested-functions)一节中介绍的嵌套函数，是在一个更大的函数内部定义和命名一部分自包含的代码块的常见方式。尽管如此，编写不需要完整的声明和命名的函数构造有时候会更有用处。这在你需要将其他函数当做一个或者多个参数传递的时候会很常见。

*闭包表达式* 就是一种编写简短并且清晰的内联式闭包的方式。闭包表达式提供了了多种语法，优化到最简单的方式来编写闭包，而且没有失去代码的清晰和意图。下面小节中举例提到的例子，就是通过数次迭代重新定义一个排序函数来展示这些优化，每一次迭代步骤中的表达式都拥有相同的功能，但是将更加简明。
‌

### 排序函数

Swift 的标准库提供了一个`sort`函数，可以用来排序一个已知类型的数组，在你提供的排序闭包函数的输出的基础上做到。当完成排序后，`sort`函数返回一个新的数组，类型和大小都跟“旧”的一样，并且里面的元素按照正确的顺序排好序。

下面的闭包表达式例子，使用`sort`按照字母倒序的顺序排序一个字符串数组，这是初始数组（未排序的）：

```
let names = ["Chris", "Alex", "Ewa", "Barry", "Daniella"]
```

sort函数接收两个参数：

- 一个已知类型的数组。
- 一个接收两个相同类型的数组元素的闭包，返回一个布尔值，表示第一个参数是在第二个参数之前还是之后。排序闭包如果返回`true`，表示第一个参数应该在第二个参数**之前**，`false`则相反。

这个例子排序一个字符串数组，因此排序闭包必须是一个`(String, String) -> Bool`签名的函数。

编写一个类型正确的普通函数就能做到，传入`sort`作为第二个参数：

```
func backwards(s1: String, s2: String) -> Bool {
 return s1 > s2
}
var reversed = sort(names, backwards)
// reversed的数组等于["Ewa", "Daniella", "Chris", "Barry", "Alex"]
```

如果第一个字符串(s1)比第二个字符串(s2)大，backwards函数返回true，表示最终的结果数组中s1应该在s2之前。对于字符串中的字符(character)来说，所谓“更大”的意思就是“出现在字母表的更后面”。也就是说字母 "B" 比字母 "A"更大，字符串 "Tom" 比字符串 "Tim" 更大（因为第二个位置的字母o在字母i后面——译者注）。这就实现了字母顺序的倒序排序，使得 "Barry" 放在了 "Alex" 之前等等。

尽管实现了功能，但是这实在是一种啰嗦的方式来编写这个本质上只是一行表达式(a > b)的事情。在这个例子中，使用闭包表达式语法来编写一个内联的排序闭包会更合适。
‌

### 闭包表达式语法

闭包表达式语法的常见形式如下:

```
  { (参数列表) -> 返回值类型 in

     一系列语句

  }
```

闭包表达式可以使用常量参数，变量参数和 inout 参数，但是不允许参数默认值。可变参数仅允许出现在参数列表的末尾。元组(Tuple)类型也可以作为参数类型和返回值类型（也就是实现多值返回——译者注）。

下面的例子展示了上面的`backwards`的闭包表达式版本：

```
reversed = sort(names, { (s1: String, s2: String) -> Bool in
 return s1 > s2
 })
```

我们注意到这个内联闭包的参数和返回值类型声明和`backwards`函数是完全一样的，都写成`(s1: String, s2: String) -> Bool`。但是，内联闭包的参数和返回值类型都写在大括号的**内部**，而不是(大括号)**之外**。

闭包的函数体(body)是通过`in`关键字开始的，这个关键字表示闭包的参数和返回值类型定义结束，接下来是闭包的函数体。

因为这个闭包的函数体是如此简短，	我们甚至可以写成单行：

```
reversed = sort(names, { (s1: String, s2: String) -> Bool in return s1 > s2 } )
```

这段代码对 `sort` 函数的调用总体上仍然保持不变。一对括号仍然包围住 `sort` 函数调用的参数列表。但是其中一个参数已经变成内联闭包。
‌

### 从上下文推断类型 

因为排序闭包是传递给一个函数也就是`sort`作为参数使用，Swift 其实可以从 `sort` 函数的第二个参数出发，推断出闭包的参数类型和返回值类型。例子中的 `sort` 函数调用预期第二个参数是一个声明为`(String, String) -> Bool`的函数，这就意味着不需要在闭包表达式里写上 `String` 类型。因为所有的类型都可以被推断出来，返回符号(->)和参数名 s1,s2 外面的括号也可以被忽略掉：

```
reversed = sort(names, { s1, s2 in return s1 > s2 } )
```

如果将闭包传递给一个函数作为内联闭包，我们总是也许可以推断出它的参数类型和返回值类型，因此，你很少需要以完整的形式来编写内联闭包。

虽然如此，如果你愿意，你还是可以明确地写上类型，我们鼓励这样的方式，这可以让你的代码的阅读者避免歧义。 在 `sort` 函数这个案例中，闭包的目的很清楚就是为了排序。因为排序的是一个字符串数组，所以代码阅读者可以安全地假设闭包是在处理 `String` 类型，我们就可以忽略掉参数类型和返回值类型声明。
‌
### 单一表达式闭包的隐式返回

单一表达式闭包可以隐式返回他们那个唯一表达式的结果，忽略声明里的 `return` 关键字，比如我们改写上面的例子：

```
reversed = sort(names, { s1, s2 in s1 > s2 } )
```

这样一来，`sort` 函数第二个参数的函数类型很清楚地表明闭包必须返回一个 Bool 值。因为闭包的函数体只包含了唯一一个表达式(s1 < s2)，并且结果是布尔类型，`return` 关键字可以被忽略掉，因为没有任何歧义。

### 参数名的速记法


Swift 自动地为内联闭包提供了参数名的快速记法，可以使用 `$0`, `$1`, `$2` 来指代闭包的参数值，以此类推。

如果你在闭包表达式里使用参数名速记法，你可以从定义里忽略掉参数列表，从预期的函数类型就可以推断出速记法里的参数的序号数字和类型。`in`关键字也可以被忽略，因为闭包表达式完全由它的函数体组成：

```
reversed = sort(names, { $0 > $1 } )
```

这里的`$0`,`$1`表示闭包的第一个和第二个`String`参数。

### 操作符函数 

这其实是一种更**简短**的编写上述闭包表达式的方式。Swift 的`String`类型定义了大于操作符(>)在字符串类型中的专有实现，并且实现为一个接收两个字符串参数并返回布尔值的函数。这跟 `sort` 函数的第二个参数的类型完全匹配。因此，你可以简单地将大于操作符传进去，Swift 会自动推断成你想使用字符串的专有实现：

```
reversed = sort(names, >)
```

更多关于操作符函数，请看[操作符函数](https://www.zybuluo.com/ghosert/note/16198#operator-functions).

‌

### 拖尾闭包（Trailing Closures ）

如果你需要传递一个闭包表达式给另一个函数作为最后一个参数，并且闭包表达式的函数体非常长，你可以将它替换成 **拖尾闭包** 的形式。所谓 **拖尾闭包** 就是将闭包表达式写在函数调用的括号之外（或者说之后）：

```
func someFunctionThatTakesAClosure(closure: () -> ()) {
 // 函数的函数体放到这里。
}
// 不使用拖尾闭包调用 someFunctionThatTakesAClosure 的样子：
someFunctionThatTakesAClosure({
 //闭包的函数体放到了这里（括号内）。
 })
// 替代地，使用拖尾闭包调用 someFunctionThatTakesAClosure 的样子：
someFunctionThatTakesAClosure() {
 // 拖尾闭包的函数体放在这里，在函数调用的小括号之外了
}
```

**注意**

```
如果函数只有一个参数，并且这个参数是一个闭包，你将它写成拖尾闭包的形式，那么你就不需要在调用函数的时候，在函数名之后写上一对括号()，括号可以忽略掉。
```

上面提到的字符串排序闭包，可以写成在 `sort` 函数括号外部的拖尾闭包：

```
reversed = sort(names) { $0 > $1 }
```

在闭包（函数体）非常长乃至无法写在一行里的时候，拖尾闭包非常有用。例如，Swift 的数组有个方法`map`接收一个闭包作为参数。数组将闭包一个一个地作用在数组的元素上，返回一个替代的映射后的新值（可能是其他类型）。映射的方式和返回值的类型，都交给闭包来指定。

在将闭包逐个作用在数组元素之后，`map`方法将返回一个包含了映射后元素的新的数组，并保持它们在相应的原始数组里的顺序。

下面是一个例子演示，使用`map`方法和一个拖尾闭包来完成一个 Int 数组到 String 数组的转换。数组`[16, 58, 510]`被用来创造一个新数组`["OneSix", "FiveEight", "FiveOneZero"]`:

```
let digitNames = [
 0: "Zero", 1: "One", 2: "Two", 3: "Three", 4: "Four",
 5: "Five", 6: "Six", 7: "Seven", 8: "Eight", 9: "Nine"
]
let numbers = [16, 58, 510]
```

上面代码首先创建了一个数字到它们英文名称的字典`digitNames`，然后定义了一个整数数组，准备转换成字符串数组。

通过将一个闭包表达式作为拖尾闭包传入数组的`map`方法，你就可以使用整数数组来创建一个字符串数组。注意到，调用`numbers.map`并不需要在`map`之后带上一对括号，这是因为`map`只有一个参数，并且这个参数是作为拖尾闭包提供的：

```
let strings = numbers.map {
 (var number) -> String in
 var output = ""
 while number > 0 {
 output = digitNames[number % 10]! + output
 number /= 10
 }
 return output
}
// strings 被自动推断为 String [] 类型
// 它的值是 ["OneSix", "FiveEight", "FiveOneZero"]
```
`map`函数将闭包作用在数组的每个元素上。你不需要指定闭包的输入参数`number`的类型，这是因为 Swift 可以从被映射的数组的值的类型自动推断出来。

在这个例子中，闭包的`number`参数被定义为[常量和变量参数](https://www.zybuluo.com/ghosert/note/16165#constant-and-variable-parameters)中描述的 **变量参数**，所以它的值可以被闭包表达式的函数体(body)所修改，而不需要再去声明一个新的局部变量，然后将`number`值赋给它。闭包表达式同时也指定了返回类型是 String，也就是将存储在映射后的新数组内的元素类型。

这个闭包表达式在每次被调用的时候，创建了一个字符串叫`output`。它使用取模操作(number % 10)来计算数字的最后一位，然后拿这一位的数字去字典`digitNames`里查找合适的字符串值。

**注意**

```
调用 digitNames 字典的下标之后有一个感叹号(!)，这是因为字典下标操作返回一个 optional 类型表示这个字典查找操作可能因为 key 不存在而失败。在上面的例子中，number % 10 的结果总是能保证落在 digitNames 字典的下标范围内，因此这里的感叹号是用来强制解引用(原文：force-unwrap)得到下标操作返回的 optional 中存储的字符串值。
```

从`digitNames`得到的字符串，添加到了`output`的**前面**，实际上以倒序的顺序构造了`number`的字符串版本。（表达式`number % 10`作用在16上返回6，作用在58上返回8，作用在510上返回0）。

数字`number`接下来除以10，这是因为它是一个数字，在除法的时候将会被取整（舍弃小数位），因此16变成1，58变成5，以及510变成51。

重复这个过程直到`number /= 10`的结果等于0，这时字符串`output`就被闭包返回，并且添加到`map`函数的输出数组中。

上面例子中对拖尾闭包语法的使用，干净整洁地将闭包支持的功能封装在函数的后面，不需要将整个闭包包装起来，放到`map`函数外面的调用括号内。


### 值的捕获 

闭包可以从它定义的周围上下文里面**捕获**常量和变量。然后闭包就可以在它的函数体里引用或者修改这些常量和变量的值，哪怕定义这些常量和变量的原始作用域已经不存在了。

Swift 中最简单的闭包形式是一个嵌套函数，嵌套在另一个函数的函数体内。一个嵌套函数可以捕获任何外部函数的参数，以及任何定义在外部函数里的常量和变量。

下面例子中有个函数叫`makeIncrementor`，它包含了一个嵌套函数`incrementor`。内嵌的`incrementor`函数从上下文里捕获了两个值：`runningTotal`和`amount`。捕获了这些值之后，`incrementor`被`makeIncrementor`作为一个闭包返回，每次调用它的时候都将以`amount`的幅度大小来递增`runningTotal`的值。

```
func makeIncrementor(forIncrement amount: Int) -> () -> Int {
 var runningTotal = 0
 func incrementor() -> Int {
 runningTotal += amount
 return runningTotal
 }
 return incrementor
}
```

`makeIncrementor`的返回值类型是`() -> Int`，也就是返回一个**函数**而不是简单的值。它返回的函数不需要任何参数，并且每次调用都返回一个 Int 值。关于如何使用函数来返回函数，请参考[将函数作为返回类型](https://www.zybuluo.com/ghosert/note/16165#function-types-as-return-types)。

`makeIncrementor`函数内部定义了一个整数变量`runningTotal`，用来保存`incrementor`中当前的`runningTotal`值，这个变量初始化为0。

`makeIncrementor`只有一个参数，外部名称为`forIncrement`。传递给这个参数的值指定了`incrementor`每次调用的时候`runningTotal`应该递增多少。

`makeIncrementor`定义了一个嵌套函数称为`incrementor`，这个函数做了真正的递增工作。`incrementor`函数只是简单地将`amount`添加到总数`runningTotal`上并返回结果。

当考虑隔离的时候，内嵌的`incrementor`可能看起来比较奇怪：

```
func incrementor() -> Int {
 runningTotal += amount
 return runningTotal
}
```

`incrementor`不接收任何参数，并且在它的函数体内引用到`runningTotal`和`amount`。它能做到这一点，就是通过从周围函数里**捕获** `runningTotal`和`amount` **已经存在**的值，并且用在自己的函数体内。

因为它并不修改`amount`，`incrementor`实际捕获和存储的是存储在`amount`变量的值的**一个拷贝**，这个值将和新的`incrementor`存储在一起。

但是，因为每次调用`incrementor`都会修改`runningTotal`变量，因此`incrementor`捕获的是当前`runningTotal`变量的**一个引用**，而不仅仅是它的初始值的一个拷贝。捕获引用而非拷贝这一点，才能保证`runningTotal`不会在对`makeIncrementor`的调用结束后消失（因为离开了`makeIncrementor`的作用域——译者注），并且保证`runningTotal`在下次`incrementor`函数被调用的时候继续有效。

**注意**

```
Swift 决定哪些值应该被捕获引用，哪些值应该被拷贝。你并不需要注释 amount 或者 runningTotal，说明它们可以在内嵌函数里被使用。当 runningTotal 不再需要被 incrementor 使用的时候，Swift 也会处理所有跟 runningTotal 变量释放引起的内存管理相关的事情。
```

这里给个例子说明`makeIncrementor`怎么用：

```
let incrementByTen = makeIncrementor(forIncrement: 10)
```

这个例子设置了一个常量称为`incrementByTen`，它指向一个`incrementor`函数，这个函数将会在每次调用的时候给`runningTotal`加上10。让我们调用这个函数几次试试：

```
incrementByTen()
// 返回10
incrementByTen()
// 返回20
incrementByTen()
// 返回30
```

如果你创建另一个`incrementor`，它会有一个自己的被保存的引用，并且这个引用指向一个全新、独立的`runningTotal`变量。在下面的例子中，`incrementBySeven`捕获了一个指向新的`runningTotal`变量的引用，这个变量跟`incrementByTen`中的那个`runningTotal`毫无联系。

```
let incrementBySeven = makeIncrementor(forIncrement: 7)
incrementBySeven()
// 返回7
incrementByTen()
// 返回40，两者独立，毫无联系
```

**注意**

如果你将闭包赋值给 class 实例的某个属性（也就是对象的某个属性——译者注），并且这个闭包引用了该实例对象或者对象中的一些成员，那么这个闭包也就**捕获**了这个对象本身。这样一来，你就创建了一个在闭包和该对象之间的**强循环引用**。 Swift 使用 **捕获列表**(capture lists)来打断这种强循环引用，更多信息请参考[闭包的强循环引用](https://www.zybuluo.com/ghosert/note/16219#strong-reference-cycles-between-class-instances)

### 闭包都是引用类型

上面的例子中，`incrementBySeven`和`incrementByTen`都是常量，但是这些常量指向的闭包仍然可以递增它们捕获的`runningTotal`变量。这是因为函数和闭包都是**引用类型**。

当你将一个函数或者闭包赋值给一个常量或者变量的时候，你实际上做的是将常量或者变量设置成一个**引用**，指向那个函数或者闭包。在上面的例子里，`incrementByTen` **指向**闭包的引用是一个常量，而不是闭包的内容本身。

这同时也意味着，如果你将闭包赋值给两个不同的常量或者变量，它们都将指向同一个闭包：

```
let alsoIncrementByTen = incrementByTen
alsoIncrementByTen()
// 返回50
```
