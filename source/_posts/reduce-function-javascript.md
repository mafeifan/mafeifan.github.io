---
title: reduce-function-javascript
date: 2017-07-20 17:56:08
tags: javascript
---

[1]:/images/0720_reduce-function-javascript/param.jpg
[2]:/images/0720_reduce-function-javascript/1-1.jpg
[3]:/images/0720_reduce-function-javascript/1-2.jpg
[4]:/images/0720_reduce-function-javascript/2-1.jpg
[5]:/images/0720_reduce-function-javascript/2-2.jpg
[6]:/images/0720_reduce-function-javascript/2-3.jpg

转载： [JavaScript中reduce()方法不完全指南](https://aotu.io/notes/2016/04/14/js-reduce/index.html)
[More From MDN](https://developer.mozilla.org/zh-CN/docs/Web/JavaScript/Reference/Global_Objects/Array/Reduce)

reduce() 方法接收一个函数作为累加器，数组中的每个值（从左到右）开始缩减，最终为一个值，是ES5中新增的又一个数组逐项处理方法，那reduce方法跟foreach、map等数组方法又有啥区别呢。

``` javascript
array.reduce(function(accumulator, currentValue, currentIndex, array), initialValue)
```

### 参数：

- callback（一个在数组中每一项上调用的函数，接受四个函数：）
 + previousValue（上一次调用回调函数时的返回值，或者初始值）
 + currentValue（当前正在处理的数组元素）
 + currentIndex（当前正在处理的数组元素下标）
 + array（调用reduce()方法的数组）
- initialValue（可选的初始值。作为第一次调用回调函数时传给previousValue的值）


### 图解参数:

抛开上面晦涩难懂的语法介绍，下面我们直接上实例：

![param][1]

<!--more-->

### 实例:

数组 arr = [1,2,3,4] 求数组的和

#### forEach 实现

``` javascript
var arr = [1,2,3,4],
sum = 0;
arr.forEach(function(e){sum += e;}); // sum = 10  just for demo
```

#### map 实现

``` javascript
var arr = [1,2,3,4],
sum = 0;
arr.map(function(obj){sum += obj});//return undefined array. sum = 10  just for demo
```

#### reduce 实现

``` javascript
var arr = [1,2,3,4],
arr.reduce(function(pre,cur){return pre + cur}); // return 10
```

没错，reduce专为累加这种操作而设计，为累加这类操作而设计的参数，十分方便。
那么问题来了，reduce方法的参数到底有哪些用法呢？


### 参数分解

1. 不传initialValue值

  ```javascript
  var arr = [1,2,3];
  arr.reduce(function(pre,cur,index,arr){debugger;return pre+cur});
  ```

  探查Arguments数组得到：

  * 第一次debugger

  ![][2]

  * 第二次debugger

  ![][3]

  * 第三次debugger

  ```
  return 6;
  ```

2. 传入initialValue值

  ```
  var arr = [1,2,3]
  arr.reduce(function(pre,cur,index,arr){debugger;return pre+cur},10);
  ```

  * 第一次debugger

  ![][4]

  * 第二次debugger

  ![][5]

  * 第三次debugger

  ![][6]

  * 第四次debugger

  ```
  return 16
  ```


  可以看出传入initialValue 会多递归一次，而initialValue的值的作用大家应该也明了了：为累加等操作传入起始值（额外的加值）。
  而callbackfn中的四个参数也可以在debugger的动态变化中查看出具有的特性。
  那么利用reduce方法还可以做哪些事情呢?

### 更多实例

求乘积

```	javascript
var pro = arr.reduce(function(pre,cur,index,arr){return pre * cur})

```

求最大值

```	javascript
var max = a.reduce(function(pre,cur,inde,arr){return pre>cur?pre:cur;});

```

综合例子

搬砖工小王拿到了这样的数据格式：`var arr = [ {name: 'brick1'}, {name: 'brick2'}, {name: 'brick3'}]`
希望得到这样的数据格式： `'brick1, brick2 & brick3'`

```javascript
var arr =  [ {name: 'brick11'}, {name: 'brick12'}, {name: 'brick13'} ]
function carryBricks(arr){
  return arr.reduce(function(prev, current, index, array){
    if (index === 0){
      return current.name;
    }
    else if (index === array.length - 1){
      return prev + ' & ' + current.name;
    }
    else {
      return prev + ', ' + current.name;
    }
  }, '');
 }
```
返回结果：brick11, brick12 & brick13

此时进一步延伸如果原来有一堆砖已经堆好，传入 initialValue值：

```javascript
var arr =  [ {name: 'brick11'}, {name: 'brick12'}, {name: 'brick13'} ]
var bricks = 'brick1, brick2, brick3, ' //已经堆好的砖
function carryBricks(arr,bricks){
  return arr.reduce(function(prev, current, index, array){
    if (index === 0){
      return prev + current.name;
    }
    else if (index === array.length - 1){
      return prev + ' & ' + current.name;
    }
      return prev + ', ' + current.name;
  }, bricks);
 }
 ```

返回结果：brick1, brick2, brick3, brick11, brick12 & brick13

### 总结

至此，我们可以很形象的归纳出来forEach、map以及reduce的不同点：

* forEach 方法是将数组中的每一个值取出做一些程序员想让他们做的事情
* map 方法 是将数组中的每一个值放入一个方法中做一些程序员想让他们做的事情后返回一个新的数组
* reduce 方法 将数组中的每一个值与前面的被返回相加的总和(初试值为数组的第一个值或者initialValue)


