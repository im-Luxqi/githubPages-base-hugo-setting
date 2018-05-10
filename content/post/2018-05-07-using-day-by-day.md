---
title: 博客日常维护 
subtitle: 千里之行，始于足下。
date: 2018-05-08
tags: ["基础杂记"]

---

刚刚开始写blog,有写markdown,hugo书写规范不太熟悉，总结常用的写法，方便日后查阅。

**坚持，加油！！！**

<!--more-->
[建站时参考的优秀blog](https://keysaim.github.io/)
##  Markdown , hugo
[markdown官网](https://www.markdowntutorial.com/)         [hugo-easy-gallery](https://github.com/liwenyip/hugo-easy-gallery/)
	
### 1.显示代码块

`www.baidu.com`

```javascript
//多行显示 ```javascript
var num1, num2, sum
num1 = prompt("Enter first number")
num2 = prompt("Enter second number")
```

{{< highlight javascript >}}
//多行显示< highlight javascript >
var num1, num2, sum
num1 = prompt("Enter first number")
num2 = prompt("Enter second number")
sum = parseInt(num1) + parseInt(num2) 
alert("Sum = " + sum)
{{</ highlight>}}

	
{{< highlight javascript "linenos=inline">}}
//多行显示< highlight javascript "linenos=inline">
var num1, num2, sum
num1 = prompt("Enter first number")
num2 = prompt("Enter second number")
{{</ highlight >}}
	
### 2.超链接
`[gallery](https://github.com/liwenyip/hugo-easy-gallery/)`


### 3.ul列表
{{< highlight javascript>}}
- 第一行
- 第二行
- 第三行
- 第四行
{{</ highlight>}}

###	4.简单表格
| Number | Next number | Previous number |
| :------ |:--- | :--- |
| Five | Six | Four |
| Ten | Eleven | Nine |
| Seven | Eight | Six |
| Two | Three | One |
how creat:
{{< highlight javascript>}}
| Number | Next number | Previous number |
| :------ |:--- | :--- |
| Five | Six | Four |
| Ten | Eleven | Nine |
| Seven | Eight | Six |
| Two | Three | One |
{{</ highlight >}}

	
###	5.显示图片

{{< gallery hover-effect="none">}}
{{< figure thumb="-thumb" link="/img/hexagon.jpg" >}}
{{< /gallery >}}

{{< gallery caption-position="bottom" caption-effect="fade" >}}
{{< figure thumb="-thumb" link="/img/sphere.jpg" caption="caption-effect(fade)" >}}
{{< /gallery >}}

{{< gallery caption-position="center" caption-effect="fade" >}}
{{< figure thumb="-thumb" link="/img/triangle.jpg" caption="caption-position(center)" >}}
{{< /gallery >}}
	




	
	
	

	