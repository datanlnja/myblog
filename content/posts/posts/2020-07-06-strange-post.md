---
title: "clickhouse: ltv using sql and clickhouse"
date: 2020-01-06T00:00:00+08:00
---

ClickHouse doesn't have window functions, but there is a workaround, arrays, which allow you to process chunks of data lying in the same structure, just like you do in Python with list. In this article I want to tell you about a couple of useful applications of the range() function, which can generate a sequence of integers in an array. And the arrayJoin function knows how to expand data from an array into a column, that's the story around which this article will be `arrayJoin(range())`


```sql
select range(10)
```

```sql
[0,1,2,3,4,5,6,7,8,9]
```

Now let's unwrap the array

```sql
select arrayJoin(range(10)) as arr
```
```sql
+---+
| 0 |
| 2 |
| 3 |
| 4 |
| 5 |
| 6 |
| 7 |
| 8 |
| 9 |
+---+
```

**Generating new values for a broken funnel**

So let's define the problem, you have a game (or other service) for which you want to build a funnel, the most common, one problem, your data has a lot of gaps and building a funnel by simple select-it will not work. I propose the following solution.

Data generation script in ClickHouse

```sql
/*This is just a small query to generate the data we need,
which is where we use arrayJoin*/

SELECT tupleElement(array, 1) AS step, /* Step */
 tupleElement(array, 2) AS UID /* User identificator */
FROM
  (SELECT arrayJoin([('step 1', 1),('step 3', 1), ('step 2', 2), ('step 3', 2)]) AS array)
```

```sql
+--------+-----+
|  step  | uid |
+--------+-----+
| step 1 |   1 |
| step 3 |   1 |
| step 2 |   2 |
| step 1 |   2 |
+--------+-----+
```

One obvious observation, we always have the last user event, and since we know that the funnel is not growing, we can focus on this event as the final one, and all we have to do is to generate the missing steps of the funnel, that's where range + arrayJoin will help us.

*Step 1: Create the necessary sequence of steps we want to see*


```sql
SELECT name,
       step
FROM
  (SELECT arrayJoin(array(('step 1', 1), ('step 2', 2), ('step 3', 3))) AS tuple,
          tupleElement(tuple, 1) AS name,
          tupleElement(tuple, 2) AS step)
```

```sql
+--------+--------------+
| events | events_index |
+--------+--------------+
| step 1 |            1 |
| step 2 |            2 |
| step 3 |            3 |
+--------+--------------+
```

*Step 2: Combine our event data with the table above, where the steps are labeled by number*

```sql
SELECT *
FROM
  (SELECT tupleElement(array, 1) AS step,
          tupleElement(array, 2) AS UID
   FROM
     (SELECT arrayJoin([('step 1', 1),('step 3', 1), ('step 2', 2), ('step 3', 2)]) AS array)) ANY
LEFT JOIN
  (SELECT step_id,
          step
   FROM
     (SELECT arrayJoin(array(('step 1', 1), ('step 2', 2), ('step 3', 3))) AS tuple,
             tupleElement(tuple, 1) AS step,
             tupleElement(tuple, 2) AS step_id)) USING step
```

```sql
+--------+-----+---------+
|  step  | uid | step_id |
+--------+-----+---------+
| step 1 |   1 |       1 |
| step 3 |   1 |       3 |
| step 2 |   2 |       2 |
| step 3 |   2 |       3 |
+--------+-----+---------+
```

Okay, now each user's eve is labeled with a number in the funnel.

*Step 3: A little magic*

```sql
SELECT UID,
       transform(1 + arrayJoin(range(toUInt8(max(step_id)))), [1, 2, 3], ['step 1',
         'step 2',
         'step 3'], 'none') AS q
FROM
  (SELECT tupleElement(array, 1) AS step,
          tupleElement(array, 2) AS UID
   FROM
     (SELECT arrayJoin([('step 1', 1),('step 3', 1), ('step 2', 2), ('step 3', 2)]) AS array)) ANY
LEFT JOIN
  (SELECT step_id,
          step
   FROM
     (SELECT arrayJoin(array(('step 1', 1), ('step 2', 2), ('step 3', 3))) AS tuple,
             tupleElement(tuple, 1) AS step,
             tupleElement(tuple, 2) AS step_id)) USING step
GROUP BY UID
```

```sql
+-----+--------+
| uid |   q    |
+-----+--------+
|   1 | step 1 |
|   1 | step 2 |
|   1 | step 3 |
|   2 | step 1 |
|   2 | step 2 |
|   2 | step 3 |
+-----+--------+
```

*max(step_id) -> toUInt8() -> range() -> arrayJoin() -> 1+ -> transform()*

First we took the maximum achieved user step, converted the value type (range requirement), then formed an array, expanded it, added 1 to start with 1 (although this is optional) and applied transform (it's kind of like a case for arrays)

So we got what we wanted from the table where part of the data was missing, we filled it with the necessary values.

P.S If suddenly you ask why it is necessary? Reasonable question, you could just group the data by events and calculate the cumulative from bottom to top, and get the same result, the problem is that the business does not stand still, and perhaps tomorrow you need to add a new feature for the user (for example, his gender), and then you will not have to add a small section of the query (Join to users), and rewrite it completely. Everything described above is typical only for ClickHouse. In other databases it is possible to realize this task a little easier.

**Prediction in SQL via linear regression**

I wrote this query when I was doing forecasting for a free-to-play game, the task was simple, to realize a forecast that could be supported by BI analysts. The essence of the method is quite simple, in video games I often use such a method of forecasting with the accumulation of ARPU/ARPPU, it is approximated by some polynomial (or logarithm) and output what is called LTV.


*Dataset generation*

```sql
SELECT tupleElement(array, 1) AS cohort,
       toUInt16(tupleElement(array, 2)) AS DAY,
       tupleElement(array, 3) AS value
FROM
  (SELECT arrayJoin([('cohort 1', 1, 10), ('cohort 1', 2, 100), ('cohort 1', 3, 200), ('cohort 1', 4, 300), ('cohort 2', 2, 10), ('cohort 2', 3, 100), ('cohort 2', 5, 200), ('cohort 2', 8, 300)]) AS array)
```

```sql
+----------+-----+-------+
|  cohort  | day | value |
+----------+-----+-------+
| cohort 1 |   1 |    10 |
| cohort 1 |   2 |   100 |
| cohort 1 |   3 |   200 |
| cohort 1 |   4 |   300 |
| cohort 2 |   2 |    10 |
| cohort 2 |   3 |   100 |
| cohort 2 |   5 |   200 |
| cohort 2 |   6 |   300 |
+----------+-----+-------+
```

An important feature, I do not take completely raw data that you are likely to work with, in my case each cohort has an already read index of the first entry. In our dataset we have, a cohort (known in advance), the day of life of the cohort, and its earnings in money. All we have to do is fill in the blanks, in the algorithm they cannot be left blank, and make a prediction.

*Step 1: Generate all days of the cohort's lifetime*

```sql
SELECT *
FROM
  (SELECT cohort,
          day1 AS DAY
   FROM
     (SELECT cohort,
             max(DAY) AS last_login,
             arrayJoin(range(last_login)) + 1 AS day1
      FROM
        (SELECT tupleElement(array, 1) AS cohort,
                tupleElement(array, 2) AS DAY,
                tupleElement(array, 3) AS value
         FROM
           (SELECT arrayJoin([('cohort 1', 1, 10), ('cohort 1', 2, 100), ('cohort 1', 3, 200), ('cohort 1', 4, 300), ('cohort 2', 2, 10), ('cohort 2', 3, 100), ('cohort 2', 5, 200), ('cohort 2', 6, 300)]) AS array))
      GROUP BY cohort)) ANY
LEFT JOIN
  (SELECT tupleElement(array, 1) AS cohort,
          toUInt16(tupleElement(array, 2)) AS DAY,
          tupleElement(array, 3) AS value
   FROM
     (SELECT arrayJoin([('cohort 1', 1, 10),('cohort 1', 2, 100), ('cohort 1', 3, 200), ('cohort 1', 4, 300),
                       ('cohort 2', 2, 10),('cohort 2', 3, 100), ('cohort 2', 5, 200), ('cohort 2', 6, 300)]) AS array)) USING (cohort,
                                                                                                                                DAY)
```

```sql
+----------+-----+-------+
|  cohort  | day | value |
+----------+-----+-------+
| cohort 2 |   1 |     0 |
| cohort 2 |   2 |    10 |
| cohort 2 |   3 |   100 |
| cohort 2 |   4 |     0 |
| cohort 2 |   5 |   200 |
| cohort 2 |   6 |   300 |
| cohort 1 |   1 |    10 |
| cohort 1 |   2 |   100 |
| cohort 1 |   3 |   200 |
| cohort 1 |   4 |   300 |
+----------+-----+-------+
```

*Step 2: Well, now let's roll everything up into arrays again...what?*

```sql
SELECT cohort,
       groupArray(DAY) AS X,
       arrayCumSum(groupArray(value)) AS y
FROM
  (SELECT *
   FROM
     (SELECT cohort,
             day1 AS DAY
      FROM
        (SELECT cohort,
                max(DAY) AS last_login,
                arrayJoin(range(last_login)) + 1 AS day1
         FROM
           (SELECT tupleElement(array, 1) AS cohort,
                   tupleElement(array, 2) AS DAY,
                   tupleElement(array, 3) AS value
            FROM
              (SELECT arrayJoin([('cohort 1', 1, 10), ('cohort 1', 2, 100), ('cohort 1', 3, 200), ('cohort 1', 4, 300), ('cohort 2', 2, 10), ('cohort 2', 3, 100), ('cohort 2', 5, 200), ('cohort 2', 8, 300)]) AS array))
         GROUP BY cohort)) ANY
   LEFT JOIN
     (SELECT tupleElement(array, 1) AS cohort,
             toUInt16(tupleElement(array, 2)) AS DAY,
             tupleElement(array, 3) AS value
      FROM
        (SELECT arrayJoin([('cohort 1', 1, 10), ('cohort 1', 2, 100), ('cohort 1', 3, 200), ('cohort 1', 4, 300), ('cohort 2', 2, 10), ('cohort 2', 3, 100), ('cohort 2', 5, 200), ('cohort 2', 8, 300)]) AS array)) USING (cohort,
                                                                                                                                                                                                                            DAY))
GROUP BY cohort
```

```sql
+----------+-------------------+--------------------------------+
|  cohort  |         X         |               y                |
+----------+-------------------+--------------------------------+
| cohort 2 | [1,2,3,4,5,6,7,8] | [0,10,110,110,310,310,310,610] |
| cohort 1 | [1,2,3,4]         | [10,110,310,610]               |
+----------+-------------------+--------------------------------+
```

And so, it's very simple here, we just pack the days and values into 2 arrays, label them X, y.

*Step 3: Forecasting*

```sql
SELECT cohort,
       groupArray(DAY) AS X,
       arrayCumSum(groupArray(value)) AS y,
       arrayMap(x -> ln(x + 2), X) AS X_ln, /* Преобразуем независимую переменную, это позволит сгладить прогноз */
 arrayReduce('simpleLinearRegression', X_ln, y) AS coef, /* Собственно расчет коэффициентов (используется аналитическое решение)*/
 /* Все последующие преобразования нужны для прогноза выручки на N день */
 length(X_ln) AS count_days,
 arrayMap(x -> ln(x + 2), range(length(X_ln) + 30)) AS days_predict,
 tupleElement(coef, 1) AS coef_a,
 tupleElement(coef, 2) AS coef_b,
 arrayMap(x -> x*coef_a + coef_b, days_predict) AS array_predict
FROM
  (SELECT *
   FROM
     (SELECT cohort,
             day1 AS DAY
      FROM
        (SELECT cohort,
                max(DAY) AS last_login,
                arrayJoin(range(last_login)) + 1 AS day1
         FROM
           (SELECT tupleElement(array, 1) AS cohort,
                   tupleElement(array, 2) AS DAY,
                   tupleElement(array, 3) AS value
            FROM
              (SELECT arrayJoin([('cohort 1', 1, 10), ('cohort 1', 2, 100), ('cohort 1', 3, 200), ('cohort 1', 4, 300), ('cohort 2', 2, 10), ('cohort 2', 3, 100), ('cohort 2', 5, 200), ('cohort 2', 8, 300)]) AS array))
         GROUP BY cohort)) ANY
   LEFT JOIN
     (SELECT tupleElement(array, 1) AS cohort,
             toUInt16(tupleElement(array, 2)) AS DAY,
             tupleElement(array, 3) AS value
      FROM
        (SELECT arrayJoin([('cohort 1', 1, 10), ('cohort 1', 2, 100), ('cohort 1', 3, 200), ('cohort 1', 4, 300), ('cohort 2', 2, 10), ('cohort 2', 3, 100), ('cohort 2', 5, 200), ('cohort 2', 8, 300)]) AS array)) USING (cohort,
                                                                                                                                                                                                                            DAY))
GROUP BY cohort
```

Well, we have the forecast, but it lies in an array and we need to unpack it. That's what we'll do.

*Step 4: Prepare data for BI*

```sql
SELECT cohort,
       arrayJoin(range(count_days + 30)) AS index_days,
       arrayElement(array_predict, index_days + 1) AS predict,
       arrayElement(y, index_days + 1) AS revenue
FROM
  (SELECT cohort,
          groupArray(DAY) AS X,
          arrayCumSum(groupArray(value)) AS y,
          arrayMap(x -> ln(x + 2), X) AS X_ln,
          arrayReduce('simpleLinearRegression', X_ln, y) AS coef,
          length(X_ln) AS count_days,
          arrayMap(x -> ln(x + 2), range(length(X_ln) + 30)) AS days_predict,
          tupleElement(coef, 1) AS coef_a,
          tupleElement(coef, 2) AS coef_b,
          arrayMap(x -> x * coef_a + coef_b, days_predict) AS array_predict
   FROM
     (SELECT *
      FROM
        (SELECT cohort,
                day1 AS DAY
         FROM
           (SELECT cohort,
                   max(DAY) AS last_login,
                   arrayJoin(range(last_login)) + 1 AS day1
            FROM
              (SELECT tupleElement(array, 1) AS cohort,
                      tupleElement(array, 2) AS DAY,
                      tupleElement(array, 3) AS value
               FROM
                 (SELECT arrayJoin([('cohort 1', 1, 10), ('cohort 1', 2, 100), ('cohort 1', 3, 200), ('cohort 1', 4, 300), ('cohort 2', 2, 10), ('cohort 2', 3, 100), ('cohort 2', 5, 200), ('cohort 2', 8, 300)]) AS array))
            GROUP BY cohort)) ANY
      LEFT JOIN
        (SELECT tupleElement(array, 1) AS cohort,
                toUInt16(tupleElement(array, 2)) AS DAY,
                tupleElement(array, 3) AS value
         FROM
           (SELECT arrayJoin([('cohort 1', 1, 10), ('cohort 1', 2, 100), ('cohort 1', 3, 200), ('cohort 1', 4, 300), ('cohort 2', 2, 10), ('cohort 2', 3, 100), ('cohort 2', 5, 200), ('cohort 2', 8, 300)]) AS array)) USING (cohort,
                                                                                                                                                                                                                               DAY))
   GROUP BY cohort)
```


This method does not claim to be super accurate, unique and that's about it. It is one way to solve a small problem, just to buy time and make a quality prediction model.
