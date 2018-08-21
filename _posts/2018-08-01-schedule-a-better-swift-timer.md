---
layout: post
title: "Schedule: 更好的 Swift Timer"
date: 2018-08-01 00:00:00.000000000 +08:00
tags: Develop
---

我不太喜欢现在的 `Timer`。

它是标准的 OC 遗老。我不是说 OC 遗老不好，它们是 `Cocoa` 框架的基石，`ObjCRuntime` 在编程中给我们带来了诡术般的能力，可你不得不承认，`Timer` 在 Swift 世界里用起来一点儿也不顺手：

## Timer

我最近在写一个日历类的应用，它需要安排大量的定时任务：每天某时某刻，每隔几年几月，在某月某日，几分几秒后，规则繁多又复杂。在 iOS 平台上想要执行定时任务最常用的方法就是使用 `Timer` 类了。但随着使用的深入，我开始对 `Timer` 类产生了厌烦，它显然对使用者不太友好。下边的代码是一个在真实世界里使用 `Timer` 的例子：

```swift
class ViewController: NSViewController {

    weak var timer: Timer?

    override func viewDidLoad() {
        super.viewDidLoad()

        // 需要使用 target-selector 形式，self 会被不可避免地强引用 
        let timer = Timer(timeInterval: 1, target: self, selector: #selector(tick(_:)), userInfo: nil, repeats: true)
        
        // 需要添加到可用的 Runloop 上，还要设置合适的 mode
        RunLoop.current.add(timer, forMode: .commonModes)
        
        // 需要持有该 timer，以便在之后释放
        self.timer = timer
    }

    // 需要用 @objc 修饰，这一点儿都不 swift
    @objc func tick(_ timer: Timer) {
        print(timer)
    }

    // 需要主动调用该方法，在 dealloc 里的 invalidate 这个 timer 显然不行
    func invalidateTimer() {
        self.timer?.invalidate()
    }
}
```

我们可以轻易地总结出 `Timer` 中不如人意的地方：

- **语法啰嗦**，`Block` 在 `iOS 10.0, macOS 10.12` 里才被加入，在这之前只能使用 `target-selector` 形式。
- **机制复杂**，`Timer` 完全依赖于 `Runloop` 机制，所以你不仅需要保证它被添加到了一个可用的 `Runloop` 上，还需要保证 `RunLoopMode ` 的正确。
- **内存隐患**，`Timer` 会不可避免地强引用 `target`，同时 `Runloop` 又将一直持有没有 `invalidate` 的 `Timer`。内存泄漏不说近在咫尺，也算唾手可得，💀。
- **可管理性，** 你没有办法暂停一个 `Timer`，也不能改变它的时间间隔，你无法重用它，能做的，只有销毁重建。

## DispatchSoureTimer

一些别的有经验的程序员们早已厌倦了这些麻烦，他们选择了 `DispatchSoureTimer` 作为定时任务的实现，事情开起来似乎变得好点了：

```swift
class ViewController: NSViewController {

    var timer: DispatchSourceTimer?

    override func viewDidLoad() {
        super.viewDidLoad()

        let timer = DispatchSource.makeTimerSource()
        timer.schedule(deadline: .now(), repeating: 1)
        timer.setEventHandler { [weal self] in
            guard let `self` = self eles { return }
            self.elapse()
        }
        timer.activate()
    }
    
    func elapse() {
    	// timer.suspend()
    	// timer.resume()
    	timer.cancel()
    }
}
```

`Dispatch` 框架在 Swift 3 里经过了完全地重写，`DispatchSoureTimer` 的 API 相比于 `Timer` 来说简洁流畅了不止一点半点，可即使这样，恼人的地方在这里依旧存在：

- **Reference**。不再像 `Timer` 一样被 `Runloop` 持有，你需要自己管理 `DispatchSoureTimer` 的实例的引用，以免被提前释放导致计时失效。
- **Over resume**。与其它 `DispatchSouce` 一样，`timer` 对它的 `suspensions` 进行计数，`suspend` 一个 `timer` 会增加该计数，`resume` 则会消耗。当你继续 `resume` 一个 `suspensions` 已经为 0 的 `time` 时，会导致程序的 crash。
- **Deallaction**。同时当一个 `DispatchSourceTimer` 被释放时，如果它的 `suspensions` 没有为 0，程序也会直接崩溃。

## Schedule

经过这么多抱怨，我不禁思考，一个好的 Timer 到底应该长什么样？在我看来，它应该：

- 拥有足够流畅的 API。流畅的表现在于接近口语，换句话说就是 **For Humans**。
- 足够强大的可管理性。至少要支持暂停，继续，取消，重新设置时间间隔。
- 逻辑清晰，概念简单。对外暴漏的 API 要直观，也就是要符合开发者的直觉，我一向认为 API 绝不应该让使用者困惑。
- **线程安全**。我知道线程安全是一个太过泛化的问题。在我看来，如果一个框架很容易出现线程问题，那么作者就应该在框架里尽力地把线程问题解决掉，而不应该暴露出来丢给使用者负责。

从这点出发，我写了 [Schedule](https://github.com/jianstm/Schedule)，一个**正确，直观，流畅，优雅**的 `Timer` 替代品。

在 Schedule 里，设置一个定时任务不能更简单了：

```swift
Schedule.after(3.seconds).do {
    print("elapse!")
}
```

### Rules

Schedule 支持多种形式的调度，比如 Interval-based:

```swift
Schedule.every(5.seconds).do { }

Schedule.after(1.hour, repeating: 1.minute).do { }

Schedule.from([1.second, 2.minutes, 3.hours]).do { }
```

Date-based 调度：

```swift
Schedule.at(when).do { }

Schedule.of(date0, date1, date2).do { }

Schedule.every(.monday, .tuesday).at("11:11").do { }

Schedule.every(.september(30)).at("10:00:00").do { }
```

Schedule 同样提供了几个基本的集合操作符，你可以使用它们进行丰富的自定义规则调度：

```swift
/// concat
let s0 = Schedule.at(birthdate)
let s1 = Schedule.every(1.year)
let birthdaySchedule = s0.concat.s1
birthdaySchedule.do { 
    print("Happy birthday")
}

/// merge
let s3 = Schedule.every(.january(1)).at("8:00")
let s4 = Schedule.every(.october(1)).at("9:00 AM")
let holiday = s3.merge(s3)
holidaySchedule.do {
    print("Happy holiday")
}

/// first
let s5 = Schedule.after(5.seconds).concat(Schedule.every(1.day))
let s6 = s5.first(10)

/// until
let s7 = Schedule.every(.monday).at(11, 12)
let s8 = s7.until(date)
```

同时 Schedule 还支持简单的自然语言解析：

```swift
Schedule.every("one hour and ten minutes").do { }
Schedule.every("1 month, 5 days and 10 hours").do { }
```

### Managemant

在 Schedule 里，所有新建的 Task 会自动被一个内部的全局变量持有，并在你主动 `cancel` 这个 `task ` 后才会被释放。所以你不需要再写 `weak var timer: Timer?`, `self.timer = timer` 之类的声明啦：

```swift
let task = Schedule.every(1.minute).do { }

task.suspend()      // 会增加一个 task 的 suspensions 计数
task.resume()       // 会减少一个 task 的 suspensions 计数，不过你不用担心过度 resume，我会帮你处理好这个
task.cancel()        // cancel 一个 task 会把它从内部的持有者对象中移除
```

Schedule 还提供了寄生机制来处理应用中最常用的场景之一：

```swift
Schedule.every(1.second).do(host: self) {
    // 每秒调用一次，直到 self 被释放，你可以把 task 跟 controller 绑定，不用再担心手动释放的问题了
}
```

你可以为同一个 `task` 来添加更多的 `action`，并在任何你想的时候删除它：

```swift
let dailyTask = Schedule.every(1.day)
dailyTask.addAction {
	print("open eyes")
}
dailyTask.addAction {
	print("get up")
}
let key = dailyTask.addAction {
	print("take a shower")
}
dailyTask.removeAction(byKey: key)
```

也可以通过 `tag` 来管理所有 `task`：

```swift
let s = Schedule.every(1.day)
let task0 = s.do(queue: myTaskQueue, tag: "log") { }
let task1 = s.do(queue: myTaskQueue, tag: "log") { }

task0.addTag("database")
task1.removeTag("log")

Task.suspend(byTag: "log")
Task.resume(byTag: "log")
Task.cancel(byTag: "log")
```

还可以观察 `task` 的实时生命周期：

```swift
let timeline = task.timeline
print(timeline.firstExecution)
print(timeline.lastExecution)
print(timeline.estimatedNextExecution)
```

设置的 `task` 的寿命：

```swift
task.setLifetime(10.hours)  // 会在 10 个小时后被 cancel
task.addLifetime(1.hours)
task.restOfLifetime == 11.hours
```

...


通过这些，我想我已经完成了出发时的期望。

Schedule 现在还是一个非常幼稚的项目(0.0.6)，😄，不过它已经很好地满足了我在项目中的需求。你可以从 [https://github.com/jianstm/Schedule)](https://github.com/jianstm/Schedule) 获取它的源码，代码的测试覆盖率几乎 90%，注释文档也基本覆盖了所有 public 的方法。如果你有任何意见或者建议，欢迎在 [GitHub](https://github.com/jianstm/Schedule) 上开一个 issues 告诉我。

如果喜欢这个项目的话，请 [star](https://github.com/jianstm/Schedule)，然后告诉你的朋友们吧！

> ❤️ Swift
