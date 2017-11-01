## 模块化灰度内部逻辑与使用指南

#### 内部逻辑

##### 模块化灰度内部逻辑设计目标:

1. 更新配置的动作不影响App的启动.

2. App下次启动时 使用新的配置.

3. 服务端不做机型 & 版本的比较逻辑 客户端进行.

##### 实现逻辑:

![内部逻辑](https://github.com/Nirvana-icy/candyImg/raw/master/iOS_AB_Test/logic.png)

> 说明: App启动 > 读取本地模块配置信息(目前存放在NSUserDefault中) > 启动不同Pods中的模块

> App启动时 异步查询CMS版本管理服务器 > 是否有更新的配置版本 > 如果服务器版本新于本地版本 > 更新本地版本号

>  > 继续在本地对比本机是否属于 服务器配置中的 适用机型 & 版本 => 如果本机属于 则更新本地 模块配置信息的内容.

##### 接口设计

![接口设计](https://github.com/Nirvana-icy/candyImg/raw/master/iOS_AB_Test/interface.png)

> 通过App启动访问接口 客户端拿到新的配置  服务端统计客户端版本/机型  配置下发数量等信息.

#### 使用指南

1. App开发阶段 App内置Dev & Stable版本的业务Pods

例如 1.4.00版本App的开发.系统内置 1.3.90 & 1.4.00两个版本的 rider_all 代码.

根据产品策略 App可以默认使用 新版本 1.4.00的rider_all 代码 或者 经过线上验证的上个版本 1.3.90 的rider_all 代码.

出现问题后 线上发布配置 回退使用1.3.90版本稳定版的 rider_all 代码.

> 或者 默认使用1.3.90的代码, 灰度逐步上线 v1.4.00的代码 -- 产品新功能不急着推广时,推荐使用此种策略.

2. 发版前, 登录 [CMS版本管理系统](http://launch.toobob.com) , 注册对应版本的内置模块信息.

为了防止 多个版本后,每个版本内置多份业务代码,日后搞不清内置模块的信息,这里我们要求发版前对该版本信息进行注册.

依靠版本管理系统 存档对应线上版本的模块版本信息.

Step One. 选择 iOS & Android > 对应App > 注册新版本

![Step One.](https://github.com/Nirvana-icy/candyImg/raw/master/iOS_AB_Test/step1.jpg)

Step Two. 填写 1.4.00 版本内置的模块版本信息 比如 rider_all 1.3.90 & 1.4.00 并 选择App内置的默认版本号.

![Step Two.](https://github.com/Nirvana-icy/candyImg/raw/master/iOS_AB_Test/step2.jpg)

3. 需要下发新配置时 > CMS版本管理系统 > 对应App > 版本管理 > 1.4.00版本 > 点击 新增配置

![Step Two.](https://github.com/Nirvana-icy/candyImg/raw/master/iOS_AB_Test/step3.jpg)

填入配置信息. iOS版本 & 手机型号 可多选,不选代表配置适用全部版本 or 机型.

4. 新增配置后 返回这个版本下面的配置信息列表 > 当前配置仅下发给白名单中的用户 > 白名单中的用户验证完毕后 > 打开发布开关 > 配置将下发给全网灰度用户

> 发布后 仅可调节 灰度比例. 如需修改配置 请新添加配置.

> 同一个App版本 新增配置后 之前配置失效 停止下发. 新增配置属于未发布状态时,之前配置亦失效. 线上环境新增配置后 如果长时间未发布 请恢复线上配置.

5. 如需添加/删除 iOS系统版本 iPhone机型信息 & 模块信息 > 可在信息配置中维护.

#### 开发注意事项

A B 两个版本封装为Pods放在同一个App内,依赖同一份三方Pods.

如果Pods改动 发生不支持B 仅支持 A的时候 我们就会在线上给自己挖了一个大坑.

> 所以: 1. 维护Pods的同学,请严格按照Pods版本号规则增加Pods版本号. 例如a.b.c 修复Bug c增加,添加feature b增加,接口变动/大版本改造 a 增加.

> 2.开发新版本需要升级Pods库时 请报备队长. 由指定人员执行 pod install/update 动作.

> 3.Podfile/podspec中明确指定适用Pods的版本.

> 4.Pods版本信息列入 版本身份证. 发版本前核对 Pods版本变动的情况.
