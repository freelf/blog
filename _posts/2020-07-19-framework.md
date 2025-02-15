---
layout: post
title: 初识iOS中的动态库和静态库
date: 2020-07-19
summary: 由于最近研究组件化后调试时二进制映射源码的功能，发现需要对开发中的动态库和静态库需要有一些了解。所以就有了这篇文章，由于只是了解，并没有深入到编译层面，所以本篇文章只是简单了解一些库的知识，并不深入。
category: Study
---

由于最近研究组件化后调试时二进制映射源码的功能，发现需要对开发中的动态库和静态库需要有一些了解。所以就有了这篇文章，由于只是了解，并没有深入到编译层面，所以本篇文章只是简单了解一些库的知识，并不深入。
## 静态库 vs 动态库
平常开发的时候会接触一些库的文件，比如`.a`、`.tbd`、`.framework`结尾的文件，甚至以前还有`.dylib`结尾的。由于以前一直没忙于工作，所以也没有细致的研究这些文件结尾的区别。只是傻傻的以为`.a`结尾的是静态库，`.framework`结尾的是动态库。现在看来以前是真的傻。现在来看一下静态库和动态库的区别。
### 静态库
如果我们的app依赖一个静态库，在链接阶段会把依赖静态库的部分合并到app的**可执行文件中**(app的**可执行文件**指的是app包内容里面同名的一个文件)。静态库的文件名后缀是`.a`或`.framework`。
### 动态库
编译app时并不会把动态库合并到app的**可执行文件中**，而是在程序启动的时候加载动态库到内存中。其中动态库分**动态链接库**和**动态加载库**两种。
* **动态链接库**：在编译阶段需要指定app需要依赖哪些动态库。当运行可执行文件时，如果系统还没有加载过这些库时，就会随着运行可执行文件的加载一起加载这些动态库。如果有多个可执行文件依赖同一个动态库，不会重复加载。
* **动态加载库**：编译阶段不需要指定app需要依赖哪些动态库。当运行过程中需要加载某个动态库时，就会用`dlopen`函数动态的把库加载到内存中使用。

所以这两个的区别就是**动态链接库**在app运行时就需要加载，**动态加载库**可以在任何时间进行加载。动态库的文件后缀是`.tbd`或`.framework`。

### 小总结
由于静态库在链接时需要链接到可执行文件中，所以会导致包体积变大。动态库因为不会添加到可执行文件中，所以相比静态库包体积会小一些。

相对于启动时间，静态库会比动态库好一些，因为如果使用很多动态库，app在冷启动时加载动态库会占用一些时间。

下面来说一下文件后缀的问题。`.a`就是静态库，`.tbd`是动态库，以前还有`.dylib`结尾的动态库。`.framework`既可以是动态库也可以是静态库，这取决于制作静态库时我们选择的Mach-O type。现在无论是静态库还是动态库都推荐使用`.framework`。接下来会展示如何制作自己的`.framework`类型的动态库和静态库。

## 如何制作使用动态库和动态库

### 制作.a静态库
![-w728](https://nightwish.oss-cn-beijing.aliyuncs.com/2020/07/19/15951446720584.jpg)
创建工程的时候选择`Static Library`选项。然后随便起个名字就好。创建完工程应该是下面这样的：
![-w270](https://nightwish.oss-cn-beijing.aliyuncs.com/2020/07/19/15951447308681.jpg)
然后写一个测试代码：

```objc
// CreateStatic.h
#import <Foundation/Foundation.h>

@interface CreateStatic : NSObject
- (void)test;
@end

// CreateStatic.m
#import "CreateStatic.h"

@implementation CreateStatic
- (void)test {
	NSLog(@"test static .a");
}
@end
```
然后对工程文件做一些配置：
![-w685](https://nightwish.oss-cn-beijing.aliyuncs.com/2020/07/19/15951450397710.jpg)
设置`Build Active Architecture Only`为`NO`，当设置为`YES`时，只会针对当前架构进行编译，当设置为`NO`时，会对所有架构进行编译。接下来编译一下就可以在Product文件夹下生成一个`.a`的静态库。
测试代码如下：

```objc
// CreateFrameworkStatic.h
#import <Foundation/Foundation.h>

NS_ASSUME_NONNULL_BEGIN

@interface CreateFrameworkStatic : NSObject
- (void)test;
@end

NS_ASSUME_NONNULL_END

// CreateFrameworkStatic.m
#import "CreateFrameworkStatic.h"

@implementation CreateFrameworkStatic
- (void)test {
	NSLog(@"test create framework static library");
}
@end

```
![-w409](https://nightwish.oss-cn-beijing.aliyuncs.com/2020/07/19/15951454438863.jpg)
通过设置是否为Debug状态和是否为真机，会生成4个`.a`文件。可以用`lipo -info`来查看`.a`文件信息。
![-w658](https://nightwish.oss-cn-beijing.aliyuncs.com/2020/07/19/15951461314997.jpg)

接下来可以通过`lipo -crate`命令把这几个静态库合并成一个：
![-w1319](https://nightwish.oss-cn-beijing.aliyuncs.com/2020/07/19/15951463781916.jpg)
可以看到合并后的`.a`文件既有真机又有模拟器的架构。

接下来创建一个工程，测试一下这个静态库：
![-w1665](https://nightwish.oss-cn-beijing.aliyuncs.com/2020/07/19/15951465074339.jpg)
可以看到静态库可以用了。

### 制作Framework静态库
![-w730](https://nightwish.oss-cn-beijing.aliyuncs.com/2020/07/19/15951468803384.jpg)

制作Framework静态库选择这个选项，创建工程。然后需要修改Build Setting选项，除了调整`Build Active Architecture Only`为`NO`，还需要调整`Mach-O Type`为`Static Library`。
![-w673](https://nightwish.oss-cn-beijing.aliyuncs.com/2020/07/19/15951469659964.jpg)
然后调整编译选项，会和原来创建`.a`静态库一样，生成四个Framework。这里偷个懒，只编译Debug+真机和Debug+模拟器选项。生成的文件目录如下：
![-w612](https://nightwish.oss-cn-beijing.aliyuncs.com/2020/07/19/15951471568877.jpg)
其中红色框框的这个文件就是二进制文件，同样可以使用`lipo -info`和`lipo -create`查看和合并库文件。
![-w1827](https://nightwish.oss-cn-beijing.aliyuncs.com/2020/07/19/15951472948918.jpg)

接下来测试一下Framework类型的静态库使用，当加到工程后，如果报`framework not found`的错误，需要调整`Build Setting`中`Library Search Path`选项：
![-w1659](https://nightwish.oss-cn-beijing.aliyuncs.com/2020/07/19/15951478283490.jpg)

### 制作Framework动态库
制作Framework动态库和Framework静态库一样，创建Framework类型的工程，然后`Mach-O Type`选择`Dynamic Library`。
测试代码如下：

```objc
// CreateFrameworkDynamicLibrary.h
#import <Foundation/Foundation.h>

NS_ASSUME_NONNULL_BEGIN

@interface CreateFrameworkDynamicLibrary : NSObject
- (void)test;
@end

NS_ASSUME_NONNULL_END

// CreateFrameworkDynamicLibrary.m
#import "CreateFrameworkDynamicLibrary.h"

@implementation CreateFrameworkDynamicLibrary
- (void)test {
	NSLog(@"test framework dynamic library");
}
@end
```

这里也只编译Debug+真机和Debug+模拟器两种：
![-w617](https://nightwish.oss-cn-beijing.aliyuncs.com/2020/07/19/15951484373137.jpg)
其中上图红色框框就是动态库二进制文件，同样可以使用`lipo -info`和`lipo -create`查看和合并库文件。
![-w1917](https://nightwish.oss-cn-beijing.aliyuncs.com/2020/07/19/15951486086859.jpg)
接下来测试一下动态库使用，同样直接拖到项目里面就可以：
![-w1657](https://nightwish.oss-cn-beijing.aliyuncs.com/2020/07/19/15951489285151.jpg)

记得在`Frameworks, Libraries, and Embedded Content`中选择`Embed & Sign`这个选项，否则会报`dyld: Library not loaded`的错误。

我们上面添加动态库的方式会在程序启动的时候加载，如果项目在编译阶段没有直接使用这个动态库，可以设置在运行时加载动态库。需要把`Build Phases`里面`Link Binary With Libraries`里面的动态库删除，只保留`Embed Frameworks`。
![-w1386](https://nightwish.oss-cn-beijing.aliyuncs.com/2020/07/19/15951548051322.jpg)

这样程序启动的时候就不会加载这个动态库，然后我们在页面里面添加一个按钮，点击按钮时加载动态库并执行动态库的方法：

```objc
- (IBAction)testDynamicLibrary:(id)sender {
	NSString *path = [NSString stringWithFormat:@"%@/%@", [NSBundle mainBundle].privateFrameworksPath, @"CreateFrameworkDynamicLibrary.framework"];
	[self loadDynamicLibraryWithPath:path];
	Class cl = NSClassFromString(@"CreateFrameworkDynamicLibrary");
	if (!cl) {
		NSLog(@"class not found");
	}
	[[cl new] performSelector:@selector(test)];
}
- (BOOL)loadDynamicLibraryWithPath:(NSString *)path {
	NSBundle *bundle = [NSBundle bundleWithPath:path];
	if (!bundle) {
		NSLog(@"%@ not found", path);
		return NO;
	}

	NSError *error;
	if (![bundle loadAndReturnError:&error]) {
		NSLog(@"Load %@ failed: %@", path, error);
		return NO;
	} else {
		NSLog(@"Load %@ success", path);
	}

	return YES;
}
```
这样也可以成功加载动态库，这样加载动态库更像是一个动态加载库。

## 总结
* 动态库和静态库其实都是一个编译产物，只是加载到app的方式不同。静态库在链接阶段参与的加载。而动态库是在app启动或app运行时加载。
* 至于文件后缀就很好理解了，静态库可以是`.a`文件也可以是`.framework`文件。动态库现在一般都是`.framework`，不过`.framework`不一定都是动态库。

## 参考
* [iOS 开发中的『库』](https://github.com/Damonvvong/DevNotes/blob/master/Notes/framework.md)
* [深入剖析iOS动态链接库](https://www.jianshu.com/p/1de663f64c05)
* [iOS动态库](https://codingpub.github.io/2019/04/26/iOS%E5%8A%A8%E6%80%81%E5%BA%93/)
* [通过dlopen使用动态库](https://www.jianshu.com/p/2e7bf6e7cf45)
* [Xcode中Link Binary With Libraries的Status含义](https://www.jianshu.com/p/9265dc07e0f8)
* [Build Active Architecture Only 设置](https://www.jianshu.com/p/882fd9fba04c)
* [What is the difference between Embedded Binaries and Linked Frameworks](https://stackoverflow.com/questions/32375687/what-is-the-difference-between-embedded-binaries-and-linked-frameworks)
* [Xcode 工程添加 “动态” Framework 的几种方式](https://kangzubin.com/xcode-add-dynamic-framework/)
* [细说iOS静态库和动态库](https://juejin.im/post/5e042f116fb9a016194b0f39)
* [库](https://casatwy.com/ku.html)
* [iOS开发之关于静态库制作及注意点](https://sevencho.github.io/archives/a9b5807a.html)
* [iOS静态库与动态库的区别与打包](https://juejin.im/post/5dc92e92f265da4d4d0d0054)
* [如何打造一个让人愉快的框架](https://onevcat.com/2016/01/create-framework/)