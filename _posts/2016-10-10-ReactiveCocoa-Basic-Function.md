---
layout: post
title: '图解 ReactiveCocoa 基本函数'
date: '2016-10-10'
header-img: "img/post-bg-android.jpg"
tags:
     - iOS
     - ReactiveCocoa
author: '稻子'
---

标签（空格分隔）： RAC FRP 函数式编程 响应式编程

> 本文内容仅适用于[ReactiveCocoa v2.5](https://github.com/ReactiveCocoa/ReactiveCocoa/tree/v2.5)

---
关于函数响应式编程（FRP），可以参考

* [What is (functional) reactive programming?](http://stackoverflow.com/questions/1028250/what-is-functional-reactive-programming/1030631#1030631)
* [Specification for a Functional Reactive Programming language](http://stackoverflow.com/questions/5875929/specification-for-a-functional-reactive-programming-language#5878525)

![图片来自互联网][1]
> Streams of values over time

ReactiveCocoa repo上最简单的一句话对FRP做了本质的描述，而repo本身提供的是APIs of composing and transforming streams of values，而Streams of values的抽象在ReactiveCocoa中应该是RACStream，而composing and transforming是本文的重点。

先解释ReactiveCocoa中的两个基本概念

* 信号（Signal） a signal is a steam of values，signals can be transformed, combined,etc.

* 订阅者（Subscriber） a subscriber subscribes to a signal. RAC lets blocks,objects, and properties subscribe to signals

##filter
```objective-c
RACSignal *signal = [@[ @1, @2, @3 ] rac_sequence].signal; signal = [signal filter:^BOOL(NSNumber *value) {
    return value.integerValue % 2;
}];
[signal subscribeNext:^(NSNumber *value) {
    NSLog(@"%@", value);
}];
```
![filter][2]

##map
```objective-c
  RACSignal *signal = [@[ @1, @2, @3 ] rac_sequence].signal;
  signal = [signal map:^id(NSNumber *value) {
    return @(value.integerValue * 2);
  }];
  [signal subscribeNext:^(NSNumber *value) {
    NSLog(@"%@", value);
  }];
```
![map][3]

##merge
```objective-c
RACSignal *signal1 = [@[ @1, @2 ] rac_sequence].signal;
RACSignal *signal2 = [@[ @4, @5 ] rac_sequence].signal;

[[signal1 merge:signal2] subscribeNext:^(NSNumber *value) {
    NSLog(@"%@", value);
}];
```
![merge][4]

##combineLatest
```objective-c
  RACSignal *signal1 = [@[ @1, @2 ] rac_sequence].signal;
  RACSignal *signal2 = [@[ @3, @4 ] rac_sequence].signal;

  [[signal1 combineLatestWith:signal2] subscribeNext:^(RACTuple *value) {
    NSLog(@"%@", value);
  }];
```
![combineLatest][5]

##combineLatest & reduce
```objective-c
  RACSignal *signal1 = [@[ @1, @2 ] rac_sequence].signal;
  RACSignal *signal2 = [@[ @3, @4 ] rac_sequence].signal;

  [[[signal1 combineLatestWith:signal2]
      reduceEach:^id(NSNumber *v1, NSNumber *v2) {
        return @(v1.integerValue * v2.integerValue);
      }] subscribeNext:^(RACTuple *value) {
    NSLog(@"%@", value);
  }];
```
![combineLatest & reduce][6]

##flatten
```objective-c
RACSignal *signal1 = [@[ @1, @2 ] rac_sequence].signal;
RACSignal *signal2 = [RACSignal return:signal1];

[[signal2 flatten] subscribeNext:^(NSNumber *value) {
    NSLog(@"%@", value);
}];
```
![flatten][7]

##flattenMap
```objective-c
RACSignal *signal = [@[ @1, @2 ] rac_sequence].signal;

[[signal flattenMap:^RACStream *(NSNumber *value) {
    return [RACSignal return:@(value.integerValue * 2)];
}] subscribeNext:^(NSNumber *value) {
    NSLog(@"%@", value);
}];
```
![flattenMap][8]

##not replay
```objective-c
RACSubject *letters = [RACSubject subject];
RACSignal *signal = letters;
[signal subscribeNext:^(id x) {
    NSLog(@"S1: %@", x);
}];
[letters sendNext:@"A"];
[signal subscribeNext:^(id x) {
    NSLog(@"S2: %@", x);
}];
[letters sendNext:@"B"];
[letters sendNext:@"C"];
```
![not replay][9]

##replay
```objective-c
RACSubject *letters = [RACReplaySubject subject];
RACSignal *signal = letters;
[signal subscribeNext:^(id x) {
  NSLog(@"S1: %@", x);
}];
[letters sendNext:@"A"];
[signal subscribeNext:^(id x) {
  NSLog(@"S2: %@", x);
}];
[letters sendNext:@"B"];
[signal subscribeNext:^(id x) {
  NSLog(@"S3: %@", x);
}];
[letters sendNext:@"C"];
```
![replay][10]

##replayLast
```objective-c
RACSubject *letters = [RACSubject subject];
RACSignal *signal = [letters replayLast];
[signal subscribeNext:^(id x) {
  NSLog(@"S1: %@", x);
}];
[letters sendNext:@"A"];
[signal subscribeNext:^(id x) {
  NSLog(@"S2: %@", x);
}];
[letters sendNext:@"B"];
[signal subscribeNext:^(id x) {
  NSLog(@"S3: %@", x);
}];
[letters sendNext:@"C"];
```
![replayLast][11]

##replayLazily
```objective-c
RACSubject *letters = [RACSubject subject];
RACSignal *signal = [letters replayLazily];
[letters sendNext:@"A"];
[signal subscribeNext:^(id x) {
  NSLog(@"S1: %@", x);
}];
[letters sendNext:@"B"];
[signal subscribeNext:^(id x) {
  NSLog(@"S2: %@", x);
}];
[letters sendNext:@"C"];
[signal subscribeNext:^(id x) {
  NSLog(@"S3: %@", x);
}];
[letters sendNext:@"D"];
```
![replayLazily][12]

##zip
```objective-c
RACSubject *letters = [RACSubject subject];
RACSubject *numbers = [RACSubject subject];

RACSignal *combined =
    [RACSignal zip:@[ letters, numbers ]
            reduce:^(NSString *letter, NSString *number) {
              return [letter stringByAppendingString:number];
            }];

// Outputs: A1 B2 C3
[combined subscribeNext:^(id x) {
  NSLog(@"%@", x);
}];

[letters sendNext:@"A"];
[letters sendNext:@"B"];
[letters sendNext:@"C"];
[numbers sendNext:@"1"];
[numbers sendNext:@"2"];
[numbers sendNext:@"3"];
```
![zip][13]

完整的源码可以在这里找到 [Demo](https://gist.github.com/foxsofter/5ece717adbd5546d0c22f8d4b1dce5ae)

  [1]: https://s3.amazonaws.com/media-p.slid.es/uploads/263775/images/1763829/687474703a2f2f692e696d6775722e636f6d2f4149696d5138432e6a7067.jpeg
  [2]: https://foxsofter.github.io/images/filter.png
  [3]: https://foxsofter.github.io/images/map.png
  [4]: https://foxsofter.github.io/images/merge.png
  [5]: https://foxsofter.github.io/images/combineLatest.png
  [6]: https://foxsofter.github.io/images/combineLatest&reduce.png
  [7]: https://foxsofter.github.io/images/flatten.png
  [8]: https://foxsofter.github.io/images/flattenMap.png
  [9]: https://foxsofter.github.io/images/not_replay.png
  [10]: https://foxsofter.github.io/images/replay.png
  [11]: https://foxsofter.github.io/images/replayLast.png
  [12]: https://foxsofter.github.io/images/replayLazily.png
  [13]: https://foxsofter.github.io/images/zip.png
	"-g"选项 : 查看日志缓冲区信息;

	"-b"选项 : 加载一个日志缓冲区, 默认是 main, 下面详解;

	"-B"选项 : 以二进制形式输出日志;



> 如有任何知识产权、版权问题或理论错误，还请指正。
>
> 转载请注明原作者及以上信息。
