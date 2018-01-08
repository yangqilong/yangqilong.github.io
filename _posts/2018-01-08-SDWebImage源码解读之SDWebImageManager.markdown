---
layout: post
title: SDWebImage源码解读之SDWebImageManager
date: 2018-01-08 13:38:00.000000000 
---
#### 前言
`SDWebImageManager`是`SDWebImage`中最核心的类了，但是源代码确是非常简单的。之所以能做到这一点，都归功于功能的良好分类。有了`SDWebImageManager`管理类，我们就能做很多其他的有意思的事情。比如给各种view绑定一个URL，就能显示图片的功能，有了Options，就能满足多种应用场景的图片下载任务。
### SDWebImageOptions
`SDWebImageOptions`作为图片选项提供了非常多的子项，用法和注意事项我都写在代码里，如下：

```bash
typedef NS_OPTIONS(NSUInteger, SDWebImageOptions) {
    /**
     * By default, when a URL fail to be downloaded, the URL is blacklisted so the library won't keep trying.
     * This flag disable this blacklisting.
     */
    /// 每一个下载都会提供一个URL，如果这个URL是错误，SD就会把它放入到黑名单之中，
    /// 黑名单中的URL是不会再次进行下载的,但是，当设置了该选项时，SD会将其在黑名单中移除，重新下载该URL，
    SDWebImageRetryFailed = 1 << 0,

    /**
     * By default, image downloads are started during UI interactions, this flags disable this feature,
     * leading to delayed download on UIScrollView deceleration for instance.
     */
    /// 一般来说，下载都是按照一定的先后顺序开始的，但是该选项能够延迟下载，也就说他的权限比较低，权限比他高的在他前边下载
    SDWebImageLowPriority = 1 << 1,

    /**
     * This flag disables on-disk caching
     */
    /// 该选项要求SD只把图片缓存到内存中，不缓存到disk中
    SDWebImageCacheMemoryOnly = 1 << 2,

    /**
     * This flag enables progressive download, the image is displayed progressively during download as a browser would do.
     * By default, the image is only displayed once completely downloaded.
     */
    /// 给下载添加进度
    SDWebImageProgressiveDownload = 1 << 3,

    /**
     * Even if the image is cached, respect the HTTP response cache control, and refresh the image from remote location if needed.
     * The disk caching will be handled by NSURLCache instead of SDWebImage leading to slight performance degradation.
     * This option helps deal with images changing behind the same request URL, e.g. Facebook graph api profile pics.
     * If a cached image is refreshed, the completion block is called once with the cached image and again with the final image.
     *
     * Use this flag only if you can't make your URLs static with embedded cache busting parameter.
     */
    /// 有这么一种使用场景，如果一个图片的资源发生了改变。但是url并没有变，我们就可以使用该选项来刷新数据了
    SDWebImageRefreshCached = 1 << 4,

    /**
     * In iOS 4+, continue the download of the image if the app goes to background. This is achieved by asking the system for
     * extra time in background to let the request finish. If the background task expires the operation will be cancelled.
     */
    /// 支持切换到后台也能下载
    SDWebImageContinueInBackground = 1 << 5,

    /**
     * Handles cookies stored in NSHTTPCookieStore by setting
     * NSMutableURLRequest.HTTPShouldHandleCookies = YES;
     */
    /// 使用Cookies
    SDWebImageHandleCookies = 1 << 6,

    /**
     * Enable to allow untrusted SSL certificates.
     * Useful for testing purposes. Use with caution in production.
     */
    /// 允许验证证书
    SDWebImageAllowInvalidSSLCertificates = 1 << 7,

    /**
     * By default, images are loaded in the order in which they were queued. This flag moves them to
     * the front of the queue.
     */
    /// 高权限
    SDWebImageHighPriority = 1 << 8,
    
    /**
     * By default, placeholder images are loaded while the image is loading. This flag will delay the loading
     * of the placeholder image until after the image has finished loading.
     */
    /// 一般情况下，placeholder image 都会在图片下载完成前显示，该选项将设置placeholder image在下载完成之后才能显示
    SDWebImageDelayPlaceholder = 1 << 9,

    /**
     * We usually don't call transformDownloadedImage delegate method on animated images,
     * as most transformation code would mangle it.
     * Use this flag to transform them anyway.
     */
    /// 使用该属性来自由改变图片，但需要使用transformDownloadedImage delegate
    SDWebImageTransformAnimatedImage = 1 << 10,
    
    /**
     * By default, image is added to the imageView after download. But in some cases, we want to
     * have the hand before setting the image (apply a filter or add it with cross-fade animation for instance)
     * Use this flag if you want to manually set the image in the completion when success
     */
    /// 该选项允许我们在图片下载完成后不会立刻给view设置图片，比较常用的使用场景是给赋值的图片添加动画
    SDWebImageAvoidAutoSetImage = 1 << 11,
    
    /**
     * By default, images are decoded respecting their original size. On iOS, this flag will scale down the
     * images to a size compatible with the constrained memory of devices.
     * If `SDWebImageProgressiveDownload` flag is set the scale down is deactivated.
     */
    /// 压缩大图片
    SDWebImageScaleDownLargeImages = 1 << 12
};

```

### Block

```bash
typedef void(^SDExternalCompletionBlock)(UIImage * _Nullable image, NSError * _Nullable error, SDImageCacheType cacheType, NSURL * _Nullable imageURL);

typedef void(^SDInternalCompletionBlock)(UIImage * _Nullable image, NSData * _Nullable data, NSError * _Nullable error, SDImageCacheType cacheType, BOOL finished, NSURL * _Nullable imageURL);

typedef NSString * _Nullable (^SDWebImageCacheKeyFilterBlock)(NSURL * _Nullable url);

```

### SDWebImageManagerDelegate

使用协议增加了我们编程的灵活性，SDWebImageManagerDelegate提供了两个方法:<br />

* 在缓存中没发现图片，控制是不是下载该图片<br />
* 自由转换图片

代码：

```bash
@protocol SDWebImageManagerDelegate <NSObject>

@optional

/**
 * Controls which image should be downloaded when the image is not found in the cache.
 *
 * @param imageManager The current `SDWebImageManager`
 * @param imageURL     The url of the image to be downloaded
 *
 * @return Return NO to prevent the downloading of the image on cache misses. If not implemented, YES is implied.
 */
- (BOOL)imageManager:(nonnull SDWebImageManager *)imageManager shouldDownloadImageForURL:(nullable NSURL *)imageURL;

/**
 * Allows to transform the image immediately after it has been downloaded and just before to cache it on disk and memory.
 * NOTE: This method is called from a global queue in order to not to block the main thread.
 *
 * @param imageManager The current `SDWebImageManager`
 * @param image        The image to transform
 * @param imageURL     The url of the image to transform
 *
 * @return The transformed image object.
 */
- (nullable UIImage *)imageManager:(nonnull SDWebImageManager *)imageManager transformDownloadedImage:(nullable UIImage *)image withURL:(nullable NSURL *)imageURL;

@end
```

### SDWebImageCombinedOperation
`SDWebImageCombinedOperation`是对每一个下载任务的封装，重要的是它提供了一个取消功能。

```bash
@interface SDWebImageCombinedOperation : NSObject <SDWebImageOperation>

@property (assign, nonatomic, getter = isCancelled) BOOL cancelled;
@property (copy, nonatomic, nullable) SDWebImageNoParamsBlock cancelBlock;
@property (strong, nonatomic, nullable) NSOperation *cacheOperation;

@end

```

### SDWebImageManager
#### 1.属性
```bash
@interface SDWebImageManager ()

@property (strong, nonatomic, readwrite, nonnull) SDImageCache *imageCache;
@property (strong, nonatomic, readwrite, nonnull) SDWebImageDownloader *imageDownloader;
@property (strong, nonatomic, nonnull) NSMutableSet<NSURL *> *failedURLs;
@property (strong, nonatomic, nonnull) NSMutableArray<SDWebImageCombinedOperation *> *runningOperations;

@end

```
#### 2.初始化（非常典型的初始化）
```bash
+ (nonnull instancetype)sharedManager {
    static dispatch_once_t once;
    static id instance;
    dispatch_once(&once, ^{
        instance = [self new];
    });
    return instance;
}

- (nonnull instancetype)init {
    SDImageCache *cache = [SDImageCache sharedImageCache];
    SDWebImageDownloader *downloader = [SDWebImageDownloader sharedDownloader];
    return [self initWithCache:cache downloader:downloader];
}

- (nonnull instancetype)initWithCache:(nonnull SDImageCache *)cache downloader:(nonnull SDWebImageDownloader *)downloader {
    if ((self = [super init])) {
        _imageCache = cache;
        _imageDownloader = downloader;
        _failedURLs = [NSMutableSet new];
        _runningOperations = [NSMutableArray new];
    }
    return self;
}

```
#### 3.URL 转 key
```bash
- (nullable NSString *)cacheKeyForURL:(nullable NSURL *)url {
    if (!url) {
        return @"";
    }

    if (self.cacheKeyFilter) {
        return self.cacheKeyFilter(url);
    } else {
        return url.absoluteString;
    }
}

```
#### 4.查看图片是否已经缓存
```bash
- (void)cachedImageExistsForURL:(nullable NSURL *)url
                     completion:(nullable SDWebImageCheckCacheCompletionBlock)completionBlock {
    NSString *key = [self cacheKeyForURL:url];
    
    BOOL isInMemoryCache = ([self.imageCache imageFromMemoryCacheForKey:key] != nil);
    
    if (isInMemoryCache) {
        // making sure we call the completion block on the main queue
        dispatch_async(dispatch_get_main_queue(), ^{
            if (completionBlock) {
                completionBlock(YES);
            }
        });
        return;
    }
    
    [self.imageCache diskImageExistsWithKey:key completion:^(BOOL isInDiskCache) {
        // the completion block of checkDiskCacheForImageWithKey:completion: is always called on the main queue, no need to further dispatch
        if (completionBlock) {
            completionBlock(isInDiskCache);
        }
    }];
}

```
#### 5.查看是否已经缓存到了硬盘
```bash
- (void)diskImageExistsForURL:(nullable NSURL *)url
                   completion:(nullable SDWebImageCheckCacheCompletionBlock)completionBlock {
    NSString *key = [self cacheKeyForURL:url];
    
    [self.imageCache diskImageExistsWithKey:key completion:^(BOOL isInDiskCache) {
        // the completion block of checkDiskCacheForImageWithKey:completion: is always called on the main queue, no need to further dispatch
        if (completionBlock) {
            completionBlock(isInDiskCache);
        }
    }];
}

```

#### 6.核心下载方法

* 处理参数相关的异常 <br \> 
* 处理复杂的逻辑 <br \> 
* 返回数据 <br \> 

```bash
- (id <SDWebImageOperation>)loadImageWithURL:(nullable NSURL *)url
                                     options:(SDWebImageOptions)options
                                    progress:(nullable SDWebImageDownloaderProgressBlock)progressBlock
                                   completed:(nullable SDInternalCompletionBlock)completedBlock {
    // Invoking this method without a completedBlock is pointless
    /// 1.如果想预先下载图片，使用[SDWebImagePrefetcher prefetchURLs]取代本方法
    /// 预下载图片是有很多种使用场景的，当我们使用SDWebImagePrefetcher下载图片后，之后使用该图片时就不用再网络上下载了。
    NSAssert(completedBlock != nil, @"If you mean to prefetch the image, use -[SDWebImagePrefetcher prefetchURLs] instead");

    // Very common mistake is to send the URL using NSString object instead of NSURL. For some strange reason, XCode won't
    // throw any warning for this type mismatch. Here we failsafe this error by allowing URLs to be passed as NSString.
    /// 2.XCode有时候经常会犯一些错误，当用户给url赋值了字符串的时候，XCode也没有报错，因此这里提供一种
    /// 错误修正的处理。
    if ([url isKindOfClass:NSString.class]) {
        url = [NSURL URLWithString:(NSString *)url];
    }

    // Prevents app crashing on argument type error like sending NSNull instead of NSURL
    /// 3.防止参数的其他错误
    if (![url isKindOfClass:NSURL.class]) {
        url = nil;
    }

    /// 4.operation会被作为该方法的返回值，但operation的类型是SDWebImageCombinedOperation，是一个封装的对象，并不是一个NSOperation
    __block SDWebImageCombinedOperation *operation = [SDWebImageCombinedOperation new];
    __weak SDWebImageCombinedOperation *weakOperation = operation;

    /// 5.在图片的下载中，会有一些下载失败的情况，这时候我们把这些下载失败的url放到一个集合中去，
    /// 也就是加入了黑名单中，默认是不会再继续下载黑名单中的url了，但是也有例外，当options被设置为
    /// SDWebImageRetryFailed的时候，会尝试进行重新下载。
    BOOL isFailedUrl = NO;
    if (url) {
        @synchronized (self.failedURLs) {
            isFailedUrl = [self.failedURLs containsObject:url];
        }
    }

    /// 6.会有两种情况让我们停止下载这个url指定的图片：
    /// - url的长度为0
    /// - options并没有选择SDWebImageRetryFailed(重新下载错误url)且这个url在黑名单之中
    /// 调用完成Block，返回operation
    if (url.absoluteString.length == 0 || (!(options & SDWebImageRetryFailed) && isFailedUrl)) {
        [self callCompletionBlockForOperation:operation completion:completedBlock error:[NSError errorWithDomain:NSURLErrorDomain code:NSURLErrorFileDoesNotExist userInfo:nil] url:url];
        return operation;
    }

    /// 7.排除了所有的错误可能后，我们就先把这个operation添加到正在运行操作的数组中
    /// 这里没有判断self.runningOperations是不是包含了operation，
    /// 说明肯定会在下边的代码中做判断，如果存在就删除operation
    @synchronized (self.runningOperations) {
        [self.runningOperations addObject:operation];
    }
    NSString *key = [self cacheKeyForURL:url];

    /// 8.self.imageCache的queryCacheOperationForKey方法是异步的获取指定key的图片，
    /// 但是这个方法的operation是同步返回的，也就是说下边的代码会直接执行到return那里。
    operation.cacheOperation = [self.imageCache queryCacheOperationForKey:key done:^(UIImage *cachedImage, NSData *cachedData, SDImageCacheType cacheType) {
        /// (8.1).这个Block会在查询完指定的key的图片后调用，由` dispatch_async(self.ioQueue, ^{`
        /// 这个可以看出，实在异步线程采用串行的方式在调用，任务在self.imageCache的ioQueue中一个一个执行，是线程安全的
        
        /// (8.2).如果每次调用loadImage方法都会生成一个operation，如果我们想取消某个下载任务
        /// 再设计上来说，只要把响应的operation.isCancelled设置为NO，那么下载就会被取消。
        if (operation.isCancelled) {
            [self safelyRemoveOperationFromRunning:operation];
            return;
        }

        /// (8.3).代码来到这里，我们就要根据是否有缓存的图片来做出响应的处理
        /// 如果没有获取到缓存图片或者需要刷新缓存图片我们应该通过网络获取图片，但是这里增加了一个额外的控制
        /// 根据delegate的imageManager:shouldDownloadImageForURL:获取是否下载的权限，返回YES，就继续下载
        if ((!cachedImage || options & SDWebImageRefreshCached) && (![self.delegate respondsToSelector:@selector(imageManager:shouldDownloadImageForURL:)] || [self.delegate imageManager:self shouldDownloadImageForURL:url])) {
            /// (8.3.1).这里需要注意了，当图片已经下载了，Options又选择了SDWebImageRefreshCached
            /// 就会触发一次completionBlock回调，这说明这个下载的回调不是只触发一次的
            /// 如果使用了dispatch_group_enter和dispatch_group_leave就一定要注意了
            if (cachedImage && options & SDWebImageRefreshCached) {
                // If image was found in the cache but SDWebImageRefreshCached is provided, notify about the cached image
                // AND try to re-download it in order to let a chance to NSURLCache to refresh it from server.
                [self callCompletionBlockForOperation:weakOperation completion:completedBlock image:cachedImage data:cachedData error:nil cacheType:cacheType finished:YES url:url];
            }

            // download if no image or requested to refresh anyway, and download allowed by delegate
            /// (8.3.2).这里是SDWebImageOptions到SDWebImageDownloaderOptions的转换
            /// 其实就是0 |= xxx
            SDWebImageDownloaderOptions downloaderOptions = 0;
            if (options & SDWebImageLowPriority) downloaderOptions |= SDWebImageDownloaderLowPriority;
            if (options & SDWebImageProgressiveDownload) downloaderOptions |= SDWebImageDownloaderProgressiveDownload;
            if (options & SDWebImageRefreshCached) downloaderOptions |= SDWebImageDownloaderUseNSURLCache;
            if (options & SDWebImageContinueInBackground) downloaderOptions |= SDWebImageDownloaderContinueInBackground;
            if (options & SDWebImageHandleCookies) downloaderOptions |= SDWebImageDownloaderHandleCookies;
            if (options & SDWebImageAllowInvalidSSLCertificates) downloaderOptions |= SDWebImageDownloaderAllowInvalidSSLCertificates;
            if (options & SDWebImageHighPriority) downloaderOptions |= SDWebImageDownloaderHighPriority;
            if (options & SDWebImageScaleDownLargeImages) downloaderOptions |= SDWebImageDownloaderScaleDownLargeImages;
            
            /// (8.3.3).已经缓存且SDWebImageRefreshCached的比较特殊
            if (cachedImage && options & SDWebImageRefreshCached) {
                // force progressive off if image already cached but forced refreshing
                /// (8.3.3.1).SDWebImageDownloaderProgressiveDownload为 1<<1 ,
                /*
                    由于当options == SDWebImageRefreshCached时，downloaderOptions |= SDWebImageDownloaderUseNSURLCache（1 << 2）
                    00000000 | 00000100 =>  00000100
                    ~SDWebImageDownloaderProgressiveDownload : ~ 00000010 => 111111101
                    00000100 & 11111101 => 00000100
                 */
                downloaderOptions &= ~SDWebImageDownloaderProgressiveDownload;
                // ignore image read from NSURLCache if image if cached but force refreshing
                /// (8.3.3.2). 00000100 | 00001000 => 00001100
                /// 通过这种位的运算，就能够给同一个值赋值两种转态
                downloaderOptions |= SDWebImageDownloaderIgnoreCachedResponse;
            }
            
            /// (8.3.4).下载图片
            SDWebImageDownloadToken *subOperationToken = [self.imageDownloader downloadImageWithURL:url options:downloaderOptions progress:progressBlock completed:^(UIImage *downloadedImage, NSData *downloadedData, NSError *error, BOOL finished) {
                __strong __typeof(weakOperation) strongOperation = weakOperation;
                if (!strongOperation || strongOperation.isCancelled) {
                    // Do nothing if the operation was cancelled
                    // See #699 for more details
                    // if we would call the completedBlock, there could be a race condition between this block and another completedBlock for the same object, so if this one is called second, we will overwrite the new data
                } else if (error) {
                    /// (8.3.4.1).发生错误就返回
                    [self callCompletionBlockForOperation:strongOperation completion:completedBlock error:error url:url];

                    /// (8.3.4.2).除了下边这几种情况之外的情况则把url加入黑名单
                    if (   error.code != NSURLErrorNotConnectedToInternet
                        && error.code != NSURLErrorCancelled
                        && error.code != NSURLErrorTimedOut
                        && error.code != NSURLErrorInternationalRoamingOff
                        && error.code != NSURLErrorDataNotAllowed
                        && error.code != NSURLErrorCannotFindHost
                        && error.code != NSURLErrorCannotConnectToHost) {
                        @synchronized (self.failedURLs) {
                            [self.failedURLs addObject:url];
                        }
                    }
                }
                else {
                    
                    /// (8.3.4.3).如果是SDWebImageRetryFailed就在黑名单中移除，不管有没有
                    if ((options & SDWebImageRetryFailed)) {
                        @synchronized (self.failedURLs) {
                            [self.failedURLs removeObject:url];
                        }
                    }
                    
                    BOOL cacheOnDisk = !(options & SDWebImageCacheMemoryOnly);

                    if (options & SDWebImageRefreshCached && cachedImage && !downloadedImage) {
                        // Image refresh hit the NSURLCache cache, do not call the completion block
                    } else if (downloadedImage && (!downloadedImage.images || (options & SDWebImageTransformAnimatedImage)) && [self.delegate respondsToSelector:@selector(imageManager:transformDownloadedImage:withURL:)]) { // 要不要修改图片，这个修改图片完全由代理来操作
                        dispatch_async(dispatch_get_global_queue(DISPATCH_QUEUE_PRIORITY_HIGH, 0), ^{
                            UIImage *transformedImage = [self.delegate imageManager:self transformDownloadedImage:downloadedImage withURL:url];

                            if (transformedImage && finished) {
                                BOOL imageWasTransformed = ![transformedImage isEqual:downloadedImage];
                                // pass nil if the image was transformed, so we can recalculate the data from the image
                                [self.imageCache storeImage:transformedImage imageData:(imageWasTransformed ? nil : downloadedData) forKey:key toDisk:cacheOnDisk completion:nil];
                            }
                            
                            [self callCompletionBlockForOperation:strongOperation completion:completedBlock image:transformedImage data:downloadedData error:nil cacheType:SDImageCacheTypeNone finished:finished url:url];
                        });
                    } else {
                        if (downloadedImage && finished) {
                            [self.imageCache storeImage:downloadedImage imageData:downloadedData forKey:key toDisk:cacheOnDisk completion:nil];
                        }
                        [self callCompletionBlockForOperation:strongOperation completion:completedBlock image:downloadedImage data:downloadedData error:nil cacheType:SDImageCacheTypeNone finished:finished url:url];
                    }
                }

                /// (8.3.4.4).完成了就把该任务在正运行着的数组中删除
                if (finished) {
                    [self safelyRemoveOperationFromRunning:strongOperation];
                }
            }];
            
            /// (8.3.5).设置取消的回调
            operation.cancelBlock = ^{
                [self.imageDownloader cancel:subOperationToken];
                __strong __typeof(weakOperation) strongOperation = weakOperation;
                [self safelyRemoveOperationFromRunning:strongOperation];
            };
        } else if (cachedImage) {
            __strong __typeof(weakOperation) strongOperation = weakOperation;
            [self callCompletionBlockForOperation:strongOperation completion:completedBlock image:cachedImage data:cachedData error:nil cacheType:cacheType finished:YES url:url];
            [self safelyRemoveOperationFromRunning:operation];
        } else {
            // Image not in cache and download disallowed by delegate
            /// (8.4).既没有缓存也下载了代理不允许的图片
            __strong __typeof(weakOperation) strongOperation = weakOperation;
            [self callCompletionBlockForOperation:strongOperation completion:completedBlock image:nil data:nil error:nil cacheType:SDImageCacheTypeNone finished:YES url:url];
            [self safelyRemoveOperationFromRunning:operation];
        }
    }];

    return operation;
}

```
#### 7.保存图片到缓存区
```bash
- (void)saveImageToCache:(nullable UIImage *)image forURL:(nullable NSURL *)url {
    if (image && url) {
        NSString *key = [self cacheKeyForURL:url];
        [self.imageCache storeImage:image forKey:key toDisk:YES completion:nil];
    }
}

```
#### 8.取消所有的下载
```bash
- (void)cancelAll {
    @synchronized (self.runningOperations) {
        NSArray<SDWebImageCombinedOperation *> *copiedOperations = [self.runningOperations copy];
        [copiedOperations makeObjectsPerformSelector:@selector(cancel)];
        [self.runningOperations removeObjectsInArray:copiedOperations];
    }
}

```
#### 9.查看是否下载完毕
```bash
- (BOOL)isRunning {
    BOOL isRunning = NO;
    @synchronized (self.runningOperations) {
        isRunning = (self.runningOperations.count > 0);
    }
    return isRunning;
}

```
#### 10.安全移除任务
```bash
- (void)safelyRemoveOperationFromRunning:(nullable SDWebImageCombinedOperation*)operation {
    @synchronized (self.runningOperations) {
        if (operation) {
            [self.runningOperations removeObject:operation];
        }
    }
}

```
#### 11.回调
```bash
- (void)callCompletionBlockForOperation:(nullable SDWebImageCombinedOperation*)operation
                             completion:(nullable SDInternalCompletionBlock)completionBlock
                                  error:(nullable NSError *)error
                                    url:(nullable NSURL *)url {
    [self callCompletionBlockForOperation:operation completion:completionBlock image:nil data:nil error:error cacheType:SDImageCacheTypeNone finished:YES url:url];
}

- (void)callCompletionBlockForOperation:(nullable SDWebImageCombinedOperation*)operation
                             completion:(nullable SDInternalCompletionBlock)completionBlock
                                  image:(nullable UIImage *)image
                                   data:(nullable NSData *)data
                                  error:(nullable NSError *)error
                              cacheType:(SDImageCacheType)cacheType
                               finished:(BOOL)finished
                                    url:(nullable NSURL *)url {
    dispatch_main_async_safe(^{
        if (operation && !operation.isCancelled && completionBlock) {
            completionBlock(image, data, error, cacheType, finished, url);
        }
    });
}

```

SDWebImage作为目前最受欢迎的图片下载第三方框架，使用率很高，地址：[SDWebImage](https://github.com/rs/SDWebImage)