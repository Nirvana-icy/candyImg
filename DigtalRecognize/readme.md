# SwiftOCR改造过程简述

## 数字识别的基本过程

![数字识别的基本过程](https://mmbiz.qpic.cn/mmbiz_png/M54fjP2zXtFcEiahyfjOCybpKcIPmqKb5BQJgVtQc0jAJTu8Xf9iapVjGBAYTz64LE7qyKmIes01SdGIqia6HicTgg/0?wx_fmt=png)

## 效果

![One_More_Thing](https://mmbiz.qpic.cn/mmbiz_gif/M54fjP2zXtFcEiahyfjOCybpKcIPmqKb58ghtBrSxTLE8tvtSDDu3F9J4baibFnI1T4aA0coJp1ztkfhylT2UB7g/0?wx_fmt=gif)

在第一,二步图像获取与预处理后

步骤三使用 [Connected-component labeling](https://en.wikipedia.org/wiki/Connected-component_labeling) 技术实现了文字的分割,这是识别前关键的一步.

步骤四,对图片进行缩放 转换为16x20的二进制数组 做为神经网络输入前的准备

步骤五,对这组数据跑神经网络

步骤六,输出运算结果 98.99的可能性是数字1.

## SwiftOCR简介

SwiftOCR是一个由Swift编写的快速简单的光学字符识别库,使用FFNN神经网络进行图像识别.针对0~9 A~Z的目标字符进行识别.

SwiftOCR的作者对为什么使用SwiftOCR代替经典的Tesseract 有如下论述:

If you want to recognize normal text like a poem or a news article, go with Tesseract, but if you want to recognize short, alphanumeric codes (e.g. gift cards), I would advise you to choose SwiftOCR because that's where it exceeds.

Tesseract is written in C++ and over 30 years old. To use it you first have to write a Objective-C++ wrapper for it. The main issue that's slowing down Tesseract is the way memory is managed. Too many memory allocations and releases slow it down.

## 改造过程

首先业务目标: 识别印刷体的手机号.

第一步,重新训练数据训练集,使用程序生成各种系统字体的0~9的数字进行训练,输出新的OCR-Network文件.

第二步,图像输入改为视频流.

第三步,优化--提高帧率.

## 优化逻辑与具体代码

1⃣️ 经过测试 识别算法运行中,仍然会有数据的到达.如果算法正在运行,则直接返回.

2⃣️ 识别算法在对数字串进行分割处理时,会调用比较重的由GPUImage库提供的Connected-component labeling,性能消耗较大.

所以首先调用iOS 9提供的文字检测API,判断输入图像中是否有文字.

有则识别文字,无则返回.

```Swift
func captureOutput(_ captureOutput: AVCaptureOutput!, didOutputSampleBuffer sampleBuffer: CMSampleBuffer!, from connection: AVCaptureConnection!) {

    // 识别中 则返回
    if self.viewModel.isOCRRecognizing {
        return   
    }

    // 截取图片
    self.imgToRecognize = XGCameraScanWrapper.cropImageFromSampleBuffer(using: sampleBuffer, croppedSizeInScreen: (self.qRScanView?.getRetangeSize())!)

    // 调用 iOS 9 文字检测API. 若没有检测到文字 => 则返回 不跑数字识别算法
    if #available(iOS 9.0, *) {
        let ciDetector = CIDetector(ofType: CIDetectorTypeText, context: nil, options: nil)
        guard let ciImgToRecognize = self.convertUIImageToCIImage(uiImage: self.imgToRecognize!) else {
            return
        }

        let features = ciDetector?.features(in: ciImgToRecognize)

        if (features?.isEmpty)! {
            self.recognizedImgView?.isHidden = true
            return
        } else {
            self.recognizedImgView?.image = self.imgToRecognize
            self.recognizedImgView?.isHidden = false
        }
    }

    // 识别图像
    self.viewModel.isOCRRecognizing = true
    self.xgDigitalRecognizeService?.recognize(self.imgToRecognize!) { recognizedString in

        if recognizedString.utf16.count >= 11 {  // 简单通过识别结果的长度进行输出判断 实际可通过正则限制结果的输出
            DispatchQueue.main.async {
                self.viewModel.phoneNumStr = recognizedString
                self.resultLablePhoneNum?.text = "手机号: " + self.viewModel.phoneNumStr
            }
        }

        self.viewModel.isOCRRecognizing = false
    }
}
```
## Other

手写字体的机器识别是一个很久远经典的问题.

也有一个近30年的标准训练集[MNIST](http://yann.lecun.com/exdb/mnist/)

可否支持手写字体的识别? 单个手写字体的识别不难,有兴趣可参考下面[Tensorflow on iOS](http://machinethink.net/blog/tensorflow-on-ios/)这篇文章.

难点是如何处理连笔书写的数字,此时通过 [Connected-component labeling](https://en.wikipedia.org/wiki/Connected-component_labeling) 技术进行文字的分割已经失效.
