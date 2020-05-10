
<br />
<br />
<br />

<a name="atHd5"></a>
### 类的整个生命周期过程

- 加载
- 连接
  - 校验
  - 准备
  - 解析
- 初始化
- 使用
- 卸载

![image.png](linkage.png)

<br />
<br />

<a name="E8x6j"></a>
### 类创建过程主要分两步骤

- 类的加载过程, 本质上是在方法区生成类的Class对象，共其他地方引用使用
- 类的实例化



<a name="l282i"></a>
#### 次序

- 次序是: 先自己的变量->父类->初始化
- 接口跟类不一样，接口只有在使用到的时候才会去初始化，而类即使没有使用到也会在它的子类初始化的时候被初始化出来



<a name="k5GKh"></a>
#### 类加载过程
加载过程主要是对class的加载，不包含实例的初始化过程，所以操作的内存区域是方法区，这个过程主要包括：

- 加载，将字节码文件加载到内存中来(比如从jar包中加载，从URL中加载)
- 连接(link)
  - 校验(verify): 校验字节码是否符合规范，比如
    - 是否以魔数0xCAFEBABE开头
    - major/minor version是否是当前JVM支持的
  - 准备(prepare)，该阶段主要是给class的初始化做内存准备(静态变量)，比如
    - 为类变量(非实例变量)分配内存
    - 为类变量赋予零值，比如int会给个0
    - 对于final的类变量，这个阶段会直接给变量赋值而不是给初始值
  - 解析(resolve)，该阶段主要是解析该类引用的其他类，并将它们加载进来
- 初始化
  - 对静态变量进行初始化
  - 这个阶段会初始化静态变量和调用静态代码块
  - 他们之间谁先谁后，取决于他们在源码中的先后顺序


<br />

<a name="izS7T"></a>
#### 实例的初始化过程

- 为实例变量分配内存，主要是堆中
- 给实例变量分配初始值
- 实例化父类
- 给实例变量赋予源码中的值, 调用实例块 (他们的先后顺序取决于他们在源码中的位置)
- 执行构造方法中剩余的代码


<br />

<a name="Yqlqk"></a>
### 实例创建的整个过程
![类的初始化guocheng.svg](process.svg)<br />
<br />
<br />

<a name="GYm9T"></a>
### 一些注意事项
<a name="R8DyH"></a>
#### 1. 被动引用不会触发初始化
被动加载是相对主动加载来说的，主动加载是Java虚拟机规范中定义的几种类型:

- 通过new/getstatic/putstatic/invokestatic指令触发的初始化
- 通过反射调用类的方法触发的初始化
- 在加载子类的时候，发现父类没有加载而触发的父类加载
- 调用类的main方法导致该类的初始化



<a name="iGhLB"></a>
##### 1.1 通过子类引用父类的静态变量，不会导致子类的初始化
```java
public class ParentClass {
    static {
        System.out.println("Parent Class static block");
    }
    public static int value = 13;
}

public class SubClass extends ParentClass {
    static {
        System.out.println("SubClass static block");
    }   
}

public class NotInitializingTest {
    public static void main(String[] args) {
		System.out.println(SubClass.value);
    }
}
```

<br />以上代码只会初始化ParentClass，不会初始化SubClass. 输出的结果:<br />

```
Parent Class static block
13
```

<br />
<br />

<a name="X2MeK"></a>
##### 1.2 通过数组定义来引用类，不会触发类的初始化
数组的生成其实并没有生成数组元素的类，生成的是数组类, 比如 "[java.lang.Integer" 这样的类.<br />

```java
public class ParentClass {
    static {
        System.out.println("Parent Class static block");
    }
    public static int value = 13;
}

public class SubClass extends ParentClass {
    static {
        System.out.println("SubClass static block");
    }   
}

public class NotInitializingTest {
    public static void main(String[] args) {
		SubClass[] subClasses = new SubClass[1];
    }
}
```

<br />运行之后发现，什么也没有输出，说明`SubClass[] subClasses = new SubClass[1];`跟 SubClass没有关系<br />

<a name="tcfKs"></a>
##### 1.3 引用常量不会触发定义常量的类的初始化
常量是只那些 final static的变量，这些变量会在准备阶段就初始化好，然后就会进入常量池里，之后其实就跟它所在的类没什么关系了. 后面其他类引用它的时候其实是从常量池中获取的.<br />

```java
public class ConstantClass {

    static {
        System.out.println("constant class init");
    }

    public static final String DEFAULT_NAME = "kobe";
}


public class NotInitializingTest {
    public static void main(String[] args) {
		System.out.println(ConstantClass.DEFAULT_NAME);
    }
}
```


<a name="sA9wT"></a>
### 2. 父子类中override方法的执行
当子类override了父类的方法的时候，在构造子类的实例阶段，父类调用的是子类的方法<br />

```java
public class ParentClass {
    public ParentClass(){
        printThree();
    }
    public void printThree(){
        System.out.println("three");
    }
}

public class SubClass extends ParentClass {
    public  int three = 3;
    public SubClass(){
        System.out.println("SubClass Constructor");
    }
    @Override
    public void printThree(){
        System.out.println(three);
    }
}

public class InitializingTest {

    public static void main(String[] args) {
        SubClass subClass = new SubClass();
        subClass.printThree();
    }
}
```

<br />输出的结果:
```
0
SubClass Constructor
3
```
这是因为在父类构造的时候，调用的是子类的printThree()，而此时three=0<br />
<br />

<a name="jVwYo"></a>
#### 一个完整的实例构造例子


```java
public class ParentClass {

    static {
        System.out.println("Parent Class static block");
    }
    public static int value = 13;


    {
        System.out.println("Parent Class instance block");
    }

    public  int k = 20;

    public ParentClass(){
        printThree();
    }
    public void printThree(){
        System.out.println("three");
    }
}


public class SubClass extends ParentClass {
    public static String staticBefore = staticP("before");

    static {
        System.out.println("SubClass static block");
    }

    public static String staticAfter = staticP("after");

    public String instanceBefore = initK("before");
    {
        System.out.println("SubClass instance block");
    }

    public String instanceAfter = initK("after");
    public int three = 3;
    public static String staticP(String i) {
        System.out.println("static param: " + i);
        return i;
    }

    SubClass() {
        System.out.println("SubClass Constructor");
    }

    public String initK(String j) {
        System.out.println("instance param:" + j);
        return j;
    }


    @Override
    public void printThree() {
        System.out.println(three);
    }


}

public class InitializingTest {

    public static void main(String[] args) {

        SubClass subClass = new SubClass();

        subClass.printThree();
    }
}
```

<br />输出的结果<br />

```
Parent Class static block
static param: before
SubClass static block
static param: after
Parent Class instance block
0
instance param:before
SubClass instance block
instance param:after
SubClass Constructor
3
```
