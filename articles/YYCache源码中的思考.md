# YYCache源码中的思考

[toc]
## 写在前面
这里并不细致讨论`YYCache`的具体实现，如有了解的需要请移步[YYCache](https://github.com/ibireme/YYCache/)

## YYCache结构
`YYCache`中，作者采用了`sqlite+file system`的形式实现了一个高性能的`cache framework`。
源码总共包含8个文件，基础结构大致如下图：

![](https://github.com/damon8to24/blog/raw/master/resources/YYCache.png)
`YYKVStorage`是最基础的操作类，负责进行sqlite和file system有关的一系列操作，不需要我们直接使用此类。其中的`YYKVStorageItem`是单个存储单元，包含了存储内容的各种信息。

`YYDiskCache`和`YYMemoryCache`分别是disk存储和memory存储。disk存储是一个线程安全、将数据保存在`sqlite`和`file system`的cache。memory存储是一个在内存中快速保存、API和表现类似`NSCache`的cache。
这里稍微提下他们的特性：
`YYMemoryCache`：

* LRU(least-recently-used)
* 可通过`cost`、`count`、`age`来进行相应控制
* 内存警告时自动移除
* 通常的时间复杂度为`O(1)`
* 与NSDictionary相反，key值只被retain

`YYDiskCache`大致与`YYMemoryCache`相同，但也有些许不同：

* 可以自动选择存储方式是`sqlite`还是`file`（依据是存储数据大小是否超过20KB，来自sqlite官方文档）

## YYCache源码中的思考

### 忽略libsqlite3.dylib以获得加速

我们可以在源码中看到这样一句话：
>You may compile the latest version of sqlite and ignore the libsqlite3.dylib in
 iOS system to get 2x~4x speed up.

原因其实是因为SQLite官网的版本比iOS中的版本要更新，对这其中也做了很多优化，所以会获得一定的加速。

PS：作者在解答中指出，倍速只是个大概值

### 为什么无论是file形式，还是sqlite形式都在数据库存储了信息？

`YYKVStorage`中的`- (BOOL)saveItemWithKey:value:filename:extendedData:`方法中，我们可以发现一个奇怪的现象。当`fileName`不为null时，操作中不仅保存了数据库，而且还以数据库的保存结果为依据，决定file形式文件的最终返回。

```
if (filename.length) {
        //存文件
        if (![self _fileWriteWithName:filename data:value]) {
            return NO;
        }
        //存数据到数据库
        if (![self _dbSaveWithKey:key value:value fileName:filename extendedData:extendedData]) {
            //如果失败删除文件
            [self _fileDeleteWithName:filename];
            return NO;
        }
        return YES;
    }
```

当我们往更深一层去探究就会发现，`_dbSaveWithKey:value:fileName:extendedData:`这个方法中有代码如下，当存在`fileName`的时候，数据库并没有保存value，也就是数据值

```
if (fileName.length == 0) {
        sqlite3_bind_blob(stmt, 4, value.bytes, (int)value.length, 0);
} else {
        sqlite3_bind_blob(stmt, 4, NULL, 0, 0);
}
```

那既然不保存value，那么取值的时候该如何取值？其实如果仔细观察作者在获取时候的应用，就可以知道原因：

```
- (YYKVStorageItem *)getItemForKey:(NSString *)key {
    if (key.length == 0) return nil;
    YYKVStorageItem *item = [self _dbGetItemWithKey:key excludeInlineData:NO];//从数据库获取数据信息
        if (item) {
        [self _dbUpdateAccessTimeWithKey:key];//更新存取时间
        if (item.filename) {
        //如果filename存在，那么此时item的value其实是没有数据的
        //所以执行下面方法，就可以读取文件当中的数据
            item.value = [self _fileReadWithName:item.filename];
            if (!item.value) {
                [self _dbDeleteItemWithKey:key];
                item = nil;
            }
        }
    }
    return item;
}
```

总结一下：这样设计是因为无论是存储在file还是DB都可以从DB中轻易获取item的相关信息，然后通过其中的key值来从不同的地方来获取对象

### 为什么在YYMemoryCache使用了互斥锁（pthread_mutex_t），而在YYDiskCache中选用了信号量（dispatch_semaphore_t）？
其实在`YYDiskCache`中作者最初的选择是自旋锁(OSSpinLock)，放弃的原因是来自这篇文章：[不再安全的 OSSpinLock](https://blog.ibireme.com/2016/01/16/spinlock_is_unsafe_in_ios/)

图片资源来自[不再安全的 OSSpinLock](https://blog.ibireme.com/2016/01/16/spinlock_is_unsafe_in_ios/)
![](https://github.com/damon8to24/blog/raw/master/resources/lock_benchmark.png)


之后作者选择了性能稍差的信号量。那为什么在`YYMemoryCache`中选择了性能差于信号量的`pthread_mutex_t`？
作者给出的解释是：

>`dispatch_semaphore` 是信号量，但当信号总量设为 1 时也可以当作锁来。在没有等待情况出现时，它的性能比 `pthread_mutex` 还要高，但一旦有等待情况出现时，性能就会下降许多。相对于 `OSSpinLock` 来说，它的优势在于等待时不会消耗 CPU 资源。对磁盘缓存来说，它比较合适。

### 关于数据获取和释放问题
在YYMemoryCache中，`- (void)_trimToCount:(NSUInteger)countLimit`方法中，当尝试加锁不成功，采用了`usleep`进行了10ms的挂起操作？（手动黑人问号）
作者给出的解释是：
>为了尽量保证所有对外的访问方法都不至于阻塞，这个对象移除的方法应当尽量避免与其他访问线程产生冲突。

那么我们其实可以得出结论，当线程产生冲突时，删除操作如果无法加锁，则挂起缓冲10ms，稍后再试，这样不会造成外部访问方法的阻塞，也能保证数据在后续的操作中能重试删除

```
while (!finish) {
    if (pthread_mutex_trylock(&_lock) == 0) {
        if (_lru->_totalCount > countLimit) {
            _YYLinkedMapNode *node = [_lru removeTailNode];
            if (node) [holder addObject:node];
        } else {
            finish = YES;
        }
        pthread_mutex_unlock(&_lock);
    } else {
        usleep(10 * 1000); //10 ms
    }
}
```

如果这个还算好理解，那么下面的可能不是那么好理解，在同一方法中，当holder拿到数据之后，做了如下处理：

```
if (holder.count) {
    dispatch_queue_t queue = _lru->_releaseOnMainThread ? dispatch_get_main_queue() : YYMemoryCacheGetReleaseQueue();
    dispatch_async(queue, ^{
        [holder count]; // release in queue
    });
}
```
`[holder count];`怎么就算是release in queue了？
其实作者的意图是这样的：当我们在此方法中持有holder，那么holder的引用计数为1，然后我们在`trashQueue`中向holder发送任意消息，holder的引用计数就为2，那么最后方法执行完，引用计数-1，等到异步线程执行之后，holder的引用计数又-1，至此，holder引用计数为0，在runloop结束前被系统释放。
那么我们就需要问了，这样做究竟有什么意义？
在作者的文章[iOS保持界面流畅的技巧](https://blog.ibireme.com/2015/11/12/smooth_user_interfaces_for_ios/)中，介绍过对象销毁也是需要消耗资源的，当容器类持有大量对象需要释放，那么就可以通过下列操作来保持界面的流畅度

>这里有个小 Tip：把对象捕获到 block 中，然后扔到后台队列去随便发送个消息以避免编译器警告，就可以让对象在后台线程销毁了。

### Why CFMutableDictionary?
我们可以观察到`YYKVStorage`和`YYMemoryCache`中都采用了`CFMutableDictionaryRef`作为容器类，为什么不是我们常用的`NSMutableDictionary`?

这就牵涉到`Core Foundation`和`Foundation`的实现方式不同：

来自：[Difference between Foundation Framework and Core Foundation Framework?](https://stackoverflow.com/questions/1843251/difference-between-foundation-framework-and-core-foundation-framework)
>Core Foundation is the C-level API, which provides CFString, CFDictionary and the like.
>Foundation is Objective-C, which provides NSString, NSDictionary, etc.

来自：[Introduction to Core Foundation Design Concepts](https://developer.apple.com/library/content/documentation/CoreFoundation/Conceptual/CFDesignConcepts/CFDesignConcepts.html#//apple_ref/doc/uid/10000122i)

>Core Foundation is a library with a set of programming interfaces conceptually derived from the Objective-C-based Foundation framework but implemented in the C language. To do this, Core Foundation implements a limited object model in C. 

这样我们就可以得出结论，在实现功能一致的情况下，使用`Core Foundation`的容器类，因为更加接近C，所以会在性能上获得一定的提升。

### inline的作用
先上源码：

```
static inline dispatch_queue_t YYMemoryCacheGetReleaseQueue() {
    return dispatch_get_global_queue(DISPATCH_QUEUE_PRIORITY_LOW, 0);
}
```
为什么这里要使用inline来定义queue？
使用inline的原因在：[内联函数 inline](https://www.jianshu.com/p/d557b0831c6a)

我们这里得到的结论是：

1. 为什么不使用宏？因为inline不需要预编译
2. 可以避免一部分压栈弹栈等函数调用的开销


