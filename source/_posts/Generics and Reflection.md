title: C#.NET的泛型与反射
date: 2015-06-30 23:25:43
tags: [泛型, C#, 反射, CLR]
categories: C#基础
---
#### 引言

在之前与同事的讨论中发现对C#泛型反射时的一些术语理解有些错误或者不够深刻，借此文对相应的知识进行一下整理。本文分为两个部分，第一部分主要简单的介绍一下 “运行时中泛型”，第二部分对泛型反射的一些术语进行解释并参照代码和输出结果进行分析和巩固。

#### CLR中的泛型

在.NET中将泛型类型或者泛型方法编译为MSIL时是包含有类型参数的元数据。也就是说在MSIL层面，只负责泛型的声明和使用，而泛型实例化是由CLR在运行时负责完成的。

泛型声明：

```cs
    public class Foo<T>
    {
        public void M1<TU>()
        {
            // 泛型的使用
            var listOne = new List<Int32>();

            Console.WriteLine(typeof(TU));
            Console.WriteLine(typeof(T));
        }
    }
```

上述泛型类将被编译成名叫 Foo\`1 < T > 的泛型类型 ,其 M1< TU > 泛型方法对应的IL代码如下：

```cs
.method public hidebysig instance void  M1<TU>() cil managed
{
  // 代码大小       40 (0x28)
  .maxstack  1
  .locals init ([0] class [mscorlib]System.Collections.Generic.List`1<int32> listOne)
  IL_0000:  nop
  IL_0001:  newobj     instance void class [mscorlib]System.Collections.Generic.List`1<int32>::.ctor()
  IL_0006:  stloc.0
  IL_0007:  ldtoken    !!TU
  IL_000c:  call       class [mscorlib]System.Type [mscorlib]System.Type::GetTypeFromHandle(valuetype [mscorlib]System.RuntimeTypeHandle)
  IL_0011:  call       void [mscorlib]System.Console::WriteLine(object)
  IL_0016:  nop
  IL_0017:  ldtoken    !T
  IL_001c:  call       class [mscorlib]System.Type [mscorlib]System.Type::GetTypeFromHandle(valuetype [mscorlib]System.RuntimeTypeHandle)
  IL_0021:  call       void [mscorlib]System.Console::WriteLine(object)
  IL_0026:  nop
  IL_0027:  ret
} // end of method Foo`1::M1
```

CLR会根据运行时提供的类型参数是值类型还是引用类型而不同，对于值类型CLR将会为每个类型创建(第一次遇到时)专用的泛型类型，对于引用类型则略有不同。所有的引用类型都会重用同一个泛型版本，大体上可以想象为所有的引用类型使用一个 Object 的泛型实例化版本。之所以这样设计，是因为引用的大小是相同的(32位系统上是32位,64位系统上是64位)，而值类型则不然。这样设计的好处是对于引用类型可以共享同一份native code, 避免“类型爆炸”。（实例化的泛型类型虽然享有独立的元数据，但是共享同一个EEClass，即其TypeHandle指向的MethodTable中的EEClass地址相同。）

#### 泛型与反射

#####一些术语

1. 开放泛型类型/方法（open generic type/method）：所有泛型参数都未绑定值的泛型类型/方法定义，例如，Dictionary< TKey, TVal >。

2. 开放构造类型/方法(open constructed type/method)：部分泛型参数绑定了具体类型的值的泛型类型/方法， 例如，Dictionary< String, TVal>。

3. 封闭构造类型/方法(close constructed type/method)：按照 ContainsGenericParameters=true 来定义，分为两种：1）封闭泛型类型/方法（close generic type/method），所有泛型参数都绑定了具体类型的值的泛型类型/方法，例如，Dictionary< String, Int32>；2）普通的类型/方法。

***注意：在泛型类型上可以调用MakeGenericType生成构造类型，在泛型方法定义上调用MakeGenericMethod生成构造方法。ContainsGenericParameters 属性提供一种标准方法来区分封闭构造类型/方法（可以实例化/调用）和开放构造类型/方法（不能实例化/调用）。***

    ContainsGenericParameters 属性递归搜索类型参数
    1、对于开放类型 A< T> 上的所有方法调用 ContainsGenericParameters 都将返回true；
    2、对于下面代码中的 listOfSomeUnknownTypeOfList 类型，由于 g1是 开放泛型类型所以listOfSomeUnknownTypeOfList.ContainsGenericParameters 返回 true。
    
    ```cs
    var gl = typeof(List<>);
    var listOfSomeUnknownTypeOfList = gl.MakeGenericType(gl);
    ```

1. 泛型类型：包括开放泛型类型、开放构造类型、封闭泛型类型

2. 泛型方法：包括开放泛型方法、开放构造方法（注：不包括开放泛型类型、开放构造类型中的非泛型开放构造方法，见后面 “开放泛型类型（及开放构造类型）上的方法反射” 部分内容）、封闭泛型方法

<!-- more -->

#####泛型类型的反射

```cs
static void Main()
{
    var g = typeof(Dictionary<,>);
    var g1 = g.MakeGenericType(new[] { typeof(string), g.GenericTypeArguments[1] });
    var g2 = g.MakeGenericType(new[] { g1.GenericTypeArguments[0], typeof(int) });
     Console.WriteLine("{0}, IsGenericType: {1}, IsGenericTypeDefinition: {2}, ContainsGenericParameters: {3}", g, g.IsGenericType, g.IsGenericTypeDefinition, g.ContainsGenericParameters);
     Console.WriteLine("{0}, IsGenericType: {1}, IsGenericTypeDefinition: {2}, ContainsGenericParameters: {3}", g1, g1.IsGenericType, g1.IsGenericTypeDefinition, g1.ContainsGenericParameters);
     Console.WriteLine("{0}, IsGenericType: {1}, IsGenericTypeDefinition: {2}, ContainsGenericParameters: {3}", g2, g2.IsGenericType, g2.IsGenericTypeDefinition, g2.ContainsGenericParameters);
}
```

输出是：

```cs
System.Collections.Generic.Dictionary`2[TKey,TValue], IsGenericType: True, IsGenericTypeDefinition: True, ContainsGenericParameters: True
System.Collections.Generic.Dictionary`2[System.String,TValue], IsGenericType: True, IsGenericTypeDefinition: False, ContainsGenericParameters: True
System.Collections.Generic.Dictionary`2[System.String,System.Int32], IsGenericType: True, IsGenericTypeDefinition: False, ContainsGenericParameters: False
```
1. g是泛型类型定义, 也就是开放泛型类型；
2. g1是开放构造类型；
3. g2是封闭构造（泛型）类型。

##### 泛型方法反射

###### 普通类型的泛型方法反射

```cs
    class Program
    {
        static void Main()
        {
            var m = typeof(Program).GetMethod("Get");
            var m1 = m.MakeGenericMethod(new[] { typeof(String), m.GetGenericArguments()[1] });
            var m2 = m.MakeGenericMethod(new[] { typeof(String), typeof(Int32) });
            Println(m);
            Println(m1);
            Println(m2);
        }

        public static void Println(MethodInfo methodInfo)
        {
            Console.WriteLine("{0}, IsGenericMethod: {1}, IsGenericMethodDefinition: {2}, ContainsGenericParameters: {3}", methodInfo, methodInfo.IsGenericMethod, methodInfo.IsGenericMethodDefinition, methodInfo.ContainsGenericParameters);
        }

        public TV Get<TK, TV>(TK key)
        {
            return default(TV);
        }
    }
```

输出是：
```cs
TV Get[TK,TV](TK), IsGenericMethod: True, IsGenericMethodDefinition: True, ContainsGenericParameters: True
TV Get[String,TV](System.String), IsGenericMethod: True, IsGenericMethodDefinition: False, ContainsGenericParameters: True
Int32 Get[String,Int32](System.String), IsGenericMethod: True, IsGenericMethodDefinition: False, ContainsGenericParameters: False
```

1. m是generic method definition，是开放泛型方法；
2. m1是开放构造方法；
3. m2是封闭构造（泛型）方法。

###### 开放泛型类型（及开放构造类型）上的方法反射

泛型类型定义的方法如果仅仅包含有类级别的泛型参数，这样的方法是[**非泛型方法，而不是泛型方法**](https://msdn.microsoft.com/zh-cn/library/twcad0zb.aspx)。结合CRL JIT在处理泛型的方式上可以更容易理解，对于泛型类型上的非泛型方法由于不包含方法级的类型化参数，在泛型实例化后是封闭的，在实例化类型的方法表（method table）里面是一个普通方法。而对于泛型类型上的泛型方法，泛型实例化以后也是开放的，这种方法是由特殊的方法表(special method table)来处理，通过调用时传入具体的类型参数进行索引得到相应的方法地址。详细的解释可参考 [how-virtual-generic-method-call-is-implemented](http://stackoverflow.com/questions/6573557/how-virtual-generic-method-call-is-implemented) 的回答。

```cs
    class Program
    {
        static void Main()
        {
            var g = typeof(Foo<>).GetMethod("Get");
            Println(g);

            var m = typeof(Foo<>).GetMethod("M1");
            var m1 = m.MakeGenericMethod(new[] { typeof(String)});
            Println(m);
            Println(m1);
        }

        public static void Println(MethodInfo methodInfo)
        {
            Console.WriteLine("{0}, IsGenericMethod: {1}, IsGenericMethodDefinition: {2}, ContainsGenericParameters: {3}", methodInfo, methodInfo.IsGenericMethod, methodInfo.IsGenericMethodDefinition, methodInfo.ContainsGenericParameters);
        }
    }

    public class Foo<T>
    {
        public void M1<V>(V t)
        {
            Console.WriteLine(typeof(V));
        }

        public T Get(String key)
        {
            return default(T);
        }
    }
```

输出是：
```cs
T Get(System.String), IsGenericMethod: False, IsGenericMethodDefinition: False, ContainsGenericParameters: True
Void M1[V](V), IsGenericMethod: True, IsGenericMethodDefinition: True, ContainsGenericParameters: True
Void M1[String](System.String), IsGenericMethod: True, IsGenericMethodDefinition: False, ContainsGenericParameters: True
```

1. g仅仅包含类级别的泛型参数，**是开放构造方法，但不是泛型方法**
2. m是泛型方法定义；
3. m1是开放构造（泛型）方法，虽然方法类型参数已经绑定到具体类型String上，但方法依附的类型是开放泛型类型Foo< T>，所以ContainsGenericParameters=true，m1是不能被直接调用的。

###### 封闭泛型类型上的方法反射

```cs
class Program
    {
        static void Main()
        {
            var g = typeof(Foo<Int32>).GetMethod("Get");
            Println(g);

            var m = typeof(Foo<Int32>).GetMethod("M1");
            var m1 = m.MakeGenericMethod(new[] { typeof(String)});
            Println(m);
            Println(m1);
        }

        public static void Println(MethodInfo methodInfo)
        {
            Console.WriteLine("{0}, IsGenericMethod: {1}, IsGenericMethodDefinition: {2}, ContainsGenericParameters: {3}", methodInfo, methodInfo.IsGenericMethod, methodInfo.IsGenericMethodDefinition, methodInfo.ContainsGenericParameters);
        }
    }

    public class Foo<T>
    {
        public void M1<V>(V t)
        {
            Console.WriteLine(typeof(V));
        }

        public T Get(String key)
        {
            return default(T);
        }
    }
```

输出是：
```cs
Int32 Get(System.String), IsGenericMethod: False, IsGenericMethodDefinition: False, ContainsGenericParameters: False
Void M1[V](V), IsGenericMethod: True, IsGenericMethodDefinition: True, ContainsGenericParameters: True
Void M1[String](System.String), IsGenericMethod: True, IsGenericMethodDefinition: False, ContainsGenericParameters: False
```

1. g是封闭构造方法（一个普通方法）；
2. m是泛型方法定义；
3. m1是封闭泛型方法。