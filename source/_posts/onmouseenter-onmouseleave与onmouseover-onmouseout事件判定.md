---
title: onmouseenter/onmouseleave与onmouseover/onmouseout事件判定
date: 2019-04-18 01:37:02
tags:
	- DOM 事件
	- HTML
categories: 学习笔记
---

# 事件的作用
 onmouseenter/onmouseleave和onmouseover/onmouseout都可以用来监听鼠标是否移入对应的元素内，但是两者在触发事件的情况下有所不同。
 先说结果：
 	onmouseenter/onmouseleave 鼠标进入到指定元素区域内触发事件，不支持冒泡，不包含子元素的区域。
 	onmouseover/onmouseout 鼠标进入指定元素触发事件，含子元素区域。

# 代码实现
 ```HTML
 <!DOCTYPE html>
<html>
	<head>
		<meta charset="UTF-8">
		<title></title>
		<style type="text/css">
			*{
				margin: 0;
				padding: 0;
			}
			#wrap{
				position: absolute;
				left: 0;
				right: 0;
				top: 0;
				bottom: 0;
				margin: auto;
				width: 400px;
				height: 400px;
				background: pink;
			}
			#inner{
				width: 300px;
				height: 300px;
				background: deeppink;
			}
			#test{
				width: 200px;
				height: 200px;
				background: blue;
			}
		</style>
	</head>
	<body>
		<div id="wrap">
			<div id="inner">
				<div id="test">
				</div>
			</div>
		</div>
	</body>
</html>
 ```
 我们先创建一个由三个div相互包裹的页面：
 ![html](/image/onmouse/onmousehtml.jpg)
 然后我们在分别为其绑定onmouseenter/onmouseleave和onmouseover/onmouseout事件
 ## onmouseenter/onmouseleave
 绑定代码如下
 ```HTML
 <script type="text/javascript">
	;window.onload = function () {
		test.onmouseenter = function () {
			console.log('test enter')
		}
		test.onmouseleave = function () {
			console.log('test leave')
		}
		inner.onmouseenter = function () {
			console.log('inner enter')
		}
		inner.onmouseleave = function () {
			console.log('inner leave')
		}
		wrap.onmouseenter = function () {
			console.log('wrap enter')
		}
		wrap.onmouseleave = function () {
			console.log('wrap leave')
		}
	}
</script>
 ```
 当我们把鼠标移入进test元素后，控制台显示
 ![mouseenter](/image/onmouse/mouseconsole_1.png)
 由此可见并不支持冒泡，onmouseleave也同理
 ![mouseleave](/image/onmouse/mouseconsole_2.png)
## onmouseover/onmouseout
 绑定代码如下
 ```HTML
 <script type="text/javascript">
	;window.onload = function () {
		test.onmouseover = function () {
			console.log('test over')
		}
		test.onmouseout = function () {
			console.log('test out')
		}
		inner.onmouseover = function () {
			console.log('inner over')
		}
		inner.onmouseout = function () {
			console.log('inner out')
		}
		wrap.onmouseover = function() {
			console.log('wrap over')
		}
		wrap.onmouseout = function () {
			console.log('wrap out')
		}
	}
</script>
 ```
 当我们把鼠标移入test中时，控制台会显示
 ![mouseover](/image/onmouse/mouseconsole_3.png)
onmouseover事件出发了冒泡，onmouseout也会同理触发。

# 注意
当我们为test开启了定位，将鼠标移入test中，即使视觉上鼠标没有移入wrap和inner元素，浏览器会从结构上判断你已经将鼠标移入了test的父级元素。
```CSS
#test{
		width: 200px;
		height: 200px;
		background: blue;
		position: relative;
		left:-400px;
		top:0px;
	}
```
 ![mouseenter](/image/onmouse/mouseconsole_1.png)