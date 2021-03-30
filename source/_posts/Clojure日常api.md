---
title: Clojure日常api
date: 2021-03-30 21:49:30
tags:
---

> 替换Vec第一个元素的值
```clojure
(assoc one_avg 0 "汇总")
```

> Reverse函数
```clojure
(reverse [1 2])
```

> conj cons into
```clojure
user=> (conj [1 2] 3)
[1 2 3]

user=> (cons 3 [1 2])
(3 1 2)

user=> (into [1 2] [3])
[1 2 3]
```
