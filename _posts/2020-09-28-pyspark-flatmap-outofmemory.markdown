---
title:  "flatMap в Spark и память"
meta_description: "Подводные камни при работе с flatMap в Spark"
date:   2020-09-28 09:10:00
---

В Apache Spark есть несколько интересных моментов, которые могут создать серьезные проблемы, если о них не знать. Так после работы  flatMap , не происходит перераспределение разделов (repartition). Обычно результатом работы этой функции является RDD с более большим количеством строк, чем исходный. Это означает, что увеличение нагрузки на память для каждого раздела, и как следствие, использование диска и к некоторым проблемам со сборщиком мусора (GC).

Что самое опасное, есть высокая вероятность что процесс Python исчерпает всю доступную ему память и исполнитель свалиться с OutOfMemoryError. И это не единственная ошибка OutOfMemoryError, которая может произойти. Вторая, даже описана в документации:

> Sometimes, you will get an OutOfMemoryError, not because your RDDs don’t fit in memory, but because the working set of one of your tasks, such as one of the reduce tasks in groupByKey, was too large. Spark’s shuffle operations (sortByKey, groupByKey, reduceByKey, join, etc) build a hash table within each task to perform the grouping, which can often be large.  

Исправить ситуацию можно по разному:

* Уменьшить данные в целом: отфильтровать не нужное, сократить кол-во столбцов;
* В ручную увеличить уровень параллелизма, вызовом `repartition()`, так чтобы входные данные для каждой задачи были меньше;
* Увеличьте буфер перемешивания, путем увеличения памяти исполнителей `spark.executor.memory`. Более тонкие настройки нужно рассматривать индивидуально.
