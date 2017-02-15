---
layout: post
title: 'iOS Flexbox 布局优化'
date: '2017-02-12'
header-img: "img/post-bg-ios.jpg"
tags:
     - iOS
     - Flexbox
     - CSSLayout
author: 'carlSQ'
---

### Flexbox
 
Flexbox 是W3C在2009年提出的一种新的前端页面布局，目前，它已经得到了所有浏览器的支持。而最早将这一页面布局方案引入iOS开发中是开源库 [AsyncDisplayKit](https://github.com/facebook/AsyncDisplayKit/)。随着[React Native](https://github.com/facebook/react-native)与[Weex](https://github.com/alibaba/weex/)在动态化领域的兴起， 让iOS开发越来越多的接触到Flexible Box 页面布局。

### Yoga
[Yoga](https://github.com/facebook/yoga) 是由C实现的Flexbox布局引擎，目前已经被用于在React Native 和 Weex 等开源项目中。但Yoga是实现了W3C标准的一个子集。

### 基于Yoga 引擎的Flexbox 布局优化
为了充分提高Flexbox的性能和易用性，UITableView Flexbox布局滑动优化，UIScrollView Flexbox ContentSize 的优化，我们维护一个开源的扩展FlexBoxLayout。 [github链接请戳我](https://github.com/LPD-iOS/CSSLayout)。

#### Flexbox 异步计算布局
现在的iOS设备都是多核的，为了充分利用设备的多核能力，所以在布局优化的时候，支持将布局的计算放在异步线程。为了减少过多线程切换的开销，这里利用RunLoop空闲时间将计算好的布局应用到视图上。
RunLoop 在运行时，当切换时会发送通知：

```objc
typedef CF_OPTIONS(CFOptionFlags, CFRunLoopActivity) {
    kCFRunLoopEntry         = (1UL << 0), // 即将进入Loop
    kCFRunLoopBeforeTimers  = (1UL << 1), // 即将处理 Timer
    kCFRunLoopBeforeSources = (1UL << 2), // 即将处理 Source
    kCFRunLoopBeforeWaiting = (1UL << 5), // 即将进入休眠
    kCFRunLoopAfterWaiting  = (1UL << 6), // 刚从休眠中唤醒
    kCFRunLoopExit          = (1UL << 7), // 即将退出Loop
};
```
创建布局计算完成事件源，当布局完成，发送事件源，唤醒MainRunLoop, 当RunLoop处理完成了所有事件，马上要休眠时,批量处理计算好的布局。

```objc
  CFRunLoopObserverRef observer;
  
  CFRunLoopRef runLoop = CFRunLoopGetMain();
  
  CFOptionFlags activities = (kCFRunLoopBeforeWaiting | kCFRunLoopExit);
  
  observer = CFRunLoopObserverCreate(NULL,
                                     activities,
                                     YES,
                                     INT_MAX,
                                     &_messageGroupRunLoopObserverCallback,
                                     NULL);
  
  if (observer) {
    CFRunLoopAddObserver(runLoop, observer, kCFRunLoopCommonModes);
    CFRelease(observer);
  }
  
  CFRunLoopSourceContext  *sourceContext = calloc(1, sizeof(CFRunLoopSourceContext));
  
  sourceContext->perform = &sourceContextCallBackLog;
  
   _runLoopSource = CFRunLoopSourceCreate(NULL, 0, sourceContext);
  
  if (_runLoopSource) {
    CFRunLoopAddSource(runLoop, _runLoopSource, kCFRunLoopCommonModes);
  }
```
当RunLoop马上要休眠时，在_messageGroupRunLoopObserverCallback函数处理布局应用。

#### UITableView Flexbox 布局滑动优化

UITableViewCell 自动计算高度 和 UITableView滑动性能一直是一个重要的性能优化。FlexBoxLayout能自动计算高度，优化了滑动性能，UITableView在滑动时的帧率接近60FPS.

```objc

 [self.tableView fb_setCellContnetViewBlockForIndexPath:^UIView *(NSIndexPath *indexPath) {
    return [[FBFeedView alloc]initWithModel:weakSelf.sections[indexPath.section][indexPath.row]];
  }];

  ....

  - (CGFloat)tableView:(UITableView *)tableView heightForRowAtIndexPath:(NSIndexPath *)indexPath {
    return [self.tableView fb_heightForIndexPath:indexPath];
  }

  - (UITableViewCell *)tableView:(UITableView *)tableView cellForRowAtIndexPath:(NSIndexPath *)indexPath {
    return [self.tableView fb_cellForIndexPath:indexPath];
  }

```
上面的代码，做了以下几件事：

##### 开发者只需要关注Cell 的ContentView
会自动重利用cell，开发者只需要实现cell的contentView具体内容.
##### 自动计算高度
会根据Flexbox布局自动计算出高度
##### 高度缓存机制
计算出的高度会自动进行缓存，当滚动cell，重利用cell，后面对用index path cell的高度询问都会命中缓存，减少cpu的计算任务。
##### 高度缓存失效机制
当数据源发生变化时，调用以下几个api

```objc
    reloadData
    insertSections:withRowAnimation:
    deleteSections:withRowAnimation:
    reloadSections:withRowAnimation:
    moveSection:toSection:
    insertRowsAtIndexPaths:withRowAnimation:
    deleteRowsAtIndexPaths:withRowAnimation:
    reloadRowsAtIndexPaths:withRowAnimation:
    moveRowAtIndexPath:toIndexPath:
```
刷新页面时，会对已有的高度进行失效处理，并可重新计算新的缓存高度。

[Demo](https://github.com/LPD-iOS/CSSLayout) 界面的刷新一直接近60FPS
![](http://upload-images.jianshu.io/upload_images/3146026-4511acac81df3074.gif?imageMogr2/auto-orient/strip)



#### UIScrollView contentSize 自动计算

```objc

 FBLayoutDiv *root = [FBLayoutDiv layoutDivWithFlexDirection:FBFlexDirectionRow
                                              justifyContent:FBJustifySpaceAround
                                                  alignItems:FBAlignCenter
                                                    children:@[div1,div2]];

  contentView.fb_contentDiv = root;

```
设置UIScrollView的fb_contentDiv属性，当Flexbox布局计算完成应用到视图上时，
在layoutSubviews函数中
UIScrollView的contentSize 会被设置大小。

### 使用FlexboxLayout
如果你能觉得这个工具能够帮到你，他是容易整合到项目的
pod "FlexBoxLayout"
[github地址](https://github.com/LPD-iOS/CSSLayout)



> 如有任何知识产权、版权问题或理论错误，还请指正。
>
> 转载请注明原作者及以上信息。
