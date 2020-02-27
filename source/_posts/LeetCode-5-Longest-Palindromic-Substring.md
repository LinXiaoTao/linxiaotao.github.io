---
title: 'LeetCode 5: Longest Palindromic Substring'
date: 2019-12-12 14:35:50
categories: LeetCode
tags:
---

### 题目描述

给定一个字符串 `s`，找到 `s` 中最长的回文子串。你可以假设 `s` 的最大长度为 1000。

### 解题思路

* 中心扩展算法

  计算以字符串中的每一个字符为中心，向两边扩展的最长回文串。

  以字符串 "babad" 为例，第一个字符 'b'，以它为中心的最长回文串就是它本身，再看第二个字符 'a'，以它为中心的最长回文串是 "bab"。

  代码如下：

  ``` java
      public String longestPalindrome2(String s) {
          if (s.length() < 2) {
              return s;
          }
          int[] ret = new int[2];
          for (int i = 0; i < s.length(); i++) {
              loop(ret, i, s);
          }
          return s.substring(ret[0], ret[1]);
      }
  
      private void loop(int[] ret, int low, String s) {
          int high = low;
          // Code 1
          while (high < s.length() - 1 && s.charAt(high + 1) == s.charAt(low)) high++;
          while (high < s.length() && low >= 0 && s.charAt(low) == s.charAt(high)) {
              high++;
              low--;
          }
          if (high - low - 1 > ret[1] - ret[0]) {
              ret[0] = low + 1;
              ret[1] = high;
          }
      }
  ```

  Code 1 处是考虑偶数回文串的情况，比如 "aa" 这种也是回文串，只要是相同字符，不管有多少位都是回文串。

  | 空间复杂度 | 时间复杂度   |
  | ---------- | ------------ |
  | O(1)       | O($$N ^ 2$$) |

  

* 动态规划

  使用一个二维数组来缓存回文串的判断，`dp[i][j]` 表示从下标 i 到 j 的字符串是否为回文串。

  状态转移方程如下：

  ``` java
  dp[i][j] = dp[i+1][j-1] && S(i) == S(j)
  ```

  代码如下：

  ``` java
      public String longestPalindrome(String s) {
  
          if (s.length() < 2) {
              return s;
          }
  
          String max = s.substring(0, 1);
          boolean[][] dp = new boolean[s.length()][s.length()];
          for (int i = s.length() - 1; i > -1; i--) {
              for (int j = i; j < s.length(); j++) {
                // Code 1
                  dp[i][j] = (s.charAt(i) == s.charAt(j))
                          && (j - i < 3 || dp[i + 1][j - 1]);
                  if (dp[i][j] && j - i + 1 > max.length()) {
                      max = s.substring(i, j + 1);
                  }
              }
          }
          return max;
      }
  ```

  Code 1 代码处中的 `j - i < 3` 是考虑 "a"、"aa" 和 "aba" 这些情况。

  | 空间复杂度 | 时间复杂度 |
  | ---------- | ---------- |
  | O($$N^2$$) | O($$N^2$$) |

  

* Manacher 算法

  这里有篇详细介绍的文章，可移步：[最长回文子串——Manacher 算法](https://segmentfault.com/a/1190000003914228)

  相对于前面两种解法，Manacher 算法有以下优点：

  * 通过在字符间插入特殊字符，使得字符串的长度变成奇数，不会影响最终结果，但不用考虑偶数回文串的情况
  * 复用已经计算好的结果
  
  代码如下：
  
  ``` java
  public String longestPalindrome(String s) {
          if (s.length() < 2) {
              return s;
          }
          StringBuilder builder = new StringBuilder();
          builder.append("#");
          for (int i = 0; i < s.length(); i++) {
              builder.append(s, i, i + 1);
              builder.append("#");
          }
          s = builder.toString();
          // 下标为 i 的字符为中点的最长回文串的半径
          int[] dp = new int[s.length()];
          builder.delete(0, builder.length());
  				
    			// 计算过的区域
          int maxRight = 0;
          int maxLen = 0;
          // maxRight 对应的回文串的中点
          int pos = 0;
  
          for (int i = 0; i < s.length(); i++) {
  
              if (i < maxRight) {
                // 2 * pos - i 表示，i 关于 pos 对称的点
                  dp[i] = Math.min(dp[2 * pos - i], maxRight - i + 1);
              } else {
                // 默认值为 1
                  dp[i] = 1;
              }
  
              while (i + dp[i] < s.length() && i - dp[i] >= 0 && s.charAt(i + dp[i]) == s.charAt(i - dp[i])) {
                // 中点扩散
                  dp[i] += 1;
              }
  
              if (i + dp[i] - 1 > maxRight) {
                  maxRight = i + dp[i] - 1;
                  pos = i;
              }
  
              maxLen = Math.max(maxLen, dp[i]);
              if (maxLen == dp[i]) {
                  if (builder.length() > 0) {
                      builder.delete(0, builder.length());
                  }
                  builder.append(s, i - dp[i] + 1, i + dp[i]);
              }
  
          }
  
          return builder.toString().replace("#", "");
  
      }
  ```
  
  | 空间复杂度 | 时间复杂度 |
  | ---------- | ---------- |
  | O($$N$$) | O($$N$$) |
  
  
  
  
  
  