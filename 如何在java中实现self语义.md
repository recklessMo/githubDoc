### 如何在java中实现self语义

#### 问题引出

在Java中，当我们需要实现builder模式或者别的流式api的时候（流式api指的是连续调用），方法需要返回this，这个this按照语义返回的是指包含方法定义的类对象！那么问题来了当我们继承一个类并且调用这个类的方法，但是这个方法返回的是一个父类的引用，我们没有办法通过这个引用来调用子类的方法，只能调用父类的方法，这在实际应用中有很大的局限性，因为我们实际上需要的是返回子类的引用。举例说明如下：



#### 问题例子

比如object的clone方法，我们如果有个子类person

```java
class Person{
  @Override
  public Person clone(){
    return (Person)super.clone();
  }
}
```

我们如果想重载clone方法，super.clone实际返回的是Person对象，但是返回的引用是object引用，所以我们需要强制类型转换一下转成Person。如果能够改写object的clone方法，让其返回一个实际调用clone方法的类型引用Person，那就不用进行强转了！



再比如在经典的builder模式里面：

```java
class PersonBuilder{
  
    private String name;
    
    public PersonBuilder withName(String name){
        this.name = name;
        return this;
    }
    
    public Person build(){
        return new Person(name);
    }

    public String getName() {
        return name;
    }

    public void setName(String name) {
        this.name = name;
    }
}
```

我们可以这么来使用

```java
Person person = new PersonBuilder().withName("rock").build();
```

现在如果我们有了新的需求，Employee extends Person，它有了一个新的属性代表职位级别，如下

```java
public class Employee extends Person{
  private int level;//职位级别
}
```

如果我们同样采用builder模式，新建一个EmployeeBuilder如下，

```java
class EmployeeBuilder extends PersonBuilder{

    private int level;

    public EmployeeBuilder withLevel(int level) {
        this.level = level;
        return this;
    }

    public Employee build(){
        return new Employee(getName(), level);
    }

    public int getLevel() {
        return level;
    }

    public void setLevel(int level) {
        this.level = level;
    }
}

```

这样使用的时候麻烦来了

```java
new EmployeeBuilder().withName("rock")
    .withLevel(1); //compiler error
```

withName是父类PersonBuilder里面的方法，返回的this，从静态类型上看是PersonBuilder对象，所以没有withLevel方法，无法调用。



#### 问题解决方案

核心问题在于java里面父类的方法在返回this的时候，this对应的类型是父类的类型，并不是实际运行的类型。所以我们可以通过java里面的泛型来模拟一下！



```java
public class Object<T>{
  protected T clone();
}

public class Person extends Object<Person>{
    
  public Person clone(){
      return super.clone();
  }
}
```

没有强转了。通过泛型我们提供给父类一个返回的类型，这样就解决了问题，父类知道返回什么样的类型了！



再看上面的builder模式~

```java
class PersonBuilder<T extends PersonBuilder<T>>{
  
    private String name;
    
    public T withName(String name){
        this.name = name;
        return (T)this;
    }
    
    public Person build(){
        return new Person(name);
    }

    public String getName() {
        return name;
    }

    public void setName(String name) {
        this.name = name;
    }
}
```

```java
class EmployeeBuilder extends PersonBuilder<EmployeeBuilder>{

    private int level;

    public EmployeeBuilder withLevel(int level) {
        this.level = level;
        return this;
    }

    public Employee build(){
        return new Employee(getName(), level);
    }

    public int getLevel() {
        return level;
    }

    public void setLevel(int level) {
        this.level = level;
    }
}
```

这样就可以正常使用了。

```java
Employee employee = new EmployeeBuilder().withName("xx").withLevel(12).build();
```



参考资料：https://www.sitepoint.com/self-types-with-javas-generics/

### over. Netty中的bootstrap也使用了这种模式



