## Задание 1
```
y1[n] = x[n - 1] + x[n] + x[n + 1]
```
Значения в `y1` считаются независимо друг от друга, а так же `x` не меняется в процессе вычисления, что очень упростит задачу.

```
y2[n] = y2[n - 2] + y2[n - 1] + x[n]
```
Здесь всё ещё `x` не меняется в процессе вычисления, но теперь очередное значение в `y2` может зависеть от двух предыдущих, что заставляет нас вычислять их последовательно.

Таким образом, первый сигнал лучше подходит под парадигму массового параллелизма на GPU.

## Задание 2

Поймем для начала, что такое:
```
(get_local_id(1) + get_local_size(1) * get_local_id(0)) % 32;
```
Исходя из описания, как рабочая группа делится на warp/wavefront-ы, а также её размера (32,32,1), мы понимаем, что будет исполняться одна "строчка" вдоль оси X(тк номер по этой оси должен меняться чаще всего) целиком на одном warp/wavefront-е (так как размер и того и другого - 32). Отсюда можно сделать вывод, что `get_local_id(1)=const` (для каждого WorkItem в warp/wavefront-е в рамках одного запуска, в рамках всего исполнения рабочей группы это значение, конечно, бегает от 0 до 31). Также теперь понятно, что `get_local_size(1)=32`, а значит `idx` по модулю 32 равен `get_local_id(1)`. 

В итоге имеем `(get_local_id(1) + get_local_size(1) * get_local_id(0)) % 32 = const` (имеется ввиду аналогичная случаю выше константность), поэтому code divergence не произойдёт (все WorkItem пойдут в одну ветку if)

## Задание 3
Все утверждения из предыдущего задания верны и здесь. Также стоит заметить, что sizeof(float)=4 байта

(a)
```
data[get_local_id(0) + get_local_size(0) * get_local_id(1)] = 1.0f;
```
Рассмотрим конкретный запуск - `get_local_id(0)` меняется от 0 до 31, `get_local_id(1)=const`, `get_local_size(0)=32`. То есть сдвиг float-указателя data на `get_local_size(0) * get_local_id(1)` по-прежнему не портит выравнивание, так как он кратен 128 (сдвиг равен `4*32*const` в байтах)

Отсюда видно, что нам нужно записать в 32 подряд идущих элемента `data`, то есть поменять 128байт, что совпадает с размером нашей кеш линии. Учтем, что это нужно было для выполнения одного куска рабочей группы, всего их 32 * 32 * 1 / 32 = 32 

Итого обращение к памяти coalesced, и нам потребуется 32 кеш линии записи в одной рабочей группе.

(b)
```
data[get_local_id(1) + get_local_size(1) * get_local_id(0)] = 1.0f;
```
Имеем `const+32*x`, где x бегает от 0 до 31. То есть нужные данные будут находиться на расстоянии 128байт друг от друга, поэтому обращение к памяти не будет coalesced.

Также вне зависимости от `const` на каждую запись нужна будет ровно одна кеш-линия записи, так как сдвиг `4*const` кратен размеру float элементов.

Итого обращение к памяти **не** coalesced, и нам потребуется 32*32=1024 кеш линии записи в одной рабочей группе.

(c)
```
data[1 + get_local_id(0) + get_local_size(0) * get_local_id(1)] = 1.0f;
```

Единственное отличие от случая (a) в том, что теперь нас сдвинули на 4 байта. Очевидно, обращение к памяти по-прежнему coalesced.

Выравнивание, конечно, испортилось, поэтому теперь нам потребуется две кеш-линии (в первой будет запись первых 31, а во второй - последнего 32-го).

Итого обращение к памяти coalesced, и нам потребуется 2*32=64 кеш линии записи в одной рабочей группе.