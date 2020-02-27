---
title: JavaScript正则表达式
date: 2018-09-08 23:20:22
categories: JavaScript
tags: 
---

* 快速字符

  | 字符 | 说明                                                |
  | ---- | --------------------------------------------------- |
  | \d   | 所有数字，相当于 [0-9]                              |
  | \D   | 非数字，相当于 ``[^0-9]``                           |
  | \s   | 空白字符，包括空格、制表符、换页符、换行符          |
  | \S   | 非空白字符                                          |
  | \w   | 单字符（字母、数字或者下划线），相当于 [A-Za-z0-9_] |
  | \W   | 非单字符                                            |

* 使用 {} 表示出现的次数

  ``` javascript
  /s{2,6}/   // s 字符出现 2 到 6 次，要注意这些情况：ssssssss，这里会当作 ssssss ss，而匹配成功
  ```

  ``` javascript
  /s{2,}/    // s 字符至少出现 2 次以上
  ```

  ``` javascript
  /s {2}/	   // s 字符精确出现 2 次
  ```

* 使用 ? 表示出现 0 次或者 1 次，等价于 {0,1}

  ``` javascript
  /apples?/	// 匹配 aaple 和 apples
  ```

  注意：如果紧跟在任何量词 *、+、? 或 {} 的后面，将会湿的量词变为非贪婪（匹配尽量少的字符），缺省使用的是贪婪模式

  ``` javascript
  /\d+/;	// 对于 123abc 将会返回 123
  /\d+?/;	// 对于 123abc 将会返回 1
  ```

* 使用 ^ 表示开头匹配，使用 $ 表示末尾匹配

  ``` javascript
  /^a/	// 匹配 apple 不匹配 banana
  /e$/	// 匹配 apple 不匹配 element
  ```

  注意和 反向字符集 的区别

  ``` javascript
  /[^abc]/	// 匹配不包括 abc 的字符
  ```

* 使用 /i 表示忽略大小写

  ``` javascript
  /abc/i
  ```

* 使用 /g 表示全局搜索

  ``` javascript
  /abc/g
  ```

* 使用 x(?=y) 正向肯定查找，匹配'x'仅仅当'x'后面跟着'y'.

  ``` javascript
  /Jack(?=Sprat)/	// 会匹配到 Jack 仅当它后面跟着 Sprat，但 Sprat 不作为匹配项返回
  ```

* 使用 x(?!y) 反向否定查找，匹配'x'仅仅当'x'后面不跟着'y'

  ``` javascript
  /Jack(?!Sprat)/	//	会匹配到 Jack 仅当它后面不跟着 Sprat
  ```

* 使用 (x) 匹配 'x' 并且记住匹配项，括号被称为 *捕获括号*

  ``` javascript
  /(foo) (bar) \1 \2/;	// 匹配 foo bar foo bar，注意空格
  ```

  如果使用 replace 功能，则需要使用 $1 等

* 使用 (?:x) 匹配 'x' 但是不记住匹配项。这种叫作非捕获括号，使得你能够定义为与正则表达式运算符一起使用的子表达式

  ``` javascript
  /foo{2}/;		// 将匹配 fooo，将 {2} 作用到 o
  /(?:foo){2}/;	// 将匹配 foofoo，将 {2} 作用到 foo
  ```