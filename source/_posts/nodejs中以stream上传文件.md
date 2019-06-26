---
title: nodejs stream 小结
date: 2019-06-13 14:54:53
tags:
---
### stream是什么
按官方文档中的描述，stream是一个抽象接口来处理流数据的，并且它提供了一些基础的api来构建和操作流对象。那么什么是流数据呢？

### 流数据
想象一条水流，它是连续不断的，你很难想象它是一批一批流动的。
再进一步，我可以想象有两个桶，水流可以从管道中一点一点溜过去，再次强调的是，这是一个连续的过程。
{% asset_img image1.jpg 图1 %}

### 流的优势与应用场景
同理，在接收到文件后，传统方式需要将文件直接读取至内存，再做处理，这样在读取较大的文件时，它所占的内存就会较大。
如下，我们将读取一个74.8MB的文件，并比较其两种方式所占的内存大小
```js
const fs = require('fs');
fs.readFile('./project_source.zip', (err, data) => {
    if (err) throw err;
    console.log(data);
});

const fs = require('fs');
fs.readFile('./project_source.zip', (err, data) => {
    if (err) throw err;
    console.log(data);
});

// 打印内存占用情况
function printMemoryUsage() {
    var info = process.memoryUsage();
    function mb(v) {
        return (v / 1024 / 1024).toFixed(2) + 'MB';
    }
    console.log('rss=%s, heapTotal=%s, heapUsed=%s', mb(info.rss), mb(info.heapTotal), mb(info.heapUsed));
}

setInterval(printMemoryUsage, 500);
```
结果如下：

| 方式   | rss(MB) | heapTotal(MB) | headUsed(MB) |
| ------ | ------- | ------------- | ------------ |
| file   | 107.10  | 12.00         | 7.06         |
| stream | 58.98   | 12.00         | 7.33         |

看起来显而易见，rss所占的内存在stream的方式下小了很多，rss是驻留集(Resident Set)大小,这个词太难理解了，实际上就是给这个应用分配了多少内存。这里顺便讲讲，v8把内存分成了一下几部分:
- 代码: 实际被执行的代码
- 栈(stack): 包含所有类型，对象的引用，说白了就是变量。
- 堆(head): 空间用于存放对象，字符串，闭包

### Eggjs中处理上传
此处我们使用egg工程来模拟一个文件的上传，并且用stream方式接收
```js
    const fileStream = await ctx.getFileStream();

    try {
        
        const fileName = fileStream.filename;
        let filePath = 'xxxx' + path.basename(fileName);
        const resp = await ctx.oss.put(filePath, fileStream);
        return this.ctx.apiSuccess({ data: resp });

    } catch (error) {
            // 必须将上传的文件流消费掉，要不然浏览器响应会卡死
            await sendToWormhole(fileStream);
            ctx.apiError(error);
    }
```