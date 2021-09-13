МЧС: прогнозирование степени пожароопасности
=================================

Соревнование алгоритмов прогнозирующих пожароопасность региона на 8 дней вперёд. 

Участникам предлагается спрогнозировать пожар в определённом регионе России используя различные, имеющиеся в открытом доступе, данные.

## Постановка задачи

Разобьём пространство на территории России на ячейки размером 0,2x0,2 градуса и для каждой ячейки поставим метку пересечения с полигоном пожара в соответствующий день. Таким образом, получим набор прямоугольников с меткой 1, если этот участок горел, и 0 в обратном случае.

Требуется разработать алгоритм, выдающий бинарную (0 или 1) вероятность пожара на 1, 2, 3, 4, 5, 6, 7 и 8 дней вперёд для заданной ячейки на определённую дату. Решение должно быть реализовано в виде программы, которая принимает на вход CSV таблицу со следующими столбцами:  
```
id - id строки для оценки предсказаний,
dt - дата,  
lon_min - минимальное значение долготы ячейки,  
lat_min - минимальное значение широты ячейки,  
lon_max - максимальное значение долготы ячейки,  
lat_max - максимальное значение широты ячейки, 
lon - долгота пожара, зафиксированного в ячейке на данную дату (пусто если пожара не было),  
lat - широта пожара, зафиксированного в ячейке на данную дату (пусто если пожара не было),  
grid_index - индекс ячейки,
type_id - тип пожара (пусто если пожара не было),  
type_name - расшифровка типа пожара (пусто если пожара не было).  
```

На выход необходимо сформировать таблицу, где для каждой ячейки (строки - `id` из входного файла) будут указаны 8 чисел (0 или 1) означающие наличие пожара через 1, 2, 3, 4, 5, 6, 7 и 8 дней соответственно.

В качестве дополнительных данных, доступных участникам как на момент обучения моделей, так и в момент инференса в проверяющей системе, могут быть использованы:

-   Фактические данные о пожарах (время и контур);
-   [ERA5](https://cds.climate.copernicus.eu/cdsapp#!/dataset/reanalysis-era5-land)  — данные реанализа "(2019): ERA5-Land hourly data from 1981 to present. Copernicus Climate Change Service (C3S) Climate Data Store (CDS)".

Также участники могут использовать для обучения моделей любые открытые источники данных, например спутниковые снимки и индексы пожароопасности. Однако стоит учитывать, что после загрузки решения в проверяющую систему у моделей не будет доступа в интернет и все необходимые для прогноза дополнительные данные нужно упаковать в docker-контейнер.


## Формат решения

В проверяющую систему необходимо отправить код алгоритма, запакованный в ZIP-архив. Решения запускаются в изолированном окружении при помощи Docker. Время и ресурсы во время тестирования ограничены. Участнику нет необходимости разбираться с технологией Docker.

### Содержимое контейнера

В корне архива обязательно должен быть файл metadata.json следующего содержания:
```json
{
    "image": "cr.msk.sbercloud.ru/aicloud-base-images-test/custom/aij2021/infire:f66e1b5f-1269",
    "entrypoint": "python3 /home/jovyan/solution.py"
}
```

Здесь `image` — поле с названием docker-образа, в котором будет запускаться решение, `entrypoint` — команда, при помощи которой запускается решение. Для решения текущей директорией будет являться корень архива. 

Для запуска решений можно использовать существующее окружение:

- `cr.msk.sbercloud.ru/aicloud-base-images-test/custom/aij2021/infire:f66e1b5f-1269` — [Dockerfile](Dockerfile) с описанием данного image и [requirements](requirements.txt) с библиотеками

Подойдет любой другой образ, доступный для загрузки из `sbercloud`. При необходимости, Вы можете подготовить свой образ, добавить в него необходимое ПО и библиотеки (см. инструкцию по созданию Docker-образов для `sbercloud`); для использования его необходимо будет опубликовать на `sbercloud`.

### Ограничения

Контейнер с решением запускается в следующих условиях:

- решению доступны ресурсы
  - 16 Гб оперативной памяти;
  - 4 vCPU;
  - 1 GPU Tesla V100.
- время на выполнение решения: 30 минут
- решение не имеет доступа к ресурсам интернета
- максимальный размер упакованного и распакованного архива с решением: 10 Гб
- максимальный размер используемого Docker-образа: 15 Гб

## Проверка качества


Качество решения оценивается на отложенной выборке - на данных за 8 дней после завершения приёма решений.  

Целью данного соревнования является прогнозирование начала пожара. Оценка как быстро этот пожар потушат зависит от многих факторов и не является приоритетной для данной задачи. Поэтому для расчёта метрики используются только метки начала пожара. После первой метки начала пожара (т. е. "1" в прогнозе) будем считать, что ячейка "горит" все оставшиеся дни:
|  | 1 | 2 | 3 | 4 | 5 | 6 | 7 | 8 |
|--|--|--|--|--|--|--|--|--|
| pred1 | 0 | 0 | 0 | 0 | 0 | 1 | 1 | 0 |
| pred1_corr |0|0|0|0|0|1|1|1|



В общем виде формула расчёта метрики:
```
total_error = 1 - sum((C**(penalty_i / max(penalty)) - 1) / (C-1)) / N

penalty_i = (-2) * (gt_corr_i - pred_corr_i), если пожар случился раньше, чем прогнозировался
penalty_i = gt_corr_i - pred_corr_i, в ином случае

где N - количество ячеек в прогнозе,
С=20 - нормировочный коэффициент, 
pred_corr_i - первый день прогнозируемого пожара для i-ой ячейки (9 если пожара не прогнозируется)
gt_corr_i - действительный день начала пожара для i-ой ячейки (9 если пожара не было)
```

Для финальной оценки можно выбрать 3 решения. По умолчанию это решения с наилучшей метрикой на public-лидерборде. Меньшее значение метрики соответствует лучшему результату. В случае одинакового значения метрики у двух или более участников приоритет имеет решение, загруженное в систему раньше. Лучшее значение метрики → 1, худшее → 0.

## Призовой фонд

1 место - 500 000 рублей;  
2 место - 250 000 рублей;  
3 место - 150 000 рублей;

Специальный приз от МЧС - 100 000 рублей.
Специальный приз выдаётся на основании заключения экспертной комиссии, оценивающей применимость решения для прогнозирования опасных пожаров в информационных системах МЧС. Комиссия учитывает, но не ограничивается только ими, следующие факторы:
- интерпретируемость модели;
- возможность определения степени важности используемых признаков;
- общее количество признаков и скорость их подготовки;
- скорость работы модели и её размер.

Участники, претендующие на спец. приз должны:
- входить в топ-10 на приватном лидерборде;
- предоставить воспроизводимое решение для обучения моделей, использовавшихся в их лучшей попытке.
