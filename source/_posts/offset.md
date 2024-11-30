---
title: offset
date: 2018-01-03 10:40:13
tags:
  - CSS
---
先来一张网上的图
![](http://oy9tlpm12.bkt.clouddn.com/3131637-50e9ea1bc0764f99.png)

当然，这个图是有浏览器的兼容问题的，不是每个浏览器都是这样定义，个人觉得有两种情况，内容盒模型(标准盒模型)，边框盒模型(IE怪异盒模型)

<!-- more -->

# 内容盒模型（标准盒模型）

`offsetWidth、offsetHeight`: 包含自身width、border、padding.(div.style.width 因为只能获取行内的数值)

`offsetLeft、offsetTop`: 对于块级元素，相对最近有定位属性的父级元素（position不为默认值,可通过el.offsetParent获取）左边的距离，如果都没有定位属性则以html元素为基准。注意，在chrome、IE8中这里的比较的是父级border的外边缘与当前元素的外边缘，而firefox、IE7、IE9比较的是父级border的内边缘与当前元素的外边缘。对于可被截断到下一行的行内元素（如 span），描述的是边界框的尺寸（使用 Element.getBoundingClientRect 来获取其位置）

`clientTop、clientLeft`: 一般返回元素的border宽度

`clientWidth、clientHeight`: Element.clientWidth 属性表示元素的内部宽度，以像素计。该属性包括内边距，但不包括垂直滚动条（如果有）、边框和外边距。该属性值会被四舍五入为一个整数。如果你需要一个小数值，可使用 element.getBoundingClientRect()。IE7/8有点不一样
`scrollWidth、scrollHeight`: 元素的scrollWidth只读属性以px为单位返回元素的内容区域宽度或元素的本身的宽度中更大的那个值。若元素的宽度大于其内容的区域（例如，元素存在滚动条时）, scrollWidth的值要大于clientWidth

`scrollLeft、scrollTop`: Element.scrollLeft 属性可以读取或设置元素滚动条到元素左边的距离。也就是滚动条位置

> 获取浏览器滚动条时要注意兼容：
> + 未声明 DTD 时（谷歌只认识他）
`document.body.scrollTop`
> + 已经声明DTD 时（IE678只认识他）
`document.documentElement.scrollTop`
> + 火狐/谷歌/ie9+以上支持的
`window.pageYOffset`

兼容获取浏览器滚动条

```javascript
window.pageYOffset || document.documentElement.scrollTop || document.body.scrollTop
```


# 边框盒模型（IE怪异盒模型）

```offsetWidth、offsetHeight```: 等于自身width, width包含padding,border

```offsetLeft、offsetTop```: 除IE7、IE9比较的是父级border的内边缘与当前元素的外边缘， firefox、chrome、ie8是父级border的外边缘与当前元素的外边缘 

```clientTop、clientLeft```: 一般返回元素的border宽度