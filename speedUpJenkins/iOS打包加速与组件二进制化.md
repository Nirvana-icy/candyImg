# iOS打包加速与组件二进制化

随着iOS模块化灰度改造方案[iOS模块化灰度 A/BTest](https://www.jianshu.com/p/e112002d9660)的实施,以及组件化拆分的完成与稳定。项目总体由DevPods,StablePods,组件Pods以及3rd Pods组成。业务开发中开发人员较多的在改变DevPods的代码,其他Pods相对稳定。此时我们可以二进制化组件来加速Jenkins打包的速度。

## 思路

Pods发布时包含源代码版本与Framework的版本，业务开发阶段Jenkins打包时仅需编译DevPods，无需重复编译其他Pods。

开发同学调试时可以方便切换Pods为源代码引入。

## 实施步骤

#### Step 1  修改Pods的 podspec文件

```ruby
Pod::Spec.new do |s|
  s.name = "XGConfig"
  s.version = "2.1.1"
  s.summary = "XGConfig."
  s.license = {"type"=>"MIT", "file"=>"FILE_LICENSE"}
  s.authors = {"name"=>"email address"}
  s.homepage = "http://xxx.com/xxx/XGConfig"

  s.ios.deployment_target    = '8.1'
  s.source       = { :git => "http://xxx/XGConfig.git", :tag => s.version.to_s }

  if ENV['IS_SOURCE'] # 代码引入
    puts '-------------------------------------------------------------------'
    puts 'Notice:XGConfig is source now'
    puts '-------------------------------------------------------------------'
    s.source_files  = 'Source/**.*'

  else  # Framework引入
    puts '-------------------------------------------------------------------'
    puts 'Notice:XGConfig is framework now'
    puts '-------------------------------------------------------------------'
    s.ios.vendored_framework   = "Products/#{s.version}/XGConfig.framework"
  end

  s.dependency 'XGFmdb/SQLCipher', '= 1.0.0'

end

```

#### Step 2 Pods发版本时, 在Pods根目录 执行 Pod package命令生成framework版本的Pods.

``` ruby
pod package XGConfig.podspec --dynamic --spec-sources=http://xxx/xgn.git,https://github.com/CocoaPods/Specs.git

```

#### Step 3 Pods根目录添加 Products/#{s.version}/文件夹 将对应版本的framework放在对应文件夹中.  与之前发布Pods的过程一样 发布新版本的Pods.

![framework](https://mmbiz.qpic.cn/mmbiz_png/M54fjP2zXtFcEiahyfjOCybpKcIPmqKb5q9Ktre4CNWjgtsPLUYW5J33Tib5nm29hDKuVMXW9EYicTgBVAt1Ibrzg/0?wx_fmt=png)

## Well Done

在主工程pod install的时候默认引入二进制版本的Pods.

需要源代码引入组件 进行代码调试的时候

pod cache clean #podsName  选择对应的的版本清除缓存

删除podfile.lock & /Pods目录

执行  IS_SOURCE =1 pod install --no-repo-update 即可切换为源代码引入.

## 下一步

可以将上述手工步骤脚本化,减少出错与重复劳动。

## One More Thing

![briefSpec](https://mmbiz.qpic.cn/mmbiz_jpg/M54fjP2zXtFcEiahyfjOCybpKcIPmqKb5pA9UNMmDE0E85x792QcjzYicxjQ4KPo7fjR0P7aJJPap9Mhml3uJJag/0?wx_fmt=jpeg)
