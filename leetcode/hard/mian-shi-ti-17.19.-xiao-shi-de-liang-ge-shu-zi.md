# 面试题 17.19. 消失的两个数字

{% embed url="https://leetcode.cn/problems/missing-two-lcci/" %}

```cpp
class Solution {
public:
    // 最开始想法：算1+n的加和，nums数组的加和，得到两数之和；然后抽屉排序找到其中一个得出结果
    // 这个想法44个用例过了41个，感觉有漏洞
    
    // 后来看题解想法，插入两个-1，还是抽屉排序，然后-1的位置就是缺失数字
    vector<int> missingTwo(vector<int>& nums) {
        nums.push_back(-1);
        nums.push_back(-1);
        for (int i = 0; i < nums.size(); i++) {
            while (nums[i] != -1 && nums[i] != i+1) swap(nums[i], nums[nums[i]-1]);
        }
        vector<int> res;
        for (int i = 0; i < nums.size(); ++i) {
            if (nums[i] == -1) res.push_back(i+1);
        }
        return res;
    }
    // 后续：这题感觉有老6
    // 这个解法空间耗时等都不太理想，尝试还是用加和方法，找到一个i就返回另一个，结果出错
    // 也就是说，-1第一次被置换到当前位置，可能并不代表缺失这个数字；后续还有可能把-1给置换到其他位置？
};
```
