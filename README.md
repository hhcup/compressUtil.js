# compressUtil.js
js图片大小压缩到指定范围

```javascript
/**
 * 图片压缩类
 * @param minSize
 * @param maxSize
 * @constructor
 */
var PhotoCompress = function (minSize, maxSize) {
    var nextQ = 0.5; // 压缩比例
    var maxQ = 1;
    var minQ = 0;

    /**
     * 将base64转换为文件
     * @param base64Codes base64编码
     * @param fileName 文件名称
     * @returns {*}
     */
    PhotoCompress.prototype.dataUrlToFile = function (base64Codes, fileName) {
        var arr = base64Codes.split(','),
            mime = arr[0].match(/:(.*?);/)[1],
            bStr = atob(arr[1]),
            n = bStr.length,
            u8arr = new Uint8Array(n);
        while (n--) {
            u8arr[n] = bStr.charCodeAt(n);
        }
        return new File([u8arr], fileName, {type: mime});
    }

    /**
     * 图片压缩
     * @param file 文件
     * @param callback 回调函数
     */
    PhotoCompress.prototype.compress = function (file, callback) {
        var self = this;
        self.imgBase64(file, function (image, canvas) {
            var base64Codes = canvas.toDataURL(file.type, nextQ); // y压缩
            var compressFile = self.dataUrlToFile(base64Codes, file.name.split('.')[0]); // 转成file文件
            var compressFileSize = compressFile.size; // 压缩后文件大小 k
            console.log("图片质量：" + nextQ);
            console.log("压缩后文件大小：" + compressFileSize / 1024);
            if (compressFileSize > maxSize) { // 压缩后文件大于最大值
                maxQ = nextQ;
                nextQ = (nextQ + minQ) / 2; // 质量降低
                self.compress(file, callback);
            } else if (compressFileSize < minSize) { // 压缩以后文件小于最小值
                minQ = nextQ;
                nextQ = (nextQ + maxQ) / 2; // 质量提高
                self.compress(file, callback);
            } else {
                callback(compressFile);
            }
        });
    }

    /**
     * 将图片转化为base64
     * @param file 文件
     * @param callback 回调函数
     */
    PhotoCompress.prototype.imgBase64 = function (file, callback) {
        // 看支持不支持FileReader
        if (!file || !window.FileReader) return;
        var image = new Image();
        // 绑定 load 事件处理器，加载完成后执行
        image.onload = function () {
            var canvas = document.createElement('canvas')
            var ctx = canvas.getContext('2d')
            ctx.clearRect(0, 0, canvas.width, canvas.height);
            canvas.width = image.width * nextQ;
            canvas.height = image.height * nextQ;
            ctx.drawImage(image, 0, 0, canvas.width, canvas.height);
            callback(image, canvas);
        };
        if (/^image/.test(file.type)) {
            // 创建一个reader
            var reader = new FileReader();
            // 将图片将转成 base64 格式
            reader.readAsDataURL(file);
            // 读取成功后的回调
            reader.onload = function () {
                // self.imgUrls.push(this.result);
                // 设置src属性，浏览器会自动加载。
                // 记住必须先绑定事件，才能设置src属性，否则会出同步问题。
                image.src = this.result;
            }
        }
    }
};
module.exports = {PhotoCompress};
```
> 说明：这边改变的图片的大小有两个地方：一是重画图片的时候，改变图片的长宽（按相同比例否者会变形），二是转base64的时候降低质量。二者缺一不可，具体步骤如下（二分法原理）：

 1. 先将图片按照一定的比例重画，这边的比例是（maxQ+minQ）/2，也就是说原来的图片长宽会缩小为一半（10001000 会变成500500），图片会相应变小，但是无法确定具体值。
 2. 将重画的图片按照一定的质量转成base64，这边也是（maxQ+minQ）/2，图片大小也会减小。
 3. 图片质量超过指定最大值，将maxQ变为（maxQ+minQ）/2，小于指定最小值则将minQ变为（maxQ+minQ）/2，重复1，2即可。
 
# 使用方法

```javacript
import {PhotoCompress} from "../../lib/compressUtil"
var maxSize = 0.5 * 1024 * 1024; // 0.5M
var minSize = 0.2 * 1024 * 1024; // 200K
var photoCompress = new PhotoCompress(minSize, maxSize);
photoCompress.compress(file, function(file) {
                var r = new FileReader()
                r.readAsDataURL(file)
                r.onload = function(e) {
                    console.log("压缩后大小：" + file.size / 1024);
                    $("#originPic").prop("src", this.result)
                }
            });
```
