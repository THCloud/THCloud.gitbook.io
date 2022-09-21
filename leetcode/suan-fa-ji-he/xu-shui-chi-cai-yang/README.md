# 蓄水池采样

一种针对不定长数据流的抽样方法。

给定一个不定长数据流（随时有增加），返回其中固定n个数据，要求每个数据抽样率相同。

```go
// 示例代码
func sample(value []string, num int) []string {
	if len(value) <= num {
		return value
	}
	result := value[:num]
	for i := num; i < len(value); i++ {
		r := rand.Intn(i)
		if r < num {
			result[r] = value[i]
		}
	}
	return result
}
```
