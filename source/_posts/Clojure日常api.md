---
title: Clojure日常api
date: 2021-03-30 21:49:30
tags:
---

> 替换Vec第一个元素的值
```clojure
(assoc one_avg 0 "汇总")

(def avg ("/" "/" "/" "/" 0.2088M 1.1259M 0.9171M 0.2088M "/" 0.9171M))
(assoc (vec avg) 0 "汇总")
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

> nth get
```clojure
user=> (get (:rows (:data {:data {:rows [1 2 3]}})) 0)
1
user=> (nth (:rows (:data {:data {:rows [1 2 3]}})) 0)
1

user=> (nth (:rows (:data {:data {:rows []}})) 0)
Execution error (IndexOutOfBoundsException) at user/eval320 (REPL:1).
null
user=> (get (:rows (:data {:data {:rows []}})) 0)
nil
```

> assoc 加线程宏
```clojure
user=> (def a {:name 123})
#'user/a

user=> (let [a (-> (assoc a :name 8) (assoc :name 9))] a)
{:name 9}

user=> (let [a (assoc a :name 8) a (assoc a :name 9)] a)
{:name 9}

user=> a
{:name 123}
```

> some when
```clojure
user=> (some #(when (>= % 3) %) (range))
3
返回集合满足条件的第一项元素
```

>map函数
```clojure
(def f (fn [x] (* 3 x)))
user=> (map f [1 2 3])
(3 6 9)
user=> (map f `(1 2 3))
(3 6 9)
```
>map some when
```clojure
(map (fn [item]
    (some #(when (= (:name item) (:name (second %)))
             (first %)) query-cols))
    all-expected-cols)
```