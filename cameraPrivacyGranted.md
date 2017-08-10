## iOS 相机权限授权过程优化

## 前言

iOS设备访问相机前的授权是一个常规功能,过程虽然常见,整理下来用户接触到的短暂过程中包含了多种路径,处理起来较为繁琐.

为了便于后续开发人员使用和优化用户体验,参考市面上现有几家App对相机授权过程的处理,在助手端业务的开发中对相机授权流程进行了一次优化。

## 用户体验地图与优化点

Path 1⃣️  首次 启动相机 > 弹出相机使用授权 > 同意 => 正常使用

Path 2⃣️️  首次 启动相机 > 弹出相机使用授权 > 拒绝 => 页面停留在照相机无法启动的黑色页面

Path 2⃣️️  优化：

Step 1: 拒绝 => 弹出权限开启引导对话框

问题 1: 关闭开启引导对话框 => 黑屏页面  

问题 2: 跳转系统设置 > 返回App => 黑屏页面

![问题 2](http://112.124.41.46/bijinglong/candyimg/raw/master/DigtalRecognize/issue_1.gif)         

问题 3: 跳转系统设置 > 开启相机权限 > 返回App => 需要重新Launch App

![问题 3](http://112.124.41.46/bijinglong/candyimg/raw/master/DigtalRecognize/issue_2.gif)    

继续优化 Step 2:  拒绝 => Pop To 前一页 > 弹出权限开启引导对话框

因为拒绝后 先Pop到了前一页

所以对于问题一 与问题二 问题都不存在了

Path 3⃣️  无授权 再次启动相机 > 弹出需要到系统设置开启对话框 如下图 微信  

![微信](http://112.124.41.46/bijinglong/candyimg/raw/master/DigtalRecognize/WechatIMG5.jpeg)

优化 Step 1: 无授权 再次启动相机 > 弹出权限开启引导对话框 => 点击跳转系统设置  如 支付宝    

![支付宝](http://112.124.41.46/bijinglong/candyimg/raw/master/DigtalRecognize/WechatIMG6.jpeg)   

这里 存在同Path 2⃣️  问题二一样的问题 采取同样的逻辑进行处理

拒绝 => Pop To 前一页 > 弹出权限开启引导对话框

## 整理后的优化方案

![整理后的优化方案](http://112.124.41.46/bijinglong/candyimg/raw/master/DigtalRecognize/final.png)

## 优化效果

![优化效果](http://112.124.41.46/bijinglong/candyimg/raw/master/DigtalRecognize/okOne.gif)

## 代码实现

```Swift
override open func viewDidLoad() {
    super.viewDidLoad()

    self.checkAndStartCamera()
}

...

extension XXCameraScanVC {

  func checkAndStartCamera() {

      let authStatus = AVCaptureDevice.authorizationStatus(forMediaType: AVMediaTypeVideo)

      switch authStatus {
      case .authorized:
          self.setupCamera()
      case .denied:
          self.alertToEncourageCameraAccessInitially()
      case .notDetermined:
          self.alertPromptToAllowCameraAccessViaSetting()
      default:
          self.alertToEncourageCameraAccessInitially()
      }
  }

  func alertPromptToAllowCameraAccessViaSetting() {

      if !AVCaptureDevice.devices(withMediaType: AVMediaTypeVideo).isEmpty {
          AVCaptureDevice.requestAccess(forMediaType: AVMediaTypeVideo) { granted in
              // 首次授权 用户没有给予照相机访问的权限 => 返回上一层处理
              DispatchQueue.main.async {
                  if granted {
                      self.setupCamera()
                  } else {
                      self.navigationController?.popViewController(animated: true)
                  }
              }
          }
      }
  }

  func alertToEncourageCameraAccessInitially() {

      // 用户首次没有给予照相机访问的权限 再次进入页面 => 首先返回到上一层
      self.navigationController?.popViewController(animated: true)
      // 其次 弹出Aler弹窗 询问是否需要在设置中 开启照相机访问权限
      let alert = UIAlertController(
          title: "IMPORTANT",
          message: "Camera access required for QR Scanning",
          preferredStyle: UIAlertControllerStyle.alert
      )
      alert.addAction(UIAlertAction(title: "Cancel", style: .default, handler: nil))
      alert.addAction(UIAlertAction(title: "Allow Camera", style: .cancel, handler: { (alert) -> Void in
          DispatchQueue.main.asyncAfter(deadline: DispatchTime.now() + 0.2) {
              UIApplication.shared.openURL(NSURL(string: UIApplicationOpenSettingsURLString)! as URL)
          }
      }))

      self.present(alert, animated: true, completion: nil)
  }
}
```

可以把相机授权的过程放在一个基类里面,以后各种camera的使用继承于此基类.客制化自己的setupCamera即可.达到使用授权过程的简化.

## 问题避免

测试过程中发现 一个偶发的Crash.

问题的路径是:点击Alert框 中的按钮通过openUrl跳转  => Sometimes Crash

[BSMatchError in Stackoverflow](https://stackoverflow.com/questions/32341851/bsmacherror-xcode-7-beta)

I had the same two error messages. In my case, the errors were appearing when I called [[UIApplication sharedApplication] openURL:url] after the user selected a button in an open UIAlertController. I assumed the alert was trying to close at the same time I was trying to open the URL. So, I introduced a slight delay and the error message went away.

```Swift
DispatchQueue.main.asyncAfter(deadline: DispatchTime.now() + 0.2) {
                UIApplication.shared.openURL(NSURL(string: UIApplicationOpenSettingsURLString)! as URL)
            }
```
