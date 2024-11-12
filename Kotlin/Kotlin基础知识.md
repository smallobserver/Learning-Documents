# Kotlin基础知识

## lambda表达式



### Kotlin中lambda表达式的变量捕获及其原理

- 什么是变量捕获?

通过上述例子，我们知道在Kotlin中既能访问final的变量也能访问或修改非final的变量。原理是怎样的呢？在此之前先抛出一个高大上的概念叫做**lambdab表达式的变量捕获**。实际上就是lambda表达式在其函数体内可以访问外部的变量，我们就称这些外部变量被lambda表达式给捕获了。有了这个概念我们可以把上面的结论变得高大上一些:

第一在Java中lambda表达式只能捕获final修饰的变量

第二在Kotlin中lambda表达式既能捕获final修饰的变量也能访问和修改非final的变量



- 变量捕获实现的原理

我们都知道函数的局部变量生命周期是属于这个函数的，当函数执行完毕，局部变量也就是销毁了，但是如果这个局部变量被lambda捕获了，那么使用这个局部变量的代码将会被存储起来等待稍后再次执行，也就是被捕获的局部变量是可以延迟生命周期的，**针对lambda表达式捕获final修饰的局部变量原理是局部变量的值和使用这个值的lambda代码会被一起存储起来；而针对于捕获非final修饰的局部变量原理是非final局部变量会被一个特殊包装器类包装起来，这样就可以通过包装器类实例去修改这个非final的变量，那么这个包装器类实例引用是final的会和lambda代码一起存储**

以上第二条结论在Kotlin的语法层面来说是正确的，但是从真正的原理上来说是错误的，只不过是Kotlin在语法层面把这个屏蔽了而已，实质的原理lambda表达式还是只能捕获final修饰变量，而为什么kotlin却能做到修改非final的变量的值，**实际上kotlin在语法层面做了一个桥接包装，它把所谓的非final的变量用一个Ref包装类包装起来，然后外部保留着Ref包装器的引用是final的，然后lambda会和这个final包装器的引用一起存储，随后在lambda内部修改变量的值实际上是通过这个final的包装器引用去修改的。**



![lambda变量捕获](.\res\lambda变量捕获.jpg)



最后通过查看Kotlin修改非final局部变量的反编译成的Java代码就是一目了然了

```kotlin
class Demo2Activity : AppCompatActivity() {

    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        setContentView(R.layout.activity_demo2)
        var count = 0//声明非final类型
        btn_click.setOnClickListener {
            println(count++)//直接访问和修改非final类型的变量
        }
    }
}
@Metadata(
   mv = {1, 1, 9},
   bv = {1, 0, 2},
   k = 1,
   d1 = {"\u0000\u0018\n\u0002\u0018\u0002\n\u0002\u0018\u0002\n\u0002\b\u0002\n\u0002\u0010\u0002\n\u0000\n\u0002\u0018\u0002\n\u0000\u0018\u00002\u00020\u0001B\u0005¢\u0006\u0002\u0010\u0002J\u0012\u0010\u0003\u001a\u00020\u00042\b\u0010\u0005\u001a\u0004\u0018\u00010\u0006H\u0014¨\u0006\u0007"},
   d2 = {"Lcom/shanbay/prettyui/prettyui/Demo2Activity;", "Landroid/support/v7/app/AppCompatActivity;", "()V", "onCreate", "", "savedInstanceState", "Landroid/os/Bundle;", "production sources for module app"}
)
public final class Demo2Activity extends AppCompatActivity {
   private HashMap _$_findViewCache;

   protected void onCreate(@Nullable Bundle savedInstanceState) {
      super.onCreate(savedInstanceState);
      this.setContentView(2131361820);
      final IntRef count = new IntRef();//IntRef特殊的包装器类的类型，final修饰的IntRef的count引用
      count.element = 0;//包装器内部的非final变量element
      ((Button)this._$_findCachedViewById(id.btn_click)).setOnClickListener((OnClickListener)(new OnClickListener() {
         public final void onClick(View it) {
            int var2 = count.element++;//直接是通过IntRef的引用直接修改内部的非final变量的值，来达到语法层面的lambda直接修改非final局部变量的值
            System.out.println(var2);
         }
      }));
   }

   public View _$_findCachedViewById(int var1) {
      if(this._$_findViewCache == null) {
         this._$_findViewCache = new HashMap();
      }

      View var2 = (View)this._$_findViewCache.get(Integer.valueOf(var1));
      if(var2 == null) {
         var2 = this.findViewById(var1);
         this._$_findViewCache.put(Integer.valueOf(var1), var2);
      }

      return var2;
   }

   public void _$_clearFindViewByIdCache() {
      if(this._$_findViewCache != null) {
         this._$_findViewCache.clear();
      }

   }
}
```

### 高阶函数和Lambda表达式

Kotlin支持将函数赋值给一个变量并把它们作为函数参数传递给其他函数。接受其他函数作为参数的函数称为*高阶函数*

Kotlin函数可以通过它的前缀`::`的声明引用，或者直接在代码块内部作为匿名函数声明，或者使用[lambda表达式语法](https://link.zhihu.com/?target=https%3A//kotlinlang.org/docs/reference/lambdas.html%23lambda-expression-syntax)，这是描述函数最紧凑方法。

Kotlin中最具有吸引力语法特性之一就是为Java 6/7 JVM和Android提供lambda表达式的支持。

考虑以下函数实例，该函数在数据库事务中执行任意操作并返回受影响的行数：

```kotlin
fun transaction(db: Database, body: (Database) -> Int): Int {
    db.beginTransaction()
    try {
        val result = body(db)
        db.setTransactionSuccessful()
        return result
    } finally {
        db.endTransaction()
    }
}
```

我们可以通过使用类似于Groovy的语法将lambda表达式作为最后一个参数传递来调用此函数：

```kotlin
val deletedRows = transaction(db) {
    it.delete("Customers", null, null)
}
```

但是Java 6 JVM不直接支持lambda表达式。那么它们如何转换为字节码？正如你所预料的，lambdas和匿名函数被编译为Function对象。

### lambda表达式实质原理

Kotlin中的lambda表达式实际上最后会编译为一个class类，这个类会去继承Kotlin中Lambda的抽象类(在kotlin.jvm.internal包中)并且实现一个FunctionN(在kotlin.jvm.functions包中)的接口(这个N是根据lambda表达式传入参数的个数决定的,目前接口N的取值为 0 <= N <= 22,也就是lambda表达式中函数传入的参数最多也只能是22个)，这个Lambda抽象类是实现了FunctionBase接口，该接口中有两个方法一个是getArity()获取lambda参数的元数，toString()实际上就是打印出Lambda表达式类型字符串，获取Lambda表达式类型字符串是通过Java中Reflection类反射来实现的。FunctionBase接口继承了Function,Serializable接口。

```kotlin
//一个简单的lambda例子
package com.mikyou.kotlin.lambda.simple

typealias Sum = (Int, Int, Int) -> Int

fun main(args: Array<String>) {
    val sum: Sum = { a, b, c ->//定义一个很简单的三个数求和的lambda表达式
        a + b + c
    }

    println(sum.invoke(1, 2, 3))
}

```

```java
//lambda调用处反编译后的代码
package com.mikyou.kotlin.lambda.simple;

import kotlin.Metadata;
import kotlin.jvm.functions.Function3;
import kotlin.jvm.internal.Intrinsics;
import org.jetbrains.annotations.NotNull;

@Metadata(
   mv = {1, 1, 10},
   bv = {1, 0, 2},
   k = 2,
   d1 = {"\u0000\u001e\n\u0000\n\u0002\u0010\u0002\n\u0000\n\u0002\u0010\u0011\n\u0002\u0010\u000e\n\u0002\b\u0002\n\u0002\u0018\u0002\n\u0002\u0010\b\n\u0000\u001a\u0019\u0010\u0000\u001a\u00020\u00012\f\u0010\u0002\u001a\b\u0012\u0004\u0012\u00020\u00040\u0003¢\u0006\u0002\u0010\u0005*:\u0010\u0006\"\u001a\u0012\u0004\u0012\u00020\b\u0012\u0004\u0012\u00020\b\u0012\u0004\u0012\u00020\b\u0012\u0004\u0012\u00020\b0\u00072\u001a\u0012\u0004\u0012\u00020\b\u0012\u0004\u0012\u00020\b\u0012\u0004\u0012\u00020\b\u0012\u0004\u0012\u00020\b0\u0007¨\u0006\t"},
   d2 = {"main", "", "args", "", "", "([Ljava/lang/String;)V", "Sum", "Lkotlin/Function3;", "", "production sources for module Lambda_main"}
)
public final class SumLambdaKt {
   public static final void main(@NotNull String[] args) {
      Intrinsics.checkParameterIsNotNull(args, "args");
      Function3 sum = (Function3)null.INSTANCE;//实例化的是FunctionN接口中Function3，因为有三个参数
      int var2 = ((Number)sum.invoke(1, 2, 3)).intValue();
      System.out.println(var2);
   }
}

//lambda反编译后的代码
package com.mikyou.kotlin.lambda.simple;

import kotlin.jvm.internal.Lambda;

@kotlin.Metadata(mv = {1, 1, 10}, bv = {1, 0, 2}, k = 3, d1 = {"\000\n\n\000\n\002\020\b\n\002\b\004\020\000\032\0020\0012\006\020\002\032\0020\0012\006\020\003\032\0020\0012\006\020\004\032\0020\001H\n¢\006\002\b\005"}, d2 = {"<anonymous>", "", "a", "b", "c", "invoke"})
final class SumLambdaKt$main$sum$1 extends Lambda implements kotlin.jvm.functions.Function3<Integer, Integer, Integer, Integer> {
    public final int invoke(int a, int b, int c) {
        return a + b + c;
    }

    public static final SumLambdaKt$main$sum$1 INSTANCE =new SumLambdaKt$main$sum$1();

    SumLambdaKt$main$sum$1() {
        super(3);//这个super传入3,也就是前面getArity获得参数的元数和函数参数个数一致
    }
}

```

注意: 对于Lambda表达式内部修改局部变量的值，只会在这个Lambda表达式被执行的时候触发。





## 伴生对象

### 描述

Kotlin中没有static关键字来使用静态方法，取而代之使用`companion object`关键字来生命，但是底层会生成一个Companion的静态类，具体实现会在其中，而原本类持有静态的Companion的实例。

kotlin版本1.6.21

```kotlin
//原始Kotlin代码
class Test{
    companion object{
        val TAG = "TAG"
        
        const val CompanionProperty = "I'm a companion property"
        
        const val num = 23
        
        val objects = Any()

        fun create(): Any = Any()

        @JvmStatic
        fun createJVM(): Any = Any()
    }
}

//反编译的java代码
public final class Test {
   @NotNull
   private static final String TAG = "TAG";
   @NotNull
   public static final String CompanionProperty = "I'm a companion property";
   public static final int num = 23;
   @NotNull
   private static final Object objects = new Object();
   @NotNull
   public static final Companion Companion = new Companion((DefaultConstructorMarker)null);
    
    // synthetic 
   public static final String access$getTAG$cp() {
        return TAG;
   }
    
   @JvmStatic
   @NotNull
   public static final Object createJVM() {
      return Companion.createJVM();
   }

   //描述注解，记录Kotlin类信息
   @Metadata(
      mv = {1, 6, 0},
      k = 1,
      d1 = {"\u0000\u001a\n\u0002\u0018\u0002\n\u0002\u0010\u0000\n\u0002\b\u0002\n\u0002\u0010\u000e\n\u0000\n\u0002\u0010\b\n\u0002\b\u0006\b\u0086\u0003\u0018\u00002\u00020\u0001B\u0007\b\u0002¢\u0006\u0002\u0010\u0002J\u0006\u0010\n\u001a\u00020\u0001J\b\u0010\u000b\u001a\u00020\u0001H\u0007R\u000e\u0010\u0003\u001a\u00020\u0004X\u0086T¢\u0006\u0002\n\u0000R\u000e\u0010\u0005\u001a\u00020\u0006X\u0086T¢\u0006\u0002\n\u0000R\u0011\u0010\u0007\u001a\u00020\u0001¢\u0006\b\n\u0000\u001a\u0004\b\b\u0010\t¨\u0006\f"},
      d2 = {"Lcom/tinyblack/Test$Companion;", "", "()V", "CompanionProperty", "", "TAG", "getTAG", "()Ljava/lang/String;", "num", "", "objects", "getObjects", "()Ljava/lang/Object;", "create", "createJVM", "lib_vehicle_debug"}
   )
   public static final class Companion {
      @NotNull
      public final String getTAG() {
         return Test.TAG;
         //字节码
         //return Test.access$getTAG$cp();
      }

      @NotNull
      public final Object getObjects() {
         return Test.objects;
      }

      @NotNull
      public final Object create() {
         return new Object();
      }

      @JvmStatic
      @NotNull
      public final Object createJVM() {
         return new Object();
      }

      private Companion() {
      }

      // $FF: synthetic method
      public Companion(DefaultConstructorMarker $constructor_marker) {
         this();
      }
   }
}    

```

### 总结

在`companion object`中，`const`定义的基础类型常量，会直接`public static final`修饰生成在原类中，java可以直接类名.访问，但是非常量类型，`private static final`生成在类中，访问需要在伴生对象`companion`中，需要通过其的get方法访问。函数略有区别，普通的函数，会仅在`companion`中生成，如果使用`@JvmStatic`注解修饰的，会在类中额外生成一个同名静态方法，但是底层还是调用到了`companion`中的同名静态方法。





## 委托

### 描述

委托模式已经证明是实现继承的一个很好的替代方式， 而 Kotlin 可以零样板代码地原生支持它。

`Derived` 类可以通过将其所有公有成员都委托给指定对象来实现一个接口 `Base`：

```kotlin
interface Base {
    fun print()
}

class BaseImpl(val x: Int) : Base {
    override fun print() { print(x) }
}

class Derived(b: Base) : Base by b

fun main() {
    val b = BaseImpl(10)
    Derived(b).print()
}
```

`Derived` 的超类型列表中的 `by`-子句表示 `b` 将会在 `Derived` 中内部存储， 并且编译器将生成转发给 `b` 的所有 `Base` 的方法。

### 覆盖由委托实现的接口成员

覆盖符合预期：编译器会使用 `override` 覆盖的实现而不是委托对象中的。如果将 `override fun printMessage() { print("abc") }` 添加到 `Derived`，那么当调用 `printMessage` 时程序会输出 *abc* 而不是 *10*：

```kotlin
interface Base {
    fun printMessage()
    fun printMessageLine()
}

class BaseImpl(val x: Int) : Base {
    override fun printMessage() { print(x) }
    override fun printMessageLine() { println(x) }
}

class Derived(b: Base) : Base by b {
    override fun printMessage() { print("abc") }
}

fun main() {
    val b = BaseImpl(10)
    Derived(b).printMessage()
    Derived(b).printMessageLine()
}
```

但请注意，以这种方式重写的成员不会在委托对象的成员中调用 ，委托对象的成员只能访问其自身对接口成员实现：

```kotlin
interface Base {
    val message: String
    fun print()
}

class BaseImpl(val x: Int) : Base {
    override val message = "BaseImpl: x = $x"
    override fun print() { println(message) }
}

class Derived(b: Base) : Base by b {
    // 在 b 的 `print` 实现中不会访问到这个属性
    override val message = "Message of Derived"
}

fun main() {
    val b = BaseImpl(10)
    val derived = Derived(b)
    derived.print()
    println(derived.message)
}
```



### 属性委托

With some common kinds of properties, even though you can implement them manually every time you need them, it is more helpful to implement them once, add them to a library, and reuse them later. For example:

- 延迟属性（*lazy* properties）: 其值只在首次访问时计算。
- 可观察属性（*observable* properties）: 监听器会收到有关此属性变更的通知。
- 把多个属性储存在一个映射（*map*）中，而不是每个存在单独的字段中。

为了涵盖这些（以及其他）情况，Kotlin 支持 *委托属性*:

```kotlin
class Example {
    var p: String by Delegate()
}
```

语法是： `val/var <属性名>: <类型> by <表达式>`。在 `by` 后面的表达式是该 *委托*， 因为属性对应的 `get()`（与 `set()`）会被委托给它的 `getValue()` 与 `setValue()` 方法。 属性的委托不必实现接口，但是需要提供一个 `getValue()` 函数（对于 `var` 属性还有 `setValue()`）。

例如:

```kotlin
import kotlin.reflect.KProperty

class Delegate {
    //thisRef：表示属性的拥有者（引用对象），即在哪个对象中使用了这个委托属性。
	//property：表示被委托的属性本身，类型为 KProperty<*>，包含属性的元数据信息。
    operator fun getValue(thisRef: Any?, property: KProperty<*>): String {
        return "$thisRef, thank you for delegating '${property.name}' to me!"
    }

    operator fun setValue(thisRef: Any?, property: KProperty<*>, value: String) {
        println("$value has been assigned to '${property.name}' in $thisRef.")
    }
}
```

当从委托到一个 `Delegate` 实例的 `p` 读取时，将调用 `Delegate` 中的 `getValue()` 函数。 它的第一个参数是读出 `p` 的对象、第二个参数保存了对 `p` 自身的描述 （例如可以取它的名称)。

```kotlin
val e = Example()
println(e.p)
```

输出结果：

```
Example@33a17727, thank you for delegating 'p' to me!
```

类似地，当我们给 `p` 赋值时，将调用 `setValue()` 函数。前两个参数相同， 第三个参数保存将要被赋予的值：

```kotlin
e.p = "NEW"
```

输出结果：

```
NEW has been assigned to 'p' in Example@33a17727.
```

可以在函数或代码块中声明一个委托属性；它不一定是类的成员。 





#### 标准委托

Kotlin 标准库为几种有用的委托提供了工厂方法。



##### 延迟属性 Lazy properties

`lazy()` 是接受一个 lambda 并返回一个 `Lazy <T>` 实例的函数，返回的实例可以作为实现延迟属性的委托。 第一次调用 `get()` 会执行已传递给 `lazy()` 的 lambda 表达式并记录结果。 后续调用 `get()` 只是返回记录的结果。

```kotlin
val lazyValue: String by lazy {
    println("computed!")
    "Hello"
}

fun main() {
    println(lazyValue)
    println(lazyValue)
}
```

默认情况下，对于 lazy 属性的求值是*同步锁的（synchronized）*：该值只在一个线程中计算，但所有线程都会看到相同的值。如果初始化委托的同步锁不是必需的，这样可以让多个线程同时执行，那么将 `LazyThreadSafetyMode.PUBLICATION` 作为参数传给 `lazy()`。

如果你确定初始化将总是发生在与属性使用位于相同的线程， 那么可以使用 `LazyThreadSafetyMode.NONE` 模式。它不会有任何线程安全的保证以及相关的开销。



##### 可观察属性 Observable properties

`Delegates.observable()` 接受两个参数：初始值与修改时处理程序（handler）。

每当我们给属性赋值时会调用该处理程序（在赋值*后*执行）。它有三个参数：被赋值的属性、旧值与新值：

```kotlin
import kotlin.properties.Delegates

class User {
    var name: String by Delegates.observable("<no name>") {
        prop, old, new ->
        println("$old -> $new")
    }
}

fun main() {
    val user = User()
    user.name = "first"
    user.name = "second"
}
```

如果你想截获赋值并*否决*它们，那么使用 `vetoable()`取代 `observable()`。 在属性被赋新值*之前*会调用传递给 `vetoable` 的处理程序。



### 委托给另一个属性

一个属性可以把它的 getter 与 setter 委托给另一个属性。这种委托对于顶层和类的属性（成员和扩展）都可用。该委托属性可以为：

- 顶层属性
- 同一个类的成员或扩展属性
- 另一个类的成员或扩展属性

为将一个属性委托给另一个属性，应在委托名称中使用 `::` 限定符，例如，`this::delegate` 或 `MyClass::delegate`。

```kotlin
var topLevelInt: Int = 0
class ClassWithDelegate(val anotherClassInt: Int)

class MyClass(var memberInt: Int, val anotherClassInstance: ClassWithDelegate) {
    var delegatedToMember: Int by this::memberInt
    var delegatedToTopLevel: Int by ::topLevelInt

    val delegatedToAnotherClass: Int by anotherClassInstance::anotherClassInt
}
var MyClass.extDelegated: Int by ::topLevelInt
```

这是很有用的，例如，当想要以一种向后兼容的方式重命名一个属性的时候：引入一个新的属性、 使用 `@Deprecated` 注解来注解旧的属性、并委托其实现。

```kotlin
class MyClass {
   var newName: Int = 0
   @Deprecated("Use 'newName' instead", ReplaceWith("newName"))
   var oldName: Int by this::newName
}
fun main() {
   val myClass = MyClass()
   // 通知：'oldName: Int' is deprecated.
   // Use 'newName' instead
   myClass.oldName = 42
   println(myClass.newName) // 42
}
```





### 将属性储存在映射中

一个常见的用例是在一个映射（map）里存储属性的值。 这经常出现在像解析 JSON 或者执行其他“动态”任务的应用中。 在这种情况下，你可以使用映射实例自身作为委托来实现委托属性。

```kotlin
class User(val map: Map<String, Any?>) {
    val name: String by map
    val age: Int     by map
}
```

在这个例子中，构造函数接受一个映射参数：

```kotlin
val user = User(mapOf(
    "name" to "John Doe",
    "age"  to 25
))
```

Delegated properties take values from this map through string keys, which are associated with the names of properties:

```kotlin
class User(val map: Map<String, Any?>) {
    val name: String by map
    val age: Int     by map
}

fun main() {
    val user = User(mapOf(
        "name" to "John Doe",
        "age"  to 25
    ))
//sampleStart
    println(user.name) // Prints "John Doe"
    println(user.age)  // Prints 25
//sampleEnd
}
```

这也适用于 `var` 属性，如果把只读的 `Map` 换成 `MutableMap` 的话：

```kotlin
class MutableUser(val map: MutableMap<String, Any?>) {
    var name: String by map
    var age: Int     by map
}
```





### 局部委托属性

你可以将局部变量声明为委托属性。 例如，你可以使一个局部变量惰性初始化：

```kotlin
fun example(computeFoo: () -> Foo) {
    val memoizedFoo by lazy(computeFoo)

    if (someCondition && memoizedFoo.isValid()) {
        memoizedFoo.doSomething()
    }
}
```

`memoizedFoo` 变量只会在第一次访问时计算。 如果 `someCondition` 失败，那么该变量根本不会计算。





### 属性委托要求

对于一个*只读*属性（即 `val` 声明的），委托必须提供一个操作符函数 `getValue()`，该函数具有以下参数：

- `thisRef` 必须与*属性所有者*类型（对于扩展属性必须是被扩展的类型）相同或者是其超类型。
- `property` 必须是类型 `KProperty<*>` 或其超类型。

`getValue()` 必须返回与属性相同的类型（或其子类型）。

```kotlin
class Resource

class Owner {
    val valResource: Resource by ResourceDelegate()
}

class ResourceDelegate {
    operator fun getValue(thisRef: Owner, property: KProperty<*>): Resource {
        return Resource()
    }
}
```

对于一个**可变**属性（即 `var` 声明的），委托必须额外提供一个操作符函数 `setValue()`， 该函数具有以下参数：

- `thisRef` 必须与*属性所有者*类型（对于扩展属性必须是被扩展的类型）相同或者是其超类型。
- `property` 必须是类型 `KProperty<*>` 或其超类型。
- `value` 必须与属性类型相同（或者是其超类型）。

```kotlin
class Resource

class Owner {
    var varResource: Resource by ResourceDelegate()
}

class ResourceDelegate(private var resource: Resource = Resource()) {
    operator fun getValue(thisRef: Owner, property: KProperty<*>): Resource {
        return resource
    }
    operator fun setValue(thisRef: Owner, property: KProperty<*>, value: Any?) {
        if (value is Resource) {
            resource = value
        }
    }
}
```

`getValue()` 或/与 `setValue()` 函数可以通过委托类的成员函数提供或者由扩展函数提供。 当你需要委托属性到原本未提供的这些函数的对象时后者会更便利。 两函数都需要用 `operator` 关键字来进行标记。

You can create delegates as anonymous objects without creating new classes, by using the interfaces `ReadOnlyProperty` and `ReadWriteProperty` from the Kotlin standard library. They provide the required methods: `getValue()` is declared in `ReadOnlyProperty`; `ReadWriteProperty` extends it and adds `setValue()`. This means you can pass a `ReadWriteProperty` whenever a `ReadOnlyProperty` is expected.

```kotlin
fun resourceDelegate(resource: Resource = Resource()): ReadWriteProperty<Any?, Resource> =
    object : ReadWriteProperty<Any?, Resource> {
        var curValue = resource 
        override fun getValue(thisRef: Any?, property: KProperty<*>): Resource = curValue
        override fun setValue(thisRef: Any?, property: KProperty<*>, value: Resource) {
            curValue = value
        }
    }

val readOnlyResource: Resource by resourceDelegate()  // ReadWriteProperty as val
var readWriteResource: Resource by resourceDelegate()
```



### Translation rules for delegated properties

在底层，Kotlin 编译器会为某些类型的委托属性生成辅助属性并委托给它们。

> For the optimization purposes, the compiler [*does not* generate auxiliary properties in several cases](https://book.kotlincn.net/text/delegated-properties.html#optimized-cases-for-delegated-properties). Learn about the optimization on the example of [delegating to another property](https://book.kotlincn.net/text/delegated-properties.html#translation-rules-when-delegating-to-another-property).
>
> <svg width="24" height="24" fill="#4dbb5f" viewBox="0 0 24 24"><path d="M21 12a9 9 0 1 1-9-9 9 9 0 0 1 9 9zM10.5 7.5A1.5 1.5 0 1 0 12 6a1.5 1.5 0 0 0-1.5 1.5zm-.5 3.54v1h1V18h2v-6a.96.96 0 0 0-.96-.96z"></path></svg>

例如，对于属性 `prop`，生成隐藏属性 `prop$delegate`，而访问器的代码只是简单地委托给这个附加属性：

```kotlin
class C {
    var prop: Type by MyDelegate()
}

// 这段是由编译器生成的相应代码：
class C {
    private val prop$delegate = MyDelegate()
    var prop: Type
        get() = prop$delegate.getValue(this, this::prop)
        set(value: Type) = prop$delegate.setValue(this, this::prop, value)
}
```

Kotlin 编译器在参数中提供了关于 `prop` 的所有必要信息：第一个参数 `this` 引用到外部类 `C` 的实例，而 `this::prop` 是 `KProperty` 类型的反射对象，该对象描述 `prop` 自身。

#### Optimized cases for delegated properties

The `$delegate` field will be omitted if a delegate is:

- A referenced property:

  ```kotlin
  class C<Type> {
      private var impl: Type = ...
      var prop: Type by ::impl
  }
  ```

- A named object:

  ```kotlin
  object NamedObject {
      operator fun getValue(thisRef: Any?, property: KProperty<*>): String = ...
  }
  
  val s: String by NamedObject
  ```

- A final `val` property with a backing field and a default getter in the same module:

  ```kotlin
  val impl: ReadOnlyProperty<Any?, String> = ...
  
  class A {
      val s: String by impl
  }
  ```

- A constant expression, enum entry, `this`, `null`. The example of `this`:

  ```kotlin
  class A {
      operator fun getValue(thisRef: Any?, property: KProperty<*>) ...
  
      val s by this
  }
  ```

#### Translation rules when delegating to another property

When delegating to another property, the Kotlin compiler generates immediate access to the referenced property. This means that the compiler doesn't generate the field `prop$delegate`. This optimization helps save memory.

Take the following code, for example:

```kotlin
class C<Type> {
    private var impl: Type = ...
    var prop: Type by ::impl
}
```

Property accessors of the `prop` variable invoke the `impl` variable directly, skipping the delegated property's `getValue`and `setValue` operators, and thus the `KProperty` reference object is not needed.

For the code above, the compiler generates the following code:

```kotlin
class C<Type> {
    private var impl: Type = ...

    var prop: Type
        get() = impl
        set(value) {
            impl = value
        }

    fun getProp$delegate(): Type = impl // This method is needed only for reflection
}
```





### 提供委托

通过定义 `provideDelegate` 操作符，可以扩展创建属性实现所委托对象的逻辑。 如果 `by` 右侧所使用的对象将 `provideDelegate` 定义为成员或扩展函数， 那么会调用该函数来创建属性委托实例。

One of the possible use cases of `provideDelegate` is to check the consistency of the property upon its initialization.

例如，如需在绑定之前检测属性名称，可以这样写：

```kotlin
class ResourceDelegate<T> : ReadOnlyProperty<MyUI, T> {
    override fun getValue(thisRef: MyUI, property: KProperty<*>): T { ... }
}

class ResourceLoader<T>(id: ResourceID<T>) {
    operator fun provideDelegate(
            thisRef: MyUI,
            prop: KProperty<*>
    ): ReadOnlyProperty<MyUI, T> {
        checkProperty(thisRef, prop.name)
        // 创建委托
        return ResourceDelegate()
    }

    private fun checkProperty(thisRef: MyUI, name: String) { …… }
}

class MyUI {
    fun <T> bindResource(id: ResourceID<T>): ResourceLoader<T> { …… }

    val image by bindResource(ResourceID.image_id)
    val text by bindResource(ResourceID.text_id)
}
```

`provideDelegate` 的参数与 `getValue` 的相同：

- `thisRef` 必须与 *属性所有者* 类型（对于扩展属性必须是被扩展的类型）相同或者是它的超类型；
- `property` 必须是类型 `KProperty<*>` 或其超类型。

在创建 `MyUI` 实例期间，为每个属性调用 `provideDelegate` 方法，并立即执行必要的验证。

如果没有这种拦截属性与其委托之间的绑定的能力，为了实现相同的功能， 你必须显式传递属性名，这不是很方便：

```kotlin
// 检测属性名称而不使用“provideDelegate”功能
class MyUI {
    val image by bindResource(ResourceID.image_id, "image")
    val text by bindResource(ResourceID.text_id, "text")
}

fun <T> MyUI.bindResource(
        id: ResourceID<T>,
        propertyName: String
): ReadOnlyProperty<MyUI, T> {
    checkProperty(this, propertyName)
   // 创建委托
}
```

在生成的代码中，会调用 `provideDelegate` 方法来初始化辅助的 `prop$delegate` 属性。 比较对于属性声明 `val prop: Type by MyDelegate()` 生成的代码与[上面](https://book.kotlincn.net/text/delegated-properties.html#translation-rules-for-delegated-properties)（当 `provideDelegate` 方法不存在时）生成的代码：

```kotlin
class C {
    var prop: Type by MyDelegate()
}

// 这段代码是当“provideDelegate”功能可用时
// 由编译器生成的代码：
class C {
    // 调用“provideDelegate”来创建额外的“delegate”属性
    private val prop$delegate = MyDelegate().provideDelegate(this, this::prop)
    var prop: Type
        get() = prop$delegate.getValue(this, this::prop)
        set(value: Type) = prop$delegate.setValue(this, this::prop, value)
}
```

请注意，`provideDelegate` 方法只影响辅助属性的创建，并不会影响为 getter 或 setter 生成的代码。

With the `PropertyDelegateProvider` interface from the standard library, you can create delegate providers without creating new classes.

```kotlin
val provider = PropertyDelegateProvider { thisRef: Any?, property ->
    ReadOnlyProperty<Any?, Int> {_, property -> 42 }
}
val delegate: Int by provider
```





### by lazy的工作原理

实际上就是一个属性委托!

我们可以把`by lazy`修饰的属性理解为是具有`lazy`委托的委托属性。

所以，`lazy`是如何工作的呢？ 让我们一起在Kotlin标准库参考中总结`lazy()`方法，如下所示：

- 1、`lazy()` 返回的是一个存储在lambda初始化器中的`Lazy<T>`类型实例。
- 2、getter的第一次调用执行传递给`lazy()`的lambda并存储其结果。
- 3、后面的话，getter调用只返回存储中的值。

> 简单地说，lazy创建一个实例，在第一次访问属性值时执行初始化，存储结果并返回存储的值。



##### 带有`lazy()`的委托属性

让我们编写一个简单的Kotlin代码来检查`lazy`的实现。

```kotlin
class Demo {
    val myName: String by lazy { "John" }
}
```

如果你将其反编译为Java代码，则可以看到以下代码：

```java
public final class Demo {
    @NotNull
    private final Lazy myName$delegate;
    
    // $FF: synthetic field
    static final KProperty[] $$delegatedProperties = ...
    @NotNull
    public final String getMyName() {
        Lazy var1 = this.myName$delegate;
        KProperty var3 = $$delegatedProperties[0];
        return (String)var1.getValue();
    }
    public Demo() {
        this.myName$delegate =
            LazyKt.lazy((Function0)null.INSTANCE);
    }
}
```

- `$delegate`后缀被拼接到字段名称后面: `myName$delegate`
- 注意`myName$delegate`的类型是`Lazy`类型不是`String`类型
- 在构造器中，`LazyKt.lazy()`函数返回值赋值给了`myName$delegate`
- `LazyKt.lazy()`方法负责执行指定的初始化块

调用`getMyName()`方法实际过程是将通过调用`myName$delegate`的`Lazy`
实例中的`getValue()`方法并返回相应的值。

##### Lazy的具体实现

`lazy()`方法返回的是一个`Lazy<T>`类型的对象，该对象处理lambda函数（初始化程序块），根据线程执行模式（LazyThreadSafetyMode）以稍微几种不同的方式执行初始化。

```kotlin
@kotlin.jvm.JvmVersion
public fun <T> lazy(
    mode: LazyThreadSafetyMode,
    initializer: () -> T
): Lazy<T> =
    when (mode) {
        LazyThreadSafetyMode.SYNCHRONIZED ->
            SynchronizedLazyImpl(initializer)
        LazyThreadSafetyMode.PUBLICATION ->
            SafePublicationLazyImpl(initializer)
        LazyThreadSafetyMode.NONE ->
            UnsafeLazyImpl(initializer)
    }
```

所有这些都负责调用给定的lambda块进行延迟初始化

**SYNCHRONIZED** → ***SynchronizedLazyImpl***

- 初始化操作仅仅在首先调用的第一个线程上执行
- 然后，其他线程将引用缓存后的值。
- 默认模式就是(LazyThreadSafetyMode.SYNCHRONIZED)

**PUBLICATION** → ***SafePublicationLazyImpl***

- 它可以同时在多个线程中调用，并且可以在全部或部分线程上同时进行初始化。
- 但是，如果某个值已由另一个线程初始化，则将返回该值而不执行初始化。

**NONE** → ***UnsafeLazyImpl***

- 只需在第一次访问时初始化它，或返回存储的值。
- 不考虑多线程，所以它不是线程安全的。

##### Lazy实现的默认行为

`SynchronizedLazyImpl`和`SafePublicationLazyImpl`，`UnsafeLazyImpl`通过以下过程执行延迟初始化。我们来看看前面的例子。

 ![lazy原理_0](.\res\lazy原理_0.webp)



- 1、将传入的初始化lambda块存储在属性的 `initializer`中

 ![lazy原理_1](.\res\lazy原理_1.webp)

- 2、通过属性`_value`来存储值。此属性最开始初始值为`UNINITIALIZED_VALUE`。

 ![lazy原理_2](.\res\lazy原理_2.webp)

- 3、在执行读取操作(属性get访问器)时，如果`_value`的值是最开始初始值`UNINITIALIZED_VALUE`，那么就会去执行 `initializer`初始化器

 ![lazy原理_3](.\res\lazy原理_3.webp)

- 4、在执行读取操作(属性get访问器)时，如果`_value`的值不是等于`UNINITIALIZED_VALUE`, 那就说明初始化操作已经执行完成了。

 ![lazy原理_4](.\res\lazy原理_4.webp)



##### SynchronizedLazyImpl

如果你没有明确指定具体模式，延迟具体实现就是`SynchronizedLazyImpl`，它默认只执行一次初始化。我们来看看它的实现代码。

```kotlin
private object UNINITIALIZED_VALUE
private class SynchronizedLazyImpl<out T>(
        initializer: () -> T,
        lock: Any? = null
) : Lazy<T>,
    Serializable {
    private var initializer: (() -> T)? = initializer
    @Volatile private var _value: Any? = UNINITIALIZED_VALUE
    // final field is required
    // to enable safe publication of constructed instance
    private val lock = lock ?: this
    override val value: T
        get() {
            val _v1 = _value
            if (_v1 !== UNINITIALIZED_VALUE) {
                @Suppress("UNCHECKED_CAST")
                return _v1 as T
            }
            return synchronized(lock) {
                val _v2 = _value
                if (_v2 !== UNINITIALIZED_VALUE) {
                    @Suppress("UNCHECKED_CAST") (_v2 as T)
                }
                else {
                    val typedValue = initializer!!()
                    _value = typedValue
                    initializer = null
                    typedValue
                }
            }
        }
    override fun isInitialized(): Boolean =
            _value !== UNINITIALIZED_VALUE
    override fun toString(): String =
            if (isInitialized()) value.toString()
            else "Lazy value not initialized yet."
    private fun writeReplace(): Any =
            InitializedLazyImpl(value)
}
```

这看起来有点复杂。但它只是多线程的一般实现。

- 使用`synchronized()`同步块执行初始化块
- 由于初始化可能已经由另一个线程完成，它会进行双重锁检测(DCL),如果已经完成了初始化，则返回存储的值。
- 如果它未初始化，它将执行lambda表达式并存储返回值。那么随后这个`initializer`将会置为`null`，因为初始化完成后就不再需要它了。

#### Kotlin的委托属性将会让你快乐

当然，延迟初始化有时会导致问题发生或通过绕过控制流并在异常情况下生成正常值来使调试变得困难。

但是，如果你对这些情况非常谨慎，那么Kotlin的延迟初始化可以使我们更加自由地避免对线程安全性和性能的担忧。

我们还研究了延迟初始化是运算符 `by` 和`lazy`函数的共同作用的结果。还有更多的委托，如`Observable`和`notNull`。如有必要，你还可以实现有趣的委托属性。





## 内联类inline

这是Kotlin标准库中的`filter`函数的简化版本的源码:

```kotlin
inline fun <T> Iterable<T>.filter(predicate: (T)->Boolean): List<T>{
    val destination = ArrayList<T>()
    for (element in this) 
        if (predicate(element))
            destination.add(element)
    return destination
}
```

这个`inline`修饰符到底有多重要呢? 假设我们有5000件商品，我们需要对已经购买的商品累计算出总价。我们可以通过以下方式完成:

```kotlin
products.filter{ it.bought }.sumByDouble { it.price }
```

在我的机器上，运行上述代码平均需要38毫秒。如果这个函数不是内联的话会是多长时间呢? 不是内联在我的机器上大概平均42毫秒。你们可以自己检查尝试下,[这里是完整源码](https://link.zhihu.com/?target=https%3A//github.com/MarcinMoskala/effective-kotlin-tests/blob/master/src/main/kotlin/org/kotlinacademy/InlineFilterBenchmark.kt). 这似乎看起来差距不是很大，但每调用一次这个函数对集合进行处理时，你都会注意到这个时间差距大约为10％左右。

当我们修改lambda表达式中的局部变量时，可以发现差距将会更大。对比下面两个函数:

```kotlin
inline fun repeat(times: Int, action: (Int) -> Unit) {
    for (index in 0 until times) {
        action(index)
    }
}

fun noinlineRepeat(times: Int, action: (Int) -> Unit) {
    for (index in 0 until times) {
        action(index)
    }
}
```

你可能已经注意到除了函数名不一样之外，唯一的区别就是第一个函数使用`inline`修饰符，而第二个函数没有。用法也是完全一样的:

```kotlin
var a = 0
repeat(100_000_000) {
    a += 1
}
var b = 0
noinlineRepeat(100_000_000) {
    b += 1
}
```

上述代码在执行时间上对比有很大的差异。内联的`repeat`函数平均运行时间是0.335ns, 而`noinlineRepeat`函数平均运行时间是153980484.884ns。大概是内联`repeat`函数运行时间的466000倍! 你们可以自己检查尝试下,[这里是完整源码](https://link.zhihu.com/?target=https%3A//github.com/MarcinMoskala/effective-kotlin-tests/blob/master/src/main/kotlin/org/kotlinacademy/InlineRepeatBenchmark.kt).

为什么这个如此重要呢? 这种性能的提升是否有其他的成本呢? 我们应该什么时候使用内联(inline)修饰符呢？这些都是重点问题，我们将尽力回答这些问题。然而这一切都需要从最基本的问题开始: 内联修饰符到底有什么作用?

### 内联修饰符有什么作用？

我们都知道函数通常是如何被调用的。先执行跳转到函数体，然后执行函数体内所有的语句，最后跳回到最初调用函数的位置。

尽管强行对函数使用`inline`修饰符标记，但是编译器将会以不同的方式来对它进行处理。在代码编译期间，它用它的主体替换这样的函数调用。 `print`函数是`inline`函数:

```kotlin
public inline fun print(message: Int) {
    System.out.print(message)
}
```

当我们在main函数中调用它时:

```kotlin
fun main(args: Array<String>) {
    print(2)
    print(2)
}
```

编译后，它将变成下面这样:

```kotlin
public static final void main(@NotNull String[] args) {
    System.out.print(2)
    System.out.print(2)
}
```

这里有一点不一样的是我们不需要跳回到另一个函数中。虽然这种影响可以忽略不计。这就是为什么你定义这样的内联函数时会在IDEA IntelliJ中发出以下警告：



![img](https://pic1.zhimg.com/v2-5b564889f60ba46ef91138358c7e20ec_1440w.jpg)



为什么IntelliJ建议我们在含有lambda表达式作为形参的函数中使用内联呢？因为当我们内联函数体时，我们不需要从参数中创建lambda表达式实例，而是可以将它们内联到函数调用中来。这个是上述`repeat`函数的调用:

```kotlin
repeat(100) { println("A") }
```

将会编译成这样：

```kotlin
for (index in 0 until 1000) {
    println("A")
}
```

正如你所看见的那样，lambda表达式的主体`println("A")`替换了内联函数`repeat`中`action(index)`的调用。让我们看另一外个例子。`filter`函数的用法:

```kotlin
val products2 = products.filter { it.bought }
```

将被替换为：

```kotlin
val destination = ArrayList<T>()
for (element in this) 
    if (predicate(element))
        destination.add(element)
val products2 = destination
```

这是一项非常重要的改进。这是因为JVM天然地不支持lambda表达式。说清楚lambda表达式是如何被编译的是件很复杂的事。但总的来说，有两种结果:

- 匿名类
- 单独的类

我们来看个例子。我们有以下lambda表达式：

```kotlin
val lambda: ()->Unit = {
    // body
}
```

它变成了JVM中的匿名类:

```java
// Java
Function0 lambda = new Function0() {
   public Object invoke() {
      // code
   }
};
```

或者它变成了单独的文件中定义的普通类：

```java
// Java
// Additional class in separate file
public class TestInlineKt$lambda implements Function0 {
   public Object invoke() {
      // code
   }
}
// Usage
Function0 lambda = new TestInlineKt$lambda()
```

第二种效率更高，我们尽可能使用这种。仅仅当我们需要使用局部变量时，第一种才是必要的。

这就是为什么当我们修改局部变量时，`repeat`和`noinlineRepeat`之间存在如此之大的运行速度差异的原因。非内联函数中的Lambda需要编译为匿名类。这是一个巨大的性能开销，从而导致它们的创建和使用都较慢。当我们使用内联函数时，我们根本不需要创建任何其他类。自己检查一下。编译这段代码并把它反编译为Java代码：

```kotlin
fun main(args: Array<String>) {
    var a = 0
    repeat(100_000_000) {
        a += 1
    }
    var b = 0
    noinlineRepeat(100_000_000) {
        b += 1
    }
}
```

你会发现一些相似的东西：

```java
/ Java
public static final void main(@NotNull String[] args) {
   int a = 0;
   int times$iv = 100000000;
   int var3 = 0;

   for(int var4 = times$iv; var3 < var4; ++var3) {
      ++a;
   }

   final IntRef b = new IntRef();
   b.element = 0;
   noinlineRepeat(100000000, (Function1)(new Function1() {
      public Object invoke(Object var1) {
         ++b.element;
         return Unit.INSTANCE;
      }
   }));
}
```

在`filter`函数例子中，使用内联函数改进效果不是那么明显，这是因为lambda表达式在非内联函数中是编译成普通的类而非匿名类。所以它的创建和使用效率还算比较高，但仍有性能开销，所以也就证明了最开始那个`filter`例子为什么只有10%的运行速度差异。



### 为什么内联类可以高性能执行

那么，内联类为什么可以和普通类更好地执行呢?

你可以像这样去实例化一个内联类

```kotlin
val period = Hours(24)
```

...实际上该类并未在编译代码中实例化！事实上，就JVM而言，实际上相当于下面这样的代码......

```java
int period = 24;
```

正如您所看到的，在此编译版本的代码中没有`Hours`概念 - 它只是将基础值分配给int类型的变量！ 同样，当您使用内联类作为函数参数的类型时也是这样的:

```kotlin
fun wait(period: Hours) { /* ... */ }
```

...它可以有效地编译成如下这样......

```kotlin
void wait(int period) { /* ... */ }
```

因此，我们的代码中内联了基础类型和基础值。换句话说，编译后的代码只使用了int整数类型，因此我们避免了在堆内存上创建和访问对象的开销成本。

但是请等一下!

还记得Hours类有一个名为toMinutes（）的函数吗？因为编译后的代码使用的是int而不是Hours对象实例，因此想像一下调用toMinutes（）函数时会发生什么呢？

```kotlin
inline class Hours(val value: Int) {
    fun toMinutes() = Minutes(value * 60)
}
```

`Hours.toMinutes（）`的编译代码如下所示：

```java
public static final int toMinutes(int $this) {
    return $this * 60;
}
```

如果我们在Kotlin中调用`Hours(24).toMinutes()`，它可以有效地编译为`toMinutes(24).`

没问题，确实可以像这样处理函数，但是类成员属性呢？如果我们希望`Hours`除了主要的基础值之外还包括其他一些数据，该怎么办？

一切事情都是有它的权衡的，那么这是其中之一 - 内联类除了基础值之外不能有任何其他成员属性。



### 集合流处理方式与经典处理方式

内联修饰符是一个非常关键的元素，它能使集合流处理的方式与基于循环的经典处理方式一样高效。它经过一次又一次的测试，在代码可读性和性能方面已经优化到极点了，并且相比之下经典处理方式总是有很大的成本。例如，下面的代码：

```kotlin
return data.filter { filterLoad(it) }.map { mapLoad(it) }
```

工作原理与下面代码相同并具有相同的执行时间：

```kotlin
val list = ArrayList<String>()
for (it in data) {
    if (filterLoad(it)) {
        val value = mapLoad(it)
        list.add(value)
    }
}
return list
```

基准测量的具体结果（[源码在这里](https://link.zhihu.com/?target=https%3A//github.com/JetBrains/kotlin-benchmarks/blob/master/src/main/kotlin/org/jetbrains/ClassListBenchmark.kt)）：

```text
Benchmark           (size) Mode  Cnt        Score    Error  Units
filterAndMap           10  avgt  200      561.249 ±      1  ns/op
filterAndMap         1000  avgt  200    29803.183 ±    127  ns/op
filterAndMap       100000  avgt  200  3859008.234 ±  50022  ns/op

filterAndMapManual     10  avgt  200      526.825 ±      1  ns/op
filterAndMapManual   1000  avgt  200    28420.161 ±     94  ns/op
filterAndMapManual 100000  avgt  200  3831213.798 ±  34858  ns/op
```

从程序的角度来看，这两个函数几乎相同。尽管从可读性的角度来看第一种方式要好很多，这就是为什么我们应该总是宁愿使用智能的集合流处理函数而不是自己去实现整个处理过程。此外如果stalib库中集合处理函数不能满足我们的需求时，请不要犹豫，自己动手编写集合处理函数。例如，当我需要转置集合中的集合时，这是我在上一个项目中添加的函数：

```kotlin
fun <E> List<List<E>>.transpose(): List<List<E>> {
    if (isEmpty()) return this

    val width = first().size
    if (any { it.size != width }) {
        throw IllegalArgumentException("All nested lists must have the same size, but sizes were ${map { it.size }}")
    }

    return (0 until width).map { col ->
        (0 until size).map { row -> this[row][col] }
    }
}
```

记得写一些单元测试：

```kotlin
class TransposeTest {

    private val list = listOf(listOf(1, 2, 3), listOf(4, 5, 6))

    @Test
    fun `Transposition of transposition is identity`() {
        Assert.assertEquals(list, list.transpose().transpose())
    }

    @Test
    fun `Simple transposition test`() {
        val transposed = listOf(listOf(1, 4), listOf(2, 5), listOf(3, 6))
        assertEquals(transposed, list.transpose())
    }
}
```

### 内联修饰符的成本

内联不应该被过度使用，因为它也是有成本的。我想在代码中打印出更多的数字2, 所以我就定义了下面这个函数:

```kotlin
inline fun twoPrintTwo() {
    print(2)
    print(2)
}
```

这对我来说可能还不够，所以我添加了这个函数:

```kotlin
inline fun twoTwoPrintTwo() {
    twoPrintTwo()
    twoPrintTwo()
}
```

还是不满意。我又定义了以下这两个函数:

```kotlin
inline fun twoTwoTwoPrintTwo() {
    twoTwoPrintTwo()
    twoTwoPrintTwo()
}

fun twoTwoTwoTwoPrintTwo() {
    twoTwoTwoPrintTwo()
    twoTwoTwoPrintTwo()
}
```

然后我决定检查编译后的代码中发生了什么，所以我将编译为JVM字节码然后将它反编译成Java代码。`twoTwoPrintTwo`函数已经很长了：

```java
public static final void twoTwoPrintTwo() {
   byte var1 = 2;
   System.out.print(var1);
   var1 = 2;
   System.out.print(var1);
   var1 = 2;
   System.out.print(var1);
   var1 = 2;
   System.out.print(var1);
}
```

但是`twoTwoTwoTwoPrintTwo`就更加恐怖了

```java
public static final void twoTwoTwoTwoPrintTwo() {
   byte var1 = 2;
   System.out.print(var1);
   var1 = 2;
   System.out.print(var1);
   var1 = 2;
   System.out.print(var1);
   var1 = 2;
   System.out.print(var1);
   var1 = 2;
   System.out.print(var1);
   var1 = 2;
   System.out.print(var1);
   var1 = 2;
   System.out.print(var1);
   var1 = 2;
   System.out.print(var1);
   var1 = 2;
   System.out.print(var1);
   var1 = 2;
   System.out.print(var1);
   var1 = 2;
   System.out.print(var1);
   var1 = 2;
   System.out.print(var1);
   var1 = 2;
   System.out.print(var1);
   var1 = 2;
   System.out.print(var1);
   var1 = 2;
   System.out.print(var1);
   var1 = 2;
   System.out.print(var1);
}
```

这说明了内联函数的主要问题: 当我们过度使用它们时，会使得代码体积不断增大。这实际上就是为什么当我们使用他们时IntelliJ会给出警告提示。



### 权衡和使用限制

首先，内联类必须包含一个基础值，这就意味它需要一个主构造器来接收 这个基础值，此外它必须是只读的(val)。你可以定义你想要的基础值变量名。

```kotlin
inline class Seconds()              // nope - needs to accept a value!
inline class Minutes(value: Int)    // nope - value needs to be a property
inline class Hours(var value: Int)  // nope - property needs to be read-only
inline class Days(val value: Int)   // yes!
inline class Months(val count: Int) // yes! - name it what you want
```

如果有需要，可以将该属性设为私有的，但构造函数必须是公有的。

```kotlin
inline class Years private constructor(val value: Int) // nope - constructor must be public
inline class Decades(private val value: Int)           // yes!
```

内联类中不能包含init block初始化块。我会在下一篇发表的文章中探讨内联类如何与Java进行互操作，这点将会彻底说明白。

```kotlin
inline class Centuries(val value: Int) {
    // nope - "Inline class cannot have an initializer block"
    init { 
        require(value >= 0)
    }
}
```

正如我们在上面发现的那样，除了一个基础值之外，我们的内联类主构造器不能包含其他任何成员属性。

```kotlin
// nope - "Inline class must have exactly one primary constructor parameter"
inline class Years(val count: Int, val startYear: Int)
```

但是呢，它的内部是可以拥有成员属性的,只要它们仅基于构造器中那个基础值计算，或者从可以静态解析的某个值或对象计算 - 来自单例，顶级对象，常量等。

```kotlin
object Conversions {
    const val MINUTES_PER_HOUR = 60    
}

inline class Hours(val value: Int) {
    val valueAsMinutes get() = value * Conversions.MINUTES_PER_HOUR
}
```

不允许类继承 - 内联类不能继承另一个类，并且它们不能被另一个类继承。 

```kotlin
open class TimeUnit
inline class Seconds(val value: Int) : TimeUnit() // nope - cannot extend classes

open inline class Minutes(val value: Int) // nope - "Inline classes can only be final"
```

如果您需要将内联类作为子类型，那很好 - 您可以实现接口而不是继承基类。

```kotlin
interface TimeUnit {
    val value: Int
}

inline class Hours(override val value: Int) : TimeUnit  // yes!
```

内联类必须在顶级声明。嵌套/内部类不能内联的。

```kotlin
class Outer {
     // nope - "Inline classes are only allowed on top level"
    inline class Inner(val value: Int)
}

inline class TopLevelInline(val value: Int) // yes!
```

目前，也不支持内联枚举类。

```kotlin
// nope - "Modifier 'inline' is not applicable to 'enum class'"
inline enum class TimeUnits(val value: Int) {
    SECONDS_PER_MINUTE(60),
    MINUTES_PER_HOUR(60),
    HOURS_PER_DAY(24)
}
```



### Type Aliases(类型别名) 与 Inline Classes(内联类)对比

因为它们都包含基础类型，所以内联类很容易与类型别名混淆。但是有一些关键的差异使它们在不同的场景下得以应用。

类型别名为基础类型提供备用名称。例如，您可以为`String`这样的常见类型添加别名，并为其指定在特定上下文中有意义的描述性名称，比如`Username`。`Username`类型的变量实际上是源代码和编译代码中String类型的变量同一个东西，只是不同名称而已。例如，您可以这样做：

```kotlin
typealias Username = String

fun validate(name: Username) {
    if(name.length < 5) {
        println("Username $name is too short.")
    }
}
```

注意到我们是可以在`name`上直接调用`.length`的，这是因为`name`实际上就是个`String`,尽管我们在声明参数类型的时候使用的是别名`Username`.

在另一面，内联类实际上是基础类型的包装器，因此当你需要使用基础值的时候，需要做拆箱操作。例如我们使用内联类来重写上面别名的例子:

```kotlin
inline class Username(val value: String)

fun validate(name: Username) {
    if (name.value.length < 5) {
        println("Username ${name.value} is too short.")
    }
}
```

注意到我们是必须这样`name.value.length`而不是`name.length`,我们必须解开这个包装器取出里面的值。

但是最大的区别在于与分配兼容性有关。内联类为你提供类型的安全性，类型别名则没有。 类型别名与其基础类型相同。例如，看如下代码：

```kotlin
typealias Username = String
typealias Password = String

fun authenticate(user: Username, pass: Password) { /* ... */ }

fun main(args: Array<String>) {
    val username: Username = "joe.user"
    val password: Password = "super-secret"
    authenticate(password, username)
}
```

在这种情况下，`Username`和`Password`仅仅是`String`另一个不同名称而已，甚至你可以将`Username`和`Password`任意调换位置。实际上，这正是我们在上面的代码中所做的 - 当我们调用authenticate（）函数时，即使我们将`Username`和`Password`位置弄反了，但编译器依然认为是合法的。

另一方面，如果你对上面同一个案例使用内联类，那么编译器将会很幸运告诉你这是不合法的：

```kotlin
inline class Username(val value: String)
inline class Password(val value: String)

fun authenticate(user: Username, pass: Password) { /* ... */ }

fun main(args: Array<String>) {
    val username: Username = Username("joe.user")
    val password: Password = Password("super-secret")
    authenticate(password, username) // <--- Compiler error here! =)
}
```



### 内联修饰符在不同方面的用法

内联修饰符因为它特殊的语法特性而发生的变化远远超过我们在本篇文章中看到的内容。它可以实化泛型类型。但是它也有一些局限性。虽然这与Effective Kotlin系列无关并且属于是另外一个话题。如果你想要我阐述更多有关它，请在Twitter或评论中表达你的想法。

### 使用场景

使用内联修饰符时最常见的场景就是把函数作为另一个函数的参数时(高阶函数)。集合或字符串处理(如`filter`,`map`或者`joinToString`)或者一些独立的函数(如`repeat`)就是很好的例子。

这就是为什么`inline`修饰符经常被库开发人员用来做一些重要优化的原因了。他们应该知道它是如何工作的，哪里还需要被改进以及使用成本是什么。当我们使用函数类型作为参数来定义自己的工具类函数时，我们也需要在项目中使用`inline`修饰符。当我们没有函数类型作为参数，没有reified实化类型参数并且也不需要非本地返回时，那么我们很可能不应该使用`inline`修饰符了。这就是为什么我们在非上述情况下使用`inline`修饰符会在Android Studio或IDEA IntelliJ得到一个警告原因。



## 拓展函数



Kotlin 能够对一个类或接口扩展新功能而无需继承该类或者使用像*装饰者*这样的设计模式。 这通过叫做*扩展*的特殊声明完成。

例如，你可以为一个你不能修改的、来自第三方库中的类或接口编写一个新的函数。 这个新增的函数就像那个原始类本来就有的函数一样，可以用寻常方式调用。 这种机制称为*扩展函数*。此外，也有*扩展属性*， 允许你为一个已经存在的类添加新的属性。

### 扩展函数

声明一个扩展函数需用一个*接收者类型*也就是被扩展的类型来作为他的前缀。 下面代码为 `MutableList<Int>` 添加一个`swap` 函数：

```kotlin
fun MutableList<Int>.swap(index1: Int, index2: Int) {
    val tmp = this[index1] // “this”对应该列表
    this[index1] = this[index2]
    this[index2] = tmp
}
```

这个 `this` 关键字在扩展函数内部对应到接收者对象（传过来的在点符号前的对象） 现在，可以对任意 `MutableList<Int>` 调用该函数了：

```kotlin
val list = mutableListOf(1, 2, 3)
list.swap(0, 2) // “swap()”内部的“this”会保存“list”的值
```

这个函数对任何 `MutableList<T>` 起作用，可以泛化它：

```kotlin
fun <T> MutableList<T>.swap(index1: Int, index2: Int) {
    val tmp = this[index1] // “this”对应该列表
    this[index1] = this[index2]
    this[index2] = tmp
}
```

为了在接收者类型表达式中使用泛型，需要在函数名前声明泛型参数。 For more information about generics, 参见[泛型函数](https://book.kotlincn.net/text/generics.html)。

### 扩展是*静态*解析的

扩展不能真正的修改他们所扩展的类。通过定义一个扩展，并没有在一个类中插入新成员， 只不过是可以通过该类型的变量用点表达式去调用这个新函数。

Extension functions are dispatched *statically*. So which extension function is called is already known at compile time based on the receiver type. For example:

```kotlin
fun main() {
//sampleStart
    open class Shape
    class Rectangle: Shape()

    fun Shape.getName() = "Shape"
    fun Rectangle.getName() = "Rectangle"

    fun printClassName(s: Shape) {
        println(s.getName())
    }

    printClassName(Rectangle())
//sampleEnd
}
```

这个例子会输出 *Shape*，因为调用的扩展函数只取决于参数 `s` 的声明类型，该类型是 `Shape` 类。

如果一个类定义有一个成员函数与一个扩展函数，而这两个函数又有相同的接收者类型、 相同的名字，并且都适用给定的参数，这种情况*总是取成员函数*。 例如：

```kotlin
fun main() {
//sampleStart
    class Example {
        fun printFunctionType() { println("Class method") }
    }

    fun Example.printFunctionType() { println("Extension function") }

    Example().printFunctionType()
//sampleEnd
}
```

这段代码输出 *Class method*。

当然，扩展函数重载同样名字但不同签名成员函数也完全可以：

```kotlin
fun main() {
//sampleStart
    class Example {
        fun printFunctionType() { println("Class method") }
    }

    fun Example.printFunctionType(i: Int) { println("Extension function #$i") }

    Example().printFunctionType(1)
//sampleEnd
}
```

### 可空接收者

注意可以为可空的接收者类型定义扩展。这样的扩展可以在对象变量上调用， 即使其值为 null。 If the receiver is `null`, then `this` is also `null`. So when defining an extension with a nullable receiver type, we recommend performing a `this == null` check inside the function body to avoid compiler errors.

可以在没有检测 null 的时候调用 Kotlin 中的 `toString()`：检测已发生在扩展函数的内部：

```kotlin
fun Any?.toString(): String {
    if (this == null) return "null"
    // 空检测之后，“this”会自动转换为非空类型，所以下面的 toString()
    // 解析为 Any 类的成员函数
    return toString()
}
```

### 扩展属性

与扩展函数类似，Kotlin 支持扩展属性：

```kotlin
val <T> List<T>.lastIndex: Int
    get() = size - 1
```

> 由于扩展没有实际的将成员插入类中，因此对扩展属性来说[幕后字段](https://book.kotlincn.net/text/properties.html#幕后字段)是无效的。这就是为什么*扩展属性不能有初始化器*。他们的行为只能由显式提供的 getter/setter 定义。
>
> <svg width="24" height="24" fill="#4dbb5f" viewBox="0 0 24 24"><path d="M21 12a9 9 0 1 1-9-9 9 9 0 0 1 9 9zM10.5 7.5A1.5 1.5 0 1 0 12 6a1.5 1.5 0 0 0-1.5 1.5zm-.5 3.54v1h1V18h2v-6a.96.96 0 0 0-.96-.96z"></path></svg>

例如:

```kotlin
val House.number = 1 // 错误：扩展属性不能有初始化器
```

### 伴生对象的扩展

如果一个类定义有一个[伴生对象](https://book.kotlincn.net/text/object-declarations.html#伴生对象) ，你也可以为伴生对象定义扩展函数与属性。就像伴生对象的常规成员一样， 可以只使用类名作为限定符来调用伴生对象的扩展成员：

```kotlin
class MyClass {
    companion object { }  // 将被称为 "Companion"
}

fun MyClass.Companion.printCompanion() { println("companion") }

fun main() {
    MyClass.printCompanion()
}
```

### 扩展的作用域

大多数情况都在顶层定义扩展——直接在包里：

```kotlin
package org.example.declarations

fun List<String>.getLongestString() { /*……*/}
```

如需使用所定义包之外的一个扩展，只需在调用方导入它：

```kotlin
package org.example.usage

import org.example.declarations.getLongestString

fun main() {
    val list = listOf("red", "green", "blue")
    list.getLongestString()
}
```

更多信息参见[导入](https://book.kotlincn.net/text/packages.html#导入)

### 扩展声明为成员

可以在一个类内部为另一个类声明扩展。在这样的扩展内部，有多个*隐式接收者*—— 其中的对象成员可以无需通过限定符访问。扩展声明所在的类的实例称为*分发接收者*，扩展方法调用所在的接收者类型的实例称为*扩展接收者*。

```kotlin
class Host(val hostname: String) {
    fun printHostname() { print(hostname) }
}

class Connection(val host: Host, val port: Int) {
    fun printPort() { print(port) }

    fun Host.printConnectionString() {
         printHostname()   // 调用 Host.printHostname()
        print(":")
         printPort()   // 调用 Connection.printPort()
    }

    fun connect() {
         /*……*/
         host.printConnectionString()   // 调用扩展函数
    }
}

fun main() {
    Connection(Host("kotl.in"), 443).connect()
    //Host("kotl.in").printConnectionString()  // 错误，该扩展函数在 Connection 外不可用
}
```

对于分发接收者与扩展接收者的成员名字冲突的情况，扩展接收者优先。要引用分发接收者的成员你可以使用 [限定的 `this` 语法](https://book.kotlincn.net/text/this-expressions.html#限定的-this)。

```kotlin
class Connection {
    fun Host.getConnectionString() {
        toString()         // 调用 Host.toString()
        this@Connection.toString()  // 调用 Connection.toString()
    }
}
```

声明为成员的扩展可以声明为 `open` 并在子类中覆盖。这意味着这些函数的分发对于分发接收者类型是虚拟的，但对于扩展接收者类型是静态的。

```kotlin
open class Base { }

class Derived : Base() { }

open class BaseCaller {
    open fun Base.printFunctionInfo() {
        println("Base extension function in BaseCaller")
    }

    open fun Derived.printFunctionInfo() {
        println("Derived extension function in BaseCaller")
    }

    fun call(b: Base) {
        b.printFunctionInfo()   // 调用扩展函数
    }
}

class DerivedCaller: BaseCaller() {
    override fun Base.printFunctionInfo() {
        println("Base extension function in DerivedCaller")
    }

    override fun Derived.printFunctionInfo() {
        println("Derived extension function in DerivedCaller")
    }
}

fun main() {
    BaseCaller().call(Base())   // “Base extension function in BaseCaller”
    DerivedCaller().call(Base())  // “Base extension function in DerivedCaller”——分发接收者虚拟解析
    DerivedCaller().call(Derived())  // “Base extension function in DerivedCaller”——扩展接收者静态解析
}
```

### 关于可见性的说明

Extensions utilize the same [visibility modifiers](https://book.kotlincn.net/text/visibility-modifiers.html) as regular functions declared in the same scope would. For example:

- 在文件顶层声明的扩展可以访问同一文件中的其他 `private` 顶层声明。
- 如果扩展是在其接收者类型外部声明的，那么它不能访问接收者的 `private` 或 `protected` 成员。



### 原理部分

顶层函数->  在kotlin底层编译后，会生成文件名+kt后缀结尾的类来作为容器包裹顶层函数，里面定义了static修饰的方法。

拓展函数，会生成 相同名字的方法，但是拓展的接受类作为第一个参数，在方法中，也是静态方法。





## 作用域函数

下面我将关于 **run、with、T.run、T.let、T.also 和 T.apply** 这些函数，并把它们称为作用域函数，因为我注意到它们的主要功能是为调用者函数提供内部作用域。

说明作用域最简单的方式是 *run* 函数

```kotlin
fun test() {
    var mood = "I am sad"

    run {
        val mood = "I am happy"
        println(mood) // I am happy
    }
    println(mood)  // I am sad
}
```

在这个例子中，在test函数的内部有一个分隔开的作用域，在这个作用域内部完全包含一个 在输出之前的**mood** 变量被重新定义并初始化为 **I am happy**的操作实现。

这个作用域函数本身似乎看起来不是很有用。但是这还有一个比作用域有趣一点是，它返回一些东西，是这个作用域内部的最后一个对象。

因此，以下的内容会变得更加整洁，我们可以将show()方法应用到两个View中，而不需要去调用两次show()方法。

```kotlin
run {
        if (firstTimeView) introView else normalView
}.show()
```



### 作用域函数的三个属性特征

为了让作用域函数更有趣，让我把他们的行为分类成三个属性特征。我将会使用这些属性特征来区分他们每一个函数。

#### 1、普通函数 VS 扩展函数 (Normal vs. extension function)

如果我们对比 **with** 和 **T.run** 这两个函数的话，他们实际上是十分相似的。下面使用他们实现相同的功能的例子.

```kotlin
with(webview.settings) {
    javaScriptEnabled = true
    databaseEnabled = true
}
// similarly
webview.settings.run {
    javaScriptEnabled = true
    databaseEnabled = true
}
```

然后他们之间唯一不同在于 **with** 是一个普通函数，而 **T.run**是一个扩展函数。

那么问题来了，它们各自使用的优点是什么？

想象一下如果 **webview.settings** 可能为空，那么下面两种方式实现如何去修改呢？

```kotlin
// Yack!(比较丑陋的实现方式)
with(webview.settings) {
      this?.javaScriptEnabled = true
      this?.databaseEnabled = true
   }
}
// Nice.(比较好的实现方式)
webview.settings?.run {
    javaScriptEnabled = true
    databaseEnabled = true
}
```

在这种情况下，显然 **T.run** 扩展函数更好，因为我们可以在使用它之前对可空性进行检查。



#### 2、this VS it 参数(This vs. it argument)

如果我们对比 **T.run** 和 **T.let** 两个函数也是非常的相似，唯一的区别在于它们接收的参数不一样。下面显示了两种功能的相同逻辑。

```kotlin
stringVariable?.run {
      println("The length of this String is $length")
}
// Similarly.
stringVariable?.let {
      println("The length of this String is ${it.length}")
}
```

如果你查看过 **T.run** 函数声明，你就会注意到T.run仅仅只是被当做了 **block: T.()** 扩展函数的调用块。因此，在其作用域内，**T** 可以被 **this** 指代。在编码过程中，在大多数情况下this是可以被省略的。因此我们上面的示例中，我们可以在println语句中直接使用 **\$length** 而不是 **\${this.lenght}**. 所以我把这个称之为传递 **this参数**

然而对于 **T.let** 函数的声明，你将会注意到 **T.let** 是传递它自己本身到函数中**block: (T)**。因此这个类似于传递一个lambda表达式作为参数。它可以在函数作用域内部使用**it**来指代. 所以我把这个称之为传递 **it参数**

从上面看，似乎**T.run**比**T.let**更加优越，因为它更隐含，但是T.let函数具有一些微妙的优势，如下所示:

- 1、**T.let**函数提供了一种更清晰的区分方式去使用给定的变量函数/成员与外部类函数/成员。

- 2、例如当**it**作为函数的参数传递时，**this**不能被省略，并且**it**写起来比**this**更简洁，更清晰。

- 3、**T.let**允许更好地命名已转换的已使用变量，即可以将it转换为其他有含义名称，而 **T.run**则不能，内部只能用**this**指代或者省略。

```kotlin
stringVariable?.let {
      nonNullString ->
      println("The non null string is $nonNullString")
}
```





#### 3、返回this VS 其他类型 (Return this vs. other type)

现在，让我们看看**T.let**和**T.also**，如果我们看看它的内部函数作用域，它们都是相同的。

```kotlin
stringVariable?.let {
      println("The length of this String is ${it.length}")
}
// Exactly the same as below
stringVariable?.also {
      println("The length of this String is ${it.length}")
}
```

然而，他们微妙的不同在于他们的返回值**T.let**返回一个不同类型的值，而**T.also**返回T类型本身，即这个。

这两个函数对于函数的链式调用都很有用，其中**T.let**让您演变操作，而**T.also**则让您对相同的变量执行操作。

简单的例子如下:

```kotlin
val original = "abc"
// Evolve the value and send to the next chain
original.let {
    println("The original String is $it") // "abc"
    it.reversed() // evolve it as parameter to send to next let
}.let {
    println("The reverse String is $it") // "cba"
    it.length  // can be evolve to other type
}.let {
    println("The length of the String is $it") // 3
}
// Wrong
// Same value is sent in the chain (printed answer is wrong)
original.also {
    println("The original String is $it") // "abc"
    it.reversed() // even if we evolve it, it is useless
}.also {
    println("The reverse String is ${it}") // "abc"
    it.length  // even if we evolve it, it is useless
}.also {
    println("The length of the String is ${it}") // "abc"
}
// Corrected for also (i.e. manipulate as original string
// Same value is sent in the chain 
original.also {
    println("The original String is $it") // "abc"
}.also {
    println("The reverse String is ${it.reversed()}") // "cba"
}.also {
    println("The length of the String is ${it.length}") // 3
}
```

**T.also**似乎看上去没有意义，因为我们可以很容易地将它们组合成一个功能块。仔细思考，它有一些很好的优点。

- 1、它可以对相同的对象提供非常清晰的分离过程，即创建更小的函数部分。

- 2、在使用之前，它可以非常强大的进行自我操作，从而实现整个链式代码的构建操作。

当两者结合在一起使用时，即一个自身演变，一个自我保留，它能使一些操作变得更加强大。

```kotlin
// Normal approach
fun makeDir(path: String): File  {
    val result = File(path)
    result.mkdirs()
    return result
}
// Improved approach
fun makeDir(path: String) = path.let{ File(it) }.also{ it.mkdirs() }
```



#### 回顾所有属性特征

通过回顾这3个属性特征，我们可以非常清楚函数的行为。让我来说明**T.apply**函数，由于我并没有以上函数中提到过它。 **T.apply**的三个属性如下

- 1、它是一个扩展函数

- 2、它是传递**this**作为参数

- 3、它是返回 **this** (即它自己本身)

因此，使用它，可以想象下它可以被用作:

```kotlin
// Normal approach
fun createInstance(args: Bundle) : MyFragment {
    val fragment = MyFragment()
    fragment.arguments = args
    return fragment
}
// Improved approach
fun createInstance(args: Bundle) 
              = MyFragment().apply { arguments = args }
```

或者我们也可以让无链对象创建链式调用。

```kotlin
// Normal approach
fun createIntent(intentData: String, intentAction: String): Intent {
    val intent = Intent()
    intent.action = intentAction
    intent.data=Uri.parse(intentData)
    return intent
}
// Improved approach, chaining
fun createIntent(intentData: String, intentAction: String) =
        Intent().apply { action = intentAction }
                .apply { data = Uri.parse(intentData) }
```





### 函数的选用

因此，显然有了这3个属性特征，我们现在可以对功能进行相应的分类。基于此，我们可以在下面构建一个决策树，以帮助确定我们想要使用哪个函数，来选择我们需要的。

![标准函数选取](.\res\标准函数选取.jpg)

​	





## Sequences(序列)

**序列(Sequences)** 共享同一个**迭代器(iterator)** ---序列允许 **map操作** 转换一个元素后，然后立马可以将这个元素传递给 **filter操作** ，而不是像**集合(lists)** 一样等待所有的元素都循环完成了**map操作**后，用一个新的集合存储起来，然后又遍历循环从新的集合取出元素完成**filter操作**。

使用first{...}或者last{...}操作符时，当使用接收一个预判断的条件 **[first or last](https://link.zhihu.com/?target=https%3A//gist.github.com/tomaszpolanski/6e5b79af2f1c3d1146bd042f3e81f2e7%23file-sequencevslist-kt-L99)** 方法时候，使用**序列(Sequences)**会产生一个小的性能提升，如果将它与其他操作符结合使用，它的性能将会得到更大的提升。

每次迭代**Sequences(序列)** 时，都会计算元素。**Lists(集合)** 中的元素只计算一次，然后存储在内存中。

将**Sequences(序列)** 作为参数传递给函数: 函数可能会多次遍历它们。在传递或者在整个使用List之前建议将**Sequences(序列)** 转换 **Lists(集合)**

想要传递一个**Sequences(序列)**，你可以使用**constrainOnce（）** - 它只允许一次遍历**Sequences(序列)**，第二次尝试遍历会抛出一个异常。 但不建议这种方法会使代码难以维护。

### 描述

序列操作又被称之为**惰性集合操作**，Sequences序列接口强大在于其操作的实现方式。序列中的元素求值都是惰性的，所以可以更加高效使用序列来对数据集中的元素进行链式操作(映射、过滤、变换等),而不需要像普通集合那样，每进行一次数据操作，都必须要开辟新的内存来存储中间结果，而实际上绝大多数的数据集合操作的需求关注点在于最后的结果而不是中间的过程，

序列是在Kotlin中操作数据集的另一种选择，它和Java8中新增的Stream很像，在Java8中我们可以把一个数据集合转换成Stream，然后再对Stream进行数据操作(映射、过滤、变换等)，序列(Sequences)可以说是用于优化集合在一些特殊场景下的工具。但是它不是用来替代集合，准确来说它起到是一个互补的作用。

序列操作分为两大类:



 ![sequences操作示意图](.\res\sequences操作示意图.png)



- 1、中间操作

序列的中间操作始终都是惰性的，一次中间操作返回的都是一个序列(Sequences)，产生的新序列内部知道如何变换原始序列中的元素。怎样说明序列的中间操作是惰性的呢？一起来看个例子:

```
 代码解读复制代码fun main(args: Array<String>) {
    (0..6)
        .asSequence()
        .map {//map返回是Sequence<T>，故它属于中间操作
            println("map: $it")
            return@map it + 1
        }
        .filter {//filter返回是Sequence<T>，故它属于中间操作
            println("filter: $it")
            return@filter it % 2 == 0
        }
}
```

运行结果:

 ![sequences示例执行结果1](.\res\sequences示例执行结果1.png)

以上例子只有中间操作没有末端操作，**通过运行结果发现map、filter中并没有输出任何提示，这也就意味着map和filter的操作被延迟了，它们只有在获取结果的时候(也即是末端操作被调用的时候)才会输出提示**。

- 2、末端操作

序列的末端操作会执行原来中间操作的所有延迟计算，欢聚，一次末端操作返回的是一个结果，返回的结果可以是集合、数字、或者从其他对象集合变换得到任意对象。上述例子加上末端操作:

```kotlin
 fun main(args: Array<String>) {
    (0..6)
        .asSequence()
        .map {//map返回是Sequence<T>，故它属于中间操作
            println("map: $it")
            return@map it + 1
        }
        .filter {//filter返回是Sequence<T>，故它属于中间操作
            println("filter: $it")
            return@filter it % 2 == 0
        }
        .count {//count返回是Int，返回的是一个结果，故它属于末端操作
            it < 6
        }
        .run {
            println("result is $this");
        }
}
```

运行结果

 ![sequences示例执行结果1](.\res\sequences示例执行结果2.png)

**注意:判别是否是中间操作还是末端操作很简单，只需要看操作符API函数返回值的类型,如果返回的是一个Sequence<T>那么这就是一个中间操作，如果返回的是一个具体的结果类型，比如Int,Boolean,或者其他任意对象,那么它就是一个末端操作**



### 怎么创建序列(Sequences)?

创建序列(Sequences)的方法主要有:

- 1、使用Iterable的扩展函数asSequence来创建。

```kotlin
//定义声明
public fun <T> Iterable<T>.asSequence(): Sequence<T> {
    return Sequence { this.iterator() }
}
//调用实现
list.asSequence()
```

- 2、使用generateSequence函数生成一个序列。

```kotlin
//定义声明
@kotlin.internal.LowPriorityInOverloadResolution
public fun <T : Any> generateSequence(seed: T?, nextFunction: (T) -> T?): Sequence<T> =
    if (seed == null)
        EmptySequence
    else
        GeneratorSequence({ seed }, nextFunction)

//调用实现，seed是序列的起始值，nextFunction迭代函数操作
val naturalNumbers = generateSequence(0) { it + 1 } //使用迭代器生成一个自然数序列
```

- 3、使用序列(Sequence<T>)的扩展函数constrainOnce生成一次性使用的序列。

```kotlin
//定义声明
public fun <T> Sequence<T>.constrainOnce(): Sequence<T> {
    // as? does not work in js
    //return this as? ConstrainedOnceSequence<T> ?: ConstrainedOnceSequence(this)
    return if (this is ConstrainedOnceSequence<T>) this else ConstrainedOnceSequence(this)
}
//调用实现
val naturalNumbers = generateSequence(0) { it + 1 }
val naturalNumbersOnce = naturalNumbers.constrainOnce()
```

**注意:只能迭代一次，如果超出一次则会抛出IllegalStateException("This sequence can be consumed only once.")异常。**



### 什么时候使用**序列(Sequences)** 

第一、数据集量级是足够大，建议使用**序列(Sequences)**。

第二、对数据集进行频繁的数据操作，类似于多个操作符链式操作，建议使用**序列(Sequences)**

第三、对于使用first{},last{}建议使用**序列(Sequences)**。

第四、当仅仅只有map操作时，**序列(Sequences)**（仅仅只有filter的时候推荐List且数据量不太大）。



### Sequences序列实现原理

```kotlin
fun main(args: Array<String>){
    (0..10)
        .asSequence()
        .map { it + 1 }
        .filter { it % 2 == 0 }
        .count { it < 6 }
        .run {
            println("by using sequence result is $this")
        }
}
```

#### 基本原理描述

**序列操作:** 基本原理是惰性求值，也就是说在进行中间操作的时候，是不会产生中间数据结果的，只有等到进行末端操作的时候才会进行求值。也就是上述例子中0~10中的每个数据元素都是先执行map操作，接着马上执行filter操作。然后下一个元素也是先执行map操作，接着马上执行filter操作。然而普通集合是所有元素都完执行map后的数据存起来，然后从存储数据集中又所有的元素执行filter操作存起来的原理。

**集合普通操作:** 针对每一次操作都会产生新的中间结果，也就是上述例子中的map操作完后会把原始数据集循环遍历一次得到最新的数据集存放在新的集合中，然后进行filter操作,遍历上一次map新集合中数据元素,最后得到最新的数据集又存在一个新的集合中。



#### 图解原理

```kotlin
//使用序列
fun main(args: Array<String>){
    (0..100)
        .asSequence()
        .map { it + 1 }
        .filter { it % 2 == 0 }
        .find { it > 3 }
}
//使用普通集合
fun main(args: Array<String>){
    (0..100)
        .map { it + 1 }
        .filter { it % 2 == 0 }
        .find { it > 3 }
}
```

![sequences原理示意图](.\res\sequences原理示意图.png)

通过以上的原理转化图，会发现使用序列会逐个元素进行操作，在进行末端操作find获得结果之前提早去除一些不必要的操作,以及find找到一个符合条件元素后，后续众多元素操作都可以省去，从而达到优化的目的。而集合普通操作，无论是哪个元素都得默认经过所有的操作。其实有些操作在获得结果之前是没有必要执行的以及可以在获得结果之前，就能感知该操作是否符合条件，如果不符合条件提前摒弃，避免不必要操作带来性能的损失。



#### 序列(Sequences)原理源码完全解析

##### 使用集合普通操作反编译源码

```kotlin
public static final void main(@NotNull String[] args) {
      Intrinsics.checkParameterIsNotNull(args, "args");
      byte var1 = 0;
      Iterable $receiver$iv = (Iterable)(new IntRange(var1, 100));
      //创建新的集合存储map后中间结果
      Collection destination$iv$iv = (Collection)(new ArrayList(CollectionsKt.collectionSizeOrDefault($receiver$iv, 10)));
      Iterator var4 = $receiver$iv.iterator();

      int it;
      //对应map操作符生成一个while循环
      while(var4.hasNext()) {
         it = ((IntIterator)var4).nextInt();
         Integer var11 = it + 1;
         //将map变换的元素加入到新集合中
         destination$iv$iv.add(var11);
      }

      $receiver$iv = (Iterable)((List)destination$iv$iv);
      //创建新的集合存储filter后中间结果
      destination$iv$iv = (Collection)(new ArrayList());
      var4 = $receiver$iv.iterator();//拿到map后新集合中的迭代器
      //对应filter操作符生成一个while循环
      while(var4.hasNext()) {
         Object element$iv$iv = var4.next();
         int it = ((Number)element$iv$iv).intValue();
         if (it % 2 == 0) {
          //将filter过滤的元素加入到新集合中
            destination$iv$iv.add(element$iv$iv);
         }
      }

      $receiver$iv = (Iterable)((List)destination$iv$iv);
      Iterator var13 = $receiver$iv.iterator();//拿到filter后新集合中的迭代器
      
      //对应find操作符生成一个while循环,最后末端操作只需要遍历filter后新集合中的迭代器，取出符合条件数据即可。
      while(var13.hasNext()) {
         Object var14 = var13.next();
         it = ((Number)var14).intValue();
         if (it > 3) {
            break;
         }
      }
   }
```

##### 使用序列(Sequences)惰性操作反编译源码

整个序列操作源码

```kotlin
public static final void main(@NotNull String[] args) {
      Intrinsics.checkParameterIsNotNull(args, "args");
      byte var1 = 0;
      //利用Sequence扩展函数实现了fitler和map中间操作，最后返回一个Sequence对象。
      Sequence var7 = SequencesKt.filter(SequencesKt.map(CollectionsKt.asSequence((Iterable)(new IntRange(var1, 100))), (Function1)null.INSTANCE), (Function1)null.INSTANCE);
      //取出经过中间操作产生的序列中的迭代器，可以发现进行map、filter中间操作共享了同一个迭代器中数据，每次操作都会产生新的迭代器对象，但是数据是和原来传入迭代器中数据共享，最后进行末端操作的时候只需要遍历这个迭代器中符合条件元素即可。
      Iterator var3 = var7.iterator();
      //对应find操作符生成一个while循环,最后末端操作只需要遍历filter后新集合中的迭代器，取出符合条件数据即可。
      while(var3.hasNext()) {
         Object var4 = var3.next();
         int it = ((Number)var4).intValue();
         if (it > 3) {
            break;
         }
      }

   }
```

抽出其中这段关键code，继续深入:

```java
SequencesKt.filter(SequencesKt.map(CollectionsKt.asSequence((Iterable)(new IntRange(var1, 100))), (Function1)null.INSTANCE), (Function1)null.INSTANCE);
```

把这段代码转化分解成三个部分:

```java
//第一部分
val collectionSequence = CollectionsKt.asSequence((Iterable)(new IntRange(var1, 100)))
//第二部分
val mapSequence = SequencesKt.map(collectionSequence, (Function1)null.INSTANCE)
//第三部分
val filterSequence = SequencesKt.filter(mapSequence, (Function1)null.INSTANCE)
```

解释第一部分代码:

第一部分反编译的源码很简单，主要是调用Iterable<T>中扩展函数将原始数据集转换成Sequence<T>对象。

```kotlin
public fun <T> Iterable<T>.asSequence(): Sequence<T> {
    return Sequence { this.iterator() }//传入外部Iterable<T>中的迭代器对象
}
```

更深入一层:

```kotlin
@kotlin.internal.InlineOnly
public inline fun <T> Sequence(crossinline iterator: () -> Iterator<T>): Sequence<T> = object : Sequence<T> {
    override fun iterator(): Iterator<T> = iterator()
}
```

通过外部传入的集合中的迭代器方法返回迭代器对象，通过一个对象表达式实例化一个Sequence<T>，Sequence<T>是一个接口，内部有个iterator()抽象函数返回一个迭代器对象，然后把传入迭代器对象作为Sequence<T>内部的迭代器，也就是相当于给迭代器加了Sequence序列的外壳，核心迭代器还是由外部传入的迭代器对象，有点偷梁换柱的概念。

解释第二部分的代码:

通过第一部分，成功将普通集合转换成序列Sequence，然后现在进行map操作，实际上调用了Sequence<T>扩展函数map来实现的

```kotlin
val mapSequence = SequencesKt.map(collectionSequence, (Function1)null.INSTANCE)
```

进入map扩展函数:

```kotlin
public fun <T, R> Sequence<T>.map(transform: (T) -> R): Sequence<R> {
    return TransformingSequence(this, transform)
}
```

会发现内部会返回一个TransformingSequence对象，该对象构造器接收一个Sequence<T>类型对象，和一个transform的lambda表达式,最后返回一个Sequence<R>类型对象。我们先暂时解析到这，后面会更加介绍。

解释第三部分的代码:

通过第二部分，进行map操作后，然后返回的还是Sequence对象，最后再把这个对象进行filter操作，filter也还是Sequence的扩展函数，最后返回还是一个Sequence对象。

```kotlin
val filterSequence = SequencesKt.filter(mapSequence, (Function1)null.INSTANCE)
```

进入filter扩展函数:

```kotlin
public fun <T> Sequence<T>.filter(predicate: (T) -> Boolean): Sequence<T> {
    return FilteringSequence(this, true, predicate)
}
```

会发现内部会返回一个FilteringSequence对象，该对象构造器接收一个Sequence<T>类型对象，和一个predicate的lambda表达式,最后返回一个Sequence<T>类型对象。我们先暂时解析到这，后面会更加介绍。

Sequences源码整体结构介绍

**代码结构图:** 图中标注的都是一个个对应各个操作符类，它们都实现Sequence<T>接口

 ![sequences操作符类图](.\res\sequences操作符类图.png)

首先，Sequence<T>是一个接口，里面只有一个抽象函数，一个返回迭代器对象的函数，可以把它当做一个迭代器对象外壳。

```kotlin
public interface Sequence<out T> {
    /**
     * Returns an [Iterator] that returns the values from the sequence.
     *
     * Throws an exception if the sequence is constrained to be iterated once and `iterator` is invoked the second time.
     */
    public operator fun iterator(): Iterator<T>
}
```

**Sequence核心类UML类图**

这里只画出了某几个常用操作符的类图



![sequencesUML图](.\res\sequencesUML图.png)



**注意: 通过上面的UML类关系图可以得到，共享同一个迭代器中的数据的原理实际上就是利用Java设计模式中的状态模式(面向对象的多态原理)(PS:个人理解应该是装饰器模式)来实现的，首先通过Iterable<T>的iterator()返回的迭代器对象去实例化Sequence，然后外部调用不同的操作符，这些操作符对应着相应的扩展函数，扩展函数内部针对每个不同操作返回实现Sequence接口的子类对象，而这些子类又根据不同操作的实现，更改了接口中iterator()抽象函数迭代器的实现，返回一个新的迭代器对象，但是迭代的数据则来源于原始迭代器中。**

接着上面TransformingSequence、FilteringSequence继续解析.

通过以上对Sequences整体结构深入分析，那么接着TransformingSequence、FilteringSequence继续解析就非常简单了。我们就以TransformingSequence为例:

```kotlin
//实现了Sequence<R>接口，重写了iterator()方法，重写迭代器的实现
internal class TransformingSequence<T, R>
constructor(private val sequence: Sequence<T>, private val transformer: (T) -> R) : Sequence<R> {
    override fun iterator(): Iterator<R> = object : Iterator<R> {//根据传入的迭代器对象中的数据，加以操作变换后，构造出一个新的迭代器对象。
        val iterator = sequence.iterator()//取得传入Sequence中的迭代器对象
        override fun next(): R {
            return transformer(iterator.next())//将原来的迭代器中数据元素做了transformer转化传入，共享同一个迭代器中的数据。
        }

        override fun hasNext(): Boolean {
            return iterator.hasNext()
        }
    }

    internal fun <E> flatten(iterator: (R) -> Iterator<E>): Sequence<E> {
        return FlatteningSequence<T, R, E>(sequence, transformer, iterator)
    }
}
```

源码分析总结

序列内部的实现原理是采用状态设计模式，根据不同的操作符的扩展函数，实例化对应的Sequence子类对象，每个子类对象重写了Sequence接口中的iterator()抽象方法，内部实现根据传入的迭代器对象中的数据元素，加以变换、过滤、合并等操作，返回一个新的迭代器对象。这就能解释为什么序列中工作原理是逐个元素执行不同的操作，而不是像普通集合所有元素先执行A操作，再所有元素执行B操作。这是因为序列内部始终维护着一个迭代器，当一个元素被迭代的时候，就需要依次执行A,B,C各个操作后，如果此时没有末端操作，那么值将会存储在C的迭代器中，依次执行，等待原始集合中共享的数据被迭代完毕，或者不满足某些条件终止迭代，最后取出C迭代器中的数据即可。



#### 对比Java中的Stream(流)

Java8中引入了流来处理集合。它们表现得看起来和Kotlin中的序列很像。

```kotlin
productsList.asSequence()
    .filter { it.bought }
    .map { it.price }
    .average()

productsList.stream()
    .filter { it.bought }
    .mapToDouble { it.price }
    .average()
    .orElse(0.0)
```

它们也都是基于惰性求值的原理并且在最后(终端)处理集合。Java中的流对于集合的处理效率几乎和Kotlin中的序列处理集合一样高。Java中的Stream流和Kotlin中的Sequence序列两者最大的差别如下所示:

- Kotlin中`Sequence`序列有更多的操作符函数(因为它们可以被定义成扩展函数)并且它们的用法也相对更简单(这是因为Kotlin的序列是已经在使用的Java流基础上设计的 - 例如我们可以使用`toList()`而不是`collect(Collectors.toList())`)
- Java的`Stream`流支持可以使用`parallel`函数以并行模式使用Java流. 当我们拥有一台具有多个内核的计算机时，这可以为我们带来巨大的性能提升。
- Kotlin的`Sequence`序列可用于通用模块、Kotlin/JS模块和Kotlin/Native模块中。

除此之外，当我们不使用并行模式时，要说Java stream和 Kotlin sequence哪个更高效，这个真的很难说。

***我的建议是仅仅将Java中的`Stream`用于计算量较大的处理以及需要启用并行模式的场景。否则其他一般场景使用Kotlin Stdlib标准库中`Sequence`序列，可以给你带来相同效率并且操作函数使用起来也很简单，代码更加简洁。***





## Kotlin中隐藏的性能开销

### Function对象

这是反编译上面的lambda表达式后的Java代码.

```java
class MyClass$myMethod$1 implements Function1 {
   // $FF: synthetic method
   // $FF: bridge method
   public Object invoke(Object var1) {
      return Integer.valueOf(this.invoke((Database)var1));//被装箱成Integer对象，这个下一节会具体讲到
   }

   public final int invoke(@NotNull Database it) {
      Intrinsics.checkParameterIsNotNull(it, "it");
      return it.delete("Customers", null, null);
   }
}
```

在你的Android dex文件中，编译为Function对象的每个lambda表达式实际上会为总方法计数添加3或4个方法。

值得高兴的是这些Function对象的新实例并不是每种情况都会创建，仅在必要的时候创建。所以这就意味着你在实际使用中，需要知道什么情况下会创建Function对象的新实例以便于给你的程序带来更好的性能:

- 对于*捕获表达式*情况，每次将lambda作为参数传递，然后执行后进行垃圾回收，就会每次创建一个新的Function实例；
- 对于*非捕获表达式*(也即是纯函数)情况，将在下次调用期间创建并复用单例函数实例。

由于我们上述的例子调用者代码使用的是*非捕获lambda*，因此它会被编译为单例而不是内部类。

```java
this.transaction(db, (Function1)MyClass$myMethod$1.INSTANCE);
```

> *如果要调用 **捕获lambda** 来减少对垃圾收集器的压力，请避免重复调用标准(非内联)高阶函数*。

### 装箱带来的性能开销

与Java8相反的是，Java8大约有[43中不同的特殊函数接口](https://link.zhihu.com/?target=https%3A//docs.oracle.com/javase/8/docs/api/java/util/function/package-summary.html)，以尽可能避免装箱和拆箱, 而Kotlin编译的Function对象仅仅实现完全通用的接口，有效地将Object类型用于任何的输入或输出值。

```kotlin
/** A function that takes 1 argument. */
public interface Function1<in P1, out R> : Function<R> {
    /** Invokes the function with the specified argument. */
    public operator fun invoke(p1: P1): R
}
```

这就意味着当函数涉及输入值或返回值是基本类型（如Int或Long）时，调用在高阶函数中作为参数传递的函数实际上将涉及**系统的装箱和拆箱**。这可能会对性能上产生不可忽视的影响，特别是在Android上。

在上面例子编译的lambda中，你可以看到结果被装箱到Integer对象。然后，调用者代码将立即将其拆箱。

> *在编写涉及使用基本类型作为输入或输出值的参数函数的标准（**非内联**）高阶函数时要小心。反复调用此参数函数将通过装箱和拆箱操作对垃圾收集器施加更大的压力*。

### 内联函数

值得庆幸的是，Kotlin提供了一个很好的语法技巧，可以避免在使用lambda表达式时带来的额外性能开销: 将高阶函数声明成 **[内联](https://link.zhihu.com/?target=https%3A//kotlinlang.org/docs/reference/inline-functions.html%23reified-type-parameters)**. 这讲使得编译器直接执行调用者代码中内联函数体中代码，完全避免了调用带来的开销。对于高阶函数，其好处甚至更大，因为作为参数传递的lambda表达式的主体也将被内联。实际效果如下：

- 声明lambda表达式时，不会实例化Function对象
- 没有装箱或拆箱操作将应用于基于原始类型的lambda输入和输出值
- 没有方法将添加到总方法计数中
- 不会执行实际的函数调用，这可以提高对此使用该函数带来的CPU占用性能

在将transaction（）函数声明为内联之后，我们的调用者代码的Java高效实现如下:

```java
db.beginTransaction();
try {
   int result$iv = db.delete("Customers", null, null);
   db.setTransactionSuccessful();
} finally {
   db.endTransaction();
}
```

然后使用这个杀手锏级别功能时，有一些地方需要注意:

- 内联函数不能直接调用自身或通过其他内联函数调用自身
- 在类中声明公有的内联函数只能访问该类的公有函数和成员变量
- 代码的大小会增加。内联多次引用代码较长的函数可以使生成的代码更大，如果这个代码较长的函数本身引用其他代码较长的内联函数，则会更多。

> *如果可能，将高阶函数声明为内联函数。保持简短，如果需要，将大段代码移动到非内联函数。 你还可以内联从代码的性能关键部分调用的函数。*

我们将在以后的文章中讨论内联函数的其他性能优势



### 内联类

自动装箱可能会导致内联失效

这段代码:

```kotlin
interface Amount { val value: Int }
inline class Points(override val value: Int) : Amount
```

在这段代码中，内联类`Points`实现了`Amount`接口。当我们调用`addToScore()`函数时，会引发一个有趣的现象，尽管...

```kotlin
fun addToScore(amount: Amount) {
    totalScore += amount.value
}
```

`addToScore()`函数可以接收任何`Amount`类型的对象。由于`Points`是`Amount`的子类型，所以我们可以传入一个`Points`类型实例对象给这个函数。

这是基本的常识，没问题吧?

但是... 假设我们的`Points`类的实例都是内联的-也就是说，在源码被编译的阶段，它们(`Points`类的实例)会被基础类型(这里是Int整数类型)给替换掉。-可是`addToScore()`函数怎么能接收一个基础类型(这里是Int整数类型)的实参呢?毕竟，基础类型`Int`并没有去实现`Amount`的接口。

#### 引用超类型时会触发自动装箱操作

正如我们所看到的那样，当我们将`Points`对象传递给接收`Amount`类型作为形参的函数式，就触发了自动装箱操作。

即使你的内联类没有去实现接口，但是必须记住一点，内联类和普通类一样，所有内联类都是`Any`的子类型。所以当你将内联类的实例赋值给`Any`类型的变量或者传递给`Any`类型作为形参的函数时，都会触发预期中的自动装箱操作。

例如，假设我们有一个可以记录日志的服务接口:

```kotlin
interface LogService {
    fun log(any: Any)
}
```

由于这个`log()`函数可以接收一个`Any`类型的实参，一旦你传入一个`Points`的实例给这个函数，那么这个实例就会触发自动装箱操作。

```kotlin
val points = Points(5)
logService.log(points) // <--- Autoboxing happens here(此处发生自动装箱操作)
```

总之一句话 - 当你使用内联类的实例（其中需要超类型）时，可能会触发自动装箱。

#### 自动装箱与泛型

当您使用具有泛型的内联类时，也会发生自动装箱。例如:

```
 代码解读复制代码val points = Points(5)

val scoreAudit = listOf(points)      // <-- Autoboxing here(此处发生自动装箱操作)

fun <T> log(item: T) {
    println(item)
}

log(points)                          // <-- Autoboxing here(此处发生自动装箱操作)
```

在使用泛型时，Kotlin为我们自动装箱是件好事，否则我们会在编译代码中会遇到类型安全的问题。例如，类似于我们之前的场景，将整数类型的值插入到`MutableList<Amount>`集合类型中是不安全的，因为整数类型并没有去实现`Amount`的接口。

而且，一旦考虑到与Java互操作时，它就会变得更加复杂，例如:

- 如果Java将`List<Points>`保存为`List<Integer>`，它是否应该可以将该类型的集合传递给如下这个Kotlin函数呢?

```kotlin
fun receive(list: List<Int>)
```

- Java将它传递给下面这个Kotlin函数又会怎么样呢？

```kotlin
fun receive(list: List<Amount>)
```

- Java能否可以构建自己的整数集合并把它传递给下面这个Kotlin函数?

```kotlin
fun receive(list: List<Points>)
```

相反，Kotlin通过自动装箱的操作来避免了内联类和泛型一起使用时的问题。

我们已经看到超类型和泛型两种场景下如何触发自动装箱操作。其实我们还有一个值得去深究的场景 - 那就是**可空性**的场景！

#### 自动装箱和可空性

当涉及到可空类型的值时，也可能会触发自动装箱操作。这个规则有点不同，主要取决于基础类型是引用类型还是基本数据类型。所以让我们一次性来搞定它们。

##### 引用类型

![内联函数触发自动装箱-引用类型](.\res\内联函数触发自动装箱-引用类型.png)

##### 基本数据类型

![内联函数触发自动装箱-基本数据类型](.\res\内联函数触发自动装箱-基本数据类型.png)

正如你所看到的那样，上面表格中对于**基本数据类型**的结果除了场景B不一样，其他的场景都和**引用类型**分析结果一样。但是这里面还是涉及到了其他很多知识，所以让我们花点时间一一分析下每一种情况。

**对于场景A.** 很容易就能分析出来。因为这里根本就没有可空类型(都是非空类型)，所以类型是内联的，正如我们所期望的那样。

**对于场景B.** 这是一种完全不同于上一个真值表中的场景，不知道你是否还记得，JVM上的`int`和`boolean`等其他基本数据类型实际上是不能为`null`的。因此，为了更好兼容`null`,Kotlin在此使用了包装类型(也就触发了自动装箱操作)

**对于场景C.** 这种场景就更有意思了。一般来说，当你有一个类似`Int`可以为空的基本数据类型时，在Kotlin中，这种基本数据类型会在编译的时候转换成Java中的基本数据类型对应的包装器类型-例如`Integer`,它(不像`int`)可以兼容`null`值。对于场景C而言，实际上在使用内联类地方编译时候却使用基础类型，因为它本身恰好是一个Java中基本包装器类型。所以在某种层面上，你可以说基础类型被自动装箱了，但是这种自动装箱操作和内联类根本就没有任何关系。

**对于场景D.** 类似于上面引用类型看到的那样，当基本类型自身为可空以及使用内联类地方为可空时，Kotlin将在编译时使用包装器类型。具体原因和引用类型同理。







### 伴生对象

Kotlin类中已经没有了静态字段或方法。取而代之的是在类的[伴生对象](https://link.zhihu.com/?target=https%3A//kotlinlang.org/docs/reference/object-declarations.html%23companion-objects)中声明与实例无关的字段和方法.

### 从其伴生对象访问私有类字段

不妨看下这个例子:

```kotlin
class MyClass private constructor() {

    private var hello = 0

    companion object {
        fun newInstance() = MyClass()
    }
}
```

编译时，伴生对象会被实现为单例类。这意味着就像任何需要从其他类访问其私有字段的Java类一样，**从伴随对象访问外部类的私有字段（或构造函数）将生成其他合成getter和setter方法**。对类字段的每次读取或写入访问都将导致伴随对象中的静态方法调用。

```text
ALOAD 1
INVOKESTATIC be/myapplication/MyClass.access$getHello$p (Lbe/myapplication/MyClass;)I
ISTORE 2
```

在Java中，我们可以通过使用`package`可见性来避免生成这些方法。然后在Kotlin中的没有`package`可见性。使用`public`和`internal`可见性将会导致Kotlin生成默认的getter和setter实例方法，以致于外部可以访问到这些字段，然而调用实例方法在技术上往往比调用静态方法更为昂贵。

> *如果需要从伴生对象重复读取或写入类字段，则可以将其值缓存在局部变量中，以避免重复的隐藏方法调用。*

### 访问伴生对象中声明的常量

在Kotlin中，您通常会在伴生对象内声明您在类中使用的“静态”常量。

```kotlin
class MyClass {

    companion object {
        private val TAG = "TAG"
    }

    fun helloWorld() {
        println(TAG)
    }
}
```

上述代码虽然看起来简洁明了，但是底层实现执行操作就十分的难看。

出于同样的原因，**访问伴生对象中声明的私有常量实际上会在伴生对象实现类中生成一个额外的合成getter方法**。

```text
GETSTATIC be/myapplication/MyClass.Companion : Lbe/myapplication/MyClass$Companion;
INVOKESTATIC be/myapplication/MyClass$Companion.access$getTAG$p (Lbe/myapplication/MyClass$Companion;)Ljava/lang/String;
ASTORE 1
```

但是更糟糕的是，合成方法实际上不会返回值，它调用的是一个Kotlin生成的getter实例方法.

```text
ALOAD 0
INVOKESPECIAL be/myapplication/MyClass$Companion.getTAG ()Ljava/lang/String;
ARETURN
```

当常量声明为public而不是private时，此getter方法是公共的并且可以直接调用，因此不需要上一步的合成方法。但是Kotlin仍然需要调用getter方法来读取常量.

那么，我们结束了吗？没有! 事实证明，为了存储常量值，Kotlin编译器是在主类级别中而不是在伴生对象内生成实际的`private static final`的字段。但是，**因为静态字段在类中声明为private，所以需要另一种合成方法来从伴生对象中访问它**。

```text
INVOKESTATIC be/myapplication/MyClass.access$getTAG$cp ()Ljava/lang/String;
ARETURN
```

并且该合成方法最后读取实际值：

```text
GETSTATIC be/myapplication/MyClass.TAG : Ljava/lang/String;
ARETURN
```

换句话说，当你从Kotlin类访问伴生对象中的私有常量字段时，而不是像Java那样直接读取静态字段，代码实际上是：

- 在伴生对象中调用静态方法
- 它将依次调用伴随对象中的实例方法
- 然后反过来调用类中的静态方法
- 读取静态字段并返回其值

```java
public final class MyClass {
    private static final String TAG = "TAG";
    public static final Companion companion = new Companion();

    // synthetic 
    public static final String access$getTAG$cp() {
        return TAG;
    }

    public static final class Companion {
        private final String getTAG() {
            return MyClass.access$getTAG$cp();
        }

        // synthetic
        public static final String access$getTAG$p(Companion c) {
            return c.getTAG();
        }
    }

    public final void helloWorld() {
        System.out.println(Companion.access$getTAG$p(companion));
    }
}
```

那么我们可以获得更轻量级的字节码吗？是的，但不是在所有情况下。

首先，通过使用**const关键字**将值声明为编译时常量，可以完全避免任何方法调用。这将直接在调用代码中有效地内联值，但是**只能将它用于原始类型和字符串**。

```kotlin
class MyClass {

    companion object {
        private const val TAG = "TAG"
    }

    fun helloWorld() {
        println(TAG)
    }
}
```

其次，可以在伴生对象的公共字段上使用[@JvmField](https://link.zhihu.com/?target=https%3A//kotlinlang.org/docs/reference/java-to-kotlin-interop.html%23instance-fields)注解，以指示编译器不生成任何getter或setter，并将其作为类中的静态字段公开，就像纯Java常量一样。事实上，这个注解是出于Java兼容性问题才创建的，如果你不需要从Java代码中访问你这个常量，我不建议你使用这个互操作注解来混乱优雅的Kotlin代码。此外，它只能用于公有字段。在Android开发环境中，可能只会使用此注解来实现`Parcelable`对象：

```kotlin
class MyClass() : Parcelable {

    companion object {
        @JvmField
        val CREATOR = creator { MyClass(it) }
    }

    private constructor(parcel: Parcel) : this()

    override fun writeToParcel(dest: Parcel, flags: Int) {}

    override fun describeContents() = 0
}
```

最后，你还可以使用ProGuard工具优化字节码，并希望它将这些链式方法调用合并在一起,但是无法绝对保证这起到理想中的作用。

> *从伴随对象中读取“静态”常量，与Java相比，在Kotlin中增加了两到三个额外的间接级别，并且将为这些常量中的每一个生成两到三个额外的方法。*
> *1、始终使用**const关键字**声明基本类型和字符串常量以避免这种情况*
> *2、对于其他类型的常量，不能使用const，因此如果需要重复访问常量，可能需要将值缓存在局部变量中*
> *3、此外，更推荐将公有的全局常量存储在它们自己的对象中而不是伴随对象中。*



### 局部函数

这是我们之前第一篇文章中没有介绍过的一种函数: 就是像正常定义普通函数的语法一样，在其他函数体内部声明该函数。这些被称为[局部函数](https://link.zhihu.com/?target=https%3A//kotlinlang.org/docs/reference/functions.html%23local-functions)，它们能访问到外部函数的作用域。

```kotlin
fun someMath(a: Int): Int {
    fun sumSquare(b: Int) = (a + b) * (a + b)

    return sumSquare(1) + sumSquare(2)
}
```

我们首先来说下局部函数最大的局限性: **局部函数不能被声明成`内联的(inline)`并且函数体内含有局部函数的函数也不能被声明成`内联的(inline)`.** 在这种情况下没有任何有效的方法可以帮助你避免函数调用的开销。

经过编译后，这些局部函数会将被转化成`Function`对象, 就类似lambda表达式一样，并且同样具有上篇文章part1中讲到的关于非内联函数存在很多的限制。反编译后的java代码:

```java
public static final int someMath(final int a) {
   Function1 sumSquare$ = new Function1(1) {
      // $FF: synthetic method
      // $FF: bridge method
      //注: 这是Function1接口生成的泛型合成方法invoke
      public Object invoke(Object var1) {
         return Integer.valueOf(this.invoke(((Number)var1).intValue()));
      }

      //注: 实例的特定方法invoke
      public final int invoke(int b) {
         return (a + b) * (a + b);
      }
   };
   return sumSquare$.invoke(1) + sumSquare$.invoke(2);
}
```

但是与lambda表达式相比，它对性能的影响要小得多: 由于该函数的实例对象是从调用方就知道的，所以它将直接调用该实例的特定方法`invoke`而不是从`Function`接口直接调用其泛型合成方法`invoke`。这就意味着**从外部函数调用局部函数时，不会进行基本类型的转换或装箱操作.** 我们可以通过看下字节码来验证一下:

```text
ALOAD 1
   ICONST_1
   INVOKEVIRTUAL be/myapplication/MyClassKt$someMath$1.invoke    (I)I
   ALOAD 1
   ICONST_2
   INVOKEVIRTUAL be/myapplication/MyClassKt$someMath$1.invoke    (I)I
   IADD //加法操作
   IRETURN
```

我们可以看到被调用两次的函数是接收一个 **`Int`** 类型的参数并且返回一个 **Int** 类型的函数，并且加法操作是立即执行的，而无需任何中间的装箱、拆箱操作。

当然，在每次方法被调用期间仍会创建一个新的Function对象。但是这个可以通过将局部函数改写为非捕获的方式来避免这种情况:

```kotlin
fun someMath(a: Int): Int {
    fun sumSquare(a: Int, b: Int) = (a + b) * (a + b)

    return sumSquare(a, 1) + sumSquare(a, 2)
}
```

现在相同的Function实例将会被复用，仍然不会进行强制的转换或装箱操作。与普通的私有函数相比，此局部函数的唯一劣势就是使用一些方法生成额外的类。

*局部函数是私有函数的替代品，其附加好处是能够访问外部函数的局部变量。然而这种好处会伴随着为外部函数每次调用创建Function对象的隐性成本，因此首选使用非捕获的局部函数。*

### 空安全

Kotlin语言的最好特性之一就是，它在可空类型和非空类型之间做出了明显清晰的界限区分。这使得编译可以通过在运行时禁止将非null或可为null的值分配给非null变量的任何代码来有效防止意外的NullPointerException.

### 非空参数的运行时检查

下面我们来声明一个使用非null字符串作为采纳数的公有函数:

```kotlin
fun sayHello(who: String) {
    println("Hello $who")
}
```

现在来看下对应的反编译后Java代码:

```java
public static final void sayHello(@NotNull String who) {
   Intrinsics.checkParameterIsNotNull(who, "who");//执行静态函数进行非空检查
   String var1 = "Hello " + who;
   System.out.println(var1);
}
```

请注意，Kotlin编译器对Java是非常友好的，可以看到在函数参数上自动添加了`@NotNull`注解，因此Java工具可以使用此注解在传递空值的时候显示警告。

但是，注解不足以强制外部调用者传入非null的值。因此，编译器还在函数的开头添加一个**静态方法调用**，该方法将检查参数，如果为**null**，则抛出`IllegalArgumentException`. 为了使不安全的调用者代码更易于修复，该函数将尽早且持续抛出异常，而不是将它置后抛出运行时的`NullPointerException`.

实际上，**每个公有的函数**都有一个对`Intrinsics.checkParameterIsNotNull()`的静态调用，该调用为**每个非null引用参数**添加。这些检查**不会被添加到私有函数中**，因为编译器保证了Kotlin类中的代码为null安全的。

这些静态调用对性能的影响几乎可以忽略不计，并且在调试和测试应用程序的时候非常有帮助。话虽如此，如果对于release版本来说你可能认为这是没必要的额外开销。在这种情况下，可以使用`-Xno-param-assertions`编译器选项或添加以下Proguard规则来禁止运行时的空检查:

```text
-assumenosideeffects class kotlin.jvm.internal.Intrinsics {
    static void checkParameterIsNotNull(java.lang.Object, java.lang.String);
}
```

### 可空的原生类型

有一点似乎众所周知，但还是在这里提醒下: 可空类型始终是引用类型。将原生类型的变量声明成**可空类型**可以防止Kotlin使用Java基本数据类型(例如**int**或**float**), 而是使用**装箱的引用类型**(例如`Integer`或`Float`),这会避免装箱和拆想操作带来的额外开销。

与Java相反的是它允许你草率地使用几乎像int变量的Integer变量，这都要归功于自动装箱和忽略了null的安全性，可是Kotlin则会强制你在使用可null的类型时编写空安全的代码，因次使用非null类型的好处就变得更显而易见了:

```kotlin
fun add(a: Int, b: Int): Int {
    return a + b
}
fun add(a: Int?, b: Int?): Int {
    return (a ?: 0) + (b ?: 0)
}
```

*尽可能使用非null的原生类型，以此来提高代码可读性和性能。*

### 关于数组

在Kotlin中存在3种类型的数组:

- `IntArray`,`FloatArray`以及其他原生类型的数组。
  最终会编译成 `int[]`,`float[]`以及其他对应基本数据类型的数组
- `Array<T>`: 非空对象引用类型的数组
  这里会涉及到原生类型的装箱过程
- `Array<T?>`: 可空对象引用类型的数组
  很明显，这里也会涉及到原生类型的装箱过程

*如果你需要一个非null原生类型的数组，最好使用`IntArray`而不是`Array<Int>`以避免装箱过程带来性能开销*

### 可变数量的参数(Varargs)

类似Java, Kotlin允许使用可变数量的参数声明函数。只是声明的语法有点不一样而已:

```kotlin
fun printDouble(vararg values: Int) {
    values.forEach { println(it * 2) }
}
```

就像在Java中一样，vararg参数实际上被编译为给定类型的数组参数。然后，可以通过三种不同的方式调用这些函数：

#### 1.传递多个参数

```kotlin
printDouble(1, 2, 3)
```

Kotlin编译器将将此代码转换为新数组的创建和初始化，就像Java编译器一样：

```java
printDouble(new int[]{1, 2, 3});
```

所以，创建新数组会产生开销，但是与Java相比，这并不是什么新鲜事。

#### 2.传递单个数组

这里不同之处就是，在Java中，可以直接将现有的数组引用作为vararg参数传递。在Kotlin中，则需要使用*伸展（spread）操作符*:

```kotlin
val values = intArrayOf(1, 2, 3)
printDouble(*values)
```

在Java中，数组引用按原样传递给函数，而无需分配额外的数组空间。然而，如你在反编译后java代码中所见，Kotlin伸展（spread）操作符的编译方式有所不同：

```kotlin
int[] values = new int[]{1, 2, 3};
printDouble(Arrays.copyOf(values, values.length));
```

调用函数时，**始终会复制现有数组**。好处是代码更安全：它允许函数修改数组而不影响调用者代码。**但是它会分配额外的内存**。

*请注意，使用Kotlin代码中可变数量的参数调用Java方法具有相同的效果。*

#### 3.传递数组和参数的混合

*Kotlin伸展（spread）运算符*的主要好处是它还允许在同一调用中将数组与其他参数混合在一起。

```kotlin
val values = intArrayOf(1, 2, 3)
printDouble(0, *values, 42)
```

上述代码将会怎样编译呢？生成代码会十分有趣:

```java
int[] values = new int[]{1, 2, 3};
IntSpreadBuilder var10000 = new IntSpreadBuilder(3);
var10000.add(0);
var10000.addSpread(values);
var10000.add(42);
printDouble(var10000.toArray());
```

除了**创建新数组**之外，还使用一个**临时生成器对象**来计算最终数组大小并填充它。这给方法调用又增加了另一笔小开销。

*即使在使用现有数组中的值时，在Kotlin中调用具有可变数量参数的函数也会增加创建新临时数组的成本。对于重复调用该函数的性能至关重要的代码，请考虑添加具有实际数组参数而不是**vararg**的方法*

感谢您的阅读，如果喜欢，请分享这篇文章。

### 泛型代理

还可以以泛型的方式声明代理函数，因此同一个代理类可以用任意的属性类型。

```kotlin
private var maxDelay: Long by SharedPreferencesDelegate<Long>()
```

但是，如果像上面例子那样使用具有原生类型属性的泛型代理的话，即便声明的原生类型为非null，每次读取或写入该属性时都避免不了**装箱和拆箱的发生**。

> *对于非null原生类型的代理属性，最好使用为该特定值类型创建特定的代理类，而不是泛型代理，以避免在每次访问该属性时产生的装箱开销*





### 补充

#### 1、为什么非捕获局部函数可以减少开销

其实关于捕获和非捕获的概念，在之前文章中也有所提及，比如在讲变量的捕获，lambda的捕获和非捕获。

这里就以上述局部函数举例，下面对比下这两个函数:

```kotlin
//改写前的捕获局部函数
fun someMath(a: Int): Int {
    fun sumSquare(b: Int) = (a + b) * (a + b)//注意:局部函数这里的a是直接引用外部函数的参数a, 
    //因为局部函数特性可以访问外部函数的作用域，这里实际上就存在了变量的捕获，所以这里sumSquare称为捕获局部函数

    return sumSquare(1) + sumSquare(2)
}
//改写前反编译后代码
 public static final int someMath(final int a) {
      //创建Function1对象$fun$sumSquare$1，所以每调用一次someMath都会创建一个Function1对象
      <undefinedtype> $fun$sumSquare$1 = new Function1() {
         // $FF: synthetic method
         // $FF: bridge method
         public Object invoke(Object var1) {
            return this.invoke(((Number)var1).intValue());
         }

         public final int invoke(int b) {
            return (a + b) * (a + b);
         }
      };
      return $fun$sumSquare$1.invoke(1) + $fun$sumSquare$1.invoke(2);
   }
```

捕获局部函数会生成额外的Function对象，所以我们为了减少性能的开销尽量使用非捕获局部函数。

```kotlin
//改写后的非捕获局部函数
fun someMath(a: Int): Int {
    //注意: 可以明显发现改写后a参数，直接由函数参数传入，而不是在局部函数直接引用外部函数的参数变量，这就是非捕获局部函数
    fun sumSquare(a: Int, b: Int) = (a + b) * (a + b)
    return sumSquare(a,1) + sumSquare(a,2)
}

//改写后反编译后代码
public static final int someMath(int a) {
    //注意:可以看到非捕获的局部函数实例是一个单例，多次调用都只会复用之前的实例不会重新创建。
    <undefinedtype> $fun$sumSquare$1 = null.INSTANCE;
    return $fun$sumSquare$1.invoke(a, 1) $fun$sumSquare$1.invoke(a, 2);
}
```

通过上述对比，应该很清楚知道了什么是捕获什么是非捕获以及为什么非捕获局部函数会减少性能的开销。

#### 2、总结下提高Kotlin代码性能开销几个点

- 局部函数是私有函数的替代品，其附加好处是能够访问外部函数的局部变量。然而这种好处会伴随着为外部函数每次调用创建Function对象的隐性成本，因此首选使用非捕获的局部函数。
- 对于release版本应用来说，特别是Android应用，可以使用`-Xno-param-assertions`编译器选项或添加以下Proguard规则来禁止运行时的空检查:

```text
-assumenosideeffects class kotlin.jvm.internal.Intrinsics {
    static void checkParameterIsNotNull(java.lang.Object, java.lang.String);
}
```

- 需要使用非null原生类型的数组时，最好使用`IntArray`而不是`Array<Int>`以避免装箱过程带来性能开销