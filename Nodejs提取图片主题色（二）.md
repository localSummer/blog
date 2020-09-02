> [Nodejs提取图片主图色（一）](https://blog.csdn.net/roamingcode/article/details/108225806)

### 如何提高颜色提取的正确率

主要是 `images`、`jpeg-js`、`pngjs` 共用，彼此之间并不冲突

```js
// node-pixels.js
'use strict';

var ndarray = require('ndarray');
// var path = require('path');
var PNG = require('pngjs').PNG; // 处理 png 文件
var jpeg = require('jpeg-js'); // 处理 jpg/jpeg 文件
var pack = require('ndarray-pack');
var GifReader = require('omggif').GifReader;
var Bitmap = require('node-bitmap');
var fs = require('fs');
var request = require('request');
var mime = require('mime-types');
var parseDataURI = require('parse-data-uri');
var images = require('images'); // 只用来规范化图片编码

function handlePNG(data, cb) {
  var pngData;
  var imageData;
  try {
    imageData = images(data);
    pngData = imageData.encode('png'); // 修复不符合png规范的图片，使其规范化，符合png格式校验程序
    var png = new PNG();
    png.parse(pngData, function (err, img_data) { // 解析png图片编码
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
  } catch (e) {
    cb(e);
    return;
  }
}

function handleJPEG(data, cb) {
  var jpegData;
  var imageData;
  try {
    imageData = images(data);
    // 修复不符合jpg规范的图片，使其规范化，符合jpg格式校验程序，并使用jpeg-js进行解码处理
    jpegData = jpeg.decode(imageData.encode('jpg'));
  } catch (e) {
    cb(e);
    return;
  }
  if (!jpegData) {
    cb(new Error('Error decoding jpeg'));
    return;
  }
  var nshape = [jpegData.height, jpegData.width, 4];
  var result = ndarray(jpegData.data, nshape);
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
      let imgReg = /\.(png|jpg|jpeg|gif|bmp)/i;
      if (!type) {
        if (
          response.getHeader !== undefined &&
          response.getHeader('content-type')
        ) {
          type = response.getHeader('content-type');
        } else if (
          response.headers !== undefined &&
          response.headers['content-type']
        ) {
          type = response.headers['content-type'];
        } else if (imgReg.test(url)) {
          let matchArr = url.match(imgReg);
          type = `image/${matchArr[1]}`;
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

### 获取色盘值并写入 excel文件

1. 将每个图片对应的色盘值写入 `xlsx` 文件，使用 [node-xlsx](https://github.com/mgcrea/node-xlsx)
2. 展示色盘值所对应的颜色（以单元格背景颜色的形态进行展示）

```js
// index.js

const xlsx = require('node-xlsx').default;
const ColorThief = require('./utils/color-thief-node'); // 同 Nodejs提取图片主题色（一）
const fs = require('fs');
const Color = require('color'); // 提供 rgb -> hex 的格式转换

const workSheetsFromFile = xlsx.parse(`${__dirname}/docs/test.xlsx`); // 解析xlsx文件获得json格式文件信息

// workSheetsFromFile[0]['data'][0][5] = '背景色';

const getPaletteColor = async () => {
    // 排除掉第一行标题行
  for (let i = 1; i < workSheetsFromFile[0]['data'].length; i++) {
    // getPalette 第一个参数为 远程连接，第二个参数为色盘值 TOP5
    await ColorThief.getPalette(workSheetsFromFile[0]['data'][i][3], 5)
      .then((palette) => {
        console.log('palette', palette);
        workSheetsFromFile[0]['data'][i][4] = JSON.stringify(palette);
        for (let j = 0; j < palette.length; j++) {
          const bgColor = Color.rgb(palette[j]).hex().slice(1); // 色值格式转换，单元格颜色设置不支持rgb格式
          workSheetsFromFile[0]['data'][i][j + 5] = { // 设置单元格背景色，设置方式见 node-xlsx 文档
            v: bgColor,
            s: {
              fill: {
                fgColor: {
                  rgb: bgColor,
                },
              },
            },
          };
        }
      })
      .catch((err) => {
        console.log(err);
        workSheetsFromFile[0]['data'][i][4] = 0;
      });
  }

  const buffer = xlsx.build([
    { name: 'Sheet1', data: workSheetsFromFile[0]['data'] },
  ]);

  fs.writeFileSync(`${__dirname}/docs/test.xlsx`, buffer);
};

getPaletteColor();
```

### 修复单元格颜色设置失败

运行上述代码发现，给单元格设置的背景色无效，原因在于 `node-xlsx` 依赖 [js-xlsx](http://sheetjs.com/) 社区版本，而只有在 付费pro 版本中才支持 样式颜色的设置。

但是业内已经有了魔改 `js-xlsx` 的方案，见[令最新JS-XLSX支持样式的改造方法](https://blog.wj2015.com/2019/05/01/js-xlsx支持样式/)

其中具体使用如下

使用下方文件覆盖  `node_modules/xlsx/xlsx.js` 文件 
[https://github.com/wangerzi/layui-excel/blob/master/src/xlsx.js](https://github.com/wangerzi/layui-excel/blob/master/src/xlsx.js)
使用下方文件覆盖  `node_modules/xlsx/jszip.js` 文件
[https://github.com/wangerzi/layui-excel/blob/master/src/jszip.js](https://github.com/wangerzi/layui-excel/blob/master/src/jszip.js)

经过上述操作后，对单元格设置的样式即可生效