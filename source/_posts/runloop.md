# runloop

当前源码使用

- https://github.com/apple/swift-corelibs-foundation.git
- https://github.com/apple/swift-corelibs-xctest.git

## runloop的构成

### 简介

runloop是一个循环或者说是保持主线程的活性来接受各类事件（source0, source1, source2)  

### 构成

NSRunloop对象是基于CFRunLoopRef对象的封装，CFRunLoopRef又是__CFRunLoop结构体定义的指针。

```objective-c
__CFRunLoop {
    CFMutableSetRef _commonModes;
    CFMutableSetRef _commonModeItems;
    CFRunLoopModeRef _currentMode;
    CFMutableSetRef _modes;
}
```

#### 结构体里面的情况

- __CFRunLoop
  - _modes
    - __CFRunLoopMode
  - _commonModes
    - CFStringRef (mode name)
  - _currentMode
    - __CFRunLoopMode
  - _commonModeItems
    - __CFRunLoopTimer
    - __CFRunLoopSource
    - __CFRunLoopObserver

#### _commonModes

主线程预置了2中mode，Mode：**kCFRunLoopDefaultMode**，**UITrackingRunLoopMode**

### _modes && _currentMode

每种mode结构

| Mode      |
| --------- |
| source0   |
| source1   |
| observers |
| timer     |

### _commonModeItems

里面存放了所有被标记为common的Source/Timer/Observer，当**CFRunLoopAddCommonMode**被调用时，一个mode被添加到了一个runloop的common modes中，随即commonModeItems里的所有源被添加到这个新的mode中。



## runloop细节窥探

当你CFRunLoopGetMain()时，会产生以下步骤

1. 参数会传入一个pthread_t
2. 并且会有一个全局的可变字典，他的名字叫做__CFRunLoops
3. 当然__CFRunLoops在首次进入的时候是空的，所以苹果先生成了一个可变的字典叫做dict，并且生成一个CFRunLoopRef类型的变量mainloop，并将且塞入到dict中
4. 这时__CFRunloops这个字典还是空的，所以苹果使用了OSAtomicCompareAndSwapPtrBarrier( void *__oldValue, void *__newValue, void * volatile *__theValue )的方法来将dict值赋给__CFRunloops
5. 接着，以传入的thread为key，苹果从__CFRunloops取出了CFRunLoopRef类型的变量loop
6. 当然假如取出的loop为空，苹果会重新的再创建一次，并将其丢入__CFRunLoops，但尚不清楚那种情况会造成步骤3中，mainloop创建的失败，这个可后续再做研究
7. 最后就是返回取出的loop

## runloop的运行细节

### 当前runloop添加一个timer源到common mode时发什么了什么

```objective-c
// 第 1 步
// 当前runloop添加一个timer源到common mode时
// 先判断modeName是不是等于Common mode -> common mode -> 复制common modes里面的内容 -> 新的timer加入_commonModeItems -> common modes里遍历每个mode（主要是default和UITracking），并将timer绑定到每个遍历的mode上
if (modeName == kCFRunLoopCommonModes) {
	CFSetRef set = rl->_commonModes ? CFSetCreateCopy(kCFAllocatorSystemDefault, rl->_commonModes) : NULL;
	if (NULL == rl->_commonModeItems) {
	    rl->_commonModeItems = CFSetCreateMutable(kCFAllocatorSystemDefault, 0, &kCFTypeSetCallBacks);
	}
	CFSetAddValue(rl->_commonModeItems, rlt);
	if (NULL != set) {
	    CFTypeRef context[2] = {rl, rlt};
	    /* add new item to all common-modes */
	    CFSetApplyFunction(set, (__CFRunLoopAddItemToCommonModes), (void *)context);
	    CFRelease(set);
	}
}
```

下面是遍历common modes,并依次绑定timer的最重要的方法

```c
CFSetApplyFunction(set, (__CFRunLoopAddItemToCommonModes), (void *)context);
```

此处有一个问题，早些年在看耀源大神的博客时，在描述CommonModes时有这样一段话

> 每当 RunLoop 的内容发生变化时，RunLoop 都会自动将 _commonModeItems 里的 Source/Observer/Timer 同步到具有 “Common” 标记的所有Mode里。

这个步骤其实是发生在***一个mode被添加到了一个runloop的common modes中时***。而我之前认为，***每当 RunLoop 的内容发生变化***指的是添加timer等这类源的时候。

当添加一个timer时，苹果只是把common modes遍历一遍，然后依次把timer加到了default mode，uitracking mode上去。无比暴力。



### runloop run的时候

下面这段代码是真的证明了runloop本质上确实只是do while罢了，当然我们需要它一直活着，来接受各种源，并且回调处理

```objective-c
int32_t result;
do {
    result = CFRunLoopRunSpecific(CFRunLoopGetCurrent(), kCFRunLoopDefaultMode, 1.0e10, false);
    CHECK_FOR_FORK();
} while (kCFRunLoopRunStopped != result && kCFRunLoopRunFinished != result);
```

1. 取出当前的mode，还是熟悉的配方

```c
CFRunLoopModeRef currentMode = __CFRunLoopFindMode(rl, modeName, false);
```

2. 通知observer，即将进入runloop

```c
if (currentMode->_observerMask & kCFRunLoopEntry ) __CFRunLoopDoObservers(rl, currentMode, kCFRunLoopEntry);
```

再往下就是__CFRunLoopRun函数的内容，即进入runloop后

3. 通知observer，即将触发timer回调

```c
if (rlm->_observerMask & kCFRunLoopBeforeTimers) {
    __CFRunLoopDoObservers(rl, rlm, kCFRunLoopBeforeTimers);
}
```

4. 通知observer，即将触发source0回调

```c
if (rlm->_observerMask & kCFRunLoopBeforeSources) {
    __CFRunLoopDoObservers(rl, rlm, kCFRunLoopBeforeSources);
}
```

5. 执行被加入的block

```c
__CFRunLoopDoBlocks(rl, rlm);
```

5. 触发source0的回调

```c
Boolean sourceHandledThisLoop = __CFRunLoopDoSources0(rl, rlm, stopAfterHandle);
```

6. 之后的处理是有分叉的,检查是否有source 1，如果有source 1的话就go to跳转，而没有的话则会顺序执行下去

```c
if (__CFRunLoopServiceMachPort(dispatchPort, &msg, sizeof(msg_buffer), &livePort, 0, &voucherState, NULL, rl, rlm)) {
    goto handle_msg;
}
```

7. 先来看顺序执行下去的情况

```c
if (!poll && (rlm->_observerMask & kCFRunLoopBeforeWaiting)) __CFRunLoopDoObservers(rl, rlm, kCFRunLoopBeforeWaiting);
	__CFRunLoopSetSleeping(rl);
```

8. 通知了observer马上就要进入睡眠了，接下来是一个内部的循环，我做了简化处理

```c
do {            
   __CFRunLoopServiceMachPort(waitSet, &msg, sizeof(msg_buffer), &livePort, poll ? 0 : TIMEOUT_INFINITY, &voucherState, &voucherCopy, rl, rlm);
   
   if (modeQueuePort != MACH_PORT_NULL && livePort == modeQueuePort) {
       // Drain the internal queue. If one of the callout blocks sets the timerFired flag, break out and service the timer.
   } else {
       // Go ahead and leave the inner loop.
       break;
   }
} while (1);
```

9. 触发timer回调

```c
/*...省略若干行代码...*/
//这个内部的循环的在livePort等于当前mode的port时，就会退出来，即退出睡眠，接下来最重要的就是假如有source 1时的情况

//MARK:触发timer 回调
if (!__CFRunLoopDoTimers(rl, rlm, mach_absolute_time())) {
    // Re-arm the next timer, because we apparently fired early
    __CFArmNextTimerInMode(rlm, rl);
}
```

10. 执行dispatch到main queue里的block

```c
__CFRUNLOOP_IS_SERVICING_THE_MAIN_DISPATCH_QUEUE__(msg);
```

11. 如果基于source1发出事件，处理这个事件

```c
(void)mach_msg(reply, MACH_SEND_MSG, reply->msgh_size, 0, MACH_PORT_NULL, 0, MACH_PORT_NULL);
CFAllocatorDeallocate(kCFAllocatorSystemDefault, reply);
```

12. 执行加入到runloop的block

```c
__CFRunLoopDoBlocks(rl, rlm);
```

13. retVal结果赋值

```c
if (sourceHandledThisLoop && stopAfterHandle) {
    retVal = kCFRunLoopRunHandledSource;
} else if (timeout_context->termTSR < mach_absolute_time()) {
    retVal = kCFRunLoopRunTimedOut;
} else if (__CFRunLoopIsStopped(rl)) {
 __CFRunLoopUnsetStopped(rl);
 retVal = kCFRunLoopRunStopped;
} else if (rlm->_stopped) {
 rlm->_stopped = false;
 retVal = kCFRunLoopRunStopped;
} else if (__CFRunLoopModeIsEmpty(rl, rlm, previousMode)) {
 retVal = kCFRunLoopRunFinished;
}

    } while (0 == retVal);
```