## iOS模块化灰度改造

## 服务能力

To 技术:

1. 新功能模块级别的 灰度发布.

2. 线上版本回退老版本的能力.

3. 一个App内 打入不同版本模块的能力.

4. 模块组装到不同App的能力. 比如司机端模块 可以单独组装为司机端App 也可与骑手端组装在一起 打包成司机 + 骑手端App.

5. 支持多个版本并行开发.

To 业务方:

1. 不同地区 运行不同版本的业务代码.

> 某些地区先试点,时机成熟后 线上动态扩大/缩小试点范围.

> 不同地区 不同市场策略 业务逻辑的实现.

2. 新旧版本A/B Test.

3. 预埋节假日模块,指定节假日不用发版 即可运行节假日模式.

## 方案概述

第一阶段:

将业务代码拆分到Pods中,不同版本的业务代码打包成不同的Pods.

由于Swift中不同Pods中的代码属于不同的Module,所以将不同的Pods组装到同一个App内不需要考虑重名的问题.

这样改造后的App = 稳定版业务代码Pods + 新开发版本业务代码Pods + 共用的三方库代码.

接入服务端配置中心后,可以在服务端控制Native用户使用的业务版本.

第二阶段:

将第一阶段业务代码拆分为不同的组件,仅对需要灰度的模块进行多版本的预埋.

客户端的主动降级保护,启动App后如果在一定时间内Crash,下次启动自动降级为稳定版本.

手动配置脚本化.

## 方案总揽

![方案总揽](https://github.com/Nirvana-icy/candyImg/raw/master/iOS_AB_Test/iOSABTest.png)

## 方案缺点

1. 包内预埋多个版本的模块 包大小增加.

2. 部分步骤目前需要手工配置xcode.

## 实施过程

### Step 1 将代码封装到Pods中管理,制作Podspec.

![Podspec](https://github.com/Nirvana-icy/candyImg/raw/master/iOS_AB_Test/podspec.png)

### Step 2 .a形式的静态库处理.

由于Pods中无法引入.a形式的静态库,需要把.a形式的静态库(比如微信支付)封装为.framework形式的动态库或者静态库.

这里我们封装为.framework形式的动态库.

[封装过程](http://www.cocoachina.com/ios/20170427/19136.html)

[WXPay封装工程维护在Gitlab](http://git.ops.com/XGN-IOS/WXPay)

编译时如果遇到找不到头文件,请检查WXPay.framework中的module.modulemap 是否包含下图中的头文件.

![WXPay modulemap](https://github.com/Nirvana-icy/candyImg/raw/master/iOS_AB_Test/modulemap.png)

或者 在 module.modulemap中指定的umbrella header文件WXPay.h中 引入需要暴漏的头文件 如下图. 也比较推荐这种方式.

![WXPay modulemap](https://github.com/Nirvana-icy/candyImg/raw/master/iOS_AB_Test/umbrella_header.png)

### Step 3 部分vendor framework完善.

比如AMap3DMap.framework AMapFoundation.framework中没有包含module.modulemap的三方framework,

编译时找不到头文件,需要手工/[脚本](http://www.jianshu.com/p/a1d2d148fdd3)添加 module.modulemap.

我们这里可采用在 podfile中添加脚本的方式 具体见下图:

![podfile](https://github.com/Nirvana-icy/candyImg/raw/master/iOS_AB_Test/podfile.png)

### Step 4 去除之前App工程Header Bridge头文件.

由于之前Swfit工程通过Header Bridge头文件去找引入OC代码的头文件.改造后引入的库封装到Pods中 以framework的形式引入工程,

在添加[module.modulemap](http://www.jianshu.com/p/a1d2d148fdd3)后无需通过header bridge头文件的方式引入头文件.

所以可以在主工程删除Header Bridge头文件.

### Step 5 header search path 添加vender framework path

在 Build Settings > framework search path 中添加 vender framework的路径

### Step 6 App内内置两个版本业务代码Pods后 xcode需要的设置

当预置两个版本业务代码Pods后,会出现Pods安装不成功.

分析一下 项目结构时 主工程 包含 Pods A & Pods B.

Pods A & Pods B包含了同样的Vendor Framework.

此时我们可以把Pods B中Vendor Framework中的库删掉.

然后在Pods B的Target > Build Settings > Framework Search Paths中指向Pods A的Vendor Framework.

达到 主工程 包含 Pods A & Pods B. Pods A & Pods B没有引入两份相同的Vendor Framework,而是共用一份Vendor Framework.

## Next Step

1. 部分手工设置脚本化.

2. 随着业务组件拆分后,灰度模块划分粒度更贴近业务.

3. 稳定模块framework化,加快编译速度.
