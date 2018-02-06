---
layout: post
title: '使用 libffi 实现 AOP'
date: '2018-02-06'
header-img: "img/post-bg-objc.jpg"
tags:
     - ObjC
author: 'Assuner'
---

## 前言

&ensp;&ensp;&ensp;&ensp;众所周知，使用runtime的提供的接口，我们可以设定原方法的`IMP`，或交换原方法和目标方法的`IMP`，以完全代替原方法的实现，或为原实现前后相当于加一段额外的代码。

```
@interface ClassA: NSObject
- (void)methodA;
+ (void)methodB;
@end

...

@implementation ClassA (Swizzle)

+ (void)load {
    Method originalMethod = class_getInstanceMethod(self, @selector(methodA));
    Method swizzledMethod = class_getInstanceMethod(self, @selector(swizzled_methodA));
    method_exchangeImplementations(originalMethod, swizzledMethod);
}

- (void)swizzled_methodA {
    ...
    [self swizzled_methodA];
    ...
}

@end

```

&ensp;&ensp;&ensp;&ensp;使用知名的AOP库 [Aspects](https://github.com/steipete/Aspects) ，可以更便捷地为原方法实现前后增加(代替)额外的执行。

```
// hook instance method
[ClassA aspect_hookSelector:@selector(methodA)
                      withOptions:AspectPositionAfter
                       usingBlock:^{...}
                            error:nil];
  
// hook class method
[object_getClass(ClassA) aspect_hookSelector:@selector(methodB)
                      withOptions:AspectPositionAfter
                       usingBlock:^{...}
                            error:nil];
```

&ensp;&ensp;&ensp;&ensp;另外，[Aspects](https://github.com/steipete/Aspects) 支持多次hook同一个方法，支持从hook返回的`id<AspectToken>`对象删除对应的hook。</br>
&ensp;&ensp;&ensp;&ensp;`IMP`即函数指针，[Aspects](https://github.com/steipete/Aspects) 的大致原理：替换原方法的`IMP`为 **消息转发函数指针**  `_objc_msgForward`或`_objc_msgForward_stret`，把原方法`IMP`添加并对应到`SEL` `aspects_originalSelector`，将`forwardInvocation:`的`IMP`替换为参数对齐的C函数`__ASPECTS_ARE_BEING_CALLED__(NSObject *self, SEL selector, NSInvocation *invocation)`的指针。在`__ASPECTS_ARE_BEING_CALLED__`函数中，替换`invocation`的`selector`为`aspects_originalSelector`，相当于要发送调用原始方法实现的消息。对于插入位置在前面，替换，后面的多个block，构建新的blockInvocation，从invocation中提取参数，最后通过`invokeWithTarget:block`来完成依次调用。有关消息转发的介绍，可以参考笔者的另一篇文章[用代码理解ObjC中的发送消息和消息转发](https://juejin.im/post/5a308f856fb9a0452b4937cc)。</br>
&ensp;&ensp;&ensp;&ensp;[Aspects](https://github.com/steipete/Aspects) 实现代码里的很多细节处理是很令人称道的，且支持hook类的单个实例对象的方法(类似于KVO的`isa-swizzlling`)。但由于对原方法调用直接进行了消息转发，到真正的`IMP`对应的函数被执行前，经历了对其他多个消息的处理，invoke block也需要额外的invocation构建开销。作者也在注释中写道，不适合对每秒钟超过1000次的方法增加切面代码。此外，使用其他方式对Aspect hook过的方法进行hook时，如直接替换为新的`IMP`，新hook得到的原始实现是`_objc_msgForward`，之前的aspect_hook会失效，新的hook也将执行异常。</br>
&ensp;&ensp;&ensp;&ensp;那么不禁要思考，有没有一种方式可以替换原方法的`IMP`为一个和原方法参数相同(type encoding)的方法的函数指针，作为壳，处理消息时，在这个壳内部拿到所有参数，最后通过函数指针直接执行“前”、“原始/替换”，“后”的多个代码块。令人惊喜的是，[libffi](https://github.com/libffi/libffi) 可以帮我们做到这一切。

## 1. libffi 简介

&ensp;&ensp;&ensp;&ensp;[libffi](https://github.com/libffi/libffi) 可以认为是实现了C语言上的runtime，简单来说，[libffi](https://github.com/libffi/libffi) 可根据 **参数类型**(`ffi_type`)，**参数个数** 生成一个 **模板**(`ffi_cif`)；可以输入 **模板**、**函数指针** 和 **参数地址** 来直接完成 **函数调用**(`ffi_call`)； **模板** 也可以生成一个所谓的 **闭包**(`ffi_closure`)，并得到指针，当执行到这个地址时，会执行到自定义的`void function(ffi_cif *cif, void *ret, void **args, void *userdata)`函数，在这里，我们可以获得所有参数的地址(包括返回值)，以及自定义数据`userdata`。当然，在这个函数里我们可以做一些额外的操作。</br>

#### 1.1 ffi_type

&ensp;&ensp;&ensp;&ensp;根据参数个数和参数类型生成的各自的ffi_type。

```
int fun1 (int a, int b) {
    return a + b;
}

int fun2 (int a, int b) {
    return 2 * a + b;
}

...

ffi_type **types;  // 参数类型
types = malloc(sizeof(ffi_type *) * 2) ;
types[0] = &ffi_type_sint;
types[1] = &ffi_type_sint;
ffi_type *retType = &ffi_type_sint;
```

#### 1.2 ffi_call

&ensp;&ensp;&ensp;&ensp; 根据ffi_type生成特定cif，输入cif、 函数指针、参数地址动态调用函数。

```
void **args = malloc(sizeof(void *) * 2);
int x = 1, y = 2;
args[0] = &x;
args[1] = &y;

int ret;

ffi_cif cif; 
// 生成模板
ffi_prep_cif(&cif, FFI_DEFAULT_ABI, 2, retType, types); 
// 动态调用fun1
ffi_call(&cif, fun1,  &ret, args);
...

// 输出: ret = 3;
```

#### 1.3 ffi_prep_closure_loc

&ensp;&ensp;&ensp;&ensp;生成closure，并产生一个函数指针imp，当执行到imp时，获得所有输入参数, 后续将执行ffi_function。

```
void ffi_function(ffi_cif *cif, void *ret, void **args, void *userdata) {
    ...
    // args为所有参数的内存地址
}

ffi_cif cif; 
// 生成模板
ffi_prep_cif(&cif, FFI_DEFAULT_ABI, 2, returnType, types);
ffi_prep_closure_loc(_closure, &_cif, ffi_function, (__bridge void *)(self), imp);

void *imp = NULL;
ffi_closure *closure = ffi_closure_alloc(sizeof(ffi_closure), (void **)&imp);

//生成ffi_closure
ffi_prep_closure_loc(closure, &cif, ffi_function, (__bridge void *)(self), stingerIMP);
```

&ensp;&ensp;&ensp;&ensp;libffi 能调用任意 C 函数的原理与`objc_msgSend`的原理类似，其底层是用汇编实现的，`ffi_call`根据模板cif和参数值，把参数都按规则塞到栈/寄存器里，调用的函数可以按规则得到参数，调用完再获取返回值，清理数据。通过其他方式调用上文中的imp，`ffi_closure`可根据栈/寄存器、模板cif拿到所有的参数，接着执行自定义函数`ffi_function`里的代码。JPBlock的实现正是利用了后一种方式，更多细节介绍可以参考 [bang: 如何动态调用 C 函数](http://blog.cnbang.net/tech/3219/)。</br>
&ensp;&ensp;&ensp;&ensp;到这里，对于如何hook ObjC方法和实现AOP，想必大家已经有了一些思路，我们可以将`ffi_closure`关联的指针替换原方法的IMP，当对象收到该方法的消息时`objc_msgSend(id self, SEL sel, ...)`，将最终执行自定义函数`void ffi_function(ffi_cif *cif, void *ret, void **args, void *userdata)`。而实现这一切的主要工作是：设计可行的结构，存储类的多个hook信息；根据包含不同参数的方法和切面block，生成包含匹配`ffi_type`的cif；替换类某个方法的实现为`ffi_closure`关联的imp，记录hook；在`ffi_function`里，根据获得的参数，动态调用原始imp和block。</br>

```
#import <Foundation/Foundation.h>
#import "StingerParams.h"

typedef NSString *STIdentifier;

typedef NS_ENUM(NSInteger, STOption) {
  STOptionAfter = 0,
  STOptionInstead = 1,
  STOptionBefore = 2,
};

@interface NSObject (Stinger)

+ (BOOL)st_hookInstanceMethod:(SEL)sel option:(STOption)option usingIdentifier:(STIdentifier)identifier withBlock:(id)block;

+ (BOOL)st_hookClassMethod:(SEL)sel option:(STOption)option usingIdentifier:(STIdentifier)identifier withBlock:(id)block;

+ (NSArray<STIdentifier> *)st_allIdentifiersForKey:(SEL)key;

+ (BOOL)st_removeHookWithIdentifier:(STIdentifier)identifier forKey:(SEL)key;

@end
```

&ensp;&ensp;&ensp;&ensp;下文将围绕一些重要的点来介绍下笔者的实现。[Stinger](https://github.com/Assuner-Lee/Stinger)

## 2 方法签名 & ffi_type

### 2.1 方法签名 -> ffi_type

&ensp;&ensp;&ensp;&ensp;对于方法的签名和type encoding，笔者在 [用代码理解ObjC中的发送消息和消息转发](https://juejin.im/post/5a308f856fb9a0452b4937cc) 一文中已经有了不少介绍。简而言之，type encoding 字符串与方法的返回类型及参数类型是一一对应的。例如：`- (void)print1:(NSString *)s;`的type encoding为`v24@0:8@16`。`v`对应`void`，`@`对应`id`(这里是self)，`:`对应`SEL`，`@`对应`id`(这里是NSString *)，另一方面，每一种参数类型都对应一种`ffi_type`，如`v`对应`ffi_type_void`, `@`对应`ffi_type_pointer`。可以用type encoding生成一个`NSMethodSignature`实例对象，利用`numberOfArguments` 和 `- (const char *)getArgumentTypeAtIndex:(NSUInteger)idx;`方法获取每一个位置上的参数类型。当然，也可以过滤掉数字来分隔字符串`v24@0:8@16`(@?为block)，得到参数类型数组(JSPatch中使用了这一方式)。接着，我们对字符和`ffi_type`做一一对应即可完成从方法签名到ffi_type的转换。
```
_args = malloc(sizeof(ffi_type *) * argumentCount) ;
for (int i = 0; i < argumentCount; i++) {
  ffi_type* current_ffi_type = ffiTypeWithType(self.signature.argumentTypes[i]);
  NSAssert(current_ffi_type, @"can't find a ffi_type of %@", self.signature.argumentTypes[i]);
  _args[i] = current_ffi_type;
}
```
### 2.2 浅谈block
#### 2.2.1 签名 & 函数指针
```
void (^block)(id<StingerParams> params, NSString *s) = ^(id<StingerParams> params, NSString *s) {
    NSLog(@"---after2 print1: %@", s);
}
```
&ensp;&ensp;&ensp;&ensp;block是一个ObjC对象，可以认为几种block类型都继承于NSBlock。block很特殊，从表面来看包含了持有了数据和对象(暂不讨论变量捕获)，并拥有可执行的代码，调用方式类似于调用C函数，等同于数据加函数。Block类型很神秘，但我们从 [opensource-apple/objc4](https://github.com/opensource-apple/objc4) 和 [oclang/docs/block](http://clang.llvm.org/docs/Block-ABI-Apple.html) 中看到Block 完整的数据结构。

```
enum {
  BLOCK_DEALLOCATING =      (0x0001),  // runtime
  BLOCK_REFCOUNT_MASK =     (0xfffe),  // runtime
  BLOCK_NEEDS_FREE =        (1 << 24), // runtime
  BLOCK_HAS_COPY_DISPOSE =  (1 << 25), // compiler
  BLOCK_HAS_CTOR =          (1 << 26), // compiler: helpers have C++ code
  BLOCK_IS_GC =             (1 << 27), // runtime
  BLOCK_IS_GLOBAL =         (1 << 28), // compiler
  BLOCK_USE_STRET =         (1 << 29), // compiler: undefined if !BLOCK_HAS_SIGNATURE
  BLOCK_HAS_SIGNATURE  =    (1 << 30)  // compiler
};

// revised new layout

#define BLOCK_DESCRIPTOR_1 1
struct Block_descriptor_1 {
  unsigned long int reserved;
  unsigned long int size;
};

#define BLOCK_DESCRIPTOR_2 1
struct Block_descriptor_2 {
  // requires BLOCK_HAS_COPY_DISPOSE
  void (*copy)(void *dst, const void *src);
  void (*dispose)(const void *);
};

#define BLOCK_DESCRIPTOR_3 1
struct Block_descriptor_3 {
  // requires BLOCK_HAS_SIGNATURE
  const char *signature;
  const char *layout;
};

struct Block_layout {
  void *isa;
  volatile int flags; // contains ref count
  int reserved;
  void (*invoke)(void *, ...);
  struct Block_descriptor_1 *descriptor;
  // imported variables
};
```

&ensp;&ensp;&ensp;&ensp;很多人大概已经看过`BlocksKit`的代码，了解到Block对象可以强转为`Block_layout`类型，通过标识符和内存地址偏移获取block的签名`signature`。

```
NSString *signatureForBlock(id block) {
  struct Block_layout *layout = (__bridge void *)block;
  if (!(layout->flags & BLOCK_HAS_SIGNATURE))
    return nil;
  
  void *descRef = layout->descriptor;
  descRef += 2 * sizeof(unsigned long int);
  
  if (layout->flags & BLOCK_HAS_COPY_DISPOSE)
    descRef += 2 * sizeof(void *);
  
  if (!descRef)
    return nil;
  
  const char *signature = (*(const char **)descRef);
  return [NSString stringWithUTF8String:signature];
}
```

```
NSString *signature = signatureForBlock(block)
// 输出 NSString：@"v24@?0@\"<StingerParams>\"8@\"NSString\"16"

```
&ensp;&ensp;&ensp;&ensp;对于Block对象的的最简签名，我们仍然可以构建`NSMethodSignature`来逐一获取，也可以通过过滤掉数字及'\\"'来获得字符数组。
```

_argumentTypes = [[NSMutableArray alloc] init];
NSInteger descNum = 0; // num of '\"' in block signature type encoding
for (int i = 0; i < _types.length; i ++) {
    unichar c = [_types characterAtIndex:i];
    NSString *arg;
    if (c == '\"') ++descNum;
    if ((descNum % 2) != 0 || (c == '\"' || isdigit(c))) {
      continue;
 }
    ...
}  
/*@"v24@?0@\"<StingerParams>\"8@\"NSString\"16"
*/ -> v,@?,@,@
```

&ensp;&ensp;&ensp;&ensp;可以看到，签名的第一位是"@?"，意味着第一个参数为blcok自己，后面的才是blcok的参数类型。同理，我们依然可以通过type encoding匹配到对应的`ffi_type`。</br>
&ensp;&ensp;&ensp;&ensp;此外，我们可以直接获取到Block对象的函数指针。

```
BlockIMP impForBlock(id block) {
  struct Block_layout *layout = (__bridge void *)block;
  return layout->invoke;
}
```

&ensp;&ensp;&ensp;&ensp;做一个简单的尝试，直接调用Block对象的包含的函数。

```
void (^block2)(NSString *s) = ^(NSString *s) {
    NSLog(@"---after2 print1: %@", s);
  };
  void (*blockIMP) (id block, NSString *s) = (void (*) (id block, NSString *s))impForBlock(block2);
  blockIMP(block2, @"tt");
  
  // 输出：---after2 print1: tt
```

>此外，实测通过`IMP _Nonnull
imp_implementationWithBlock(id _Nonnull block)`获得的函数指针对应的参数并不包含Block对象自身，意味着签名发生了变化。

#### * 为block对象增加可用方法

&ensp;&ensp;&ensp;&ensp;通过一些方式，我们可以觉得Block对象拥有了新的实例方法。

```
NSString *signature = [block signature];
void *blockIMP = [block blockIMP];
```

&ensp;&ensp;&ensp;&ensp;做法是在`STBlock`里为`NSBlock`类增加实例方法。

```
typedef void *BlockIMP;

@interface STBlock : NSObject
+ (instancetype)new NS_UNAVAILABLE;
- (instancetype)init NS_UNAVAILABLE;

- (NSString *)signature;
- (BlockIMP)blockIMP;

NSString *signatureForBlock(id block);
BlockIMP impForBlock(id block);
@end
```

```
#define NSBlock NSClassFromString(@"NSBlock")

void addInstanceMethodForBlock(SEL sel) {
  Method m = class_getInstanceMethod(STBlock.class, sel);
  if (!m) return;
  IMP imp = method_getImplementation(m);
  const char *typeEncoding = method_getTypeEncoding(m);
  class_addMethod(NSBlock, sel, imp, typeEncoding);
}

@implementation STBlock

+ (void)load {
  addInstanceMethodForBlock(@selector(signature));
  addInstanceMethodForBlock(@selector(blockIMP));
}

...

@end
```

>这样做，可以为Block对象增加可处理的消息。但如果在其他类的load方法里尝试调用，可能会遇到STBlock类里load方法未加载的问题。

## 3 存储hook信息 & 生成两个ffi_cif对象

### 3.1 StingerInfo

&ensp;&ensp;&ensp;&ensp;这里使用简单的对象来存储单个hook信息。

```
@protocol StingerInfo <NSObject>
@required
@property (nonatomic, copy) id block;
@property (nonatomic, assign) STOption option;
@property (nonatomic, copy) STIdentifier identifier;

@optional
+ (instancetype)infoWithOption:(STOption)option withIdentifier:(STIdentifier)identifier withBlock:(id)block;
@end

@interface StingerInfo : NSObject <StingerInfo>
@end
```

### 3.2 StingerInfoPool

```
typedef void *StingerIMP;

@protocol StingerInfoPool <NSObject>

@required
@property (nonatomic, strong, readonly) NSMutableArray<id<StingerInfo>> *beforeInfos;
@property (nonatomic, strong, readonly) NSMutableArray<id<StingerInfo>> *insteadInfos;
@property (nonatomic, strong, readonly) NSMutableArray<id<StingerInfo>> *afterInfos;
@property (nonatomic, strong, readonly) NSMutableArray<NSString *> *identifiers;

@property (nonatomic, copy) NSString *typeEncoding;
@property (nonatomic) IMP originalIMP;
@property (nonatomic) SEL sel;

- (StingerIMP)stingerIMP;
- (BOOL)addInfo:(id<StingerInfo>)info;
- (BOOL)removeInfoForIdentifier:(STIdentifier)identifier;

@optional
@property (nonatomic, weak) Class cls;
+ (instancetype)poolWithTypeEncoding:(NSString *)typeEncoding originalIMP:(IMP)imp selector:(SEL)sel;
@end

@interface StingerInfoPool : NSObject <StingerInfoPool>
@end
```

#### 3.2.1 管理StingerInfo

&ensp;&ensp;&ensp;&ensp;这里利用三个数组来存储某个类hook位置在原实现前、替换、实现后的`id<StingerInfo>`对象，并保存了原始imp。添加和删除`id<StingerInfo>`对象的操作是线程安全的。

#### 3.2.2 生成方法调用模板 cif

&ensp;&ensp;&ensp;&ensp;根据原始方法提供的type encoding，生成各个参数对应的ffi_type，继而生成cif对象，最后调用`ffi_prep_closure_loc`相当于生成空壳函数`StingerIMP`。调用`StingerIMP`将最终执行到自定义的`static void ffi_function(ffi_cif *cif, void *ret, void **args, void *userdata)`函数，此函数可获得调用`StingerIMP`时获得的所有参数。

```
- (StingerIMP)stingerIMP {
  ffi_type *returnType = ffiTypeWithType(self.signature.returnType);
  NSAssert(returnType, @"can't find a ffi_type of %@", self.signature.returnType);
  
  NSUInteger argumentCount = self.signature.argumentTypes.count;
  StingerIMP stingerIMP = NULL;
  _args = malloc(sizeof(ffi_type *) * argumentCount) ;
  
  for (int i = 0; i < argumentCount; i++) {
    ffi_type* current_ffi_type = ffiTypeWithType(self.signature.argumentTypes[i]);
    NSAssert(current_ffi_type, @"can't find a ffi_type of %@", self.signature.argumentTypes[i]);
    _args[i] = current_ffi_type;
  }
  
  _closure = ffi_closure_alloc(sizeof(ffi_closure), (void **)&stingerIMP);
  
  if(ffi_prep_cif(&_cif, FFI_DEFAULT_ABI, (unsigned int)argumentCount, returnType, _args) == FFI_OK) {
    if (ffi_prep_closure_loc(_closure, &_cif, ffi_function, (__bridge void *)(self), stingerIMP) != FFI_OK) {
      NSAssert(NO, @"genarate IMP failed");
    }
  } else {
    NSAssert(NO, @"FUCK");
  }
  
  [self _genarateBlockCif];
  return stingerIMP;
}
```

#### 3.2.3 生成block调用模板 blockCif

&ensp;&ensp;&ensp;&ensp;与前面生成方法调用模板cif类似，只不过这里没有生成壳子`ffi_closure`。值得注意的是，这里把原始方法`type encoing`的第0位(`@ self`)和第1位(`: SEL`)替换为(`@？block`)和(`@ id<StingerParams>`)。意味着，限定了切面Block对象的签名类型。

```
- (void)_genarateBlockCif {
  ffi_type *returnType = ffiTypeWithType(self.signature.returnType);
  
  NSUInteger argumentCount = self.signature.argumentTypes.count;
  _blockArgs = malloc(sizeof(ffi_type *) *argumentCount);
  
  ffi_type *current_ffi_type_0 = ffiTypeWithType(@"@?");
  _blockArgs[0] = current_ffi_type_0;
  ffi_type *current_ffi_type_1 = ffiTypeWithType(@"@");
  _blockArgs[1] = current_ffi_type_1;
  
  for (int i = 2; i < argumentCount; i++){
    ffi_type* current_ffi_type = ffiTypeWithType(self.signature.argumentTypes[i]);
    _blockArgs[i] = current_ffi_type;
  }
  
  if(ffi_prep_cif(&_blockCif, FFI_DEFAULT_ABI, (unsigned int)argumentCount, returnType, _blockArgs) != FFI_OK) {
    NSAssert(NO, @"FUCK");
  }
}
```

> 在非instead位置，block的返回值可以为任意；写block时，block的第0位(不考虑block自身)参数类型应该为id，后面接的是与原方法对应的参数。

#### 3.2.4 ffi_function !!!

&ensp;&ensp;&ensp;&ensp;在这个函数里，获取到了调用原始方法时的所有入参的内存地址，先是根据`block_cif`模板生成新的参数集`innerArgs`，第0位留给`Block`对象，第1位留给`StingerParams`对象，从第2位开始复制原始的参数。</br>
&ensp;&ensp;&ensp;&ensp;以下是完成切面代码和原始imp执行的过程：</br>
&ensp;&ensp;&ensp;&ensp;1. 利用`ffi_call(&(self->_blockCif), impForBlock(block), NULL, innerArgs);`完成所有切面位置在前block的调用。使用block模板blockCif和innerArgs。</br>
&ensp;&ensp;&ensp;&ensp;2. 利用`ffi_call(cif, (void (*)(void))self.originalIMP / impForBlock(block), ret, args);`完成对原始IMP或替换位置block imp的调用。使用原始模板cif和原始参数args，并可能产生返回值。</br>
&ensp;&ensp;&ensp;&ensp;3. 利用`ffi_call(&(self->_blockCif), impForBlock(block), NULL, innerArgs);`完成所有切面位置在后的block的调用。使用block模板blockCif和innerArgs。</br>

```
static void ffi_function(ffi_cif *cif, void *ret, void **args, void *userdata) {
  StingerInfoPool *self = (__bridge StingerInfoPool *)userdata;
  NSUInteger count = self.signature.argumentTypes.count;
  void **innerArgs = malloc(count * sizeof(*innerArgs));
  
  StingerParams *params = [[StingerParams alloc] init];
  void **slf = args[0];
  params.slf = (__bridge id)(*slf);
  params.sel = self.sel;
  [params addOriginalIMP:self.originalIMP];
  NSInvocation *originalInvocation = [NSInvocation invocationWithMethodSignature:self.ns_signature];
  for (int i = 0; i < count; i ++) {
    [originalInvocation setArgument:args[i] atIndex:i];
  }
  [params addOriginalInvocation:originalInvocation];
  
  innerArgs[1] = &params;
  memcpy(innerArgs + 2, args + 2, (count - 2) * sizeof(*args));
  
  #define ffi_call_infos(infos) \
    for (id<StingerInfo> info in infos) { \
    id block = info.block; \
    innerArgs[0] = &block; \
    ffi_call(&(self->_blockCif), impForBlock(block), NULL, innerArgs); \
  }  \
  // before hooks
  ffi_call_infos(self.beforeInfos);
  // instead hooks
  if (self.insteadInfos.count) {
    id <StingerInfo> info = self.insteadInfos[0];
    id block = info.block;
    innerArgs[0] = &block;
    ffi_call(&(self->_blockCif), impForBlock(block), ret, innerArgs);
  } else {
    // original IMP
    ffi_call(cif, (void (*)(void))self.originalIMP, ret, args);
  }
  // after hooks
  ffi_call_infos(self.afterInfos);
  
  free(innerArgs);
}
```

> 注：StingerParams 对象包含了消息接收者slf，当前消息的selector sel， 还包含了可调用原始方法的invocation(使用invokeUsingIMP:完成调用)，该invocation仅适合在替换方法且需要原始返回值作参数时调用。其他hook直接使用optionBefore或after即可, 不用关注该invocation。</br>

```
#import <Foundation/Foundation.h>

#define ST_NO_RET NULL

@protocol StingerParams
@required
@property (nonatomic, unsafe_unretained) id slf;
@property (nonatomic) SEL sel;

- (void)invokeAndGetOriginalRetValue:(void *)retLoc;
@end

@interface StingerParams : NSObject <StingerParams>
- (void)addOriginalInvocation:(NSInvocation *)invocation;
- (void)addOriginalIMP:(IMP)imp;
@end
```

## 4 替换方法实现 & 记录HOOK

&ensp;&ensp;&ensp;&ensp;思路是对某个类以SEL sel为键关联一个`id<StingerInfoPool>`对象，第一次hook，新建该对象，尝试替换原方法实现为`ffi_prep_closure_loc`关联的IMP，后续hook时，将直接添加hook info到关联的`id<StingerInfoPool>`对象中。</br>
&ensp;&ensp;&ensp;&ensp;关于条件，最主要的就是两点，第一点就是对于某个类中(父类)的某个SEL sel要能找到对应Method m及IMP imp；第二点即切面block与原方法的签名是匹配的，且切面block的签名是符合要求的(isMatched方法)。

```

#import "Stinger.h"
#import <objc/runtime.h>
#import "StingerInfo.h"
#import "StingerInfoPool.h"
#import "STBlock.h"
#import "STMethodSignature.h"

@implementation NSObject (Stinger)

#pragma - public

+ (BOOL)st_hookInstanceMethod:(SEL)sel option:(STOption)option usingIdentifier:(STIdentifier)identifier withBlock:(id)block {
  return hook(self, sel, option, identifier, block);
}

+ (BOOL)st_hookClassMethod:(SEL)sel option:(STOption)option usingIdentifier:(STIdentifier)identifier withBlock:(id)block {
  return hook(object_getClass(self), sel, option, identifier, block);
}

+ (NSArray<STIdentifier> *)st_allIdentifiersForKey:(SEL)key {
  NSMutableArray *mArray = [[NSMutableArray alloc] init];
  @synchronized(self) {
    [mArray addObjectsFromArray:getAllIdentifiers(self, key)];
    [mArray addObjectsFromArray:getAllIdentifiers(object_getClass(self), key)];
  }
  return [mArray copy];
}

+ (BOOL)st_removeHookWithIdentifier:(STIdentifier)identifier forKey:(SEL)key {
  BOOL hasRemoved = NO;
  @synchronized(self) {
    id<StingerInfoPool> infoPool = getStingerInfoPool(self, key);
    if ([infoPool removeInfoForIdentifier:identifier]) {
      hasRemoved = YES;
    }
    infoPool = getStingerInfoPool(object_getClass(self), key);
    if ([infoPool removeInfoForIdentifier:identifier]) {
      hasRemoved = YES;
    }
  }
  return hasRemoved;
}

#pragma - inline functions

NS_INLINE BOOL hook(Class cls, SEL sel, STOption option, STIdentifier identifier, id block) {
  NSCParameterAssert(cls);
  NSCParameterAssert(sel);
  NSCParameterAssert(option == 0 || option == 1 || option == 2);
  NSCParameterAssert(identifier);
  NSCParameterAssert(block);
  Method m = class_getInstanceMethod(cls, sel);
  NSCAssert(m, @"SEL (%@) doesn't has a imp in Class (%@) originally", NSStringFromSelector(sel), cls);
  if (!m) return NO;
  const char * typeEncoding = method_getTypeEncoding(m);
  STMethodSignature *methodSignature = [[STMethodSignature alloc] initWithObjCTypes:[NSString stringWithUTF8String:typeEncoding]];
  STMethodSignature *blockSignature = [[STMethodSignature alloc] initWithObjCTypes:signatureForBlock(block)];
  if (! isMatched(methodSignature, blockSignature, option, cls, sel, identifier)) {
    return NO;
  }

  IMP originalImp = method_getImplementation(m);
  
  @synchronized(cls) {
    StingerInfo *info = [StingerInfo infoWithOption:option withIdentifier:identifier withBlock:block];
    id<StingerInfoPool> infoPool = getStingerInfoPool(cls, sel);
    
    if (infoPool) {
      return [infoPool addInfo:info];
    }
    
    infoPool = [StingerInfoPool poolWithTypeEncoding:[NSString stringWithUTF8String:typeEncoding] originalIMP:originalImp selector:sel];
    infoPool.cls = cls;
    
    IMP stingerIMP = [infoPool stingerIMP];
    
    if (!(class_addMethod(cls, sel, stingerIMP, typeEncoding))) {
      class_replaceMethod(cls, sel, stingerIMP, typeEncoding);
    }
    const char * st_original_SelName = [[@"st_original_" stringByAppendingString:NSStringFromSelector(sel)] UTF8String];
    class_addMethod(cls, sel_registerName(st_original_SelName), originalImp, typeEncoding);
    
    setStingerInfoPool(cls, sel, infoPool);
    return [infoPool addInfo:info];
  }
}

NS_INLINE id<StingerInfoPool> getStingerInfoPool(Class cls, SEL key) {
  NSCParameterAssert(cls);
  NSCParameterAssert(key);
  return objc_getAssociatedObject(cls, key);
}

NS_INLINE void setStingerInfoPool(Class cls, SEL key, id<StingerInfoPool> infoPool) {
  NSCParameterAssert(cls);
  NSCParameterAssert(key);
  objc_setAssociatedObject(cls, key, infoPool, OBJC_ASSOCIATION_RETAIN);
}

NS_INLINE NSArray<STIdentifier> * getAllIdentifiers(Class cls, SEL key) {
  NSCParameterAssert(cls);
  NSCParameterAssert(key);
  id<StingerInfoPool> infoPool = getStingerInfoPool(cls, key);
  return infoPool.identifiers;
}


NS_INLINE BOOL isMatched(STMethodSignature *methodSignature, STMethodSignature *blockSignature, STOption option, Class cls, SEL sel, NSString *identifier) {
  //argument count
  if (methodSignature.argumentTypes.count != blockSignature.argumentTypes.count) {
    NSCAssert(NO, @"count of arguments isn't equal. Class: (%@), SEL: (%@), Identifier: (%@)", cls, NSStringFromSelector(sel), identifier);
    return NO;
  };
  // loc 1 should be id<StingerParams>.
  if (![blockSignature.argumentTypes[1] isEqualToString:@"@"]) {
     NSCAssert(NO, @"argument 1 should be object type. Class: (%@), SEL: (%@), Identifier: (%@)", cls, NSStringFromSelector(sel), identifier);
    return NO;
  }
  // from loc 2.
  for (NSInteger i = 2; i < methodSignature.argumentTypes.count; i++) {
    if (![blockSignature.argumentTypes[i] isEqualToString:methodSignature.argumentTypes[i]]) {
      NSCAssert(NO, @"argument (%zd) type isn't equal. Class: (%@), SEL: (%@), Identifier: (%@)", i, cls, NSStringFromSelector(sel), identifier);
      return NO;
    }
  }
  // when STOptionInstead, returnType
  if (option == STOptionInstead && ![blockSignature.returnType isEqualToString:methodSignature.returnType]) {
    NSCAssert(NO, @"return type isn't equal. Class: (%@), SEL: (%@), Identifier: (%@)", cls, NSStringFromSelector(sel), identifier);
    return NO;
  }
  
  return YES;
}

@end
```

## * 使用示例

```
import UIKit;

@interface ASViewController : UIViewController

- (void)print1:(NSString *)s;

- (NSString *)print2:(NSString *)s;

@end
```

```
#import "ASViewController+hook.h"

@implementation ASViewController (hook)

+ (void)load {
  /*
   * hook @selector(print1:)
   */
  [self st_hookInstanceMethod:@selector(print1:) option:STOptionBefore usingIdentifier:@"hook_print1_before1" withBlock:^(id<StingerParams> params, NSString *s) {
    NSLog(@"---before1 print1: %@", s);
  }];
  
  [self st_hookInstanceMethod:@selector(print1:) option:STOptionBefore usingIdentifier:@"hook_print1_before2" withBlock:^(id<StingerParams> params, NSString *s) {
    NSLog(@"---before2 print1: %@", s);
  }];
  
  [self st_hookInstanceMethod:@selector(print1:) option:STOptionAfter usingIdentifier:@"hook_print1_after1" withBlock:^(id<StingerParams> params, NSString *s) {
    NSLog(@"---after1 print1: %@", s);
  }];
  
  [self st_hookInstanceMethod:@selector(print1:) option:STOptionAfter usingIdentifier:@"hook_print1_after2" withBlock:^(id<StingerParams> params, NSString *s) {
    NSLog(@"---after2 print1: %@", s);
  }];
  
  /*
   * hook @selector(print2:)
   */
  __block NSString *oldRet, *newRet;
  [self st_hookInstanceMethod:@selector(print2:) option:STOptionInstead usingIdentifier:@"hook_print2_instead" withBlock:^NSString * (id<StingerParams> params, NSString *s) {
    [params invokeAndGetOriginalRetValue:&oldRet];
    newRet = [oldRet stringByAppendingString:@" ++ new-st_instead"];
    NSLog(@"---instead print2 old ret: (%@) / new ret: (%@)", oldRet, newRet);
    return newRet;
  }];
  
  [self st_hookInstanceMethod:@selector(print2:) option:STOptionAfter usingIdentifier:@"hook_print2_after1" withBlock:^(id<StingerParams> params, NSString *s) {
    NSLog(@"---after1 print2 self:%@ SEL: %@ p: %@",[params slf], NSStringFromSelector([params sel]), s);
  }];
}
@end

```

> Stinger用法与Aspects很相似，但收到消息后，由于block和原始IMP直接使用函数指针进行调用，不处理额外的消息，不用实例化诸多NSInvocation对象，两个lib_cif对象在hook后也即准备好，相比aspects，实测有5%到50%左右的速度提升。使用其他方式hook时，仍能保证st_hook的有效性。

### 谢谢观看，水平有限，如有错误，请指正。

[https://github.com/Assuner-Lee/Stinger](https://github.com/Assuner-Lee/Stinger)
![](https://github.com/Assuner-Lee/resource/blob/master/Stinger-2.jpg)

### 参考资料

[https://github.com/opensource-apple/objc4](https://github.com/opensource-apple/objc4)</br>
[http://blog.cnbang.net/tech/3219/](http://blog.cnbang.net/tech/3219/)</br>
[https://juejin.im/post/5a308f856fb9a0452b4937cc](https://juejin.im/post/5a308f856fb9a0452b4937cc)</br>
[https://github.com/mikeash/MABlockClosure](https://github.com/mikeash/MABlockClosure)</br>


> 如有任何知识产权、版权问题或理论错误，还请指正。
>
> 转载请注明原作者及以上信息。
