---
title: "clikhouse: changing arrays to JSON"
date: 2022-01-06T00:00:00+08:00
---

Imagine you have 10 columns with long arrays that you would like to put together into one json field.

```sql
SELECT
    *,
    arrayMap(x -> x + 1, range(length(params1))) as len,
    arrayMap(x ->  arrayFilter(y -> y != '', x),arrayMap(x -> array(arrayElement(params1, x),
                        arrayElement(params2, x),
                        arrayElement(params3, x),
                        arrayElement(params4, x),
                        arrayElement(params5, x),
                        arrayElement(params6, x),
                        arrayElement(params7, x),
                        arrayElement(params8, x),
                        arrayElement(params9, x),
                        arrayElement(params10, x)), len)) as array_array,
        arrayMap(x -> x[-1], array_array) as value,
        arrayMap(x -> arrayStringConcat(x, '_'), arrayMap(x -> arrayPopBack(x), array_array)) as key,
        arrayMap(x -> arrayStringConcat(x, '":"'), arrayMap(x -> array(arrayElement(key, x), arrayElement(value, x)), len)) as tmp,
        replaceAll(replaceOne(replaceOne(toString(tmp), '[', '{'), ']', '}'), '\'', '"') as tmp_json,
        if(tmp_json = '{"":""}', '', tmp_json) as params_json
FROM table
```
