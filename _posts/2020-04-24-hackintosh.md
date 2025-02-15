---
layout: post
title: 近期黑苹果折腾小记
date: 2020-04-24
summary: 一直以来都想装一台黑苹果折腾一下，去年10月终于组装了一台。在近期也把电脑折腾完美了。
category: Hackintosh
---

一直以来都想装一台黑苹果折腾一下，去年10月终于组装了一台。在近期也把电脑折腾完美了。配置如下：

| 主板              | 技嘉Z390m GAMING      |
|-----------------|---------------------|
| CPU             | i7 9700k            |
| GPU             | 蓝宝石5700xt 超白金       |
| 主SSD(macOS)     | 三星 970 Pro M.2 500G |
| 副SSD(Windows10) | Intel 660P M.2 1T   |
| 内存              | 芝奇幻光戟 16G  * 2      |
| 电源              | 海韵FOCUS GX750 750w  |
| 水冷              | 酷冷至尊冰神B240          |
| 网卡蓝牙(黑苹果免驱)     | BCM94360CS2         |
| 显示器             | LG 27UL850          |
| 机箱              | 酷冷至尊睿Lite3.1红色      |

其中显卡最开始是蓝宝石Vega56，因为黑屏问题换成了5700xt。
桌面整理如下，秀一下我4K的壁纸，哈哈
![WechatIMG15](https://nightwish.oss-cn-beijing.aliyuncs.com/2020/04/24/wechatimg15.jpeg)


## 发现问题
去年10月装完之后发现电脑有静电或者开关音响会自动黑屏，然后就只能重启。因为静电的出现频率不高，把音响和主机插到两个不同的插线板上，开关音响就不会导致黑屏了。所以当时并没有在意，也就不了了之了(主要是不想折腾了，哈哈)。直到今年年初因为疫情的原因在家办公，加上春天天气干燥，电脑经常黑屏，搞得心情烦躁，终于下定决心来找一下问题。
## 确定黑屏原因
本着花钱节约时间的原则，约了京东的电脑上门维修服务，维修人员来了之后检测了一下告诉我是地线问题。我也不知道他是如何检测的，只拿着电表笔告诉我火线和底线之间应该有电压，但是测量出来我家里的没有。本着怀疑的态度，我自己买了一个电压表，回来检测了一下，家里的地线是正常的。于是联系京东售后重新检测，为了排除问题，维修人员把我电脑带到了维修站去检测，终于发现是显卡的问题，因为换一个其他显卡就不会出现这种问题。
## 解决显卡问题
因为最初装机时显卡是闲鱼买的二手的蓝宝石Vega56，所以当天立即联系卖家解决问题。卖家当时买了京东的延保服务，2年之内有问题可以直接换新。所以需要我把显卡寄给延保服务进行检测。在这里强烈谴责这个延保服务工作效率，检测问题就需要15个工作日。后面检测确实有问题，进行换新，因为Vega56没有现货了，所以自己补了差价，买了5700xt。收到货后放到电脑里面，把黑苹果引导配置重新折腾了一番，终于可以正常开机，并且没有了黑屏问题了。本以为到这里就算解决了显卡问题，哪知道前方还有坑在等我。
第二天早上想测试一下5700xt的性能，毕竟A卡高端卡。下载了Geekbench开始跑分，跑OpenCL刚开始10秒中，电脑直接卡住不动，然后自己重启了。当时我心里想可能需要预热一下，又跑了一次，结果还是同样的情况。切到Windows系统，下载Windows版本的Geekbench重新跑(这里表扬一下Geekbench的开发者，支持了Windows，让我排除了系统问题)，还是一样卡死，只是Windows不会重启。在这之后还下载了国产跑分软件鲁大师跑了一下，鲁大师告诉我我的显卡很厉害，超过了99%的电脑，我呵呵了。虽然我平常不用OpenCL的软件(从一个群友那里得知Adobe全家桶好像都用了OpenCL算法)，但是也不允许电脑有这种缺陷额，于是联系京东换货。京东客服告诉我他们需要检测有问题才能换货。我去，当时心里就想又检测，上次检测15个工作日，不会又来15个工作日吧。好在客服告诉我2-3个工作日就可以检测完成。于是尽快的把显卡寄给了京东售后，这里要夸一下京东的售后处理速度，第二天就给我说可以换货了，然后给我寄了一个新的显卡。装到电脑上，跑分不会卡住重启，黑屏的问题也解决了。
## 优化5700xt在黑苹果下的跑分
虽然跑分没问题了，但是5700xt在macOS下跑分只有3w左右，Windows下面可以到6w5左右。远景也有一些优化的帖子，但是我感觉小白不容易看懂。自己照着帖子研究了一段时间，终于明白了其中的一些原理。
因为5700xt这块显卡还没有苹果机器在用，只是在macOS10.15.1系统里面支持了Navi架构(5700xt是Navi10的架构)。所以想要正确驱动5700xt需要自己注册正确的RomId和EFIVersion进系统，只要正确注入了，跑分基本上可以到65000左右。注入的教程可以看[xjn大佬的优化贴](https://bbs.pcbeta.org/viewthread-1839725-1-1.html)。我自己是用SSDT注入的，[SSDT文件地址](
https://nightwish.oss-cn-beijing.aliyuncs.com/SSDT-GPU.aml)，这个SSDT的作用有3个：
1. Change HECI To IMEI
2. Change PEGP To GFX0
3. 注入5700xt信息

如果要用SSDT需要自己修改以下的代码，把ATY,EFIVersionRomB和ATY,EFIVersionB替换成自己显卡的RomId，自己显卡的RomId可以通过Windows下的GPU-Z软件查看，剩下的几个不用修改。
![-w438](https://nightwish.oss-cn-beijing.aliyuncs.com/2020/04/24/15876995810760.jpg)


用了这个SSDT还需要在config文件里面注入Change GFX0 To IGPU的补丁，也就是下图这样：
![-w783](https://nightwish.oss-cn-beijing.aliyuncs.com/2020/04/24/15876515157523.jpg)
这三个改名的其实是WhateverGreen的一部分，看xjn大佬的帖子，WhateverGreen其实干了以下几件事：
1. AMD框架等
2. HECI, IGPU, PEGP的重命名
3. 核显驱动
4. AGDP黑屏补丁

AMD框架会影响独显的发挥，重命名的事我们自己做了，去掉核显驱动核显就可以拉满1.2GHz，AGDP黑屏补丁不能去掉。所以在我的EFI里面用了[VegaGraphicsFixup](https://github.com/jyavenard/Vega5KFixup/releases)这个kext，没用WhateverGreen。这个里面只保留了AGDP黑屏补丁。通过这样设置5700xt在黑苹果下的跑分就和Windows下面一样了。下面对比一下优化前和优化后独显的跑分：
![未命名](https://nightwish.oss-cn-beijing.aliyuncs.com/2020/04/24/wei-ming-ming.png)

下面是核显1.2GHz的满载：
![-w400](https://nightwish.oss-cn-beijing.aliyuncs.com/2020/04/24/15876531696127.jpg)

到这里已经可以说独显跑分正常，核显可以拉满性能。如果是视频工作者的朋友可以接着往下看，独显的跑分虽然正常了，但是独显还是可以接着优化的。独显默认的超频很小，可以通过注入一些内容，让独显在渲染视频时可以发挥更大的性能。

## 5700xt超频优化
我的蓝宝石5700xt超白金，GPU最高频率是2100MHz，超频最高超到2150MHz，这也太少了。我们可以通过在config里面注入一些信息调整。调整超频需要用到一个excel表，表的内容大概如下：
![-w1272](https://nightwish.oss-cn-beijing.aliyuncs.com/2020/04/24/15877000843764.jpg)
这里面是显卡的Bios信息，调整完这些信息可以在左下角获得一个Output值，把Output的值注入到config的DeviceProperties里面。
![-w1068](https://nightwish.oss-cn-beijing.aliyuncs.com/2020/04/24/15877003531451.jpg)
其中1是显卡的Device Path，每个人的主板不一样，对应的地址也不一样，可以通过Hackintool获取
![-w1298](https://nightwish.oss-cn-beijing.aliyuncs.com/2020/04/24/15877005016001.jpg)
找到PCIe下的显卡，然后复制设备地址出来，添加到DeviceProperties里面。
2是注入的Key，添加PP_PhmSoftPowerPlayTable这个key，value类型是data，然后把前面Excel表中的Output值填进去就可以了。
下面来说一下如何调整这个表，首先需要获取Rom信息，这个需要两个工具，并且需要再Windows下操作。其中一个工具是GPU-Z，用来导出显卡Bios信息，另外一个是MorePowerTool，用来查看Rom信息。
1. 通过GPU-Z导出显卡Bios信息：
![](https://nightwish.oss-cn-beijing.aliyuncs.com/2020/04/24/15877011140377.jpg)

2. 通过MorePowerTool加载导出的信息：
![](https://nightwish.oss-cn-beijing.aliyuncs.com/2020/04/24/15877011748826.jpg)
点击load加载导出的文件，然后对上面5个标签展示的页面分别截图保存，比如我的就是下面这样的：
![-w842](https://nightwish.oss-cn-beijing.aliyuncs.com/2020/04/24/15877019678461.jpg)

这些信息对应上面Excel表中的信息，可以先把上面Excel表中的每个字段都按照当前显卡信息填一遍，然后再去调整超频。我自己调整的超频是在上面这些信息的基础上调整了表中的5个值：
1. Max Memory Clock： 从1900调整到了2000，表中的Max Memory Clock对应的是上图的Memory Maximum Clock，而且表中对应值=上图中的值 * 2，我的显卡信息对应的是950，表中的就是1900，我调整到了2000。
2. Max GPU Clock (MHz)：从2150调整到了2300
3. Power Limit (%) Maximum：从50调整到了90
4. Power Limit GPU (W)：从220调整到了250
5. MaxVoltageSoc：从1050调整到了1250

其他的都是按照上图默认的填进去的，调整完可以从左下角的Output把值复制出来，填到config的PP_PhmSoftPowerPlayTable里面。我调整的这几个值是从xjn大佬调整的结果，因为到现在为止，苹果系统没有开5700xt的传感器接口，所以不能瞎填，还是在大佬们的调整结果上自己改一些东西。我是自己加了风扇控制，如果直接用[这个帖子](http://bbs.pcbeta.com/viewthread-1836920-1-1.html)里面调整完的结果，显卡的风扇会一直转。我调整到显卡温度到65度才会开始转，这样静音效果比较好。
如果用别人已经调好了值，自己想在这种结果上修改，可以看一下Excel表中最后的Output怎么来的：
![-w1206](https://nightwish.oss-cn-beijing.aliyuncs.com/2020/04/24/15877026565671.jpg)
可以看到Output的值其实就是左面Output上面所有单元格的值拼起来，我们修改右侧的一些值，左侧的带颜色的单元格的值就会变化，最后Output的值也会变化，其中左侧单元格颜色对应的就是右面单元格每行相对应颜色的值，也是一个函数，下面随便来举一个例子：
![-w1202](https://nightwish.oss-cn-beijing.aliyuncs.com/2020/04/24/15877029035511.jpg)
看上图中的FC，函数对应的是AD4，意思就是这个单元格的值就是AD列，第4行的值，右侧第4行的颜色和左侧FC单元格的颜色是一样的，再看AD4的值就是FC，再点到AD4上，看到对应的函数：
![-w1204](https://nightwish.oss-cn-beijing.aliyuncs.com/2020/04/24/15877030733739.jpg)
AD4取的值是AC4的后两位，再去看AC4的值是怎么来的：
![-w1207](https://nightwish.oss-cn-beijing.aliyuncs.com/2020/04/24/15877031455907.jpg)
可以看到AC4的值是AB4的16进制数，并且格式化成4位，2300的16进制可以算一下：
![-w572](https://nightwish.oss-cn-beijing.aliyuncs.com/2020/04/24/15877032104748.jpg)
可以看到2300的16进制是8fc，然后格式化成4位，高位加0，就是08FC，通过上面一系列可以得到，我们修改AB4单元格的值可以影响到左侧的Output值。所以拿到别人调整好的Output值，想在调整好的值上面修改一些东西可以这样对应一下去修改。
## 总结
如果没有这场疫情，可能自己永远都不会想着去解决黑屏的问题，还是用着那个有问题的显卡。所以以后应该直面一些问题，问题总会解决的。只是需要些时间嘛。

再说后面调整显卡，其实自己完全可以将就着用，直接把其他大佬调整的值抄过来。但是用抄过来的值显卡风扇会一直转，像我这么追求完美的人怎么会允许风扇一直转(不经意间又自夸了一下，哈哈)。只能自己去探索一下了，本来以为很难的事情，经过探索那个Excel表发现，其实大佬只是调整了超频的几个值，风扇的自动控制是关的，只要把风扇的自动控制打开就可以达到自己想要的效果了。所以有些东西看起来很难，但是稍微去研究一下就会发现其实还蛮简单的。

有的人说了，追求完美直接去买白苹果啊，折腾啥黑苹果。哎，没办法，我就想折腾一些自己没玩过的东西。黑果可以撸代码，还可以切Windows玩游戏，爽歪歪。白果显卡太垃圾，升级不合算，还是黑果香～
















