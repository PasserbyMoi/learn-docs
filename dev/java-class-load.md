## 类加载

### 加载时机

类从被加载到虚拟机内存中开始，到卸载出内存为止，它的整个生命周期包括：

加载（Loading）、验证（Verification）、准备（Preparation）、解析（Resolution）、初始化（Initializa-tion）、使用（Using）和卸载（Unloading）7个阶段。其中验证、准备、解析3个部分统称为连接（Linking），这7个阶段。![img](https://cdn.cnbj1.fds.api.mi-img.com/book/images/6f3a1499a83105fff7ce5898ebdf5b64?thumb=1024x&scale=auto)

> 加载、验证、准备、初始化和卸载这5个阶段的顺序是确定的，而解析阶段则不一定：它在某些情况下可以在初始化阶段之后再开始，这是为了支持Java语言的运行时绑定（也称为动态绑定或晚期绑定）。

**只有5种情况必须立即对类进行“初始化：**

- 遇到new、getstatic、putstatic或invokestatic这4条字节码指令时，如果类没有进行过初始化，则需要先触发其初始化。

  **生成这4条指令的最常见的Java代码场景是：** 

  - 使用new关键字实例化对象的时候
  - 读取或设置一个类的静态字段（被final修饰、已在编译期把结果放入常量池的静态字段除外）的时候
  - 调用一个类的静态方法的时候。

- 使用java.lang.reflect包的方法对类进行反射调用的时候，如果类没有进行过初始化，则需要先触发其初始化。

- 当初始化一个类的时候，如果发现其父类还没有进行过初始化，则需要先触发其父类的初始化。

- 当虚拟机启动时，用户需要指定一个要执行的主类（包含main()方法的那个类），虚拟机会先初始化这个主类。

- 当使用JDK 1.7的动态语言支持时，如果一个java.lang.invoke.MethodHandle实例最后的解析结果REF_get-Static、REF_putStatic、REF_invokeStatic的方法句柄，并且这个方法句柄所对应的类没有进行过初始化，则需要先触发其初始化。

> 这5种场景中的行为称为对一个类进行主动引用。除此之外，所有引用类的方式都不会触发初始化，称为被动引用。

**被动引用的例子：**

- 通过子类引用父类的静态字段，不会导致子类初始化
```c
  /*** 被动使用类字段演示一：通过子类引用父类的静态字段，不会导致子类初始化 **/
  public class SuperClass {    
      static {        
          System.out.println("SuperClass init!");    
      }    
      public static int value = 123;
  }
  public class SubClass extends SuperClass {    
      static {        
          System.out.println("SubClass init!");    
      }
  }
  /*** 非主动使用类字段演示 **/
  public class NotInitialization {    
      public static void main(String[] args) {
          System.out.println(SubClass.value);
      }
  }

// 输出：对于静态字段，只有直接定义这个字段的类才会被初始化。至于是否要触发子类的加载和验证，在虚拟机规范中并未明确规定，这点取决于虚拟机的具体实现。对于Sun HotSpot虚拟机来说，可通过-XX:+TraceClassLoading参数观察到此操作会导致子类的加载。
SuperClass init！
```
- 通过数组定义来引用类，不会触发此类的初始化
```c
  /*** 被动使用类字段演示二：* 通过数组定义来引用类，不会触发此类的初始化**/
  public class NotInitialization { 
      public static void main(String[] args) {        
          SuperClass[] sca = new SuperClass[10];    
      }
  }

// 无输出，虚拟机会在初始化阶段，虚拟机会自动生成直接继承于java.lang.Object的子类"[SuperClass"，创建动作由字节码指令newarray触发。这个类代表了一个元素类型为SuperClass的数组，封装了数组元素的属性和方法
```
- 常量在编译阶段会存入调用类的常量池中，本质上并没有直接引用到定义常量的类，因此不会触发定义常量的类的初始化。

```c
/*** 被动使用类字段演示三：
* 常量在编译阶段会存入调用类的常量池中，本质上并没有直接引用到定义常量的类，因此不会触发定义常量的类的初始化。**/
public class ConstClass {    
    static {        
        System.out.println("ConstClass init!");    
    }    
    public static final String HELLOWORLD = "hello world";
}
/*** 非主动使用类字段演示**/
public class NotInitialization {    
    public static void main(String[] args) {       
        System.out.println(ConstClass.HELLOWORLD);    
    }
}

// 无输出，编译阶段通过常量传播优化，将常量的值“hello world”存储到了NotInitialization类的常量池中，以后NotInitialization对常量ConstClass.HELLOWORLD的引用实际都被转化为NotInitialization类对自身常量池的引用。
```

### 类加载过程







**接口的加载过程**：

接口也有初始化过程，这点与类是一致的，编译器会为接口生成 “<clinit>()” 类构造器，用于初始化接口中所定义的成员变量。

接口与类真正有所区别的是前面讲述的5种“有且仅有”需要开始初始化场景中的第3种：当一个类在初始化时，要求其父类全部都已经初始化过了，但是一个接口在初始化时，并不要求其父接口全部都完成了初始化，只有在真正使用到父接口的时候（如引用接口中定义的常量）才会初始化。