# 【力扣】777. 在LR字符串中交换相邻字符

**题目：** 777. 在LR字符串中交换相邻字符

**难度：** 中等

**提交时间：** 2022-10-02


### 问题描述

在一个由 'L' , 'R' 和 'X' 三个字符组成的字符串（例如"RXXLRXRXL"）中进行移动操作。一次移动操作指用一个"LX"替换一个"XL"，或者用一个"XR"替换一个"RX"。现给定起始字符串start和结束字符串end，请编写代码，当且仅当存在一系列移动操作使得start可以转换成end时， 返回True。

```示例 

示例 :

输入: start = "RXXLRXRXL", end = "XRLXXRRLX"
输出: True
解释:
我们可以通过以下几步将start转换成end:
RXXLRXRXL ->
XRXLRXRXL ->
XRLXRXRXL ->
XRLXXRRXL ->
XRLXXRRLX
 

提示：

1 <= len(start) = len(end) <= 10000。
start和end中的字符串仅限于'L', 'R'和'X'。
```

>来源：力扣（LeetCode）
>链接：https://leetcode.cn/problems/swap-adjacent-in-lr-string
>著作权归领扣网络所有。商业转载请联系官方授权，非商业转载请注明出处。

### 解法

#### 方法一：双指针，脑筋急转弯

 **思路与算法** 

假设end字符串是由start字符串转换而来，那么必须满足两个条件：
- start和end字符串剔除掉'X'之后相同
- L字符在start字符串中的的位置一定在end的右边；R字符在start字符串中的位置一定在end的左边；

如果不满足其中一个，则返回false;


**代码**

```java
class Solution {
    public boolean canTransform(String start, String end) {
        if(!start.replace("X","").equals(end.replace("X",""))) return false;
        int len = start.length();
        for(int i =0,j=0;i<len;++i){
            if(start.charAt(i)=='X') continue;
            while(end.charAt(j)=='X')++j;
            if(i!=j&&(start.charAt(i)=='L')==(i<j)){
                return false;
            }
            ++j;
        }
        return true;
    }
}
```

**执行结果**

![执行结果](img/20221002214902.png)


**复杂度分析**
- 时间复杂度：O(n)，n是字符串的长度
- 空间复杂度：O(1)

### 总结

解题逻辑理解之后，就是编码，下面这段代码无敌

```java
      i!=j&&(start.charAt(i)=='L')==(i<j)          
```

灵佬无敌！附一个灵佬力扣主页（https://leetcode.cn/u/endlesscheng/）

### 深入研究

类似一道题：[2337.移动片段得到字符串](https://leetcode.cn/problems/move-pieces-to-obtain-a-string/)可以研究一下







