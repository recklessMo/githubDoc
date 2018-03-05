### java中的协变，逆变和不变

先说一下逆变协变和不变的概念。

如果A和B是类型，f是类型变化函数，<=代表子类型关系(如果 A<=B 代表A是B的子类型)。

1. f是协变的：如果A<=B的时候有f(A) <= f(B)
2. f是逆变的：如果A<=B的时候有f(B) <= f(A)
3. f是不变的：如果A<=B的时候，1和2都不满足

下面举几个例子来说明：

###### 泛型

```java
f(A) = List<A>, class List<T> {...}

A = String;
B = Object;
f(A) = List<String>
f(B) = List<Object>

ArrayList<String> stringList = new ArrayList<Object>();//编译不过
ArrayList<Object> objectList = new ArrayList<String>();//编译不过
```

A如果是String，B如果是Object。那么f(A)是List<String>， f(B)是List<Object>。由于List<String>和List<Object>没有父子关系，所以我们说泛型List<T>是不变的。



###### 数组

```java
f(A) = A[]

A = String;
B = Object;
f(A) = A[];
f(B) = B[];

Object[] objects = new String[1];//编译通过
```

A如果是String，B如果是Object。那么f(A)是A[]，f(B)是B[]。由于String[] <= Object[]。所以数组是协变的！



###### 方法调用

returnType = method(parameterType)

上面这个表达式有两个要求

1. 传递给method的参数必须是parameterType的子类
2. method的返回类型必须是returnType的子类。

比如，举例子：

```java
Number[] method(ArrayList<Number> list){...}

Integer[] result = method(new ArrayList<Integer>());//编译不通过
Number[] result = method(new ArrayList<Integer>());//编译不通过
Object[] result = method(new ArrayList<Object>());//编译不通过

Number[] result = method(new ArrayList<Number>());//ok
Object[] result = method(new ArrayList<Number>());//ok
```



###### 子类重写

如下的例子

```java
Super sup = new Sub();
Number n = sup.method(1);
```

Super和Sub的定义如下

```java
class Super{
  Number method(Number n){...}
}

class Sub extends Super{
    @Override
  	Number method(Number n);
}
```

对于重写，也就是java的多态，可以按照下面的方式来理解

```java
class Super{
  Number method(Number n){
    if(this instanceof Sub){
      return ((Sub)this).method(n);//1
    }else{
      ...
    }
  }
}
```

可以看到，通过Super引用来调用method方法的时候。实际上会调用到子类的重写方法上。要想在1处编译通过，那么子类的重写方法的参数类型必须是Number的父类型，返回的类型必须是Number的子类型。如下：

```java
class Super{
  A method(B b){...}
}

class Sub extends Super{
  subof(A) method(superof(B));
}
```

注意在java中，java不支持子类方法参数类型的逆变，只支持返回类型的协变！



https://stackoverflow.com/questions/8481301/covariance-invariance-and-contravariance-explained-in-plain-english

https://stackoverflow.com/questions/2501023/demonstrate-covariance-and-contravariance-in-java