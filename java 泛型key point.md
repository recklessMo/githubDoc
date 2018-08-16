### java 泛型key point

##### 参考资料

https://stackoverflow.com/questions/6783316/list-vs-listobject





#####List，List<?>， List<T>,  List<? extends Object>， List<Object> 区别

###### List和List<Object>区别？List逃避编译器检查，而且能够创建数组，属于具体类型。java 5以前使用，现在尽量不要使用，因为不安全。

```java
public void unsafeAdd(List list, Object o){
    list.add(o);
}
//如果用原生List的话
List<Integer> a = new ArrayList<Integer>();
unsafeAdd(a, "XX");//通过，类型安全消失！ 所以我们需要用泛型来保证泛型安全！
```

```java
public void unsafeAdd(List<Object> list, Object o){
    list.add(o);
}

List<Integer> a = new ArrayList<Integer>();
unsafeAdd(a, "xx");//不通过，List<Integer>并不是List<Object>的子类型
```



###### List<?>和List<? extends Object>区别

大部分情况下都是一致的，但是List<?>是具体化的。一些小区别如下，平常感觉不到

```java
List<?>[] xx = {};//可以定义数组	
List<? extends Object>[] yy = {};//泛型数组不允许

boolean b1 = (y instanceof List<?>);//可以
boolean b2 = (y instanceof List<? extends Object>);//不可以
```

 







##### 逆变和协变---核心就是java是类型安全的原则

数组是协变的，Integer是number的子类型，所以Integer[]是number[]的子类型

###### java中数组为什么要设计成协变呢？

这是对java早期没有泛型的一种妥协，有些多态方法比如sort在没有泛型的情况下，必须以object[]类型来作为参数，这样sort方法就可以适用于int[]  double[]等类型，这算是java遗留下来的一个静态类型漏洞！但是由于java的数组存在运行时类型检查，所以即使发生了协变，在往数组中插入不符合类型的元素的时候会报异常！代码如下

```java 
Number[] a = new Integer[10];
a[0] = 2.1; //error ArrayStoreException，a记住了自己的运行时类型是Integer，所以不能复制一个double类型

Number[] b = new Number[10];
Object[] c = b;
c[0] = "111";//error ArrayStoreException, 尝试通过切换引用，让b中可以存放完全不相关的类型
```



###### java中泛型不能弄成协变的，为什么呢？核心就是原则上不应该，而且泛型没有运行时类型检查

刚才说到数组的协变是早期的妥协，本身就是一种错误，只是数组有了运行时类型检查才防止了这个风险，如果泛型允许协变的话。那么就会出现类型安全的问题了，因为java静态语言重视的就是类型问题。如下代码

```java
List<Number> a = new ArrayList<Integer>();
List<Object> b = a;
b.add("123456");//不报错，这样造成的结果就是明明是装number的list，结果混进了一个string
```



###### 那么我们在java中如何处理协变的需求呢？我们有通配符

比如有针对List<Number>进行打印的方法。如下

```java
void print(List<Number> data);

List<Number> a = new ArrayList<Number>();
List<Integer> b = new ArrayList<Integer>();
List<Double> c = new ArrayList<Double>();
print(a);//ok
print(b);//not ok 
print(c);//not ok
```

我们想利用这个方法传递参数List<Integer>和List<Double>，显然是不行的。因为泛型不协变。这个时候我们利用通配符。修改上面的方法至：

```java
void print(List<? extends> data)

List<Number> a = new ArrayList<Number>();
List<Integer> b = new ArrayList<Integer>();
List<Double> c = new ArrayList<Double>();
print(a);//ok
print(b);//ok 
print(c);//ok
```



###### 那么如何理解<? extends> 和 <? supers>呢？PECS原则

用继承树的思想来理解extends和supers。

```java
List<? extends Number> a = new ArrayList<Integer>();
a.set(12);//not OK。extends只能get，因为set的话 不能确定类型安全
Number b = a.get(0);//OK 能get
```

```java
List<? supers Number> a = new ArrayList<Object>();
a.set(12);//ok
a.set(12.0);//ok

Integer x = a.get(0);//not ok 
Object x = a.get(0);//ok
```







