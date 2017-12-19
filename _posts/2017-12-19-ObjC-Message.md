---
layout: post
title: '用代码理解 ObjC 中的发送消息和消息转发'
date: '2017-12-19'
header-img: "img/post-bg-objc.jpg"
tags:
     - ObjC
author: 'Assuner'
---

今天我们主要以看代码写代码的形式聊聊消ObjC中的发送消息和消息转发。

当我们向一个对象(实例对象、类对象)发送一条消息时，对象可能是处理不了的，结果就是程序发生crash。当然，通过消息转发可以预防crash。现在我们就带着几个困惑：消息发送和处理的机制是什么样的？消息转发执行的时机和包含的步骤是什么样的？(为什么实际步骤是2步而不是很多人认为的3步)？消息转发的一些细节是什么样的？下面是我分析一些开源代码并通过自己的代码实践，得出的自己的一些理解和心得。

```
 id null = [NSNull null];
 [null setObject:@2 forKey:@"2"];

 2017-12-08 10:40:34.678705+0800 test[8809:225907] -[NSNull setObject:forKey:]: 
unrecognized selector sent to instance 0x10bc2def0
```

## 尝试理解开源代码

##### 发送消息

```
 void/id objc_msgSend(void /* id self, SEL op, ... */ )   //返回值为结构体及浮点数时方法名有所不同_stret / _fpret
/*
 * Sends a message with a simple return value to an instance of a class.
 * 
 * @param self A pointer to the instance of the class that is to receive the message.
 * @param op The selector of the method that handles the message.
 * @param ... 
 *   A variable argument list containing the arguments to the method.
 * 
 * @return The return value of the method.
```

objc_msgSend的实现在objc-msg-x86.64.s文件中的汇编代码如下：

###### id objc_msgSend(id self, SEL _cmd,...)

```
/********************************************************************
 *
 * id objc_msgSend(id self, SEL	_cmd,...);
 *
 ********************************************************************/

	ENTRY	_objc_msgSend
	MESSENGER_START

	NilTest	NORMAL

	GetIsaFast NORMAL		// r11 = self->isa
	CacheLookup NORMAL		// calls IMP on success

	NilTestSupport	NORMAL

	GetIsaSupport	NORMAL

// cache miss: go search the method lists
LCacheMiss:
	// isa still in r11
	MethodTableLookup %a1, %a2	// r11 = IMP
	cmp	%r11, %r11		// set eq (nonstret) for forwarding
	jmp	*%r11			// goto *imp

	END_ENTRY	_objc_msgSend

	
	ENTRY _objc_msgSend_fixup
	int3
	END_ENTRY _objc_msgSend_fixup
```

上文中的一些宏如下：

###### GetIsaFast

```
.macro GetIsaFast
.if $0 != STRET
	testb	$$1, %a1b
	PN
	jnz	LGetIsaSlow_f
	movq	$$0x00007ffffffffff8, %r11
	andq	(%a1), %r11
.else
	testb	$$1, %a2b
	PN
	jnz	LGetIsaSlow_f
	movq	$$0x00007ffffffffff8, %r11
	andq	(%a2), %r11
.endif
LGetIsaDone:	
.endmacro
```

###### NilTest

```
.macro NilTest //藏
.if $0 == SUPER  ||  $0 == SUPER_STRET
	error super dispatch does not test for nil
.endif

.if $0 != STRET
	testq	%a1, %a1
.else
	testq	%a2, %a2
.endif
	PN
	jz	LNilTestSlow_f
.endmacro

```

###### CacheLookup

```
.macro	CacheLookup
.if $0 != STRET  &&  $0 != SUPER_STRET  &&  $0 != SUPER2_STRET
	movq	%a2, %r10		// r10 = _cmd
.else
	movq	%a3, %r10		// r10 = _cmd
.endif
	andl	24(%r11), %r10d		// r10 = _cmd & class->cache.mask
	shlq	$$4, %r10		// r10 = offset = (_cmd & mask)<<4
	addq	16(%r11), %r10		// r10 = class->cache.buckets + offset

.if $0 != STRET  &&  $0 != SUPER_STRET  &&  $0 != SUPER2_STRET
	cmpq	(%r10), %a2		// if (bucket->sel != _cmd)
.else
	cmpq	(%r10), %a3		// if (bucket->sel != _cmd)
.endif
	jne 	1f			//     scan more
	// CacheHit must always be preceded by a not-taken `jne` instruction
	CacheHit $0			// call or return imp

1:
	// loop
	cmpq	$$1, (%r10)
	jbe	3f			// if (bucket->sel <= 1) wrap or miss

	addq	$$16, %r10		// bucket++
2:	
.if $0 != STRET  &&  $0 != SUPER_STRET  &&  $0 != SUPER2_STRET
	cmpq	(%r10), %a2		// if (bucket->sel != _cmd)
.else
	cmpq	(%r10), %a3		// if (bucket->sel != _cmd)
.endif
	jne 	1b			//     scan more
	// CacheHit must always be preceded by a not-taken `jne` instruction
	CacheHit $0			// call or return imp

3:
	// wrap or miss
	jb	LCacheMiss_f		// if (bucket->sel < 1) cache miss
	// wrap
	movq	8(%r10), %r10		// bucket->imp is really first bucket
	jmp 	2f

	// Clone scanning loop to miss instead of hang when cache is corrupt.
	// The slow path may detect any corruption and halt later.

1:
	// loop
	cmpq	$$1, (%r10)
	jbe	3f			// if (bucket->sel <= 1) wrap or miss

	addq	$$16, %r10		// bucket++
2:	
.if $0 != STRET  &&  $0 != SUPER_STRET  &&  $0 != SUPER2_STRET
	cmpq	(%r10), %a2		// if (bucket->sel != _cmd)
.else
	cmpq	(%r10), %a3		// if (bucket->sel != _cmd)
.endif
	jne 	1b			//     scan more
	// CacheHit must always be preceded by a not-taken `jne` instruction
	CacheHit $0			// call or return imp

3:
	// double wrap or miss
	jmp	LCacheMiss_f

.endmacro
```

###### MethodTableLookup

```
.macro MethodTableLookup

	MESSENGER_END_SLOW
	
	SaveRegisters

	// _class_lookupMethodAndLoadCache3(receiver, selector, class)

	movq	$0, %a1
	movq	$1, %a2
	movq	%r11, %a3
	call	__class_lookupMethodAndLoadCache3

	// IMP is now in %rax
	movq	%rax, %r11

	RestoreRegisters

.endmacro
```

使用开源代码里最底层的runtime api，可以把上述过程下尽可能的逐行写成如下伪代码，如下

###### id objc_msgSend(id self, SEL _cmd,...)

```
id objc_msgSend(id self, SEL _cmd,...) {
  ① if (!self) return nil; 
  ② Class cls = self->getIsa();
     IMP imp = nil;
  ③ imp = cache_getImp(cls, sel);   
     if (imp) return imp;  
  ④ imp = _class_lookupMethodAndLoadCache3(self, _cmd, cls); 
 return imp;
}

IMP _class_lookupMethodAndLoadCache3(id obj, SEL sel, Class cls)
{        
    return lookUpImpOrForward(cls, sel, obj, 
                              YES/*initialize*/, NO/*cache*/, YES/*resolver*/);
}  // 跳过了“乐观的”无锁的查找cache过程
```

 ① `NilTest`宏，判断对象是否为nil，若为 nil，直接返回 nil。
 ② `GetIsaFast`宏快速获取到对象的 isa 指针地址（不同处理器架构存放的位置不同）
 ③ `CacheLookup`宏`_cache_getImp(Class cls, SEL sel)`包含并调用了这块代码。尝试寻找sel对应的IMP，有可能返回`_objc_msgForward_impcache`(?下文会讲到。
 ④ `MethodTableLookup`最终调用了lookUpImpOrForward方法，尝试找`method_array_t`里所有`method_list_t`中的包含sel的method_t的IMP。有可能返回`_objc_msgForward_impcache`(?下文会讲到)。
此外，我们可以猜测ObjC中IMP的定义为
`typedef id (*IMP)(...)`或者`id (*IMP)(id object, SEL sel,...)` (返回值也可能为结构体或浮点数)。

##### IMP lookUpImpOrForward(Class cls, SEL sel, id inst,vbool initialize, bool cache, bool resolver) 实现

```
IMP lookUpImpOrForward(Class cls, SEL sel, id inst, 
                       bool initialize, bool cache, bool resolver)
{
    Class curClass;
    IMP methodPC = nil;
    Method meth;
    bool triedResolver = NO;

    methodListLock.assertUnlocked();

    // Optimistic cache lookup
    if (cache) {
        methodPC = _cache_getImp(cls, sel);
        if (methodPC) return methodPC;    
    }

    // Check for freed class
    if (cls == _class_getFreedObjectClass())
        return (IMP) _freedHandler;

    // Check for +initialize
    if (initialize  &&  !cls->isInitialized()) {
        _class_initialize (_class_getNonMetaClass(cls, inst));
        // If sel == initialize, _class_initialize will send +initialize and 
        // then the messenger will send +initialize again after this 
        // procedure finishes. Of course, if this is not being called 
        // from the messenger then it won't happen. 2778172
    }

    // The lock is held to make method-lookup + cache-fill atomic 
    // with respect to method addition. Otherwise, a category could 
    // be added but ignored indefinitely because the cache was re-filled 
    // with the old value after the cache flush on behalf of the category.
 retry:
    methodListLock.lock();

    // Ignore GC selectors
    if (ignoreSelector(sel)) {
        methodPC = _cache_addIgnoredEntry(cls, sel);
        goto done;
    }

    // Try this class's cache.

    methodPC = _cache_getImp(cls, sel);
    if (methodPC) goto done;

    // Try this class's method lists.

    meth = _class_getMethodNoSuper_nolock(cls, sel);
    if (meth) {
        log_and_fill_cache(cls, cls, meth, sel);
        methodPC = method_getImplementation(meth);
        goto done;
    }

    // Try superclass caches and method lists.

    curClass = cls;
    while ((curClass = curClass->superclass)) {
        // Superclass cache.
        meth = _cache_getMethod(curClass, sel, _objc_msgForward_impcache);
        if (meth) {
            if (meth != (Method)1) {
                // Found the method in a superclass. Cache it in this class.
                log_and_fill_cache(cls, curClass, meth, sel);
                methodPC = method_getImplementation(meth);
                goto done;
            }
            else {
                // Found a forward:: entry in a superclass.
                // Stop searching, but don't cache yet; call method 
                // resolver for this class first.
                break;
            }
        }

        // Superclass method list.
        meth = _class_getMethodNoSuper_nolock(curClass, sel);
        if (meth) {
            log_and_fill_cache(cls, curClass, meth, sel);
            methodPC = method_getImplementation(meth);
            goto done;
        }
    }

    // No implementation found. Try method resolver once.

    if (resolver  &&  !triedResolver) {
        methodListLock.unlock();
        _class_resolveMethod(cls, sel, inst);
        triedResolver = YES;
        goto retry;
    }

    // No implementation found, and method resolver didn't help. 
    // Use forwarding.

    _cache_addForwardEntry(cls, sel);
    methodPC = _objc_msgForward_impcache;

 done:
    methodListLock.unlock();

    // paranoia: look for ignored selectors with non-ignored implementations
    assert(!(ignoreSelector(sel)  &&  methodPC != (IMP)&_objc_ignored_method));

    return methodPC;
}
```

通过关键点简述这个函数的查找过程.

执行起点a

*起点a 方法列表加锁(查询读取和动态添加修改方法实现互斥)，尝试忽略GC sel 

1. 从`cache_t`中寻找sel对应的IMP，如果找到，直接返回, 可能直接返回`_objc_msgForward_impcache`；
2. 在所有方法列表中(自身，categorys)使用二分法或遍历逐一寻找以`name`属性值为sel的`method_t`(Method)，如果找到，以sel为键把method存入`cache_t`, 直接执行mehtod里的IMP;

```
static method_t *search_method_list(const method_list_t *mlist, SEL sel) //藏
{
    int methodListIsFixedUp = mlist->isFixedUp();
    int methodListHasExpectedSize = mlist->entsize() == sizeof(method_t);
    
    if (__builtin_expect(methodListIsFixedUp && methodListHasExpectedSize, 1)) {
        return findMethodInSortedMethodList(sel, mlist);
    } else {
        // Linear search of unsorted method list
        for (auto& meth : *mlist) {
            if (meth.name == sel) return &meth;
        }
    }

    return nil;
}
```

3. 循环父类直到NSObject(父类为nil)，通过`_cache_getMethod`方法(返回1，IMP或nil)在父类的`cache_t`寻找以sel为键的`method_t`, 如果此时`method_t`不为1(imp属性为`_objc_msgForward_impcache时method`为1)，证明父类有执行该方法的记录，加入自己的缓存，直接调用，若为1，停止寻找。然后在父类的所有方法列表里继续寻找，如果找到IMP，加入自己的缓存并执行。
4. 如果没有找到，尝试调用自身的` _class_resolveMethod`动态为类对象或元类对象里添加方法实现。如果成功添加了method，记录已经添加过，重新从起点a出发执行;

```
void _class_resolveMethod(Class cls, SEL sel, id inst)
{
    if (! cls->isMetaClass()) {
        // try [cls resolveInstanceMethod:sel]
        _class_resolveInstanceMethod(cls, sel, inst);
    } 
    else {
        // try [nonMetaClass resolveClassMethod:sel]
        // and [cls resolveInstanceMethod:sel]
        _class_resolveClassMethod(cls, sel, inst);
        if (!lookUpImpOrNil(cls, sel, inst, 
                            NO/*initialize*/, YES/*cache*/, NO/*resolver*/)) 
        {
            _class_resolveInstanceMethod(cls, sel, inst);
        }
    }
}
```

**如果这时候还没找到sel对应的IMP imp, 把`_objc_msgForward_impcache`当做sel的实现一块加入到缓存中，并返回`_objc_msgForward_impcache`。**这也意味着，如果下次再收到该sel消息，将从缓存中直接返回`_objc_msgForward_impcache`。
 
```
 void _cache_addForwardEntry(Class cls, SEL sel) //藏
{
    cache_entry *smt;
  
    smt = (cache_entry *)malloc(sizeof(cache_entry));
    smt->name = sel;
    smt->imp = _objc_msgForward_impcache;
    if (! _cache_fill(cls, (Method)smt, sel)) {  // fixme hack
        // Entry not added to cache. Don't leak the method struct.
        free(smt);
    }
}
 ```

#### `_objc_msgForward_impcache`是什么？

上文中已经可以看出，当某种类型的对象第一次处理SEL sel消息过程中，无论如何也找不到对应的IMP imp时，便使得`_objc_msgForward_impcache`作为sel对应的imp计入缓存(下一次直接从缓存中返回)并返回。没错，它就是消息转发的函数指针，也就是说，无法顺利找到该类sel对应的实现imp时，将执行消息转发对应的imp。**从上面也可以看出，严格意义上来讲，`_class_resolveMethod ` 因为并不是`_objc_msgForward_impcache`触发的，并不能算作消息转发的后续步骤； 消息转发后，该种对象/类对象再次处理到同名消息，将直接进行消息转发(从cache_t中拿到sel对应的imp， 即`_objc_msgForward_impcache`)**。

```
/********************************************************************
*
* id _objc_msgForward(id self, SEL _cmd,...);
*
* _objc_msgForward and _objc_msgForward_stret are the externally-callable
*   functions returned by things like method_getImplementation().
* _objc_msgForward_impcache is the function pointer actually stored in
*   method caches.
*
********************************************************************/

	.non_lazy_symbol_pointer
L_forward_handler:
	.indirect_symbol __objc_forward_handler
	.long 0
L_forward_stret_handler:
	.indirect_symbol __objc_forward_stret_handler
	.long 0

	STATIC_ENTRY	__objc_msgForward_impcache
	// Method cache version
	
	// THIS IS NOT A CALLABLE C FUNCTION
	// Out-of-band condition register is NE for stret, EQ otherwise.

	MESSENGER_START
	nop
	MESSENGER_END_SLOW

	jne	__objc_msgForward_stret
	jmp	__objc_msgForward
	
	END_ENTRY	_objc_msgForward_impcache

	
	ENTRY	__objc_msgForward
	// Non-struct return version

	call	1f
1:	popl	%edx
	movl	L_forward_handler-1b(%edx), %edx
	jmp	*(%edx)

	END_ENTRY	__objc_msgForward


	ENTRY	__objc_msgForward_stret
	// Struct return version

	call	1f
1:	popl	%edx
	movl	L_forward_stret_handler-1b(%edx), %edx
	jmp	*(%edx)

	END_ENTRY	__objc_msgForward_stret
```

从源码中可以看出，`_objc_msgForward_impcache` 只是个内部的函数指针，会根据根据此时 CPU 的状态寄存器的内容来继续执行 `_objc_msgForward`或者`_objc_msgForward_stret`, 这两个才是真正的调用的消息转发的函数;且，对应的处理过程在`_forward_handler或_forward_stret_handler`里。在开源代码里，我们找到了一个默认的handler实现。貌似输出了我们熟悉的 unrecognized selector sent to instance *，但真的会执行这样薄弱的东西吗？

```
__attribute__((noreturn)) void 
objc_defaultForwardHandler(id self, SEL sel)
{
    _objc_fatal("%c[%s %s]: unrecognized selector sent to instance %p "
                "(no message forward handler is installed)", 
                class_isMetaClass(object_getClass(self)) ? '+' : '-', 
                object_getClassName(self), sel_getName(sel), self);
}
```

## 代码实践

以下是一段会发生crash的代码；

```
- (void)viewDidLoad {
  [super viewDidLoad];
  id obj = [ASClassB new];
  
  [obj performSelector:@selector(exampleInvoke:) withObject:@"1"];
  //[obj performSelector:@selector(exampleInvoke:) withObject:@"1"];
}
```

我们在该位置打上断点；

![屏幕快照 2017-12-10 下午6.33.19.png](https://user-gold-cdn.xitu.io/2017/12/13/1604db09c40d7846?w=1240&h=194&f=png&s=76364)

调试栏执行`call (void)instrumentObjcMessageSends(YES)`, 继续

![屏幕快照 2017-12-10 下午6.34.01.png](https://user-gold-cdn.xitu.io/2017/12/13/1604db09c39d0a7c?w=1090&h=92&f=png&s=27940)

在private/tmp文件夹中找到msgSends开头的文件，便知道所有的发送的消息和对象的日志。(下图截取了一部分)

![屏幕快照 2017-12-10 下午6.43.51.png](https://user-gold-cdn.xitu.io/2017/12/13/1604db09c60762e2?w=940&h=368&f=png&s=87440)

可以看到，通过`performSelector:`向ASClassA发送`exampleInvoke:`消息后，陆续调用了`resolveInstanceMethod:``forwardingTargetForSelector:``methodSignatureForSelector:``class``doesNotRecognizeSelector:`方法。

```
// Replaced by CF (throws an NSException)
- (void)doesNotRecognizeSelector:(SEL)sel {
    _objc_fatal("-[%s %s]: unrecognized selector sent to instance %p", 
                object_getClassName(self), sel_getName(sel), self);
}
```

通过源码发现，是`doesNotRecognizeSelector:`抛出异常终止了程序并给出了提示!可以猜测，实际上那个名字为default的handler并没有执行。那么如何验证上述消息转发过程呢，很简单，我们可以写一层层的简单的消息转发来防止crash。

##### forwardingTargetForSelector:

```
#import "ASClassB.h"
#import "ASClassA.h"
#import <objc/runtime.h>

@implementation ASClassB

- (id)forwardingTargetForSelector:(SEL)aSelector {
  if (aSelector == @selector(exampleInvoke:)) {
    return [ASClassA new];
  }
  return [super forwardingTargetForSelector:aSelector];
}
@end

@implementation ASClassA

- (void)exampleInvoke:(NSString *)text {
  NSLog(@"ASClassA receive exampleIncoke:");
}
@end
```

我们重写了ASClassB的`forwardingTargetForSelector:`方法，尝试把消息转发给实际上已经实现了`exampleInvoke:`的ASClass类的一个对象。和上文调试步骤一样，我们对objA执行两次方法。

![屏幕快照 2017-12-10 下午7.12.32.png](https://user-gold-cdn.xitu.io/2017/12/13/1604db09c22ab99c?w=1240&h=159&f=png&s=81019)

执行结果：

![屏幕快照 2017-12-10 下午7.18.19.png](https://user-gold-cdn.xitu.io/2017/12/13/1604db09c71a897b?w=1206&h=126&f=png&s=33005)

第一次：

```
- ASClassB NSObject performSelector:withObject:
+ ASClassB NSObject resolveInstanceMethod:
+ ASClassB NSObject resolveInstanceMethod:
- ASClassB ASClassB forwardingTargetForSelector:
- ASClassB ASClassB forwardingTargetForSelector:
+ ASClassA NSObject initialize
+ ASClassA NSObject new
- ASClassA NSObject init
- ASClassA ASClassA exampleInvoke:
```

第二次：

```
- ASClassB NSObject performSelector:withObject:
- ASClassB ASClassB forwardingTargetForSelector:
- ASClassB ASClassB forwardingTargetForSelector:
+ ASClassA NSObject new
- ASClassA NSObject init
- ASClassA ASClassA exampleInvoke:
```

可以发现，第一点，没有执行`methodSignatureForSelector:`方法，因为`forwardingTargetForSelector:`方法已经返回了能正确处理消息的对象；第二点，obj第二次收到`exampleInvoke:`消息时，直接进行进行了消息转发。原因正是上文中提到的首次未找到sel对应的imp时，直接把消息转发的imp和sel一块放在了类对象/元对象的`cache_t`中。

##### methodSignatureForSelector: & forwardInvocation:

实测，在未重写`forwardingTargetForSelector:`或该方法提供对象不能处理该消息时(返回nil无效)，便会陆续执行`methodSignatureForSelector:`和 `forwardInvocation:`方法。

```
#import "ASClassB.h"
#import "ASClassA.h"
#import <objc/runtime.h>

@implementation ASClassB

- (NSMethodSignature *)methodSignatureForSelector:(SEL)aSelector {
  if (aSelector == @selector(exampleInvoke:)) {
    return [NSMethodSignature signatureWithObjCTypes:"v@:"];
  }
  return [super methodSignatureForSelector:aSelector];
}

- (void)forwardInvocation:(NSInvocation *)anInvocation {
  if (anInvocation.selector == @selector(exampleInvoke:)) {
     [anInvocation invokeWithTarget:[ASClassA new]];
  } else {
    [super forwardInvocation:anInvocation];
  }
}anInvocation invokeWithTarget:[ASClassA new]];
}
@end
```

这个简单的demo可以实现正确的消息转发。通过重写`methodSignatureForSelector:`方法返回一个可用的方法签名，通过`forwardInvocation:`将incovation（后面介绍）完成一个完整的发送消息过程。我们甚至可以重写这两个方法完成所有未知消息的消息转发, 不再crash。

```
- (NSMethodSignature *)methodSignatureForSelector:(SEL)aSelector {
    return [NSMethodSignature signatureWithObjCTypes:"v@:"];
}

- (void)forwardInvocation:(NSInvocation *)anInvocation {
  [anInvocation invokeWithTarget:nil];
#if DEBUG
  NSLog(@"[%@ %@] unrecognized selector sent to instance %@", self.class, NSStringFromSelector(anInvocation.selector), self);
  [NSException raise:@"UnrecognizedSelector" format:@"[%@ %@] unrecognized selector sent to instance %@", self.class, NSStringFromSelector(anInvocation.selector), self];
#endif
}
```

后来我们也看到了`forwardInvocation:`的调用过程

```
- ASClassB ASClassB forwardInvocation:
+ NSInvocation NSInvocation _invocationWithMethodSignature:frame:
+ NSInvocation NSObject alloc
- NSMethodSignature NSObject retain
- NSMethodSignature NSMethodSignature frameLength
- NSMethodSignature NSMethodSignature _frameDescriptor
- NSMethodSignature NSMethodSignature frameLength
- NSMethodSignature NSMethodSignature _frameDescriptor
- NSInvocation NSObject autorelease
- ASClassB ASClassB forwardInvocation:
- NSInvocation NSInvocation invokeWithTarget:
- NSInvocation NSInvocation setArgument:atIndex:
- NSMethodSignature NSMethodSignature numberOfArguments
- NSMethodSignature NSMethodSignature _frameDescriptor
- NSMethodSignature NSMethodSignature _argInfo:
- NSMethodSignature NSMethodSignature _frameDescriptor
- NSInvocation NSInvocation invoke

```

提到几个点，`invokeWithTarget:`在这里，是可以转发给nil的，毕竟nil收到任何消息后会直接返回nil。然后注意到，在这里的invocation调用过程，此处的`methodSignature`的`types`只需设成`"v@:"或"v@"`(如果不取SEL)，相当于`- (id)m；只要不在anInvocation里取和设方法参数，并不会发生数组越界，也不会影响多个变量传递给新的target，系统执行时应该把参数放置在了一个更高效的位置，incocation取时也只相当于一个懒加载的getter;  另外，NSNull+NullSafe扩展采用了遍历所有类来寻找能响应未知消息的类对象来转发消息，并做了缓存优化。

#### 简单讲下 NSMethodSignature  & NSInvocation

###### NSMethodSignature

A record of the type information for the return value and parameters of a method. 官方文档定义：一个对于方法返回值和参数的记录。

```
Method m = class_getInstanceMethod(NSString.class, @selector(initWithFormat:));
const char *c = method_getTypeEncoding(m);
NSMethodSignature* sg = [[NSString new] methodSignatureForSelector:@selector(initWithFormat:)];
```

输出c和m, 得到：

```
(lldb) po c
"@24@0:8@16"

(lldb) po sg
<NSMethodSignature: 0x600000273880>
    number of arguments = 3
    frame size = 224
    is special struct return? NO
    return value: -------- -------- -------- --------
        type encoding (@) '@'
        flags {isObject}
        modifiers {}
        frame {offset = 0, offset adjust = 0, size = 8, size adjust = 0}
        memory {offset = 0, size = 8}
    argument 0: -------- -------- -------- --------
        type encoding (@) '@'
        flags {isObject}
        modifiers {}
        frame {offset = 0, offset adjust = 0, size = 8, size adjust = 0}
        memory {offset = 0, size = 8}
    argument 1: -------- -------- -------- --------
        type encoding (:) ':'
        flags {}
        modifiers {}
        frame {offset = 8, offset adjust = 0, size = 8, size adjust = 0}
        memory {offset = 0, size = 8}
    argument 2: -------- -------- -------- --------
        type encoding (@) '@'
        flags {isObject}
        modifiers {}
        frame {offset = 16, offset adjust = 0, size = 8, size adjust = 0}
        memory {offset = 0, size = 8}

```

c = `"@24@0:8@16"`, 数字代表着相对于地址的偏移量，由下边的sg可以看出，第一位@代表返回值(实际是argument -1)，第二位 argument 0是id self, argument 1是SEL sel, argument 2是id arg。为什么会这样，我们接下来会验证，这仿佛又与` id objc_msgSend(id self, SEL op, ... */ ) `的参数顺序是一致的...可以认为方法签名就是个方法的模板记录。关于type encoding，有以下资料:

```
#define _C_ID       '@'
#define _C_CLASS    '#'
#define _C_SEL      ':'
#define _C_CHR      'c'
#define _C_UCHR     'C'
#define _C_SHT      's'
#define _C_USHT     'S'
#define _C_INT      'i'
#define _C_UINT     'I'
#define _C_LNG      'l'
#define _C_ULNG     'L'
#define _C_LNG_LNG  'q'
#define _C_ULNG_LNG 'Q'
#define _C_FLT      'f'
#define _C_DBL      'd'
#define _C_BFLD     'b'
#define _C_BOOL     'B'
#define _C_VOID     'v'
#define _C_UNDEF    '?'
#define _C_PTR      '^'
#define _C_CHARPTR  '*'
#define _C_ATOM     '%'
#define _C_ARY_B    '['
#define _C_ARY_E    ']'
#define _C_UNION_B  '('
#define _C_UNION_E  ')'
#define _C_STRUCT_B '{'
#define _C_STRUCT_E '}'
#define _C_VECTOR   '!'
#define _C_CONST    'r'
```

总之这些不同字符代表不同类型啦。例如':'代表SEL，证明了argument 1确实是sel，@代表'id'等。例如-(BOOL)isKindOfClass:(Class)cls;的type encoding为"B@:#"。

###### NSInvocation。

An Objective-C message rendered as an object. 
呈现为对象的消息，可以存储消息的所有配置和直接调用给任意对象(真tm是万物皆对象啊)。
输出上文中得到的anInvocation:

```

//type: @v:@

id obj = [ASClassB new];
[obj performSelector:@selector(exampleInvoke:) withObject:@"1"];

----------------------------------------
id x;
id y;
id z;
[anInvocation getArgument:&x atIndex:0];
[anInvocation getArgument:&y atIndex:1];
[anInvocation getArgument:&z atIndex:2];
---------------------------------------- 

(lldb) po anInvocation
<NSInvocation: 0x604000460780>
return value: {v} void
target: {@} 0x6040000036e0
selector: {:} exampleInvoke:
argument 2: {@} 0x10e8ec340

(lldb) po x
<ASClassB: 0x60400000eb10>

(lldb) po anInvocation.selector
"exampleInvoke:"

(lldb) po NSStringFromSelector(y)
exampleInvoke:

(lldb) po z
1

(lldb) po anInvocation.methodSignature
<NSMethodSignature: 0x604000464c40>
    number of arguments = 3
    frame size = 224
    is special struct return? NO
    return value: -------- -------- -------- --------
        type encoding (v) 'v'
        flags {}
        modifiers {}
        frame {offset = 0, offset adjust = 0, size = 0, size adjust = 0}
        memory {offset = 0, size = 0}
    argument 0: -------- -------- -------- --------
        type encoding (@) '@'
        flags {isObject}
        modifiers {}
        frame {offset = 0, offset adjust = 0, size = 8, size adjust = 0}
        memory {offset = 0, size = 8}
    argument 1: -------- -------- -------- --------
        type encoding (:) ':'
        flags {}
        modifiers {}
        frame {offset = 8, offset adjust = 0, size = 8, size adjust = 0}
        memory {offset = 0, size = 8}
    argument 2: -------- -------- -------- --------
        type encoding (@) '@'
        flags {isObject}
        modifiers {}
        frame {offset = 16, offset adjust = 0, size = 8, size adjust = 0}
        memory {offset = 0, size = 8}
```

由此可以看出上文描述方法签名前几位位置代表的意义是完全正确的。

此外我们也可以自己手动构建invocation，实现多参数方法的动态执行。总之这个类很强大，后续文章我们还会提到。

```
NSString *text = @"string";
SEL sel = @selector(stringByAppendingString:);
NSMethodSignature *sg = [text methodSignatureForSelector:sel];
NSInvocation *invocation = [NSInvocation invocationWithMethodSignature:sg];
invocation.target = text;
invocation.selector = sel;
id p = @"SS";
[invocation setArgument:&p atIndex:2];
id r;
[invocation invoke];
if (sg.methodReturnLength) {
  [invocation getReturnValue:&r];
}
-----------------------------------------------
(lldb) po r
stringSS

(lldb) 
```

和上面分析的一样，方法的参数index从2开始。

##### 尝试手动触发消息转发

前面我们已经知道，如果method的imp为`__objc_msgForward`, 将直接触发消息转发。

下面我们直接替换ASClassA的`@selector(print)`的实现为`__objc_msgForward`,然后替换该类@selector(forwardInvocation:)对应的imp为我们自己实现的函数。

```
@implementation ASClassA
- (void)print {
  NSLog(@"ASClassA print");
}
```
```
void forward(id obj, SEL sel, NSInvocation *invo) {
  if (invo.selector == @selector(print)) {
    NSLog(@"hahhahahahhaha");
  }
}

- (void)viewDidLoad {
  [super viewDidLoad];
  class_replaceMethod(ASClassA.class, @selector(print), _objc_msgForward, "v@:");
  
  class_replaceMethod(ASClassA.class, @selector(forwardInvocation:), (IMP)forward,"v@:@");
  ASClassA *obj = [ASClassA new];
  [obj performSelector:@selector(print)];
}
```

结果为:

```
(lldb) call (void)instrumentObjcMessageSends(YES)
2017-12-10 23:20:47.625463+0800 test[12136:765892] hahhahahahhaha
(lldb) 
```

执行过程为：

```
 ASClassA NSObject performSelector:
- ASClassA ASClassA print
- ASClassA NSObject forwardingTargetForSelector:
- ASClassA NSObject forwardingTargetForSelector:
- ASClassA NSObject methodSignatureForSelector:
- ASClassA NSObject methodSignatureForSelector:
...
- ASClassA ASClassA forwardInvocation:

```

print方法直接跳到了我们的自定义函数代码实现上，消息转发成功。上述只是一个简单的例子，如果自定义的函数里根据每个invocation的SEL名字动态化新建一个包含完整代码完全不同的invocation，功能将会异常强大。实际上JSPatch的某些核心部分也正是使用了这种方式直接替换掉某些类里的方法实现。

#### 谢谢观看！！如有问题请多指教！！

###### 参考文献

[https://github.com/RetVal/objc-runtime](https://github.com/RetVal/objc-runtime)
[https://github.com/opensource-apple/objc4](https://github.com/opensource-apple/objc4)
[https://developer.apple.com/documentation](https://developer.apple.com/documentation)
[可以参考的反编译代码](http://yulingtianxia.com/blog/2016/06/15/Objective-C-Message-Sending-and-Forwarding/)


> 如有任何知识产权、版权问题或理论错误，还请指正。
>
> 转载请注明原作者及以上信息。
