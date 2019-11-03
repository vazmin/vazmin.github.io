---
title:  "Threejs Object3d 灰度处理"
date:   2019-11-03 16:00:00 +0800
categories:
  - js
tags:
  - threejs
  - gray processing
---

![选择角色](/assets/images/2019-11-03/gray.png){:height="667" width="357"}

如图，将未得到的角色模型调整为灰色。

> 灰度化，在RGB模型中，如果R=G=B时，则彩色表示一种灰度颜色，其中R=G=B的值叫灰度值。
> 因此，灰度图像每个像素只需一个字节存放灰度值（又称强度值、亮度值），灰度范围为0-255。
> 一般有分量法、最大值法、平均值法、加权平均法四种方法对彩色图像进行灰度化。

`Mesh.geometry.attributes.color`记录`Mesh`模型的RGB数据，color的类型为`BufferAttribute`。
`BufferAttribute`中的array则是rgb值数组，`BufferAttribute.array[0-2]`记录一个rgb。

本文采用平均值做灰度处理: `gray = (r + g + b) / 3`。


```javascript
/**
 * 将 BufferGeometry 每个面的颜色转灰度
 * @param {TypedArray} array BufferAttribute.color
 */
export const rgbToGrayscale = (array) => {
    for (let i = 0; i < array.length; i += 3) {
        let avg = (array[i] + array[i + 1] + array[i + 2]) / 3;
        array[i] = avg;
        array[i + 1] = avg;
        array[i + 2] = avg;
    }
}
```


另附：

![play](/assets/images/2019-11-03/play.gif){:height="667" width="357"}
