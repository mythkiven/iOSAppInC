
用纯 C 实现了构建一个简单的 iOS App。

主要包含3个文件：
```
main.c：实现oc中main.m的功能，设置自动释放池，调用UIApplicationMain函数；
MKAppDelegate.c: 实现oc中AppDelegate.m的功能，设置window,controller;
MKView.c: 类似oc中一个纯代码的view，绘制简单的视图。
```
实现的效果如下图：



其中需要注意的几点简单说下：

### 1、直接用 C 写的函数，在 main 中报 SIGABRT 错误。

```
是由于我们写的类，并没有被 dyld 注册到 table 中，所以到了 main 这里，就找不到 AppDelegate。
用 __attribute__((constructor)) 修饰以保证函数在 main 之前执行。
```

###  2、@selector 在纯 C 环境用不了怎么办？

```
实例：
[tableView cellForRowAtIndexPath:indexPath];

在 OC 环境中写法:
objc_msgSend(tableView, @selector(cellForRowAtIndexPath:), indexPath);

在纯 C 环境中写法：
objc_msgSend(tableView, sel_getUid("cellForRowAtIndexPath:"), indexPath);
```
参考： https://developer.apple.com/documentation/objectivec/1418625-sel_getuid?language=objc


### 3、main 中的 @autoreleasepool 如何处理?

```
实际中用过 NSAutoreleasePool 的应该了解， @autoreleasepool 其实是语法糖，在编译时会展开为: NSAutoreleasePool，并调用 drain。

所以在 C 中，调用亦如此：
id autoreleasePool = objc_msgSend(objc_msgSend(objc_getClass("NSAutoreleasePool"), sel_registerName("alloc")), sel_registerName("init"));
objc_msgSend(autoreleasePool, sel_registerName("drain"));
```

https://developer.apple.com/documentation/foundation/nsautoreleasepool/1520553-drain


### 4、如何获取屏幕的 size ?
```

这里也是我花时间相对多一些的地方，尝试直接调用 objc_msgSend ，返回的 id 用不了的。

翻文档发现丰富的接口，可以用 objc_msgSend_stret，来获取 react 值：
objc_msgSend
#Sends a message with a simple return value to an instance of a class.
objc_msgSend_fpret
#Sends a message with a floating-point return value to an instance of a class.
objc_msgSendSuper
#Sends a message with a simple return value to the superclass of an instance of a class.
objc_msgSendSuper_stret
#Sends a message with a data-structure return value to the superclass of an instance of a class.


但。。。还是不行: objc_msgSendSuper_stret 返回值是 void。。。还好已有大神遇到过这个问题，并给出了答案：

UIView *view = [[UIView alloc] initWithFrame:CGRectZero];
CGRect (*sendRectFn)(id receiver, SEL operation);
sendRectFn = (CGRect(*)(id, SEL))objc_msgSend_stret;
CGRect frame = sendRectFn(view, @selector(frame));

转为我们所需的C代码就是：
id const screen = objc_msgSend((id)objc_getClass("UIScreen"), sel_getUid("mainScreen"));
CGRect (*sendRectFn)(id receiver, SEL operation);
sendRectFn = (CGRect(*)(id, SEL))objc_msgSend_stret;
CGRect screenBounds = sendRectFn(screen, sel_getUid("bounds"));

```

参考：https://developer.apple.com/documentation/objectivec/1456730-objc_msgsend_stret?language=objc

参考：http://blog.lazerwalker.com/objective-c,/code/2013/10/12/the-objective-c-runtime-and-objc-msgsend-stret.html

 
### 测试环境:

```
Xcode: Version 10.3 (10G8)
SDK: iOS 12.4
模拟器：iPhone X
```


其他没啥特别的了，绘图用 CGContextRef，objc_msgSend应该都比较熟悉，代码本身也比较简单。。

注解了一部分代码，不太熟悉消息机制调用的同学可以对照着看下。
