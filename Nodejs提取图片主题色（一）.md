### 需求

1. 一个可以正常访问的图片链接
2. 排除掉图片数据值中灰色系rgb的值（即 `r=g=b` 的值）
3. 统计剩余的rgb值，得出颜色排行

>这里主要参考了 [https://github.com/lokesh/color-thief](https://github.com/lokesh/color-thief) 开源项目
>
> [Nodejs提取图片主题色（二）](https://blog.csdn.net/roamingcode/article/details/108286167)

### 使用方式

```js
ColorThief.getColor('https://lokeshdhakar.com/projects/color-thief/images/image-1.jpg')
  .then((color) => {
    console.log('color', color); // 获得主题色（色值占比最多）
  })
  .catch((err) => {
    console.log(err);
  });

ColorThief.getPalette('https://lokeshdhakar.com/projects/color-thief/images/image-1.jpg', 5)
  .then((palette) => {
    console.log('palette', palette); // 获得主题色排行（色值占比前5的值）
  })
  .catch((err) => {
    console.log(err);
  });
```

### 如何排除掉图片数据值中灰色系rgb的值

**这里是在对图像色值做统计的地方做拦截处理 -> createPixelArray 方法**

```js
// 该文件是基于 color-thief 提供的 color-thief-node.js 上做的处理

const quantize = require('quantize');
const getPixels = require('./node-pixels');

function createPixelArray(imgData, pixelCount, quality) {
    const pixels = imgData;
    const pixelArray = [];

    for (let i = 0, offset, r, g, b, a; i < pixelCount; i = i + quality) {
        offset = i * 4;
        r = pixels[offset + 0];
        g = pixels[offset + 1];
        b = pixels[offset + 2];
        a = pixels[offset + 3];

        if (typeof a === 'undefined' || a >= 125) {
            // 排除灰色系值
            if (r !== g || r !== b) {
                pixelArray.push([r, g, b]);
            }
        }
    }
    return pixelArray;
}

function validateOptions(options) {
    let { colorCount, quality } = options;

    if (typeof colorCount === 'undefined' || !Number.isInteger(colorCount)) {
        colorCount = 10;
    } else if (colorCount === 1 ) {
        throw new Error('colorCount should be between 2 and 20. To get one color, call getColor() instead of getPalette()');
    } else {
        colorCount = Math.max(colorCount, 2);
        colorCount = Math.min(colorCount, 20);
    }

    if (typeof quality === 'undefined' || !Number.isInteger(quality) || quality < 1) {
        quality = 10;
    }

    return {
        colorCount,
        quality
    }
}

function loadImg(img) {
    return new Promise((resolve, reject) => {
        getPixels(img, function(err, data) {
            if(err) {
                reject(err)
            } else {
                resolve(data);
            }
        })
    });
}

function getColor(img, quality) {
    return new Promise((resolve, reject) => {
        getPalette(img, 5, quality)
            .then(palette => {
                resolve(palette[0]);
            })
            .catch(err => {
                reject(err);
            })
    });

}

function getPalette(img, colorCount = 10, quality = 10) {
    const options = validateOptions({
        colorCount,
        quality
    });

    return new Promise((resolve, reject) => {
        loadImg(img)
            .then(imgData => {
                const pixelCount = imgData.shape[0] * imgData.shape[1];
                const pixelArray = createPixelArray(imgData.data, pixelCount, options.quality);

                const cmap = quantize(pixelArray, options.colorCount);
                const palette = cmap? cmap.palette() : null;

                resolve(palette);
            })
            .catch(err => {
                reject(err);
            })
    });
}

module.exports = {
    getColor,
    getPalette
};
```

在上面的文件中，除了修改了排除灰色系值的逻辑，还有一点不同

```js
const getPixels = require('./node-pixels');
```

原文使用

```js
const getPixels = require('get-pixels');
```

我们需要对 `get-pixels` 这个包进行一些额外的修改

**修改的原因在于**

1. 图片的 `mimeType` 可能带有字符编码，比如 `image/jpeg;charset=UTF-8`，但期望得到的是 `image/jpeg`
2. 针对 `jpg、jpeg` 的图片处理经常报错，尤其是经过第三方处理过的图片，比如（经过裁剪、拼接），其中报错多为`throw new Error("SOI not found");`，意思是该图片不符合jpg图片规范

```js
// node-pixels.js
'use strict';

var ndarray = require('ndarray');
var path = require('path');
var PNG = require('pngjs').PNG;
// var jpeg = require('jpeg-js'); // 对于 jpg、jpeg 格式的文件，不再使用jpeg-js进行解码处理，转而使用 images 这个包
var pack = require('ndarray-pack');
var GifReader = require('omggif').GifReader;
var Bitmap = require('node-bitmap');
var fs = require('fs');
var request = require('request');
var mime = require('mime-types');
var parseDataURI = require('parse-data-uri');
var images = require('images'); // 使用 images 处理jpg、jpeg

function handlePNG(data, cb) {
  var png = new PNG();
  png.parse(data, function (err, img_data) {
    if (err) {
      cb(err);
      return;
    }
    cb(
      null,
      ndarray(
        new Uint8Array(img_data.data),
        [img_data.width | 0, img_data.height | 0, 4],
        [4, (4 * img_data.width) | 0, 1],
        0
      )
    );
  });
}

function handleJPEG(data, cb) {
  var jpegData;
  var imageData;
  try {
    // jpegData = jpeg.decode(data); 不再使用jpeg-js进行解码处理，使用 images 进行处理
    imageData = images(data);
    jpegData = imageData.encode('jpg'); // 重新以jpg格式进行编码，用于修复问题 2
  } catch (e) {
    cb(e);
    return;
  }
  if (!jpegData) {
    cb(new Error('Error decoding jpeg'));
    return;
  }
  var nshape = [imageData.height(), imageData.width(), 4];
  var result = ndarray(jpegData, nshape);
  cb(null, result.transpose(1, 0));
}

function handleGIF(data, cb) {
  var reader;
  try {
    reader = new GifReader(data);
  } catch (err) {
    cb(err);
    return;
  }
  if (reader.numFrames() > 0) {
    var nshape = [reader.numFrames(), reader.height, reader.width, 4];
    try {
      var ndata = new Uint8Array(nshape[0] * nshape[1] * nshape[2] * nshape[3]);
    } catch (err) {
      cb(err);
      return;
    }
    var result = ndarray(ndata, nshape);
    try {
      for (var i = 0; i < reader.numFrames(); ++i) {
        reader.decodeAndBlitFrameRGBA(
          i,
          ndata.subarray(result.index(i, 0, 0, 0), result.index(i + 1, 0, 0, 0))
        );
      }
    } catch (err) {
      cb(err);
      return;
    }
    cb(null, result.transpose(0, 2, 1));
  } else {
    var nshape = [reader.height, reader.width, 4];
    var ndata = new Uint8Array(nshape[0] * nshape[1] * nshape[2]);
    var result = ndarray(ndata, nshape);
    try {
      reader.decodeAndBlitFrameRGBA(0, ndata);
    } catch (err) {
      cb(err);
      return;
    }
    cb(null, result.transpose(1, 0));
  }
}

function handleBMP(data, cb) {
  var bmp = new Bitmap(data);
  try {
    bmp.init();
  } catch (e) {
    cb(e);
    return;
  }
  var bmpData = bmp.getData();
  var nshape = [bmpData.getHeight(), bmpData.getWidth(), 4];
  var ndata = new Uint8Array(nshape[0] * nshape[1] * nshape[2]);
  var result = ndarray(ndata, nshape);
  pack(bmpData, result);
  cb(null, result.transpose(1, 0));
}

function doParse(mimeType, data, cb) {
  // 增加mime的类型的处理，用于修复问题 1
  if (mimeType.includes(';')) {
    mimeType = mimeType.split(';')[0];
  }
  switch (mimeType) {
    case 'image/png':
      handlePNG(data, cb);
      break;

    case 'image/jpg':
    case 'image/jpeg':
      handleJPEG(data, cb);
      break;

    case 'image/gif':
      handleGIF(data, cb);
      break;

    case 'image/bmp':
      handleBMP(data, cb);
      break;

    default:
      cb(new Error('Unsupported file type: ' + mimeType));
  }
}

module.exports = function getPixels(url, type, cb) {
  if (!cb) {
    cb = type;
    type = '';
  }
  if (Buffer.isBuffer(url)) {
    if (!type) {
      cb(new Error('Invalid file type'));
      return;
    }
    doParse(type, url, cb);
  } else if (url.indexOf('data:') === 0) {
    try {
      var buffer = parseDataURI(url);
      if (buffer) {
        process.nextTick(function () {
          doParse(type || buffer.mimeType, buffer.data, cb);
        });
      } else {
        process.nextTick(function () {
          cb(new Error('Error parsing data URI'));
        });
      }
    } catch (err) {
      process.nextTick(function () {
        cb(err);
      });
    }
  } else if (url.indexOf('http://') === 0 || url.indexOf('https://') === 0) {
    request({ url: url, encoding: null }, function (err, response, body) {
      if (err) {
        cb(err);
        return;
      }

      type = type;
      if (!type) {
        if (response.getHeader !== undefined) {
          type = response.getHeader('content-type');
        } else if (response.headers !== undefined) {
          type = response.headers['content-type'];
        }
      }
      if (!type) {
        cb(new Error('Invalid content-type'));
        return;
      }
      doParse(type, body, cb);
    });
  } else {
    fs.readFile(url, function (err, data) {
      if (err) {
        cb(err);
        return;
      }
      type = type || mime.lookup(url);
      if (!type) {
        cb(new Error('Invalid file type'));
        return;
      }
      doParse(type, data, cb);
    });
  }
};

```

在修复 `问题2` 的时候使用了 `images`  这个包，这样处理后可以得到下面的色盘值

```json
[ [ 195, 170, 172 ],
  [ 36, 154, 139 ],
  [ 184, 35, 149 ],
  [ 174, 186, 34 ],
  [ 95, 171, 171 ] ]
```

经过与实际图片的对比，发现这个色盘值大多不符合，且差距明显

其中具体问题就在 `images` 这里，经过代码优化后产生了如下结果，该值与图片是相符的

```json
[ [ 125, 190, 192 ],
  [ 213, 191, 131 ],
  [ 53, 37, 27 ],
  [ 129, 123, 58 ],
  [ 42, 125, 148 ] ]
```

代码优化见 [Nodejs提取图片主题色（二）](https://blog.csdn.net/roamingcode/article/details/108286167)







