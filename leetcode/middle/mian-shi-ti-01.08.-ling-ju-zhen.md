# 面试题 01.08. 零矩阵

{% embed url="https://leetcode.cn/problems/zero-matrix-lcci/" %}

```cpp
class Solution {
public:
    // 思路：
    // 1.用两个标记位，标记首行，首列是否有0
    // 2.遍历非首行首列元素，如果遇到0，将0移到该元素对应的行列首元素位置
    // 3.遍历首行首列，遇到0就将对应的列和行置0
    // 4.通过两个标记为决定首行首列是否为0
    void setZeroes(vector<vector<int>>& matrix) {
        bool flag1 = false;
        bool flag2 = false;
        for (auto& v: matrix[0]) {flag1 = flag1 ? true : v == 0;}
        for (auto& v: matrix) {flag2 = flag2 ? true : v[0] == 0;}
        for (int i = 1; i < matrix.size(); ++i) {
            for (int j = 1; j < matrix[i].size(); j++) {
                if (matrix[i][j] == 0) {
                    matrix[0][j] = 0;
                    matrix[i][0] = 0;
                }
            }
        }
        for (int i = 1; i < matrix.size(); ++i) {
            if (matrix[i][0] == 0) {
                for (int j = 0; j < matrix[i].size(); ++j) matrix[i][j] = 0;
            }
        }

        for (int j = 0; j < matrix[0].size(); ++j) {
            if (matrix[0][j] == 0) {
                for (int i = 0; i < matrix.size(); ++i) matrix[i][j] = 0;
            }
        }

        if (flag1) {
            for (int i = 0; i < matrix[0].size(); ++i) matrix[0][i] = 0;
        }
        if (flag2) {
            for (int i = 0; i < matrix.size(); ++i) matrix[i][0] = 0;
        }
    }
};
```
