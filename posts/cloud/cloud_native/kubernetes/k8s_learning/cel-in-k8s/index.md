# Kubernetes - CEL 语法


## 1 概述

Kubernetes 中 Validate 规则都是通过 [**Common Expression Language**](https://github.com/google/cel-go) 来表达的。

CEL 表达式通常是一行简短的代码，能够内联到 Kubernetes 资源的属性中。不同的资源可能有着不同的输入，因此当你使用 CEL 时，需要先确认表达式的输入是什么。

例如，在 CRD 的 Validate 中，`self` 和 `oldSelf` 变量代表着当前对象和修改前的对象。

```python
# 判断 replicas 是否合法
self.minReplicas <= self.replicas && self.replicas <= self.maxReplicas

# 判断 map 中是否包含 'Available' 的 key
'Available' in self.stateCounts
```

## 2 CEL 库

### 2.1 List 操作

`indexOf` 和 `lastIndexOf` 用于获取 list 中的某个元素的下标。

```python
# 判断 list 第一个元素的名字正不正确
names.indexOf('should-be-first') == 1
```

`min` 和 `max` 用于获取 list 中元素的最小值或最大值，`sum` 用于计算所有元素之和。

```python
# 判断 weight 之和是不是 1
items.map(x, x.weight).sum() == 1.0

# 判断两个优先级不重合
lowPriorities.map(x, x.priority).max() < highPriorities.map(x, x.priority).min()
```

`isSorted` 用于判断 list 中所有元素是否按字母序排序。

```python
names.isSorted()
```

### 2.2 Regex 库

通过 `match` 函数来判断字符串是否匹配 Regex。

```python
# 判断 'MY_ENV' 的 值是否合法
self.envars.filter(e, e.name = 'MY_ENV').all(e, e.value.matches('^[a-zA-Z]*$')
```

通过 `find` 和 `findAll` 来根据 Regex 查找。

```python
# 找到字符串中第一个数字
"abc 123".find('[0-9]*')

# 检查所有数字和小于 100
"1, 2, 3, 4".findAll('[0-9]*').map(x, int(x)).sum() < 100
```

### 2.3 URL 库

`isURL(string)` 检查字符串是否是合法的 URL，通过 Go 的 `net/url` 包。

`url(string)` 将字符串转换为 URL，然后可以调用一系列相关的函数。

```python
url('https://example.com:80/').getHost()
url('https://example.com/path with spaces/').getEscapedPath()
```

## 3 类型检查

CEL 是一种动态类型的语言，也就是能够处理动态的类型，而不需要知道输入的类型结构是怎么样的。

通过 `has()` 函数可以判断某个字段是否存在（可访问的）。

```python
# 使用 namex 或者 name
has(object.namex) ? object.namex == 'special' : request.name == 'special'
```

## 参考

* 官方文档：[**Common Expression Language in Kubernetes**](https://kubernetes.io/docs/reference/using-api/cel/)
