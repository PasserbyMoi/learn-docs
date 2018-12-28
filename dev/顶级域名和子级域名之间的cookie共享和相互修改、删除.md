2016-09-18 发布

# 顶级域名和子级域名之间的cookie共享和相互修改、删除

最近项目中刚好涉及到了主域名和子域名之间的共享和相互修改、删除，也就借此机会总结一下常用的几个场景，这里代码以`PHP`为例来说明，域名的话就拿顶级域名和二级域名为例，其他的场景都是类似哈！

# 设置COOKIE

## 顶级域名

顶级域名只能设置`domain`为顶级域名，不能设置为二级域名或者三级域名等等，否则`cookie`无法生成。

> 如`yangbai.com`能设置`domain`为`yangbai.com`或者`www.yangbai.com`，但不能设置`domain`为`login.yangbai.com`，这样`cookie`不会生成。

以下面的代码为例：

```
setcookie("name1", "yangbai", time() + 1000);//yangbai.com自己可以看到
setcookie("name2", "yangbai", time() + 1000, "/", "www.yangbai.com");//*.www.yangbai.com都可以看到
setcookie("name3", "yangbai", time() + 1000, "/", "yangbai.com");//*.yangbai.com都可以看到
setcookie("name4", "yangbai", time() + 1000, "/", "login.youzan.com");//设置无效
```

> 设置`domain`的时候，`.yangbai.com`和`yangbai.com`是一样的。
> 未指定`domain`时，默认的`domain`为用哪个域名访问就是哪个。

执行后，`www.yangbai.com`在浏览器的`cookie`情况如下图：

![图片描述](https://segmentfault.com/img/bVDeNV?w=2413&h=238)

二级域名`login.yangbai.com`和`game.yangbai.com`浏览器的`cookie`情况如下图：

![clipboard.png](https://segmentfault.com/img/bVDeOc?w=1208&h=70)

总的来说，顶级域名设置的`cookie`可以共享【需要指定`domain`主域名的`host`】给二级域名，也可以自己私有【不指定`domain`】。

## 二级域名

拿`game.yangbai.com`为例，代码如下：

```
setcookie("game", "yangbai");//只有自己可以看到
setcookie("game1", "yangbai", time() + 1000, "/", "yangbai.com");//*.yangbai.com都可以看到
setcookie("game2", "yangbai", time() + 1000, "/", "chip.game.yangbai.com");//设置无效
```

执行后，`game.yangbai.com`在浏览器的`cookie`情况如下图：

![图片描述](https://segmentfault.com/img/bVDfC3?w=2411&h=170)

> 总的来说，设置`cookie`的话只能在本域名下或者`domain`级别高于自身的域名下才会生效！

# 读取COOKIE

有了上面的例子和实践，大概的规则如下：

- 二级域名能读取设置了`domain`为顶级域名或者自身的`cookie`，不能读取其他二级域名`domain`的`cookie`。例如：要想`cookie`在多个二级域名中共享，需要设置`domain`为顶级域名，这样就可以在所有二级域名里面或者到这个`cookie`的值了。
- 顶级域名只能获取到`domain`设置为顶级域名的`cookie`，`domain`设置为其他子级域名的无法获取。

# 修改COOKIE

## 顶级域名

顶级域名的`cookie`在顶级域名或者非顶级域名【需要设置domain为顶级域名才可以】都可以修改。代码如下：

```
#为所有二级域名设置一个cookie
setcookie("name", "yangbai", time() + 1000, "/", "yangbai.com");

#在game.yangbai.com下面修改这个cookie值
setcookie("name", "yangbai11", time() + 1000, "/", "yangbai.com");
```

## 二级域名

修改二级域名自身生成的`cookie`不需要设置`domain`，直接设置即可。代码如下：

```
#修改game.yangbai.com下面自身的cookie值
setcookie("game", "chip", time() + 10000);
```

# 删除COOKIE

删除cookie理解为是修改cookie的一种特殊场景，只需将`expire`设置为过期、值设置为`null`即可，代码如下：

```
#删除yangbai.com下面的cookie值
setcookie("name", null, time() - 1000, "/", "yangbai.com");

#删除game.yangbai.com下面自身的cookie值
setcookie("game", null, time() - 1000);
```