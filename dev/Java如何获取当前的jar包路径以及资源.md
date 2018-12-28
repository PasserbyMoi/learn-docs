# Java如何获取当前的jar包路径以及资源

**一：使用类路径**

```
1 String path = this.getClass().getProtectionDomain().getCodeSource().getLocation().getPath();
```

或者

```
1 String path = this.getClass().getProtectionDomain().getCodeSource().getLocation().getFile();
```

因为程序已经被打包成jar包，所以getPath()和getFile()在这里的返回值是一样的。都是/xxx/xxx.jar这种形式。如果路径包含Unicode字符，还需要将路径转码

```
path = java.net.URLDecoder.decode(path, "UTF-8");
```

**二：使用JVM**

```
String path = System.getProperty("java.class.path");
```

利用了java运行时的系统属性来得到jar文件位置，也是/xxx/xxx.jar这种形式。

------

------

 

这样，我们就获得了jar包的位置，但是这还不够，我们需要的是jar包的目录。

使用

```
1 int firstIndex = path.lastIndexOf(System.getProperty("path.separator")) + 1;
2 int lastIndex = path.lastIndexOf(File.separator) + 1;
3 path = path.substring(firstIndex, lastIndex);
```

来得到目录。

path.separator在Windows系统下得到;（分号）,在Linux下得到:(冒号)。也就是环境变量中常用来分割路径的两个符号，比如在Windows下我们经常设置环境变量PATH=xxxx\xxx;xxx\xxx;这里获得的就是这个分号。

File.separator则是/（斜杠）与\（反斜杠），Windows下是\（反斜杠），Linux下是/（斜杠）。

**如何加载jar包中的资源。**

\1. 比如说我要得到背景图片，源代码中它是

/src/UI/image/background.jpg

那么在jar包中它的路径应该是

/UI/image/background.jpg

路径最前面的/表示根目录，即绝对路径，若没有最左边的/，则表示相对路径。使用哪种方法看自己的需求，这里使用了绝对路径。

加载的时候使用

```
1 java.net.URL fileURL = this.getClass().getResource("/UI/image/background.jpg");
2 javax.swing.Image backGround = new ImageIcon(fileURL).getImage();
```

即可以获得该图片资源。

\2. 有时候，我们需要加载文本资源或音乐资源，而文本在Java中是以流对象存在的，这时我们就要使用

```
InputStream in = this.getClass().getResourceAsStream("/UI/image/background.txt");
```

加载该资源。

PS:注意这里两种方法的区别，第一种是先得到该文件的路径，再加载该文件资源。第二种则是直接加载该对象。

3.有时候，我们有一些资源类，其中的资源对象都是pulic static final修饰的，这里可以采用这样的方法初始化。

比如说我有一个ImageSource类用来加载各种图片资源，那么可以如下使用

```
1 public class ImageSource
2 {
3     static
4     {
5         URL fileURL = ImageSource.class.getResource(“/UI/image/background.jpg”);
6         BACK_GROUND = new ImageIcon(fileURL).getImage();
7     }
8     public static final Image BACK_GROUND;
9 }
```

这里不能用构造函数初始化，因为构造函数和对象相关，而static变量和对象是无关的，只和类相关。在Java的语法中，类中的static块是不依赖类对象的，因此可以初始化statc对象。同时，static块中不能使用this，这里使用了ImageSource.class代替。