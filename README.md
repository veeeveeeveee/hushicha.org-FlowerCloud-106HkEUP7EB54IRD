
# 简介


![image](https://img2024.cnblogs.com/blog/1084317/202411/1084317-20241106105125524-663990694.png)


泛型参考资料烂大街，基本资料不再赘述，比如泛型接口/委托/方法的使用，逆变与协变。


泛型好处有如下几点


0. 代码重用
算法重用，只需要预先定义好算法，排序，搜索，交换，比较等。任何类型都可以用同一套逻辑
1. 类型安全
编译器保证不会将int传给string
2. 简单清晰
减少了类型转换代码
3. 性能更强
减少装箱/拆箱，泛型算法更优异。


## 为什么说泛型性能更强？


主要在于装箱带来的托管堆分配问题以及性能损失


1. 值类型装箱会额外占用内存



```
            var a = new List<int>()
            {
                1,2, 3, 4
            };
            var b = new ArrayList()
            {
                1,2,3,4
            };

```

变量a:72kb
![image](https://img2024.cnblogs.com/blog/1084317/202411/1084317-20241106165728209-927747978.png)
变量b:184kb
![image](https://img2024.cnblogs.com/blog/1084317/202411/1084317-20241106165741792-299864992.png)


2. 装箱/拆箱会消耗额外的CPU



```
	public void ArrayTest()
	{
		Stopwatch stopwatch = Stopwatch.StartNew();
		stopwatch.Start();
		ArrayList arrayList = new ArrayList();
		for (int i = 0; i < 10000000; i++)
		{
			arrayList.Add(i);
			_ = (int)arrayList[i];
		}
		stopwatch.Stop();
		Console.WriteLine($"array time is {stopwatch.ElapsedMilliseconds}");
	}

	public void ListTest()
	{
		Stopwatch stopwatch = Stopwatch.StartNew();
		stopwatch.Start();
		List<int> list = new List<int>();
		for (int i = 0; i < 10000000; i++)
		{
			list.Add(i);
			_ = list[i];
		}
		stopwatch.Stop();
		Console.WriteLine($"list time is {stopwatch.ElapsedMilliseconds}");
	}

```

![image](https://img2024.cnblogs.com/blog/1084317/202411/1084317-20241107144411581-1509823768.png)


如此巨大的差异，无疑会造成GC的管理成本增加以及额外的CPU消耗。



> 思考一个问题，如果是引用类型的实参。差距还会如此之大吗？
> 如果差距不大，那我们使用泛型的理由又是什么呢？


# 开放/封闭类型


CLR中有多种**类型对象** ,比如引用类型，值类型，接口类型和委托类型，以及泛型类型。


根据创建行为，他们又被分为开放类型/封闭类型



> 为什么要说到这个？ 泛型的一个有优点就是代码复用，只要定义好算法。剩下的只要往里填就好了。比如List\<\>开放给任意实参，大家都可以复用同一套算法。


举个例子


1. 开放类型是指类型参数尚未被指定,他们不能被实例化 List\<\>,Dictionary\<,\>,interface 。它们只是搭建好了基础框架，开放不同的实参



```
            Type it = typeof(ITest);
            Activator.CreateInstance(it);//创建失败

            Type di = typeof(Dictionary<,>);
            Activator.CreateInstance(di);//创建失败

```

2. 封闭类型是指类型已经被指定,是可以被实例化 List\>,String 就是封闭类型。它们只接受特定含义的实参



```
            Type li = typeof(List<string>);
            Activator.CreateInstance(li);//创建成功

```

# 代码爆炸


所以当我们使用开放类型时，都会面临一个问题。在JIT编译阶段，CLR会获取泛型的IL，再寻找对应的实参替换，生成合适的本机代码。
但这么做有一个缺点，要为每一种不同的泛型类型/方法组合生成，各种各种的本机代码。这将明显增加程序的Assembly，从而损害性能
CLR为了缓解该现象，做了一个特殊的优化：共享方法体


1. 相同类型实参，共用一套方法
如果一个Assembly中使用了List另外一个Assembly也使用了List
那么CLR只会生成一套本机代码。
2. 引用类型实参，共用一套方法
List与List 实参都是引用类型，它们的值都是托管堆上的指针引用。因此CLR对指针都可以用同一套方式来操作
值类型就不行了，比如int与long. 一个占用4字节，一个占用8字节。占用的内存不长不一样，导致无法用同一套逻辑来复用


## 眼见为实1



示例代码

```
    internal class Program
    {
        static void Main(string[] args)
        {
            var a = new Test<string>();
            var b = new Test();
            
            Debugger.Break();
        }
    }

    public class Test<T>
    {
        public void Add(T value)
        {
		
        }
        public void Remove(T value)
        {

        }
    }

```


变量a：
![image](https://img2024.cnblogs.com/blog/1084317/202411/1084317-20241106151912771-553369751.png)


变量b
![image](https://img2024.cnblogs.com/blog/1084317/202411/1084317-20241106151854028-602281304.png)


仔细观察发现，它们的EEClass完全一致，它们的Add/Remove方法的MethodDesc也完全一直。这印证了上面的说法，引用类型实参引用同一套方法。


## 眼见为实2



点击查看代码

```
    internal class Program
    {
        static void Main(string[] args)
        {
            var a = new Test<int>();
            var b = new Test<long>();
            var c = new Test();
            
            Debugger.Break();
        }
    }

    public class Test<T>
    {
        public void Add(T value)
        {

        }
        public void Remove(T value)
        {

        }
    }

    public struct MyStruct
    {
        public int Age;
    }

```


我们再把引用类型换为值类型，再看看它们的方法表。
变量a:
![image](https://img2024.cnblogs.com/blog/1084317/202411/1084317-20241106152629575-890210511.png)
变量b:
![image](https://img2024.cnblogs.com/blog/1084317/202411/1084317-20241106152654933-2110068571.png)
变量c:
![image](https://img2024.cnblogs.com/blog/1084317/202411/1084317-20241106152712268-2032554662.png)


一眼就能看出，它们的MethodDesc完全不一样。这说明在Assembly中。CLR为泛型生成了3套方法。



> 细心的朋友可能会发现，引用类型的实参变成了一个叫System.\_\_Canon的类型。CLR 内部使用 System.\_\_Canon 来给所有的引用类型做“占位符”使用
> 有兴趣的小伙伴可以参考它的源码：coreclr\\System.Private.CoreLib\\src\\System\_\_Canon.cs


## 为什么值类型无法共用同一套方法？


其实很好理解，引用类型的指针长度是固定的(32位4byte,64位8byte)，而值类型的长度不一样。导致值类型生成的底层汇编无法统一处理。因此值类型无法复用同一套方法。


### 眼见为实



点击查看代码

```
    internal class Program
    {
        static void Main(string[] args)
        {
            var a = new Test<int>();
            a.Add(1);
            var b = new Test<long>();
            b.Add(1);

            var c = new Test<string>();
            c.Add("");
            var d = new Test();
            d.Add(null);
            
            Debugger.Break();
        }
    }

    public class Test<T>
    {
        public void Add(T value)
        {
            var s = value;
        }
        public void Remove(T value)
        {

        }
    }

```



```
//变量a
00007FFBAF7B7435  mov         eax,dword ptr [rbp+58h]  
00007FFBAF7B7438  mov         dword ptr [rbp+2Ch],eax    //int 类型步长4 2ch

//变量b
00007FFBAF7B7FD7  mov         rax,qword ptr [rbp+58h]  
00007FFBAF7B7FDB  mov         qword ptr [rbp+28h],rax  //long 类型步长8 28h 汇编不一致

//变量c
00007FFBAF7B8087  mov         rax,qword ptr [rbp+58h]  
00007FFBAF7B808B  mov         qword ptr [rbp+28h],rax  // 28h

//变量d
00007FFBAF7B8087  mov         rax,qword ptr [rbp+58h]  
00007FFBAF7B808B  mov         qword ptr [rbp+28h],rax  // 28h 引用类型地址步长一致，汇编也一致。

```

# 泛型的数学计算


在.NET 7之前，如果我们要利用泛型进行数学运算。是无法实现的。只能通过dynamic来曲线救国


![image](https://img2024.cnblogs.com/blog/1084317/202411/1084317-20241107091815593-2034411679.png)


.NET 7中，引入了新的数学相关泛型接口，并提供了接口的默认实现。
![image](https://img2024.cnblogs.com/blog/1084317/202411/1084317-20241106200246969-1356538525.png)



> [https://learn.microsoft.com/zh\-cn/dotnet/standard/generics/math](https://github.com)


## 数学计算接口的底层实现


C\#层：
相加的操作主要靠IAdditionOperators接口。
![image](https://img2024.cnblogs.com/blog/1084317/202411/1084317-20241107092058398-1269788711.png)


IL层：
\+操作符被JIT编译成了op\_Addition抽象方法
![image](https://img2024.cnblogs.com/blog/1084317/202411/1084317-20241107135732550-1206507221.png)


对于int来说，会调用int的实现
System.Int32\.System.Numerics.IAdditionOperators
![image](https://img2024.cnblogs.com/blog/1084317/202411/1084317-20241107141219277-922247390.png)


对于long来说，会调用long的实现
System.Int64\.System.Numerics.IAdditionOperators
![image](https://img2024.cnblogs.com/blog/1084317/202411/1084317-20241107141323467-1420921333.png)


从原理上来说很简单,BCL实现了基本值类型的所有\+\-\*/操作，只要在泛型中做好约束，JIT会自动调用相应的实现。


# 结论


一路无话，无非打打杀杀。
泛型，用就完事了。就是要稍微注意(硬盘比程序员便宜多了)值类型泛型造成的代码爆炸。


 本博客参考[slower加速器](https://jisuanqi.org)。转载请注明出处！
