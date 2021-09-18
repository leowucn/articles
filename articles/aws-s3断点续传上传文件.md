# 上传文件到s3
---

## 前言

aws s3是aws的对象文件存储服务。可以上传几乎任何大小和类型的文件。对于上传大文件而言，断点续传十分重要，在这篇文章我将记录一下我实现断点续传的过程。实例代码采用typescript。

## 创建ManagedUpload

文件上传第一步需要调用aws api创建ManagedUpload对象。代码如下：

```typescript
    private getManagedUpload(file: any): S3.ManagedUpload {
        const { bucket: Bucket, accessControlLevel: ACL } = this.options;
        const baseParams = {
            Bucket,
            Key: file.keyPath,
        };

        const params: S3.PutObjectRequest = {
            ...baseParams,
            Body: file.fileObject,
            ContentType: mime.getType(file.absolutePath) || '',
            CacheControl: this.getCacheControlValue(file.absolutePath),
        };
        if (ACL) {
            params.ACL = ACL;
        }
        return this.s3.upload(params);
    }
```

上述代码摘自实现的一部分，可供参考。

这里创建出来的managedUpload是aws sdk的一个类，它具有`send()`方法，调用该方法可以进行文件上传，直觉上再调用它的`abort()`即可实现上传和暂停。但是再次调用`send()`方法后请求会出现`404`错误。原因是调用`abort()`方法会在云端清理之前上传的文件分块，所以再次调用`send()`方法会出错。

上面之所以再次调用`send()`方法出错，是因为调用`abort()`后，aws服务端自动完成了文件终止上传后的错误处理，导致之前的分块丢失。解决办法也很简单，只要把这个错误处理留给用户自己出来就可以再次调用`send()`方法实现断点续传了。

改造后的`getManagedUpload()`方法如下：

```typescript
    private getManagedUpload(file: any): S3.ManagedUpload {
        const { bucket: Bucket, accessControlLevel: ACL } = this.options;
        const baseParams = {
            Bucket,
            Key: file.keyPath,
        };

        const params: S3.PutObjectRequest = {
            ...baseParams,
            Body: file.fileObject,
            ContentType: mime.getType(file.absolutePath) || '',
            CacheControl: this.getCacheControlValue(file.absolutePath),
        };
        if (ACL) {
            params.ACL = ACL;
        }
        const options: S3.ManagedUpload.ManagedUploadOptions = {
            // 该选项可以用于支持 暂停/继续 上传
            leavePartsOnError: true,
        };

        return this.s3.upload(params, options);
    }
```

改造后的代码和之前的区别是，我们增加了选项`leavePartsOnError`，该选项默认值为`false`，设置为`true`表示上传出错时由用户处理错误。

这里需要额外补充一下。设置`leavePartsOnError: true`后，如果中断了上传，那么之前上传的文件分块在s3上是不见的，但是用户仍然需要为这部分数据支付费用。再次上传也只会上传缺少的分块。

在设置`leavePartsOnError: true`后，中断出现错误由用户处理，假如用户真的想要中断上传，已经上传的数据如何清理呢？查了很多文档没有找到方案，后来咨询了aws技术人员，如下：

清理 S3 分段上传失败数据的最佳实践为为您的 S3 桶配置生命周期规则，您可以使用该规则指示 S3 停止未在启动后的指定天数内完成的分段上传。当分段上传未在规定的时限内完成时，便能够执行中止操作，S3 将中止分段上传，并且会删除与分段上传相关的分段，包括分段上传失败的数据，详细说明您可以参考文档 [1]，而关于生命周期规则的配置，您可以参考以下：

```json
{
    "Rules": [
        {
            "ID": "AbortIncompleteMultipartUpload",
            "Status": "Enabled",
            "Filter": {
                "Prefix": ""
            },
            "AbortIncompleteMultipartUpload": {
                "DaysAfterInitiation": 7
            }
        }
    ]
}
```

上面json中的`Prefix`字段表示该生命周期规则适用于拥有特定 prefix 的对象。`DaysAfterInitiation`表示分段上传未在 7 天内完成时，S3 会终止分段上传并删除相关的分段

 可以调用`putBucketLifecycleConfiguration` 函数来为 S3 桶配置生命周期规则。

结束~

 
