---
layout: post
title: "block hook"
date: 2020-09-15 11:30
header-style: text
author: "wbq"
tags: 
  - block
---


## 导言

block作为oc极为重要的一部分从来都是面试和开发乐此不疲的话题与工具。

关于block的数据结构和实现原理网上大大小小真的已经很多了，这里不再叙述，最近有业务需求需要hook block 进行一些无埋点的监控，所以顺便记录下学习过程。



## 方案



### 当前`blockhook`的方案大致有两种:

#### [1.基于libffi的blockhook](https://github.com/yulingtianxia/BlockHook)

libffi的原理和用法这里不予赘述了。

大致流程:

1. 根据 block 对象的签名，使用 `ffi_prep_cif` 构建` block->invoke `函数的模板 `cif`
2. 使用 `ffi_closure`，根据 cif 动态定义函数 `replacementInvoke`，绑定到`ClosureFunc`(一个具体实现的函数实体上)
3. 将` block->invoke `替换为 `replacementInvoke`，原始的 `block->invoke` 存放在 `originInvoke`
4. 在 `ClosureFunc` 中通过hook位置动态调用 `originInvoke` 函数和执行 hook 的逻辑。





#### [2.基于方法转发的blockhook](https://github.com/welcommand/FishBind)

大致流程:

1. 通过`runtime`交换block对象的方法转发的方法。
2. 将` block->invoke `替换为`(IMP)_objc_msgForward`，重新拷贝原block生成`newBlock`，通过关联对象进行保存(block地址为key)。
3. 在自定义的`方法转发`方法中拿到`newBlock`的 `originInvoke` 函数进行调用和执行 hook 的逻辑。





## 问题总结

1.其实两种方法在思路是差不多的，都是通过'曲线救国'的方式，找到那个中间的桥接函数，进行替换。区别在于实现方法不同。

上述的第一种方法，优点是目前比较稳定，功能强大，且兼容了在各种环境下的很多问题，缺点是要引libffi。

第二种的方式较轻量，且不用引libffi，但没有像第一种经过大量的验证，兼容性有待测试。



2.在第二种方式中，我发现作者对于原`originInvoke`的处理麻烦了，不需要重新new一份新的内存和拷贝。直接保存`originInvoke`的通过关联对象保存原函数指针，在调用过程中再换回来就可以。

伪代码:

```objective-c
-(void)blockhook{
    //交换方法..
    ...
    //保存block原实现
		[self saveOriginInvoke:block->invoke];	
    //替换block实现
    block->invoke = _objc_msgForward; 
}

-(void)bh_forwardInvocation:(NSInvocation *)invocation{
    
    //hook逻辑
    ...
    //替换回原实现
    block->invoke = [self getOriginInvoke];
    //执行原逻辑函数
    [invocation invokeWithTarget:block];
}

```



3.将target设置为iOS13,运行时`GlobalBlock`类型会出现invoke替换问题，而另外两张类型的block没问题。

![image.png](https://upload-images.jianshu.io/upload_images/2782305-b4586ec4d33fc875.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)



开始也是百思不得其解，想了各种办法，想试着通过结构体的地址偏移量去修改invoke依然换不掉，开始觉得可能是底层偷偷把地址换了，看了汇编把寄存器值打印出来发现地址也对，但写入就是坏地址访问。

![image.png](https://upload-images.jianshu.io/upload_images/2782305-4934167dcbfecf44.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


走投无路之际看到了第一位作者的文章才恍然大悟，不得不佩服大佬的才华。

[BlockHook and Memory Safety](http://yulingtianxia.com/blog/2020/05/30/BlockHook-and-Memory-Safety/) 文中提到如何解决 GlobalBlock 没有写权限的问题。用到了虚拟内存的一些相关api去修改内存页的读写权限，经过代码运行，我也证实了确实苹果在iOS13将GlobalBlock权限进行了限制，只有只读权限，至于为啥求大佬们解惑。以下是解决方案

```c++
vm_prot_t protectInvokeVMIfNeed(void *address) {
    vm_address_t addr = (vm_address_t)address;
    vm_size_t vmsize = 0;
    mach_port_t object = 0;
#if defined(__LP64__) && __LP64__
    vm_region_basic_info_data_64_t info;
    mach_msg_type_number_t infoCnt = VM_REGION_BASIC_INFO_COUNT_64;
    kern_return_t ret = vm_region_64(mach_task_self(), &addr, &vmsize, VM_REGION_BASIC_INFO, (vm_region_info_t)&info, &infoCnt, &object);
#else
    vm_region_basic_info_data_t info;
    mach_msg_type_number_t infoCnt = VM_REGION_BASIC_INFO_COUNT;
    kern_return_t ret = vm_region(mach_task_self(), &addr, &vmsize, VM_REGION_BASIC_INFO, (vm_region_info_t)&info, &infoCnt, &object);
#endif
    if (ret != KERN_SUCCESS) {
        NSLog(@"vm_region block invoke pointer failed! ret:%d, addr:%p", ret, address);
        return VM_PROT_NONE;
    }
    vm_prot_t protection = info.protection;
    if ((protection&VM_PROT_WRITE) == 0) {
        ret = vm_protect(mach_task_self(), (vm_address_t)address, sizeof(address), false, protection|VM_PROT_WRITE);
        if (ret != KERN_SUCCESS) {
            NSLog(@"vm_protect block invoke pointer VM_PROT_WRITE failed! ret:%d, addr:%p", ret, address);
            return VM_PROT_NONE;
        }
    }
    return protection;
}
```

核心函数是下面这个，直接贴文档。

```
Function - Set access privilege attribute for a region of virtual memory.

SYNOPSIS
kern_return_t   vm_protect
                 (vm_task_t           target_task,（需要修改的内存空间区域）
                  vm_address_t            address,（起始地址）
                  vm_size_t                  size,（地址大小）
                  boolean_t           set_maximum, （这个没太看懂）
                  vm_prot_t        new_protection); (赋予的新的权限)
PARAMETERS
target_task
[in task send right] The port for the task whose address space contains the region.
address
[in scalar] The starting address for the region.
size
[in scalar] The number of bytes in the region.
set_maximum
[in scalar] Maximum/current indicator. If true, the new protection sets the maximum protection for the region. If false, the new protection sets the current protection for the region. If the maximum protection is set below the current protection, the current protection is also reset to the new maximum.
new_protection
[in scalar] The new protection for the region. Valid values are obtained by or'ing together the following values:
VM_PROT_READ
Allows read access.
VM_PROT_WRITE
Allows write access.
VM_PROT_EXECUTE
Allows execute access.
```

至此在iOS13也可以开心的玩耍了。

剩下的实现包括读取参数，获取方法签名包装这些相关文档已经太多了，就不多赘述了。

看别人的方案就是这样。内心的os都是:哇还可以这么玩，为什么我想不到😭



我自己也基于第二种方法,并且结合以上的问题，做了些修改，撸了一个分类。目前自己在项目中debug用用感觉还不错

[超简易版BlockHook](https://github.com/Wbqqqq/BQBlockHook)



本次学习非常感谢:

[BlockHook](https://github.com/yulingtianxia/BlockHook)

[FishBind](https://github.com/welcommand/FishBind)

[BlockHook and Memory Safety](http://yulingtianxia.com/blog/2020/05/30/BlockHook-and-Memory-Safety/)

[Hook Objective-C Block with Libffi](http://yulingtianxia.com/blog/2018/02/28/Hook-Objective-C-Block-with-Libffi/)

