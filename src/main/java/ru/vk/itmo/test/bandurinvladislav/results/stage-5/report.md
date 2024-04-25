# Stage-5

Задача этого этапа - сделать взаимодействие между нодами асинхронным.

### Пропусная способность

После нагрузки сервера с помощью _wrk_, я получил следующие результаты:

- [get_35k](profile_wrk%2Fget_35k)
- [put_45k](profile_wrk%2Fput_45k)

И у Get, и у Put `RPS` уменьшился на `5000` по сравнению с прошлым этапом. Нагрузку с большим количеством запросов
в секунду сервер не выдерживал. \
Сначала мне показалось это немного странным, поэтому я открыл _visualvm_ и посмотрел, чем заняты потоки во время нагрузки. \
![threads.png](profile_png%2Fthreads.png) \
Видно, что 2/3 потоков, которые приходятся на одно ядро процессора очень много простаивают, пока один берёт работу на
себя. Думаю, что в реальной жизни, когда ноды находятся на разных физических машинах, переход на асинхронное взаимодействие
должен быть быстрее.

## Что же изменилось на профиле?

### Аллокации
Get и Put - Появились новые аллокации, связанные с добавление `CompletableFuture`, суммарно они занимают около 3% 
от всех аллокаций. Помимо этого есть улучшения в методах _successResponse_ и _getClientsByKey_, т.к. я убрал `SteamAPI` в этих методах.

### CPU
У обоих методов достаточно много накладных расходов из-за `CompletableFuture`, внутри которых они обрабатывают и 
отправляют пришедные ответы.
Кроме того, у Get сильно возросла доля, занимаемая методом `invokeLocal` (20% -> 40%) \
![get_cpu.png](profile_png%2Fget_cpu.png) \
Думаю, это связано с тем, что при асинхронном заимодействии, когда мы дублируем выполнение запроса на несколько нод, 
получается ситуация, когда они одновременно вызывают `invokeLocal`, от чего он и начал есть больше ресурсов.

### Lock
Блокировок тоже стало сильно больше:
1) Увеличились количество ожиданий в `ThreadPoolExecutor.getTask`. Кажется, это как раз связано с тем, что ядер на
моём ноутбуке не хватает на все потоки и происходит сильная конкуренция за ресурсы.
2) Добавились новые локи, связанные с работой с `CompletableFuture`. \
![get_lock.png](profile_png%2Fget_lock.png)

## Вывод
Переход на асинхронное взаимодействие между нодами увеличил накладные расходы по всем параметрам и показал меньшую
производительность по сравнению с прошлым этапом. Однако для того, чтобы оценить его реальную работоспособность, нужно
проводить тестирование на нескольких физических машинах.