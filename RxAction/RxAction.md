### RxSwift在兔波波助手中的应用

### GUI编程的特点

1⃣️  展现数据

2⃣️  响应用户操作

 > view = render(state) + handle(event)，view 本身只做两件事，给 state 包一层漂亮的外衣，同时对用户的操作做出响应。

### 兔波波助手中的应用

#### 概述

> MVVM with RAC in 美团:

![MVVM_RAC](https://raw.githubusercontent.com/Nirvana-icy/candyImg/master/RxAction/MVVM_RAC.png)

![Error_Handling](https://raw.githubusercontent.com/Nirvana-icy/candyImg/master/RxAction/Error_Handling.png)

模式与美团MVVM+RAC相同,我们尝试采用 MVVM + RxSwift 来满足GUI编程的特点.

viewDidLoad时,绑定view与viewModel.这样viewModel中的数据变化时,界面也随之变化.

当用户操作界面时 viewController 捕获到这些事件,然后调用viewModel中的特定方法,这些方法最终导致viewModel中数据的变化,再次反馈到界面上.

经过初步实践 抽离出以下使用姿势:

1⃣️  viewController 四件套

1.  handleDataChange()

2.  handleUIEvent()

3.  configUI()

4.  makeConstraints()

```Swift
//  TBBWayBillInputVC.swift

override func viewDidLoad() {
    super.viewDidLoad()

    // Do any additional setup after loading the view.
    self.configUI()            // 配置UI
    self.handleDataChange()    // 响应数据变化
    self.handleUIEvent()       // 响应UI操作
}
```

2⃣️  使用Variable包装需要在viewModel和view中绑定的变量

```Swift
//  TBBWayBillInputVM.swift

struct TBBWayBillInputVM {
    var wayBillStrVariable: Variable<String> = Variable("")
    var phoneNumStrVariable: Variable<String> = Variable("")
}
```

这样 viewModel中的变量就可以通过 .asObservable() 在VC中进行与UI元素的绑定.

```Swift
// 响应 viewModel.wayBillStrVariable 变化
        self.viewModel.wayBillStrVariable.asObservable()
            .observeOn(MainScheduler.instance)    // Tips: We should always handle data changes in UI with main thread.
            .filter { !$0.isEmpty }
            .bind(to: self.wayBillTextField.rx.text)
            .disposed(by: disposeBag)
```

viewModel中的变量 也可通过.value 完成变量值的更新

> self.viewModel.wayBillStrVariable.value = text

#### 展现数据 & 响应用户操作

 1⃣️  展现数据

```Swift
    func handleDataChange() {
        // 响应 viewModel.wayBillStrVariable 变化
        self.viewModel.wayBillStrVariable.asObservable()
            .observeOn(MainScheduler.instance)    // Tips: We should always handle data changes in UI with main thread.
            .filter { !$0.isEmpty }
            .bind(to: self.wayBillTextField.rx.text)
            .disposed(by: disposeBag)

        // tipsStrVariable.value 发生改变后 => tipsLabel.text 随之改变
        self.viewModel.tbbPdaOcrScanViewModel.tipsStrVariable.asObservable()
            .observeOn(MainScheduler.instance)    // Tips: We should always handle data changes in UI with main thread.
            .bind(to: self.scanView.tipsLabel.rx.text)
            .disposed(by: disposeBag)
    }
```

    2⃣️  响应用户操作

```Swift
    func handleUIEvent() {
        // 记录 wayBillTextField 内容输入
        self.wayBillTextField.rx.text.orEmpty.changed
        .subscribe(onNext: { [unowned self] text in
            self.viewModel.wayBillStrVariable.value = text
          })
        .disposed(by: disposeBag)

        // 闪光灯按钮
        self.scanView.lightControlBtn.rx.tap
            .subscribe(onNext: { [unowned self] in
                self.viewModel.isTorchOn = !self.viewModel.isTorchOn
                self.scanView.lightControlBtn.setTitle(self.viewModel.isTorchOn ? "关闭闪光灯" : "打开闪光灯", for: .normal)
                self.setTorch(torch: self.viewModel.isTorchOn)
            })
            .disposed(by: disposeBag)
      }
```

🌄  Show Demo & Code

#### One more thing - 处理页面连跳 with RxSwift

![One_More_Thing](https://raw.githubusercontent.com/Nirvana-icy/candyImg/master/RxAction/RxAction.gif)

页面A 跳 B 跳 C Pop to A

避免 block 套 block or Pop时遍历VC

with RxSwift

```Swift
//  TBBWayBillInputVC.swift

// 发起手机号/条码 扫描录入流程
        self.wayBillScanInputBtn.rx.tap
            .flatMap { [unowned self] () -> Observable<(TBBOcrRecognizeResult)> in
                let tbbPdaOcrScanVC = TBBPdaOcrScanVC(isSetupBarCodeRecognize: true, isSetupDigitalRecognize: true)
                tbbPdaOcrScanVC.delegate = self
                self.navigationController?.pushViewController(tbbPdaOcrScanVC, animated: true)
                return tbbPdaOcrScanVC.pushInPhoneNumConfirmVC
            }
            .filter { !$0.barCodeStr.isEmpty }    // 当ocrRecognizeResultVariable发生变化 且 barCodeStr不为空时 => pushInPhoneNumConfirmVC
            .flatMap { [unowned self] (tbbOcrRecognizeResult) -> Observable<(Bool)>  in
                let tbbPhoneNumConfirmVC = TBBPhoneNumConfirmVC(tbbOcrRecognizeResult: tbbOcrRecognizeResult)
                tbbPhoneNumConfirmVC.delegate = self
                self.navigationController?.pushViewController(tbbPhoneNumConfirmVC, animated: true)
                return tbbPhoneNumConfirmVC.shouldPopBackToInputVC
            }
            .filter { $0 == true }    // 仅当 true == shouldPopBackToInputVC时 => popBackToInputVC
            .subscribe(onNext: { [unowned self] _ in
                _ = self.navigationController?.popToViewController(self, animated: true)
            })
            .disposed(by: disposeBag)
```

这样 A 跳 B 跳 C Pop To A 的逻辑都在 A 里面.  

C pop to A 直接这样写就可以了  self.navigationController?.popToViewController(self, animated: true).

问题 1⃣️  B 跳 C 时需要传递值时怎么办? A怎么知道B要传什么值给C?

> Observable<(TBBOcrRecognizeResult)

问题 2⃣️  C pop to A 时 C 的值怎样带回给 A 并触发 A 的刷新动作?

> 为了清晰 C pop back to A时 => 建议采用代理模式 来 刷新A

### RxSwift介绍

limboy关于RxSwift的介绍写的很好,下面的内容摘取自 [是时候学习 RxSwift了](http://limboy.me/tech/2016/12/11/time-to-learn-rxswift.html)

#### 是什么

> 在说 RxSwift 之前，先来说下 Rx， ReactiveX 是一种编程模型，最初由微软开发，结合了观察者模式、迭代器模式和函数式编程的精华，来更方便地处理异步数据流。其中最重要的一个概念是 Observable。

> Object-C时代对应的是ReactiveCocoa. ReactiveCocoa是Github在制作Github客户端时开源的一个副产物，缩写为RAC。它是Objective-C语言下FRP思想的一个优秀实例，后续版本也支持了Swift语言。

> Swift语言的推出为iOS界的函数式编程爱好者迎来了曙光。著名的FRP开源库Rx系列也新增了RxSwift，保持其接口与ReactiveX.net、RxJava、RxJS接口保持一致。

#### 理解 Observable

举个简单的例子，当别人在跟你说话时，你就是那个观察者，别人就是那个 Observable，它有几个特点:

* 可能会不断地跟你说话。（onNext:）

* 可能会说错话。（onError:）

* 结束会说话。（onCompleted）

你在听到对方说的话后，也可以有几种反应：

* 根据说的话，做相应的事，比如对方让你借本书给他。（subscribe）

* 把对方说的话，加工下再传达给其他人，比如对方说小张好像不太舒服，你传达给其他人时就变成了小张失恋了。（map:）

* 参考其他人说的话再做处理，比如 A 说某家店很好吃，B 说某家店一般般，你需要结合两个人的意见再做定夺。（zip:）

#### RxSwift Workflow

大致分为这么几个阶段：先把 Native Object 变成 Observable，再通过 Observable 内置的各种强大的转换和组合能力变成新的 Observable，最后消费新的 Observable 的数据。

![Trello](https://raw.githubusercontent.com/Nirvana-icy/candyImg/master/RxAction/RxSwfitWorkflow.png)

##### Native Object -> Observable  

1⃣️  .rx extension

假设需要处理点击事件，正常的做法是给 Tap Gesture 添加一个 Target-Action，然后在那里实现具体的逻辑，这样的问题在于需要重新取名字，而且丢失了上下文。RxSwift (确切说是 RxCocoa) 给系统的诸多原生控件（包括像 URLSession）提供了 rx 扩展，所以点击的处理就变成了这样：

```Swift
let tapBackground = UITapGestureRecognizer()

tapBackground.rx.event
    .subscribe(onNext: { [weak self] _ in
        self?.view.endEditing(true)
    })
    .addDisposableTo(disposeBag)

view.addGestureRecognizer(tapBackground)
```

2⃣️  Observable.create

通过这个方法，可以将 Native 的 object 包装成 Observable，比如对网络请求的封装：

```Swift
public func response(_ request: URLRequest) -> Observable<(Data, HTTPURLResponse)> {
	return Observable.create { observer in
		let task = self.dataTaskWithRequest(request) { (data, response, error) in
			observer.on(.next(data, httpResponse))
			observer.on(.completed)
		}

		task.resume()

		return Disposables.create {
			task.cancel()
		}
	}
}
```

出于代码的简洁，略去了对 error 的处理，使用姿势类似:

```Swift
let disposeBag = DisposeBag()

response(aRequest)
  .subscribe(onNext: { data in
    print(data)
  })
  .addDisposableTo(disposeBag)
```

这里有两个注意点：

* Observerable 返回的是一个 Disposable，表示「可扔掉」的，扔哪里呢，就扔到刚刚创建的袋子里，这样当袋子被回收（dealloc）时，会顺便执行一下 Disposable.dispose()，之前创建 Disposable 时申请的资源就会被一并释放掉。

> 实际项目中 我们可以在项目中的BaseVC中 统一声明一个disposeBag变量, 其他继承于BaseVC的使用了RxSwfit的ViewController 都可以把返回的Disposable 扔到这个disposeBag中.

TBBPdaBaseVC

```Swift
class TBBPdaBaseVC: UIViewController {

    let disposeBag = DisposeBag()
  }

  // MARK: ViewController 四件套
```

TBBWayBillInputVC

```Swift
class TBBWayBillInputVC: TBBPdaBaseVC {
}

extension TBBWayBillInputVC {
    func handleDataChange() {
        // 响应 viewModel.wayBillStrVariable 变化
        self.viewModel.wayBillStrVariable.asObservable()
            .observeOn(MainScheduler.instance)    // Tips: We should always handle data changes in UI with main thread.
            .filter { !$0.isEmpty }
            .bind(to: self.wayBillTextField.rx.text)
            .disposed(by: disposeBag)

        // 响应 phoneNumStrVariable 变化
        self.viewModel.phoneNumStrVariable.asObservable()
            .observeOn(MainScheduler.instance)    // Tips: We should always handle data changes in UI with main thread.
            .bind(to: self.phoneNumTextField.rx.text)
            .disposed(by: disposeBag)
    }
  }
```

* 如果有多个 subscriber 来 subscribe response(aRequest) 那么会创建多个请求，从代码也可以看得出来，来一个 observer 就创建一个 task，然后执行。这很有可能不是我们想要的，如何让多个 subscriber 共享一个结果，这个后面会提到。

3⃣️  Variable()

Variable(value) 可以把 value 变成一个 Observable，不过前提是使用新的赋值方式 aVariable.value = newValue，来看个 Demo

```Swift
let magicNumber = 42

let magicNumberVariable = Variable(magicNumber)
magicNumberVariable.asObservable().subscribe(onNext: {
    print("magic number is \($0)")
})

magicNumberVariable.value = 73

// output
//
// magic number is 42
// magic number is 73
```

起初看到时，觉得还蛮神奇的，跟进去看了下，发现是通过 subject 来做的，大意是把 value 存到一个内部变量 _value 里，当调用 value 方法时，先更新 _value 值，然后调用内部的 _subject.on(.next(newValue)) 方法告知 subscriber。

4⃣️  Subject

Subject 简单来说是一个可以主动发射数据的 Observable，多了 onNext(value), onError(error), ‘onCompleted’ 方法，可谓全能型选手。

```Swift
let disposeBag = DisposeBag()
let subject = PublishSubject<String>()

subject.addObserver("1").addDisposableTo(disposeBag)
subject.onNext("🐶")
subject.onNext("🐱")

subject.addObserver("2").addDisposableTo(disposeBag)
subject.onNext("🅰️")
subject.onNext("🅱️")
```

记得在 RAC 时代，subject 是一个不太推荐使用的功能，因为过于强大了，容易失控。RxSwift 里倒是没有太提及，但还是少用为佳。

##### Observable -> New Observable

Observable 的强大不仅在于它能实时更新 value，还在于它能被修改／过滤／组合等，这样就能随心所欲地构造自己想要的数据，还不用担心数据发生变化了却不知道的情况。

1⃣️  Combine

Combine 就是把多个 Observable 组合起来使用，比如 zip

zip 对应现实中的例子就是拉链，拉链需要两个元素这样才能拉上去，这里也一样，只有当两个 Observable 都有了新的值时，subscribe 才会被触发。

```Swift
let stringSubject = PublishSubject<String>()
let intSubject = PublishSubject<Int>()

Observable.zip(stringSubject, intSubject) { stringElement, intElement in
	"\(stringElement) \(intElement)"
	}
	.subscribe(onNext: { print($0) })
	.addDisposableTo(disposeBag)

stringSubject.onNext("🅰️")
stringSubject.onNext("🅱️")

intSubject.onNext(1)
intSubject.onNext(2)

// output
//
// 🅰️ 1
// 🅱️ 2
```

如果这里 intSubject 始终没有执行 onNext，那么将不会有输出，就像拉链掉了一边的链子就拉不上了。

除了 zip，还有其他的 combine 的姿势，比如 combineLatest / switchLatest 等。

2⃣️  Transform

这是最常见的操作了，对一个 Observable 的数值做一些小改动，然后产出新的值，依旧是一个 Observable。

```Swift
let disposeBag = DisposeBag()
Observable.of(1, 2, 3)
    .map { $0 * $0 }
    .subscribe(onNext: { print($0) })
    .addDisposableTo(disposeBag)
```

3⃣️  Filter

Filter 的作用是对 Observable 传过来的数据进行过滤，只有符合条件的才有资格被 subscribe。

```Swift
extension TBBWayBillInputVC {
    func handleDataChange() {
        // 响应 viewModel.wayBillStrVariable 变化
        self.viewModel.wayBillStrVariable.asObservable()
            .observeOn(MainScheduler.instance)    // Tips: We should always handle data changes in UI with main thread.
            .filter { !$0.isEmpty }
            .bind(to: self.wayBillTextField.rx.text)
            .disposed(by: disposeBag)

        // 响应 phoneNumStrVariable 变化
        self.viewModel.phoneNumStrVariable.asObservable()
            .observeOn(MainScheduler.instance)    // Tips: We should always handle data changes in UI with main thread.
            .bind(to: self.phoneNumTextField.rx.text)
            .disposed(by: disposeBag)
    }
}
```

### 参考资料 & 更多内容请参考：

[是时候学习 RxSwift了](http://limboy.me/tech/2016/12/11/time-to-learn-rxswift.html)

[iOS开发下的函数响应式编程](http://williamzang.com/blog/2016/06/27/ios-kai-fa-xia-de-han-shu-xiang-ying-shi-bian-cheng/)  

[The Right Way to Architect iOS App with Swift](http://limboy.me/tech/2017/06/22/the-right-way-to-ios-architecture.html)
