# 面试题 01.02. 判定是否互为字符重排

{% embed url="https://leetcode.cn/problems/check-permutation-lcci/" %}

```cpp
class Solution {
public:
    // bool CheckPermutation(string s1, string s2) {
    //     sort(s1.begin(), s1.end());
    //     sort(s2.begin(), s2.end());
    //     return s1 == s2;
    // }
    bool CheckPermutation(string s1, string s2) {
        if (s1.size() != s2.size()) return false;
        vector<int> v(128, 0);
        for (int i = 0; i < s1.size(); ++i) v[s1[i]]++;
        for (int i = 0; i < s2.size(); ++i) if (v[s2[i]] > 0) v[s2[i]]--;
        return std::accumulate(v.begin(), v.end(), 0) == 0;
    }
};
```
