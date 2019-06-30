### 日常LeetCode刷题总结 
---
刷题的时光总是快乐的~
---

#### 2019.06.20 星期四  
456. 132 Pattern   
缘起：群里发的头条抖音算法岗面试算法题（原题）  
知识点：单调栈  


3. Longest Substring Without Repeating Characters  
缘起：同学提起  
知识点：unordered_map 以及下标过期策略  

69. Sqrt(x)  
缘起：听B站课  
知识点：牛顿迭代法 和 二分法  
收获：  
1）二分法的小技巧：  
```cpp
if (check(mid)) {
    left = mid;
}
else {
    right = mid-1;
}
这种情况 int mid = left + right + 1 >> 1;需要+1，否则mid可能会一直保持不变，所以left也会保持不变，陷入死循环。    
if (check(mid)) {
    right = mid;
}
else {
    left = mid+1;
}
这种情况 int mid = left + right >> 1; 不需要
```
2）牛顿迭代法思想：以直代曲，既然切线能在极小的范围能近似的代替曲线来简化运算，那么就将这一思想运用彻底，反正事情肯定朝着好的方向去发展的。  
`f(x) = x2 - a` 根据f(x)在x0处的一阶泰勒展开，f'(x) = 2x; f(x) = f(x0)/0! + (f'(x0)/1!)*(f(x0)-x0)令其为0 算出x0继续迭代  


34. Find First and Last Position of Element in Sorted Array  
缘起：之前看到过面试有考，也刚刚复习到二分  
知识点：lower_bound和upper_bound的写法，都是与right那个有关，所以正好印证了上个结论。


#### 2019.06.30 星期日 
239. Sliding Window Maximum    
缘起：最近在学习彻底掌握单调栈单调队列类似的算法  
知识点：学习使用单调队列  
单调队列思想：  
* 队首保持着想要的答案 
* 队列中尽量消除无用的元素  
单调队列的实现： 
使用的基本数据结构：deque  
方法一<记录下标验证过期>：  
队首确实一直保持着想要的答案，但是在这个sliding window的时候可能最大值会滑出窗口，所以可以通过记录下标来验证是否过期。  

方法二<通过对比对首元素和num[i-k]>：
正如前一种方法所言，需要额外记录下标，这种方法就是更直接的去比较最大值是否就是刚刚过期的那个值。  
```cpp
class MonotonicQueue
    {
        public:
            void pop() {Q.pop_front();}
            void push(int val) {
                while (!Q.empty() && val > Q.back()) Q.pop_back();
                Q.push_back(val);
            }
            int getMax()
            {
                return Q.front();
            }
        private:
            deque<int> Q ;
    };
```
