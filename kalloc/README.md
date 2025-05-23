
![2025-05-02](https://github.com/user-attachments/assets/bf4b8b3d-261e-43be-afb5-7f9119c05e9a)

Youtube-запись от `2025-05-02`: https://youtu.be/VJPe9A_gpBQ

# Цена «естественных прав» на память в ядре Linux • Щупаем средствами C

## Потребляем страницами, а не байтами

- Непрерывная адресация памяти побайтово — удобная абстрация прикладного уровня. На системном она разрушается.
- Страницы — это требование процессора. То есть аппаратуры. Именно процессор  (точнее, Memory Management Unit внутри него) может работать только страницами.
- Страница — минимальный кусок для системы управления памятью.
- `getconf PAGESIZE` покажет реальный размер страницы в системе.
- Выделить, заблокировать, освободить — это всё относится к страницам. А точные байты уже внутри страниц.
- В микроконтроллерах (и их операционках), а также в очень старых системах страниц нет.


В общем, надо уметь смотреть на память через страницы.


### Как расточительно расходуется память


Добавим один байт



Добавим «сколько-то» байтов



Добавим мегабайт


- `sbrk(0)` из `<unistd.h>` покажет адрес текущей границы кучи («добавит ничего») #legacy
- Адрес каждый раз разный!
- От мегабайта включается `mmap()`

`strace -e trace=brk,mmap,munmap ./a.out` увидим интересующие нас вызовы

## Думаем о о цене «естественных прав» на память

- Мы перестаём брать память нахаляву
- И начинаем её «покупать»
- Платим временем и разными качествами

### Непрерывность? Тогда мало, зато быстро

- `kmalloc()` запрашивает у ядра физически непрерывный кусок виртуальной памяти. А его может не быть.
- Поэтому ядро может в момент запроса «навести порядок», прибраться, избавиться от лишнего. Это занимает время.
- В сообщении «можешь потратить на это время» и есть суть `GFP_KERNEL`.
- А вот если указать `GFP_ATOMIC`, то ядро отдаст только ту память, что уже есть, и никакой чистки не будет (что ОК для особых режимов, где нельзя спать).


Добиться **не**выделения памяти через `kmalloc()`


Проверить, есть ли отличия между режимами `GFP_KERNEL` и `GFP_ATOMIC`


Есть стандартная ошибка, стандартная проверка. Надо использовать.

```c
if (!buf) {
    pr_err("[k|v]malloc failed\n");
    return -ENOMEM;
}
```

- Освобождать — `kfree(buf)`
- `sudo cat /proc/slabinfo` покажет всю память, выделенную через `kalloc()`
- Адреса напрямую не увидеть. Но можно логировать самостоятельно, конечно.
- Вообще-то и обычный `malloc()` по умолчанию ограничен обычно 128 КБ. Просто если у него просят больше, он подключает механику `mmap()`, и на прикладном уровне это не видно.
- Ещё мы на прикладном уровне не замечаем для `malloc()` вызов `brk()` — а он как раз и увеличивает незаметно для нас размер кучи, если это потребуется. В ядре ничего такого не дождёшься.
- И с `mmap()` то же самое, если запрошено ну очень уж много.


Всё это очень примерно. Реальная схема затейливей. Но для начала норм.


- Можно провести эксперимент: какого размера `kmalloc()` сработает, а какого уже скажет «кря». Поскольку запросы по выделению страниц идут степенями двойки, то и размеры можно каждый раз удваивать. С `vmalloc()` то же самое. И с постраничным выделением. Результат, конечно, зависит от машины.
- `kmalloc()` самый быстрый. Ну да, он же к диску не ходит. Отличие в разы, иногда на порядки.
- `krealloc()` тоже есть. Очевидно, это инструмент для сильных духом. Ну и переназначать, как всегда, можно только после проверки.


Проверить, поменялись ли адреса после `krealloc()`


### Объём? Тогда медленно и дискретно

- Нужно больше 128 КБ и при этом не для устройства? Надо брать `vmalloc()`
- Он медленней. Не подходит для железок (ну, очевидно). Из особых режимов — лучше не рисковать.


Попытаться добиться **не**выделения памяти через `vmalloc()`


Docker


- Освобождать — `vfree(buf)`
- `sudo cat /proc/vmallocinfo` покажет всю память, выделенную через `valloc()`
- В системе таблиц `vmalloc()` нужно иногда разбираться — но это не первоочередная задача. Самое интересное — наверное, ситуация, когда память надо напрямую передать в user-space. Это потом.

### И объём, и непрерывность, и сэкономить? Тогда оптом

- Нужно много и подряд? Ещё и с выравниванием? И не по страницу на байт? Надо бронировать память страницами! Тут-то они нам и потребуются.
- Соответствующие функции работают со структурами `struct page`. К каждой странице памяти ядро намертво пришивает такую структуру уже при старте. То есть это снова способ абстрагироваться от физики (но не сильно). Иначе говоря, метаданные.


Посмотреть на `struct page`


- Выделение страниц — такое же получение указателя. Точно так же и работать. Структуры хранить, массивы, такое. Физические адреса узнавать. Устройствам передавать. Плюс можно кусок из нескольких страниц разбить указателями постранично.


Давайте хотя бы строчку запишем


### А как же стек? Да сколько там того стека…

- Стек ядра — обычно 8 КБ на один процесс в ядре (на ARM64 — 16 КБ). Совсем кроха. Не наиспользуешься. А мы-то привыкли к 8 МБ на один пользовательский процесс!
- Привычки здесь очень подводят. Даже не осознаём.


Выход за пределы стека — смерть ядра

