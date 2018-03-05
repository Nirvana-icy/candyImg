## 开发阶段的自我检查 之 单元测试

#### Q:开发阶段我们可以进行哪些自我检查呢?

🌇  例如:刚写好一个轮播图组件

✅  我们会造一些轮播图的数据 > 跑一把 看看效果

✅  修改一下数据 只有一张图片 > 重新加载 看下效果

✅  数据为空 > 重新加载 看下效果

✅  换用不同尺寸的图片 > 再次进入 看下效果

#### 是的 在开发阶段我们经常会做代码的自我检查

#### A:从开发的过程来分 我们可以把开发阶段的自我检查划分为：

test func > test class > test service || test UI

#### 相应的我们可以在开发阶段进行:

func test > class test > service test || UI test

#### 从后往前看 我们是否可以更系统的进行开发阶段的自我检查:

1⃣️UI Test我们做的很多 => 开发完毕 > mock数据 > 人肉看一下不同数据时 UI和功能是否正常

2⃣️service test？=> 服务端通过接口测试做的比较多

3⃣️  再往前看 class test & func test => 不易触及 进行自我检查的次数比较少

---

当你的输出是一个App ➡️  我们可以通过UI进行直接的交互

当你的输出是一个Service ➡️  我们可以通过接口进行直接的交互

当你的输出是一个Class or function ➡️  我们可以通过什么进行直接的交互呢?

#### 如果你是AFNetworking的作者 如何进行测试？

1⃣️  写一个Example App 通过人机交互的方式 进行每一项功能的验证？

2⃣️  通过单元测试框架 调用每一个待测试的方法 进行功能的验证

#### 今天我们的主题就是这里

通过单元测试 我们可以方便系统的进行 func test & class test

从而可以更系统的进行开发阶段的自我检查 交付测试团队更加健壮的产出

#### 什么是单元测试呢?

1⃣️  最小可测单元_测试自己写的对外提供功能or计算过程

2⃣️  不是为了发现bug，是为了提高开发效率，为了我们代码健康的可持续发展

🌰  一些调试很久的疑难问题  往往根源是一些简单低级的小错误引起的

#### 单元测试的优缺点

#### 优点：

1⃣️  单元测试能保证在加入新功能或修改旧功能时代码的正确性

2⃣️  单元测试保证在整个开发流程中代码都会被测试，更容易及早发现问题，降低风险

场景:

1⃣️  发版前修改了基础组件的某个功能 ➡️  通过单元测试 可以方便低成本的进行回归测试

2⃣️  代码重构后 ➡️  功能影响的摸底与评估

3⃣️  开发过程中 直接测试某个方法 ➡️  无需build & run整个工程

#### 缺点：

单元测试不能减少研发的代码量，反而会花费很多精力在编写单元测试上，增加了开发成本，而且对开发人员的要求也会更高

#### 单元测试由谁来写?

最好是开发者本人

#### 如何写单元测试?

每一个Test Case的写法有三个步骤：①Mock对象，准备测试数据。②调用要测试的目标API。③验证输出和行为是否正确

#### iOS项目如何进行单元测试呢?

1⃣️  XCTest in XCode

2⃣️  Kiwi for OC (3rd framework)

3⃣️  Quick for Swift or OC (3rd framework)

#### XCTest Demo:

![TestCaseWritenByXCTest](https://raw.githubusercontent.com/Nirvana-icy/candyImg/master/UnitTest/TestCaseWritenByXCTest.png)

#### Quick Demo:

![TestCaseWritenByQuick](https://raw.githubusercontent.com/Nirvana-icy/candyImg/master/UnitTest/TestCaseWritenByQuick.png)

#### 单元测试应用案例(学习样本)

![XCTestWorkInAF](https://raw.githubusercontent.com/Nirvana-icy/candyImg/master/UnitTest/XCTestWorkInAF.png)

![QuickWorkinRAC](https://raw.githubusercontent.com/Nirvana-icy/candyImg/master/UnitTest/QuickWorkinRAC.png)

#### 单元测试落地计划:

1⃣️  基础服务  覆盖

2⃣️  公共组件  覆盖

3⃣️  基础业务viewModel 覆盖

4⃣️  重要业务viewModel 覆盖

#### 参考资料:

1⃣️  [iOS单元测试基本介绍](https://baiduhidevios.github.io/2016/03/20/iOS单元测试/)

2⃣️  [Quick](https://github.com/Quick/Quick)

3⃣️  [Quick使用配置](http://www.jianshu.com/p/95e84dcada56)
