---
layout: post
title: Objective-C Runtime —— 基础数据结构
date: 2018-05-14 17:48:48
summary: iOS工作两年了，几乎每次面试都会被问到Runtime的东西，自己一直没有系统的学习过Runtime。正好这段时间比较闲，自己也想深入的学习下iOS。所以那就先了解下基础的东西吧。以前听到Runtime一直感觉很高深，什么黑魔法啊一堆名词，其实去了解下的话，没有那么神奇。
category: Study
---

iOS工作两年了，几乎每次面试都会被问到Runtime的东西，自己一直没有系统的学习过Runtime。正好这段时间比较闲，自己也想深入的学习下iOS。所以那就先了解下基础的东西吧。以前听到Runtime一直感觉很高深，什么黑魔法啊一堆名词，其实去了解下的话，没有那么神奇。这篇文章分三大部分来介绍Runtime，是一个循序渐进的过程。第一呢，我们需要了解Objective-C的对象模型，然后才能去理解消息发送的过程，再然后再去了解下Runtime都能干什么，毕竟如果一个技术没有落地的应用，那它就没啥价值了。这篇文章基于objc4-723源码进行分析，GitHub有一个可调试库，[地址](https://github.com/RetVal/objc-runtime)。
<!-- more -->
# 简介
Runtime是什么？官方文档第一段就给出了答案。
> The Objective-C language defers as many decisions as it can from compile time and link time to runtime. Whenever possible, it does things dynamically. This means that the language requires not just a compiler, but also a runtime system to execute the compiled code. The runtime system acts as a kind of operating system for the Objective-C language; it’s what makes the language work.

其实Runtime就是一个运行时库，我们在编译期调用的函数最终都会通过Runtime库去进行分发。Runtime的核心就是发送消息。Objective-C从三种层级和Runtime交互。

1. Objective-C源代码。<br/>
一般情况，我们只需编写Objective-C代码，Runtime会默默的工作。当编译包含Objective-C代码时，编译器会创建实现动态特性的数据结构和函数调用。主要的函数就是发送消息。
3. Runtime函数。<br/>
Runtime系统是由一系列函数和数据结构构成的，它有一系列公共接口。头文件在/usr/include/objc目录下。许多函数允许你用C函数来实现Objective-C中的功能。但是谁会去费这个力气呢？一般在写一些与其它语言桥接或者debug时会用到。在[Objective-C Runtime Reference](https://developer.apple.com/documentation/objectivec/objective_c_runtime)中有对函数的详细解释。
2. NSObject方法。<br/>
在写Objective-C时，我们的对象大多数都继承自NSObject(NSProxy除外)。所以，我们可以通过NSObject的一些方法来与Runtime交互。比如：NSObject提供了一系列自省方法来查询Runtime信息。<br/>


```
- (BOOL)isKindOfClass:(Class)aClass;
- (BOOL)isMemberOfClass:(Class)aClass;
- (BOOL)conformsToProtocol:(Protocol *)aProtocol;
- (BOOL)respondsToSelector:(SEL)aSelector;
- (Class)superclass;
+ (Class)class OBJC_SWIFT_UNAVAILABLE("use 'aClass.self' instead");
```


# 基础数据结构
## 类和对象的关系
由于我们在Objective-C中大多数都继承NSObject类，那我们就以NSObject为切入点来分析对象的数据结构。NSObject定义如下：

```
@interface NSObject <NSObject> {
    Class isa  OBJC_ISA_AVAILABILITY;
}
```
我们看到它有一个Class变量。那我们在看一下Class的定义：

```
/// An opaque type that represents an Objective-C class.
typedef struct objc_class *Class;
```
可以看到他是一个objc_class指针，那objc_class是什么呢？在objc-runtime-new.h中找到了objc_class的定义，下面是一个简化版的定义：

```
struct objc_class : objc_object {
    // Class ISA;
    Class superclass;
    cache_t cache;             // formerly cache pointer and vtable
    class_data_bits_t bits;    // class_rw_t * plus custom rr/alloc flags

    class_rw_t *data() { 
        return bits.data();
    }
};
```
我们可以看到objc_class是一个继承objc_object的结构体，下面是一个简化版objc_object定义，在文件中，我们还发现了id的定义。

```
struct objc_object {
private:
    isa_t isa;
};
typedef struct objc_object *id;
```
从上面可以看出来，id是一个objc_object类型的指针，以前一直以为id和NSObject *等价，好无知啊。从上面看NSObject和objc_object的定义很像，只不过NSObject的isa是一个Class，而objc_object的isa是一个isa_t。下面，我们来看下isa_t是一个什么东西。isa_t定义如下：

```
union isa_t 
{
    isa_t() { }
    isa_t(uintptr_t value) : bits(value) { }

    Class cls;
    uintptr_t bits;

#if SUPPORT_PACKED_ISA

    // extra_rc must be the MSB-most field (so it matches carry/overflow flags)
    // nonpointer must be the LSB (fixme or get rid of it)
    // shiftcls must occupy the same bits that a real class pointer would
    // bits + RC_ONE is equivalent to extra_rc + 1
    // RC_HALF is the high bit of extra_rc (i.e. half of its range)

    // future expansion:
    // uintptr_t fast_rr : 1;     // no r/r overrides
    // uintptr_t lock : 2;        // lock for atomic property, @synch
    // uintptr_t extraBytes : 1;  // allocated with extra bytes

# if __arm64__
#   define ISA_MASK        0x0000000ffffffff8ULL
#   define ISA_MAGIC_MASK  0x000003f000000001ULL
#   define ISA_MAGIC_VALUE 0x000001a000000001ULL
    struct {
        uintptr_t nonpointer        : 1;
        uintptr_t has_assoc         : 1;
        uintptr_t has_cxx_dtor      : 1;
        uintptr_t shiftcls          : 33; // MACH_VM_MAX_ADDRESS 0x1000000000
        uintptr_t magic             : 6;
        uintptr_t weakly_referenced : 1;
        uintptr_t deallocating      : 1;
        uintptr_t has_sidetable_rc  : 1;
        uintptr_t extra_rc          : 19;
#       define RC_ONE   (1ULL<<45)
#       define RC_HALF  (1ULL<<18)
    };

# elif __x86_64__
#   define ISA_MASK        0x00007ffffffffff8ULL
#   define ISA_MAGIC_MASK  0x001f800000000001ULL
#   define ISA_MAGIC_VALUE 0x001d800000000001ULL
    struct {
        uintptr_t nonpointer        : 1;
        uintptr_t has_assoc         : 1;
        uintptr_t has_cxx_dtor      : 1;
        uintptr_t shiftcls          : 44; // MACH_VM_MAX_ADDRESS 0x7fffffe00000
        uintptr_t magic             : 6;
        uintptr_t weakly_referenced : 1;
        uintptr_t deallocating      : 1;
        uintptr_t has_sidetable_rc  : 1;
        uintptr_t extra_rc          : 8;
#       define RC_ONE   (1ULL<<56)
#       define RC_HALF  (1ULL<<7)
    };
```
可以看出isa_t是一个union类型结构体，也就是说isa_t，cls，bits和结构体共用一块内存结构。isa总共占64为内存结构(取决于结构体)。好了，把上面的源码转换成类图就是下面这样子了。
![图片来自冰霜的blog](https://nightwish.oss-cn-beijing.aliyuncs.com/1526375149.png)
> 图片来自冰霜的blog

从上面源码我们可以看出，Objective-C中的对象都是一个objc_object结构体，objc_class继承objc_object，所以在Objective-C中，**类也是一个对象。**在objc_class中，除了isa还有superclass，cache，bits三个变量，分别表示父类指针，方法缓存，还有一个是类的数据区域。object和NSObject都包含一个objc_class的isa。

类的isa指向的是什么呢？这就引出了元类(meta-class)的概念。关于元类可以看这篇文章[What is a meta-class in Objective-C?](http://www.cocoawithlove.com/2010/01/what-is-meta-class-in-objective-c.html)。

为什么会有元类呢？

因为在Objective-C中，对象的方法并没有存储在对象的结构体中(如果每个对象都存储自己的方法，那我们程序中无数对象就都要存储自己的方法，那内存肯定就不够用了)。当我们调用实例方法时，它通过自己的isa查找到对应的类，然后在class_data_bits_t结构体中查找对应的方法实现。每一个objc_class也有一个superClass指向自己的父类，可以查找到继承的方法。那么如果调用实例方法怎么查找呢？这时，就需要引入元类来保证无论调用类方法和实例方法，都可以以相同的方式来查找。

* 调用实例方法，通过对象的isa在类中获取方法。
* 调用类方法，通过类的isa在元类中获取方法。

类和元类的关系如下图：
![类和元类](https://nightwish.oss-cn-beijing.aliyuncs.com/1526378403.png)
>图片来自[Classes and metaclasses](http://www.sealiesoftware.com/blog/archive/2009/04/14/objc_explain_Classes_and_metaclasses.html)

## isa指针
### isa初始化
下面我们来一步步解析类结构，先从isa指针来看起。上面我们说了isa是一个isa_t的union。那么我们看下isa是如何初始化的。

```
inline void 
objc_object::initInstanceIsa(Class cls, bool hasCxxDtor)
{
    assert(!cls->instancesRequireRawIsa());
    assert(hasCxxDtor == cls->hasCxxDtor());

    initIsa(cls, true, hasCxxDtor);
}
inline void 
objc_object::initIsa(Class cls, bool nonpointer, bool hasCxxDtor) 
{ 
    assert(!isTaggedPointer()); 
    
    if (!nonpointer) {
        isa.cls = cls;
    } else {
        assert(!DisableNonpointerIsa);
        assert(!cls->instancesRequireRawIsa());

        isa_t newisa(0);

#if SUPPORT_INDEXED_ISA
        assert(cls->classArrayIndex() > 0);
        newisa.bits = ISA_INDEX_MAGIC_VALUE;
        // isa.magic is part of ISA_MAGIC_VALUE
        // isa.nonpointer is part of ISA_MAGIC_VALUE
        newisa.has_cxx_dtor = hasCxxDtor;
        newisa.indexcls = (uintptr_t)cls->classArrayIndex();
#else
        newisa.bits = ISA_MAGIC_VALUE;
        // isa.magic is part of ISA_MAGIC_VALUE
        // isa.nonpointer is part of ISA_MAGIC_VALUE
        newisa.has_cxx_dtor = hasCxxDtor;
        newisa.shiftcls = (uintptr_t)cls >> 3;
#endif
        isa = newisa;
    }
}
```
这里调用initIsa的第二个参数nonpointer传入了true，所以会直接进入else。
>关于SUPPORT_INDEXED_ISA应该是对watchOS的优化，所以会走到#else

我们看下isa_t内存布局：
![](https://nightwish.oss-cn-beijing.aliyuncs.com/1526463593.png)
>图片来自[霜神blog](https://www.jianshu.com/p/9d649ce6d0b8)

ISA_MAGIC_VALUE = 0x000001a000000001ULL，转换成二进制是11010000000000000000000000000000000000001，所以把ISA_MAGIC_VALUE赋值给bits，isa_t的结构如下图：
![](https://nightwish.oss-cn-beijing.aliyuncs.com/1526464002.png)
>图片来自[霜神blog](https://www.jianshu.com/p/9d649ce6d0b8)

从上图可以看出，调用`newisa.bits = ISA_MAGIC_VALUE;`这句代码是给nonpointer和magic进行了赋值。
#### nonpointer和magic
* nonpointer表示是否开启指针优化，1表示开启指针优化，0表示一个raw isa，也就是isa_t这个union没有结构体部分，访问对象的isa直接返回cls指针。不过现在都会开启指针优化。关于这个指针优化，后面会有一篇单独的文章来分析，就是关于Tagged Pointer技术。
* magic判断对象是否初始化完成，在arm64中0x16是调试器判断当前对象是真的对象还是没有初始化的空间。

#### has_cxx_dtor
经过`newisa.has_cxx_dtor = hasCxxDtor;`这段代码，会设置isa_t的has_cxx_dtor，这一位表示当前对象有 C++ 或者 ObjC 的析构器(destructor)，如果没有析构器就会快速释放内存。设置完后结构如下：
![](https://nightwish.oss-cn-beijing.aliyuncs.com/1526465717.png)
>图片修改自[霜神blog](https://www.jianshu.com/p/9d649ce6d0b8)

#### shiftcls
通过上面两行代码设置了nonpointer，magic，has_cxx_dtor。最后一行代码把当前对象的类指针存入了isa中。

```
newisa.shiftcls = (uintptr_t)cls >> 3;
```
为什么右移三位呢，引用Draveness的blog中一段话来解释下：
>将当前地址右移三位的主要原因是用于将 Class 指针中无用的后三位清除减小内存的消耗，因为类的指针要按照字节（8 bits）对齐内存，其指针后三位都是没有意义的 0。

>绝大多数机器的架构都是 [byte-addressable](https://en.wikipedia.org/wiki/Byte_addressing) 的，但是对象的内存地址必须对齐到字节的倍数，这样可以提高代码运行的性能，在 iPhone5s 中虚拟地址为 33 位，所以用于对齐的最后三位比特为 000，我们只会用其中的 30 位来表示对象的地址。

Objective-C中类的指针后三位也是0，在`_class_createInstanceFromZone`中打印类的地址可以看到最后一位不是0就是8，所以类的指针后三位是0。
![](https://nightwish.oss-cn-beijing.aliyuncs.com/1526483409.png)
所以上面cls>>3没有问题。我们尝试运行下面代码来形象说明下，理解的可以更加深刻。

```
NSString *binaryWithInteger(NSUInteger decInt) {
    NSString *string = @"";
    NSUInteger x = decInt;
    while (x > 0) {
        string = [[NSString stringWithFormat:@"%lu", x&1] stringByAppendingString:string];
        x = x >> 1;
    }
    return string;
}
int main(int argc, const char * argv[]) {
    @autoreleasepool {
        struct objc_object *object = (__bridge struct objc_object *)([NSObject new]);
        NSLog(@"%@", binaryWithInteger(object->isa));
        NSLog(@"%@", binaryWithInteger([NSObject class]));
    }
    return 0;
}
//打印结果
object_isa 0000000001011101100000000000000100000000011101000000000100111001 // 补全64位
class  100000000011101000000000100111000

```
打印出来object的isa的内容如下图所示：
![](https://nightwish.oss-cn-beijing.aliyuncs.com/1526539598.png)
>图片修改自[霜神blog](https://www.jianshu.com/p/9d649ce6d0b8)

其中浅蓝色部分是类指针，和上面打印的class指针右移三位一样。这就验证了我们之前对于初始化isa是设置了nonpointer，magic，shiftclass的分析。我们也可以通过这种方式分析上面类结构那个图，验证下NSObject的isa是指向NSObject元类，NSObject元类的isa指向自己，superclass指向NSObject类对象。感兴趣的可以自己验证下，在这里就不去分析了。
### isa总结
下面我们总结下isa是如何初始化的。首先把ISA_MAGIC_VALUE赋值给bits设置了nonpoint和magic，然后设置has_cxx_dtor，最后把类指针右移三位设置给shiftcls。下面我们集中说一下结构体所有位的作用。

| 位                | 代表信息                                                       |
| ----------------- | -------------------------------------------------------------- |
| nonpointer        | 0 表示普通的 isa 指针，1 表示使用优化                          |
| has_assoc         | 表示该对象是否包含 associated object，如果没有，则析构时会更快 |
| has_cxx_dtor      | 表示该对象是否有 C++ 或 ARC 的析构函数，如果没有，则析构时更快 |
| shiftcls          | 类的指针                                                       |
| magic             | 固定值，用于在调试时分辨对象是否未完成初始化                   |
| weakly_referenced | 表示该对象是否有过 weak 对象，如果没有，则析构时更快           |
| deallocating      | 表示该对象是否正在析构                                         |
| has_sidetable_rc  | 表示该对象的引用计数值是否过大无法存储在 isa 指针              |
| extra_rc          | 存储引用计数值减一后的结果                                     |
>解释来自http://www.sealiesoftware.com/blog/archive/2013/09/24/objc_explain_Non-pointer_isa.html

## cache_t
cache_t就比较简单了，先来看下具体代码
```
struct cache_t {
    struct bucket_t *_buckets;
    mask_t _mask;
    mask_t _occupied;
}
struct bucket_t {
private:
    cache_key_t _key;
    IMP _imp;
}
```
* _buckets是一个散列表，用来存储方法缓存。bucket_t是一个key和IMP的结构体。
* _mask分配bucket的总数。
* _occupied表示目前占用缓存bucket的个数。

Cache的作用主要是优化方法调用的性能，当对象收到消息后，会优先去Cache中查找，如果Cache中没有才会去类的方法列表找。如果没有Cache，我们每次调用方法都去类的方法找，会严重影响性能。
## class_data_bits_t
### class_data_bits_t结构体解析
在类的结构体中，isa我们已经解释过了，superclass是一个指向父类的指针，cache是为了加速方法调用，下面我们来分析一下class_data_bits_t，一个类的方法，属性和遵循的协议都在这个里面。我们先看下代码：

```
struct class_data_bits_t {

    // Values are the FAST_ flags above.
    uintptr_t bits;
}
```
它只有一个64位bits用来存储和类有关的信息。它有一个data()方法来返回class_rw_t *指针。

```
class_rw_t* data() {
        return (class_rw_t *)(bits & FAST_DATA_MASK);
}
```
其中FAST_DATA_MASK = 0x00007ffffffffff8UL，bits与FAST_DATA_MASK位运算，取bits的[3,47]位转成class_rw_t。
>在 x86_64 架构上，Mac OS 只使用了其中的 47 位来为对象分配地址。而且由于地址要按字节在内存中按字节对齐，所以掩码的后三位都是 0。

因为class_rw_t *指针只存于[3,47]，所以后面三位可以存储当前类的的其他信息。
![](https://nightwish.oss-cn-beijing.aliyuncs.com/1526550108.png)
>图片来自[深入解析 ObjC 中方法的结构](https://github.com/Draveness/analyze/blob/master/contents/objc/%E6%B7%B1%E5%85%A5%E8%A7%A3%E6%9E%90%20ObjC%20%E4%B8%AD%E6%96%B9%E6%B3%95%E7%9A%84%E7%BB%93%E6%9E%84.md)


```
// class is a Swift class
#define FAST_IS_SWIFT           (1UL<<0)
// class or superclass has default retain/release/autorelease/retainCount/
//   _tryRetain/_isDeallocating/retainWeakReference/allowsWeakReference
#define FAST_HAS_DEFAULT_RR     (1UL<<1)
// class's instances requires raw isa
#define FAST_REQUIRES_RAW_ISA   (1UL<<2)
```
* FAST_IS_SWIFT 用于判断 Swift 类
* FAST_HAS_DEFAULT_RR 当前类或者父类含有默认的 retain/release/autorelease/retainCount/_tryRetain/_isDeallocating/retainWeakReference/allowsWeakReference 方法
* FAST_REQUIRES_RAW_ISA 当前类的实例需要 raw isa

当调用objc_class的data()方法也会返回class_rw_t *指针，这个方法也是调用了class_data_bits_t的data()方法。
### class_rw_t和class_ro_t
我们来看下class_rw_t的结构：

```
struct class_rw_t {
    // Be warned that Symbolication knows the layout of this structure.
    uint32_t flags;
    uint32_t version;

    const class_ro_t *ro;

    method_array_t methods;
    property_array_t properties;
    protocol_array_t protocols;

    Class firstSubclass;
    Class nextSiblingClass;

    char *demangledName;
};
```
终于看到了和方法，属性，协议有关的东西。可以看到class_rw_t包含一个class_ro_t *指针，class_ro_t中存储了当前类在编译期就确定的属性，方法和遵守的协议。

```
struct class_ro_t {
    uint32_t flags;
    uint32_t instanceStart;
    uint32_t instanceSize;
    uint32_t reserved;
    const uint8_t * ivarLayout;
    const char * name;
    method_list_t * baseMethodList;
    protocol_list_t * baseProtocols;
    const ivar_list_t * ivars;

    const uint8_t * weakIvarLayout;
    property_list_t *baseProperties;
};
```
下面我们说下class_rw_t和class_ro_t的关系，然后再来验证。在编译期间，类结构中的class_data_bits_t *data指向的是class_ro_t *指针，用类图来表示一下：
![](https://nightwish.oss-cn-beijing.aliyuncs.com/1526613551.png)
>图片来自[深入解析 ObjC 中方法的结构](https://github.com/Draveness/analyze/blob/master/contents/objc/%E6%B7%B1%E5%85%A5%E8%A7%A3%E6%9E%90%20ObjC%20%E4%B8%AD%E6%96%B9%E6%B3%95%E7%9A%84%E7%BB%93%E6%9E%84.md)

然后在运行时会调用realizeClass方法：

```
ro = (const class_ro_t *)cls->data(); // 调用class_data_bits_t的data()方法，强制转换成class_ro_t *指针。
class_rw_t *rw = (class_rw_t *)calloc(sizeof(class_rw_t), 1); 初始化class_rw_t结构体。
rw->ro = ro; //设置ro
rw->flags = RW_REALIZED|RW_REALIZING; //设置flag
cls->setData(rw); // 把class_rw_t设置为class_data_bits_t旳data
```
经过上面的方法类图会变为下面这样的：
![](https://nightwish.oss-cn-beijing.aliyuncs.com/1526607686.png)
>图片来自[深入解析 ObjC 中方法的结构](https://github.com/Draveness/analyze/blob/master/contents/objc/%E6%B7%B1%E5%85%A5%E8%A7%A3%E6%9E%90%20ObjC%20%E4%B8%AD%E6%96%B9%E6%B3%95%E7%9A%84%E7%BB%93%E6%9E%84.md)

代码运行到这里class_rw_t中的方法，属性和协议列表是空的。这时需要调用methodizeClass方法来将类自己实现的方法(包括分类)，属性，协议加载到method，property，protocol列表中。
### 验证
#### 验证类在编译期结构
下面我们来分析一下TestObject在初始化过程中内存的修改，下面是TestObject的代码：

```
//TestObject.h
#import <Foundation/Foundation.h>

@interface TestObject : NSObject
-(void)sayHello;
@end

#import "TestObject.h"

//TestObject.m
@implementation TestObject
- (void)sayHello
{
    NSLog(@"Hello");
}
@end

//main.m
int main(int argc, const char * argv[]) {
    @autoreleasepool {
        Class cls = [TestObject class];
        NSLog(@"%p\n",cls);
    }
    return 0;
}
```
因为**类在内存中的地址是在编译期确定的，只要代码不变，类在内存中的位置不变。**我们先来运行一次拿到类在内存中的位置。下面是打印结果：

```
0x100002750
```
接下来在_objc_init方法打一个断点，这个方法是运行时初始化之前调用的。
![](https://nightwish.oss-cn-beijing.aliyuncs.com/1526611021.png)
然后在lldb中输入下面的命令：
![](https://nightwish.oss-cn-beijing.aliyuncs.com/1526611896.png)
通过上面的图我们可以看到，在class_ro_t中，只有baseMethodList和name有值，其他的都没值。因为类里面只有一个方法，没有属性，协议什么的。下面我们看下baseMethodList里面的内容：
![](https://nightwish.oss-cn-beijing.aliyuncs.com/1526612249.png)
我们通过$5->get(0)可以获取到一个method_t结构体，可以看到就是我们在类中的sayHello方法。
#### 验证realizeClass
下面我们来验证下realizeClass的作用，该方法的主要作用就是对类进行第一次初始化，包括：
* 分配可读写内存空间
* 返回真正的类结构


```
static Class realizeClass(Class cls)
```
我们需要在这个方法打一个条件断点：
![](https://nightwish.oss-cn-beijing.aliyuncs.com/1526612802.png)
这里使用指针相等的方式，来确定当前类是TestObject，之所以不用`[NSStringFromClass(cls) isEqualToString:@"TestObject"]`，是因为在这个时间点上这些方法还不能用。
![](https://nightwish.oss-cn-beijing.aliyuncs.com/1526613287.png)
这个断点设置在这里，因为TestObject是一个正常类，所以会走else方法。运行代码时，因为每个类都会走这个方法，只有类是TestObject时才会进断点，所以可能会等会才进断点。
在这时打印类结构，其中布局依然会是下图这样：
![](https://nightwish.oss-cn-beijing.aliyuncs.com/1526613551.png)
>图片来自[深入解析 ObjC 中方法的结构](https://github.com/Draveness/analyze/blob/master/contents/objc/%E6%B7%B1%E5%85%A5%E8%A7%A3%E6%9E%90%20ObjC%20%E4%B8%AD%E6%96%B9%E6%B3%95%E7%9A%84%E7%BB%93%E6%9E%84.md)

当运行完else中代码，我们再来打印下结构：
![](https://nightwish.oss-cn-beijing.aliyuncs.com/1526614493.png)
通过打印可以看出来，当运行完else中代码，类的class_ro_t和class_rw_t都被正确设置了，但是class_rw_t中的methods，properties和protocols均未设置，这些将在`methodizeClass`中设置。
下面我们就去验证下：
![](https://nightwish.oss-cn-beijing.aliyuncs.com/1526615057.png)
当代码运行到这一行，我们再执行下面的命令：
![](https://nightwish.oss-cn-beijing.aliyuncs.com/1526615225.png)
在这里调用了`method_array_t`的`attachLists`方法，将`baseMethods`中的方法添加到`methods`数组之后。我们访问`methods`才会获取当前类的实例方法。
#### list_array_tt和method_t
##### list_array_tt
当我们再看class_rw_t时，发现它们的方法，属性，协议列表分别是method_list_t，property_array_t和protocol_array_t。他们都继承自list_array_tt。list_array_tt是一个泛型结构体，用于存储一些元数据，其实是一个二维数组。
```
template <typename Element, typename List>
class list_array_tt {
    struct array_t {
        uint32_t count;
        List* lists[0];
    };
}
class method_array_t : 
    public list_array_tt<method_t, method_list_t>
```
其中Element表示元数据类型，比如method_t，而List表示用来转载元数据的一维数组，比如method_list_t。
>list_array_tt有三种状态：
1. 自身为空，可以类比为`[[]]`
2. 它只有一个指针，指向一个元数据的集合，可以类比为`[[1, 2]]`
3. 它有多个指针，指向多个元数据的集合，可以类比为`[[1, 2], [3, 4]]`
当一个类刚创建时，它可能处于1，2状态，如果使用`class_addMethod`或者category添加方法，就会进入3状态，而且一旦进入3状态，就再也不可能回到其他状态。即使新增的方法移除掉

##### method_t
下面来看下method_t的结构，method还是比较简单的，都比较容易理解。有关类型可以看官方文档的[Type Encodings](https://developer.apple.com/library/content/documentation/Cocoa/Conceptual/ObjCRuntimeGuide/Articles/ocrtTypeEncodings.html#//apple_ref/doc/uid/TP4000804。)

```
struct method_t {
    SEL name; // 方法名
    const char *types; // 类型   
    IMP imp; // 实现指针IMP
}
```
我们上面TestObject的method_t就是下面这样的：

```
name = "sayHello"
types = 0x0000000100001f9b "v16@0:8"
imp = 0x0000000100001d10 (debug-objc`-[TestObject sayHello] at TestObject.m:11)
```
# 总结
1. isa是一个union，里面包含一个结构体。每一个字段代表一段信息，其中它的类信息存在于shiftcls中。我们可以通过object_getClass()函数获取一个对象的类信息。这里需要注意一点，对象的`class`实例方法会调用object_getClass()方法，所以对于实例方法`object_getClass()`和`class`等价，但是同样还有一个`class`类方法，这个方法返回self，直接返回类对象。
2. 类的内存中的位置在编译期间就决定了，在之后修改代码，也不会改变内存中的位置。
3. 类的方法在编译期间存在于class_ro_t中，直到通过realizeClass方法后才放到class_rw_t指向的class_ro_t中，这样我们在运行时为class_rw_t添加方法，也不会影响类的只读结构。
4. 在class_ro_t中的属性运行时就不会改变了，运行时添加方法，会修改class_rw_t的方法属性，对于方法添加，我们在消息发送的文章再来分析。

```
 +(Class)class {
    return self;
 }
 -(Class)class {
    return object_getClass(self);
 }
```
# 参考资料
* [Objective-C Runtime Programming Guide](https://developer.apple.com/library/content/documentation/Cocoa/Conceptual/ObjCRuntimeGuide/Introduction/Introduction.html#//apple_ref/doc/uid/TP40008048-CH1-SW1)
* [从 NSObject 的初始化了解 isa](https://github.com/Draveness/analyze/blob/master/contents/objc/%E4%BB%8E%20NSObject%20%E7%9A%84%E5%88%9D%E5%A7%8B%E5%8C%96%E4%BA%86%E8%A7%A3%20isa.md)
* [深入解析 ObjC 中方法的结构](https://github.com/Draveness/analyze/blob/master/contents/objc/%E6%B7%B1%E5%85%A5%E8%A7%A3%E6%9E%90%20ObjC%20%E4%B8%AD%E6%96%B9%E6%B3%95%E7%9A%84%E7%BB%93%E6%9E%84.md)
* [神经病院Objective-C Runtime入院第一天——isa和Class](https://www.jianshu.com/p/9d649ce6d0b8)
* [用 isa 承载对象的类信息](https://www.desgard.com/isa/)
* [结合 category 工作原理分析 OC2.0 中的 runtime](https://bestswifter.com/runtime-category/)
* [Non-pointer isa](http://www.sealiesoftware.com/blog/archive/2013/09/24/objc_explain_Non-pointer_isa.html)
* [What is a meta-class in Objective-C?](http://www.cocoawithlove.com/2010/01/what-is-meta-class-in-objective-c.html)




