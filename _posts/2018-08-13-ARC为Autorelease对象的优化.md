---
title:  "ARC为Autorelease对象的优化"
date:   2018-07-31 15:13:21
categories: ARC Sorce Prob
---

## 一个问题
> 什么样的对象是Autorelease对象？

说到这个问题我们都知道除了以`alloc/new/copy/mutableCopy`或者方法名以`alloc/new/copy/mutableCopy`开头的驼峰法方法创建的对象都是autorelease对象。

### why？

我们先看一个例子：

```Objective-C

NSDictionary* dic = [NSDictionary dictionary];
//MRC
[dic retain];
...
...
[dic release];

```

NSDictionary的`dictionary`方法是一个`convenience`构造方法。它在类的内部帮你创建了一个字典返回出来。秉承着谁创建谁释放的原则，这个在类内部创建的对象实例，应该走完构造方法之后就被释放了。这样就没有办法在外部引用或者使用了。但是又不能不释放，不然就内存泄漏了。怎么办？聪明的小伙伴都知道autoreleasepool可以延迟对象的释放（直到当前线程的runloop结束时调用对应的自动释放池的`drain`）。所以通过这种方式new出来的对象会被加入到AutoreleasePool中，留给外部调用者引用它的机会。

话说ARC不是为我们的内存管理做了很多优化么，那么这种情况它有做什么优化吗？

## 各种情况下的autorelease对象 

### 事前准备
我们先创建`Foo`这个类，通过它我们将模拟四种（有无外部引用、是否是属性）情况下的autorelease对象，并通过源码追踪它的行为。

```Objective-C

@interface Foo : NSObject

+ (instancetype)createFoo;

@end

@implementation Foo

+ (instancetype)createFoo {
    return [[self alloc] init];
}

int main(int argc, char * argv[]) {
	...
	[self test]
}

```


### 无引用非属性
```Objective-C

- (void)testFoo {
    [Foo createFoo];
}

```
通过Xcode的`Product->Perform Action->Assemble“ViewController.m”`，我们可以看到OC源文件(viewController.m)最终被编编译生产的汇编代码，这里就能详细的查看到底编译器在我们的代码背后插入了哪些代码.

```C
LPC0_5:
	add	r2, pc
	.loc	3 20 5                  @ /Users/wansheng/Desktop/testDemo/testDemo/ViewController.m:20:5
	ldr	r2, [r2]
	ldr	r1, [r1]
	str	r0, [sp, #4]            @ 4-byte Spill
	mov	r0, r2
	ldr	r2, [sp, #4]            @ 4-byte Reload
	blx	r2
	.loc	3 20 5 is_stmt 0 discriminator 1 @ /Users/wansheng/Desktop/testDemo/testDemo/ViewController.m:20:5
	@ InlineAsm Start
	mov	r7, r7	@ marker for objc_retainAutoreleaseReturnValue
	@ InlineAsm End
	.loc	3 20 5 discriminator 2  @ /Users/wansheng/Desktop/testDemo/testDemo/ViewController.m:20:5
	bl	_objc_unsafeClaimAutoreleasedReturnValue
	movw	r1, :lower16:(L__unnamed_cfstring_-(LPC0_6+4))
	movt	r1, :upper16:(L__unnamed_cfstring_-(LPC0_6+4))

"+[Foo Foo]":
Lfunc_begin0:
	.loc	3 18 0                  @ /Users/wansheng/Desktop/testDemo/testDemo/Foo.m:18:0
@ BB#0:
	push	{r7, lr}
	mov	r7, sp
	sub	sp, #8
	@DEBUG_VALUE: +[Foo Foo]:self <- [%SP+4]
	@DEBUG_VALUE: +[Foo Foo]:_cmd <- [%SP+0]
	str	r0, [sp, #4]
	str	r1, [sp]
Ltmp0:
	.loc	3 19 13 prologue_end    @ /Users/wansheng/Desktop/testDemo/testDemo/Foo.m:19:13
	movw	r0, :lower16:(L_OBJC_CLASSLIST_REFERENCES_$_-(LPC0_0+4))
	movt	r0, :upper16:(L_OBJC_CLASSLIST_REFERENCES_$_-(LPC0_0+4))
LPC0_0:
	add	r0, pc
	ldr	r0, [r0]
	movw	r1, :lower16:(L_OBJC_SELECTOR_REFERENCES_-(LPC0_1+4))
	movt	r1, :upper16:(L_OBJC_SELECTOR_REFERENCES_-(LPC0_1+4))
LPC0_1:
	add	r1, pc
	ldr	r1, [r1]
	bl	_objc_msgSend
	.loc	3 19 12 is_stmt 0       @ /Users/wansheng/Desktop/testDemo/testDemo/Foo.m:19:12
	movw	r1, :lower16:(L_OBJC_SELECTOR_REFERENCES_.2-(LPC0_2+4))
	movt	r1, :upper16:(L_OBJC_SELECTOR_REFERENCES_.2-(LPC0_2+4))
LPC0_2:
	add	r1, pc
	ldr	r1, [r1]
	.loc	3 19 12 discriminator 1 @ /Users/wansheng/Desktop/testDemo/testDemo/Foo.m:19:12
	bl	_objc_msgSend
	.loc	3 19 5 discriminator 2  @ /Users/wansheng/Desktop/testDemo/testDemo/Foo.m:19:5
	add	sp, #8
Ltmp1:
	pop.w	{r7, lr}
	b.w	_objc_autoreleaseReturnValue	

```

> 注意这里的mov r7, r7.这行代码，具体的含义在后面我们会说到。

同样的方式再查看下Foo.m里面`createFoo`方法的代码。

```c

"+[Foo createFoo]":
Lfunc_begin0:
	.loc	3 18 0                  @ /Users/wansheng/Desktop/testDemo/testDemo/Foo.m:18:0
@ BB#0:
	push	{r7, lr}
	mov	r7, sp
	sub	sp, #8
	@DEBUG_VALUE: +[Foo Foo]:self <- [%SP+4]
	@DEBUG_VALUE: +[Foo Foo]:_cmd <- [%SP+0]
	str	r0, [sp, #4]
	str	r1, [sp]
Ltmp0:
	.loc	3 19 13 prologue_end    @ /Users/wansheng/Desktop/testDemo/testDemo/Foo.m:19:13
	movw	r0, :lower16:(L_OBJC_CLASSLIST_REFERENCES_$_-(LPC0_0+4))
	movt	r0, :upper16:(L_OBJC_CLASSLIST_REFERENCES_$_-(LPC0_0+4))
LPC0_0:
	add	r0, pc
	ldr	r0, [r0]
	movw	r1, :lower16:(L_OBJC_SELECTOR_REFERENCES_-(LPC0_1+4))
	movt	r1, :upper16:(L_OBJC_SELECTOR_REFERENCES_-(LPC0_1+4))
LPC0_1:
	add	r1, pc
	ldr	r1, [r1]
	bl	_objc_msgSend
	.loc	3 19 12 is_stmt 0       @ /Users/wansheng/Desktop/testDemo/testDemo/Foo.m:19:12
	movw	r1, :lower16:(L_OBJC_SELECTOR_REFERENCES_.2-(LPC0_2+4))
	movt	r1, :upper16:(L_OBJC_SELECTOR_REFERENCES_.2-(LPC0_2+4))
LPC0_2:
	add	r1, pc
	ldr	r1, [r1]
	.loc	3 19 12 discriminator 1 @ /Users/wansheng/Desktop/testDemo/testDemo/Foo.m:19:12
	bl	_objc_msgSend
	.loc	3 19 5 discriminator 2  @ /Users/wansheng/Desktop/testDemo/testDemo/Foo.m:19:5
	add	sp, #8
Ltmp1:
	pop.w	{r7, lr}
	b.w	_objc_autoreleaseReturnValue
```

将重要的代码简化之后，之前的被编译器优化为：

```Objective-C

- (void)testFoo { 
    objc_unsafeClaimAutoreleasedReturnValue([Foo createFoo]); 
}

+ (instancetype)createFoo {
    id temp = [self new];  
    return objc_autoreleaseReturnValue(temp); 
} 

```

让我们从内到外看看这些代码都做了什么。以下代码都可以在[objc720]()中找到。

#### objc_autoreleaseReturnValue

```C++

id objc_autoreleaseReturnValue(id obj)
{
    if (prepareOptimizedReturn(ReturnAtPlus1)) return obj;
    
    return objc_autorelease(obj);
}

```
看到if判断里面的`Optimized`大概能猜到是做了优化。在if判断为true直接返回obj。并没有对obj做autorelease操作。在外部才对对象做了`objc_autorelease`操作。我们先看看它判断了什么。

##### prepareOptimizedReturn

```C++

enum ReturnDisposition : bool {
    ReturnAtPlus0 = false, ReturnAtPlus1 = true
};

// Try to prepare for optimized return with the given disposition (+0 or +1).
// Returns true if the optimized path is successful.
// Otherwise the return value must be retained and/or autoreleased as usual.
static ALWAYS_INLINE bool 
prepareOptimizedReturn(ReturnDisposition disposition)
{
    assert(getReturnDisposition() == ReturnAtPlus0);

    if (callerAcceptsOptimizedReturn(__builtin_return_address(0))) {
        if (disposition) setReturnDisposition(disposition);
        return true;
    }

    return false;
}

static ALWAYS_INLINE ReturnDisposition 
getReturnDisposition()
{
    return (ReturnDisposition)(uintptr_t)tls_get_direct(RETURN_DISPOSITION_KEY);
}


static ALWAYS_INLINE void 
setReturnDisposition(ReturnDisposition disposition)
{
    tls_set_direct(RETURN_DISPOSITION_KEY, (void*)(uintptr_t)disposition);
}

```
先来解释下`ReturnAtPlus0 `和`ReturnAtPlus1`,这两个枚举可以理解为未优化和已优化。再看下面两个getter setter方法。分别是基于TLS（线程局部存储）通过key`RETURN_DISPOSITION_KEY `存取优化标记`ReturnDisposition `
那么TLS是什么呢。

##### TLS

TLS（Thread Local Storage）是系统在内存中为线程自身单独开辟的一片空间，以 `<Key,Value>` 的形式存储线程自己独享的变量。

再回到上面的代码中，如果调用方是MRC就不需要优化了，所以需要知道调用方目前处于什么状态，`callerAcceptsOptimizedReturn `的左右就是检查调用方是否是ARC。

##### __builtin_return_address(0)

__builtin_return_address(0)让我们可以根据调用栈获取主调方的地址，0表示当前函数，每+1向外层跳一次。这里写`__builtin_return_address(0)`而不是`__builtin_return_address(1)`的原因是`prepareOptimizedReturn ()`声明关键字`static ALWAYS_INLINE bool`表明它是一个`内联函数`在编译器会自动插入到调用的地方。


##### callerAcceptsOptimizedReturn

```c++
// arm
static ALWAYS_INLINE bool 
callerAcceptsOptimizedReturn(const void *ra)
{
    // if the low bit is set, we're returning to thumb mode
    if ((uintptr_t)ra & 1) {
        // 3f 46          mov r7, r7
        // we mask off the low bit via subtraction
        // 16-bit instructions are well-aligned
        if (*(uint16_t *)((uint8_t *)ra - 1) == 0x463f) {
            return true;
        }
    } else {
        // 07 70 a0 e1    mov r7, r7
        // 32-bit instructions may be only 16-bit aligned
        if (*(unaligned_uint32_t *)ra == 0xe1a07007) {
            return true;
        }
    }
    return false;
}

```
因为不同的CPU架构的对齐方式和偏移量不同，所以这个方法针对各种CPU架构都做了单独的实现。这里以arm为例。首先先检查ra寄存器(存入的是pc的值（程序运行处的地址）)低位是否置位1.如果不是，就进入thumb指令集模式，再对齐低16/32位来判断是否是mov r7, r7.还记得上面看汇编的时候让大家注意的`mov r7, r7`么。这里就是通过这种方式来判断调用方是否支持`ARC`

总结下各个架构CPU判断ARC的方式

* x86_64: 被调方通过查找指令`mov rax, rdi`，根据下一条指令方法判断是否跳转调用了 `objc_retainAutoreleasedReturnValue` 或者 `objc_unsafeClaimAutoreleasedReturnValue`
* i386: 被调方通过在帧指针寄存器中查找指令`movl %ebp, %ebp`
* armv7: 被调方通过在帧指针寄存器中查找指令`mov r7, r7`
* arm64: 被调方通过在帧指针寄存器中查找指令`mov x29, x29`

```c

@ InlineAsm Start
	mov	r7, r7	@ marker for objc_retainAutoreleaseReturnValue
@ InlineAsm End

```

所以通过这样的判断，就可以知道调用方是否处于ARC环境下。
理清了这个方法内部整个调用流程，这样可以基本梳理出`testFoo()`之后内部的处理如下：


可以看到，runtime经过判断是否是ARC环境下，帮我们做了对应的优化，如果是，则不加入autoreleasePool直接返回对象，仅仅在TLS中标记已经优化。不论是否优化，最终创建的对象都传递给了`objc_unsafeClaimAutoreleasedReturnValue`,这个方法又是做什么的？我们来看下代码：

```c++
// Accept a value returned through a +0 autoreleasing convention for use at +0.
id
objc_unsafeClaimAutoreleasedReturnValue(id obj)
{
    if (acceptOptimizedReturn() == ReturnAtPlus0) return obj;

    return objc_releaseAndReturn(obj);
}

static ALWAYS_INLINE ReturnDisposition 
acceptOptimizedReturn()
{
    ReturnDisposition disposition = getReturnDisposition();
    setReturnDisposition(ReturnAtPlus0);  // reset to the unoptimized state
    return disposition;
}

```

总结一下，这个方法通过TLS中的优化位判断这个对象是否被优化过。如果被优化过，则将优化位置位为`false`然后release这个对象。反之则不做操作，因为这个对象是一个`autorelease`对象。请注意，我们当前的讨论是在无外部引用非属性的情况下。所以创建完对象就直接release了。

接下来我们来看看其他几种情况。

### 有外部引用非属性

```Objective-C

- (void)testFoo {
    Foo * myfoo = [Foo createFoo];
}

```

同样，通过Xcode查看汇编代码，简化后得出：

```Objective-C

+ (instancetype)createFoo  {
    id temp = [self  new]; 
    return objc_autoreleaseReturnValue(temp); 
} 

- (void)testFoo { 
    id temp = objc_retainAutoreleasedReturnValue([Foo createFoo]); 
    objc_storeStrong(&temp,nil); 
}

```

首先，`createFoo`方法中和之前无变化，不需要关注。可以看到testFoo方法中的代码发生了变化。
首先通过`objc_retainAutoreleasedReturnValue`创建了`temp`这个临时变量。然后通过`objc_storeStrong`对它的地址做了处理。我们来看下这两个方法分别做了什么。

#### objc_retainAutoreleasedReturnValue

```c++
id
objc_retainAutoreleasedReturnValue(id obj)
{
    if (acceptOptimizedReturn() == ReturnAtPlus1) return obj;

    return objc_retain(obj);
}
```
这个方法通过判断TLS中是否标记了已优化，如果优化了就直接返回，并置位为false。反之，则对对象进行retain操作。

跟`objc_unsafeClaimAutoreleasedReturnValue`一样，对于类便利构造方法返回的接收方，都会根据TLS中的优化标记进行判断。这个标记位总是在创建时置位，在接收时复位。所以这就是一个线程中多个变量仅需要一个标志位来处理优化的原因。置位和复位总是成对出现在代码逻辑中。

而之后的`objc_storeStrong`

```c++
void
objc_storeStrong(id *location, id obj)
{
    id prev = *location;
    if (obj == prev) {
        return;
    }
    objc_retain(obj);
    *location = obj;
    objc_release(prev);
}
```
它其实是用于将obj的所有权进行转让，交由location。在这里的调用`objc_storeStrong(&temp,nil); `,相当于将temp进行了release操作，并置为nil。

### 有外部引用且为属性

```Objective-C

- (void)testFoo {
    Foo * myfoo = [Foo createFoo];
    self.myfoo = myfoo;
}
```

简化后：

```
- (void)testFoo {
    id temp = _objc_retainAutoreleasedReturnValue([Foo createFoo]); 
    objc_storeStrong(&_myfoo, temp);
    objc_storeStrong(&temp,nil);
}

```

因为将临时变量myfoo赋值给了属性self.myfoo。所以`objc_storeStrong(&_myfoo, temp)`相当于调用了`_myfoo`的`set`方法，最后再release了temp。

### 无外部引用且为属性


```Objective-C

- (void)testFoo {
    self.myfoo = [Foo createFoo];
}
```

简化后：

```
- (void)testFoo {
    id temp = _objc_retainAutoreleasedReturnValue([Foo createFoo]); 
    objc_storeStrong(&_myfoo, temp);
    objc_release(temp)
}

```

### 总结

* Runtime会在运行时根据调用方是否处于ARC环境下，决定是否对autorelease对象进行优化。
* 优化标记存在当前线程的TLS中。
* 优化标记的置位和复位成对出现。