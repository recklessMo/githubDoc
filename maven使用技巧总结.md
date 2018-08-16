### maven使用技巧总结

参考资料：

<http://www.infoq.com/cn/news/2011/01/xxb-maven-3-pom-refactoring>

https://www.jianshu.com/p/49acf1246eff

https://blog.csdn.net/ruanjian1111ban/article/details/80890539

通常我们使用的比较多的结构都是父子结构的工程。下面是列举出来的一些最佳实践：

#####利用properties来消除版本的重复：

比如依赖的spring相关的项目有很多，一般版本都是同一个，就可以用properties来消除重复，这样修改起来也方便。

```xml
<properties>
  <spring.version>2.5</spring.version>
</properties>
<depencencies>
  <dependency>
    <groupId>org.springframework</groupId>
    <artifactid>spring-context</artifactId>
    <version>${spring.version}</version>
  </dependency>
  <dependency>
    <groupId>org.springframework</groupId>
    <artifactid>spring-core</artifactId>
    <version>${spring.version}</version>
  </dependency>
</depencencies>
```



#####消除多模块依赖配置重复

父子结构的maven工程中，父pom文件里面的dependencyManagement的作用是为了解决整个项目里面的依赖问题，避免出现依赖的版本不一致的情况，子项目的pom里面可以不用写具体的版本，直接引用依赖，子pom会根据继承关系向上寻找第一个遇到dependencyManagement里面的依赖版本。

```xml
<dependencyManagement>
  <dependencies>
    <dependency>
      <groupId>junit</groupId>
      <artifactid>junit</artifactId>
      <version>4.8.2</version>
      <scope>test</scope>
    </dependency>
    <dependency>
      <groupId>log4j</groupId>
      <artifactid>log4j</artifactId>
      <version>1.2.16</version>
    </dependency>
  </dependencies>
</dependencyManagement>
```

```xml
<dependency>
    <groupId>junit</groupId>
    <artifactid>junit</artifactId>
</dependency>
<dependency>
    <groupId>log4j</groupId>
    <artifactid>log4j</artifactId>
</dependency>
```



##### 消除多模块插件依赖重复

类似于管理依赖，我们可以使用pluginManagement元素来管理插件。常见的场景是我们希望项目所有的模块的maven compile plugin都使用java1.8，以及指定源文件的编码为UTF8，于是我们可以在父pom中配置pluginManagement，这个配置会被应用到所有的子模块的maven-compile-plugin之中，由于maven内置了maven-compile-plugin与生命周期的绑定，因此子模块就不再需要任何配置了。但是某些包特有的插件还是需要独立配置，比如maven-war-plugin用来打web包。

```xml
<pluginManagement>
    <plugins>
        <plugin>
            <groupId>org.apache.maven.plugins</groupId>
            <artifactId>maven-source-plugin</artifactId>
            <version>2.1</version>
            <configuration>
                <attach>true</attach>
            </configuration>
            <executions>
                <execution>
                    <phase>compile</phase>
                    <goals>
                        <goal>jar</goal>
                    </goals>
                </execution>
            </executions>
        </plugin>
    </plugins>
</pluginManagement>
```

```xml
<plugins>
    <plugin>
        <groupId>org.apache.maven.plugins</groupId>
        <artifactId>maven-source-plugin</artifactId>
    </plugin>
</plugins>
```



##### maven多module项目的引用

多module项目一般都有一个父亲工程，父亲工程通过dependencyManagement来管理依赖，我们可以在父亲pom中通过添加子模块的依赖。然后在子模块的pom中指定依赖就可以。但是这样可能造成一个问题，就是如果我想引用整个项目的时候，直接指定父亲依赖会下不到子依赖的包，因为在父亲依赖中是用dependencyManagement来进行依赖配置的。导致找不到依赖。这个时候可以选在父亲模块中再建立一个pom专门用于子类的引用。