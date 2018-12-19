# 从Protocol Extension看Swift 派发机制

## Protocol中的诡异现象
相信大家都知道，在Swift Protocol中是可以通过Extension提供默认实现方法的，并且无需在声明中定义。这也是Swift Protocol如此强大的基石，广大的Swifter可以利用其灵活的特点轻松实现POP，不过此文重点不是介绍如何面向协议编程，有兴趣的同学可以参阅喵神出品的[面向协议编程与 Cocoa 的邂逅 (上)](https://onevcat.com/2016/11/pop-cocoa-1/)
下文要介绍的是在使用Protocol开发中我们可能会遇到（也许遇到过）的坑，比如使用Protocol作为类型声明、泛型约束，并且调用其Extension中的默认实现，就有可能出现不符合我们预期的结果。如下示例：(其中注释为对应语句输出结果)


```swift
protocol Drawable {
    func draw()
}

extension Drawable {
    func draw() {
        print("\(self) Drawing something")
    }
    
    func commit() {
        print("\(self) Commit something")
    }
}

struct Triangle: Drawable {
    func draw() {
        print("\(self) Drawing triangle")
    }
    
    func commit() {
        print("\(self) Commit triangle")
    }
}

let triangle1: Drawable = Triangle()
let triangle2: Triangle = Triangle()
let triangles: [Drawable] = [triangle1, triangle2]

triangle1.draw() // *Triangle() Drawing triangle*
triangle1.commit() // *Triangle() Commit something*

triangle2.draw() // *Triangle() Drawing triangle*
triangle2.commit() // *Triangle() Commit triangle*

triangles.forEach {$0.draw(); $0.commit()}
/**
	*Triangle() Drawing triangle*
	*Triangle() Commit something*
	*Triangle() Drawing triangle*
	*Triangle() Commit something*
*/
```


上面的代码中，我们的预期应该是执行自己重写的实现（输出`Triangle() Commit triangle`），但从结果上看执行的却是协议中的默认实现。其实导致这个差异的原因是Protocol中的函数派发方式，在Swift Protocol中，如果该方法在定义中有声明，则采用动态派发，如果在定义中未声明该方法，则默认使用静态派发的方式。
在平常开发中大部分人也不会太关注函数派发机制、函数调用栈帧等，其实如果对其中的一些原理稍有了解的话，在面对一些奇奇怪怪的诡异问题时也就能大概知其原因，至少可以提供一个解决的思路

#### 动态派发 <[Dynamic dispatch - Wikipedia](https://en.wikipedia.org/wiki/Dynamic_dispatch)>

> In computer science, dynamic dispatch is the process of selecting which implementation of a polymorphic operation (method or function) to call at run time. It is commonly employed in, and considered a prime characteristic of, object-oriented programming (OOP) languages and systems.  

#### 静态派发  <[Static dispatch - Wikipedia](https://en.wikipedia.org/wiki/Static_dispatch)>
> In computing, static dispatch is a form of polymorphism fully resolved during compile time. It is a form of method dispatch, which describes how a language or environment will select which implementation of a method or function to use.  

## 常见的函数派发方式

1. Direct Dispatch （直接派发）
	1. 派发速度最快
	2. 调用的指令集较少
	3. 编译器可以对其进行优化，如函数内联
2. Table Dispatch  （函数表派发）
	1. 速度比直接派发慢一点(查表和跳转会损耗部分性能)
	2. 实现动态派发的常用方式
	3. 使用数组存储函数指针，添加的函数都会插入改表中
	4. 运行时根据此表决定调用的函数
3. Message Dispatch （消息机制派发）
	1. 可以在运行时改变函数行为（Method Swizzling, isa-Swizzling）
	2. 消息派发通过Cache机制可以提高查找命中率，派发性能基本和函数表派发差不多

* C/C++  (Direct Dispatch)， 通过virtual修饰可改为函数表派发
* Java  (Table Dispatch )，可以通过final修饰可改为直接派发
* Objective-C  (Message Dispatch)，兼容C函数调用方式

## Swift中的派发机制
Swift相对ObjcC来说，其在派发机制上的改变是显著的，在ObjC中所有的方法最终都会转成对`objc_msgSend`函数的调用， 其通过Runtime查找对应的IMP具体实现函数进行调用。而Swift因为其严格的类型安全机制，可以向编译器保证其声明类型的安全，这样的话， 编译器就可以建立函数表，在调用方法时只需要通过索引就可以找到具体函数执行，其效率上是比ObjC的方法查找高一些的
上面有提到Swift会通过建立函数表的方式进行方法调用，但进一步来说，Swift中函数的派发方式是受以下因素影响的：
1. [函数修饰符指定](#)
2. [类型和声明的作用域](#)
3. [编译器优化](#)

### 函数修饰符指定派发

#### dynamic
Swift文档中有明确指出，使用`dynamic`修饰时会通过ObjC运行时机制进行消息派发，并且`dynamic`修饰可以让extension中的函数被`override`，
用`dynamic`修饰的函数必须同时也被`@objc`修饰，否则编译会通不过`'dynamic' instance method 'xxxx()' must also be '@objc'`

> Apply this modifier to any member of a class that can be represented by Objective-C. When you mark a member declaration with the dynamic modifier, access to that member is always dynamically dispatched using the Objective-C runtime. Access to that member is never inlined or devirtualized by the compiler.  
> Because declarations marked with the dynamic modifier are dispatched using the Objective-C runtime, they must be marked with the objc attribute.  

```swift
class Goat: Animal {
    @objc dynamic func eatGrass() { }
}
```

#### @objc & @nonobjc
使用`@objc`修饰的函数可以被ObjC运行时捕获，例如Target-Action指定selector时需要显示的声明此函数可以被运行时捕获
`@nonobjc`刚好相反，其修饰作用就是指明该函数禁止使用消息派发，不让这个函数注册到Runtime中
```swift
class Goat: Animal {
    @objc func eatGrass() { }
    
    @nonobjc func drink() { }
}
```

#### final
`final`修饰的类或函数 ，指明其函数使用直接派发的方式，运行时无法捕获该函数，因此也就失去了动态特性
```swift
final class Cat: Animal {
    
    func stretching() { }
    
}

class Jellyfish: Animal {
    
    final func illuminate() {}
}
```

#### final @objc
在Swift中可以同时使用`final`和`@objc`l来修饰函数，其结果就是在直调用的时候会采用直接派发的方式，同时又可以响应selector运行时派发
```swift
class Jellyfish: Animal {
    
    @objc final func illuminate() {}
}
```

#### @inline
在Swift中，可以通过`@inline`声明内联函数，有两种形式，`@inline(never)` 和`@inline(__always)`，可以使用`@inline`修饰告诉编译器使用直接派发的方式
```swift
class Horse: Animal {
    
    @inline(__always) func run() { }
    @inline(never) func stop() { }
}
```


### 类型和声明的作用域

#### 值类型的函数总是采用直接派发的方式 （Struct/Enum）
```swift
struct Card {
    func reverse() {
        print("Reverse card")
    }
}

enum Currency {
    case hkd
    case usd
    
    func exchangeRate(to cy: Currency) -> Double {
        fatalError("Just an example")
    }
}
```

#### Protocol & Class （非继承至NSObject）的Extension 默认使用直接派发
```swift
protocol Render { }
extension Render {
    func render() { }
}

class Actor { }
extension Actor {
    func cry() { }
}
```

#### NSObject派生类声明的函数默认使用函数表进行派发
```swift
class MessageHelper: NSObject {
    func send() { }
    func read() { }
}
```

#### NSObject派生类Extension中函数默认使用消息机制派发
```swift
class MessageHelper: NSObject { }
extension MessageHelper {
    func send() { }
    func read() { }
}
```

#### Protocol定义在声明里，并有默认实现的函数会使用函数表进行派发
```swift
protocol Render {
    func render()
}
extension Render {
    func render() { }
}
```

### 编译器优化
编译器会通过`whole-module optimization`检查继承关系， 如果一个函数从来没有被`override`并且在编译时期就可以确定执行，那么Swift可能会采用直接派发的方式（另外，这个优化也会导致KVO失效，原因是如果绑定的属性未使用`dynamic`修饰的话，其`getter`和`setter`方法会被优化成直接派发）
```swift
class MessageHelper {
    
    /// 若使用KVO监听此属性会失效
    var timestamp: String = "xxx"
    
    /// 使用KVO监听需要dynamic显式修饰
    @objc dynamic var msgeID: String = "xx"
    
    /// 此方法可能会被优化成直接派发的方式执行
    func send() { }
}
```

## 小结
总的来说，Swift在不同的场景下选择了不同的派发方式，其中有 `Static Dispatch`，同时为了保留ObjC中的一些动态特性也存在`Dynamic Dispatch`。当然由于其复杂的派发方式，也带来了一些问题，其中包括像：

> 1.   
> Swift采用直接派发的方式对方法调用的性能带来了提升，同时也损失了部分Runtime的动态特性  
>   
> 2.   
> Swift在NSObject派生类中使用函数表派发，此举是否真正的带来了性能上的提升，并且在Extension中仍是采用消息机制，对这个做法存在疑惑  
>   
> 3.   
> 通过Selector绑定的方法或KVO监听的属性值还需要显示修饰成`@objc`或`dynamic`， 在编译器优化上是否可以检测到自动注册到Runtime中去？  
>   

### 参考文档
> [Swift ReferenceManual](https://docs.swift.org/swift-book/ReferenceManual/)  
> [Swifter - Swift 必备 tips](https://swifter.tips/objc-dynamic/)  
> [Method Dispatch in Swift - RaizException](https://www.raizlabs.com/dev/2016/12/swift-method-dispatch/?utm_campaign=This%2BWeek%2Bin%2BSwift&utm_medium=email&utm_source=This_Week_in_Swift_114)  
