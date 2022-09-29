# 面试题 01.09. 字符串轮转

{% embed url="https://leetcode.cn/problems/string-rotation-lcci/" %}

```cpp
class Solution {
public:
    bool isFlipedString(string s1, string s2) {
        return s1.size() == s2.size() && (s2+s2).find(s1) != std::string::npos;
    }
};
```
