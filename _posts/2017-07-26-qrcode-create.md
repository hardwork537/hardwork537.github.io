---
layout: post
title: qrcode create
category: web
tags: [web]
description: 二维码的应用实践
---

## 解析字符串

调用 str_split 把字符串切割到 QRinput->input 中
生成的QRinput对象内容如下(字符串为 20170726tencent):

```
QRinput::__set_state(array(
   'items' => 
  array (
    0 => 
    QRinputItem::__set_state(array(
       'mode' => 0,
       'size' => 8,
       'data' => 
      array (
        0 => '2',
        1 => '0',
        2 => '1',
        3 => '7',
        4 => '0',
        5 => '7',
        6 => '2',
        7 => '6',
      ),
       'bstream' => NULL,
    )),
    1 => 
    QRinputItem::__set_state(array(
       'mode' => 2,
       'size' => 7,
       'data' => 
      array (
        0 => 't',
        1 => 'e',
        2 => 'n',
        3 => 'c',
        4 => 'e',
        5 => 'n',
        6 => 't',
      ),
       'bstream' => NULL,
    )),
  ),
   'version' => 0,
   'level' => 2,
))
```

## 编码处理

对上面产生的QRinput对象进行一系列编码处理 : 数据编码、结束符和补齐码、纠错码、掩码、转成二进制等，结果如下 : 

```
array (
  0 => '1111111000111001101111111',
  1 => '1000001010000000101000001',
  2 => '1011101000010001101011101',
  3 => '1011101011000000101011101',
  4 => '1011101011011011101011101',
  5 => '1000001001011001101000001',
  6 => '1111111010101010101111111',
  7 => '0000000011001101100000000',
  8 => '0101111010000100011011010',
  9 => '0010110000111100110001001',
  10 => '0111111011101101111111011',
  11 => '1111000011001100110001000',
  12 => '0100011000110101110000110',
  13 => '1000010010110010110110011',
  14 => '1110111101001110111011000',
  15 => '1011110001000001010110101',
  16 => '1011011010001010111111000',
  17 => '0000000010000010100010111',
  18 => '1111111000111001101010011',
  19 => '1000001010100110100010100',
  20 => '1011101010100101111110001',
  21 => '1011101011011010101100001',
  22 => '1011101001100111000001111',
  23 => '1000001010100000110010101',
  24 => '1111111001010101110110011',
)
```
## 画图(使用gd库)
- $times 是相对于像素的放大倍数，主要是作图、画圆方便(对一个像素点无法画圆)
- 不要使用 ImageCreate、ImageCopyResized 函数，效果不好(虽然速度快);用 

(1). 创建画板 
```
imagecreatetruecolor($imgW*$times, $imgH*$times);
```
(2). 用白色填充整个画布
```
imagefill($base_image, 0, 0, $col[0]);
```
(3). 填充值为1的像素点(这里是画圆)，四个角不填充
```
imagefilledellipse($source,$pointX,$pointY,$diameter,$diameter,$color);
```
(4). 填充三个定位角
(5). 拉伸或缩小到指定尺寸
```
imagecopyresampled($dest_image, $target_image, 0, 0, 0, 0, $imgW * $pixelPerPoint, $imgH * $pixelPerPoint, $imgW * $times, $imgH * $times);
```
(6). 用看点logo填充右下角
```
$logo = imagecreatefromstring(file_get_contents('kd_logo.png'));
$logo_width = imagesx($logo);
$logo_height = imagesy($logo);
$from_width = ($imgW - 7 - $outerFrame) * $pixelPerPoint;

imagecopyresampled($dest_image, $logo, $from_width, $from_width, 0, 0, 7 * $pixelPerPoint, 7 * $pixelPerPoint, $logo_width, $logo_height);
```

## 显示或保存图片
```
if ($filename === false) {
	Header("Content-type: image/png");
	ImagePng($image);
} else {
	if($saveandprint===TRUE){
		ImagePng($image, $filename);
		header("Content-type: image/png");
		ImagePng($image);
	}else{
		ImagePng($image, $filename);
	}
}
```

## 样例展示
!(demo1)[/images/qrcode/qrcode1.png]

!(demo2)[/images/qrcode/qrcode2.png]

相关链接 : [二维码的生成原理](http://aibenlin.com/web/2017/03/22/qrcode.html)
相关链接 : [php demo代码](https://github.com/hardwork537/phpqrcode)