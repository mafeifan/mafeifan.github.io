---
title: codewar
date: 2017-08-09 22:13:58
tags: 算法
---

记录在 http://www.codewars.com 做的算法题

<!--more-->

### Persistent Bugger

Write a function, persistence, that takes in a positive parameter num and returns its multiplicative persistence, which is the number of times you must multiply the digits in num until you reach a single digit.

		persistence(39) === 3 // because 3*9 = 27, 2*7 = 14, 1*4=4
		                       // and 4 has only one digit

		persistence(999) === 4 // because 9*9*9 = 729, 7*2*9 = 126,
		                        // 1*2*6 = 12, and finally 1*2 = 2

		persistence(4) === 0 // because 4 is already a one-digit number


注意和提示
* 返回的是连乘的次数，有时候一个很长的数不一定能执行多次，比如254123421。因为2*5=10 出现了0.所以结果是2。
* 考虑到前后数的连续性，可使用JS提供的reduce函数，[详见](/2017/07/20/reduce-function-javascript/)


答案：

```javascript
function persistence(num) {
   var times = 0
   
   num = num.toString()
   
   while (num.length > 1) {
     times++
     num = num.split('').map(Number).reduce((a, b) => a * b).toString()
   }
   
   return times
}
```


### Find the divisors!

Create a function named divisors that takes an integer and returns an array with all of the integer's divisors(except for 1 and the number itself). If the number is prime return the string '(integer) is prime' (use Either String a in Haskell and Result<Vec<u32>, String> in Rust).

该题目是返回一个数的所有因数
 
注意和提示
* prime指质数

答案：

```javascript
function divisors(integer) {
  let max = Math.round(integer/2)
  // 当然完全可以用array类型
  const set  = new Set()
  for (let i=2; i <= max; i++) {
    if (integer % i === 0) {
      set.add(i)
    } 
  }
  return set.size ? Array.from(set) : `${integer} is prime`
};
```

### Find The Parity Outlier

You are given an array (which will have a length of at least 3, but could be very large) containing integers. The array is either entirely comprised of odd integers or entirely comprised of even integers except for a single integer N. Write a method that takes the array as an argument and returns N.

从至少包含三个数的数组中找出唯一的奇数或偶数

For example:

[2, 4, 0, 100, 4, 11, 2602, 36] Should return: 11

[160, 3, 1719, 19, 11, 13, -21] Should return: 160


注意和提示
* prime指质数

答案：

```javascript
function findOutlier(integers){
  let firstNum  = Math.abs(integers[0]) % 2
  let secondNum = Math.abs(integers[1]) % 2
  // 前俩数一奇数一偶数
  if (firstNum !== secondNum){
      if (Math.abs(integers[2]) % 2 === firstNum) {
        return integers[1]
      }else {
        return integers[0]
      }
  }
  //前两数都是奇数或都是偶数
  for(let i=0; i < integers.length - 2; i++) {
    if (Math.abs(integers[i+2]) % 2 !== firstNum) {
      return integers[i+2]
    }
  }
}
```


- 思路

先取出前两个数的奇偶性，如果同为奇数或偶数，则直接返回第三个数。不然的话一个个判断吧

更简洁的写法，分别提出来所有的奇数和偶数，但是效率不高

```javascript
function findOutlier(integers){
  const even = integers.filter(int => int % 2 === 0);
  const odd  = integers.filter(int => int % 2 !== 0);
  return even.length === 1 ? even[0] : odd[0];
}
```