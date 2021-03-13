---
title: Clojure的并发-Agent深入分析和Actor
date: 2021-03-06 18:39:39
tags:
---

> 参数解构

```clojure
(defn- update-dashboard-parameters
  [{:keys [type interval_day default], :as parameter}]
  (cond
    ;;当前日期的单个日期
    (and (= "date/single" type) (not (nil? interval_day)))
    (TimeUtil/diff interval_day)

    ;;当前日期的日期范围
    (and (= "date/range" type) (not (nil? interval_day)))
    (TimeUtil/diff_date_range interval_day)
    :else
    default))

(update-dashboard-parameters parameter)
```

