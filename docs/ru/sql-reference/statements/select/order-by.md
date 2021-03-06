# Секция ORDER BY {#select-order-by}

Секция `ORDER BY` содержит список выражений, к каждому из которых также может быть приписано `DESC` или `ASC` (направление сортировки). Если ничего не приписано - это аналогично приписыванию `ASC`. `ASC` - сортировка по возрастанию, `DESC` - сортировка по убыванию. Обозначение направления сортировки действует на одно выражение, а не на весь список. Пример: `ORDER BY Visits DESC, SearchPhrase`

Строки, для которых список выражений, по которым производится сортировка, принимает одинаковые значения, выводятся в произвольном порядке, который может быть также недетерминированным (каждый раз разным).
Если секция ORDER BY отсутствует, то, аналогично, порядок, в котором идут строки, не определён, и может быть недетерминированным.

## Сортировка специальных значений {#sorting-of-special-values}

Существует два подхода к участию `NaN` и `NULL` в порядке сортировки:

-   По умолчанию или с модификатором `NULLS LAST`: сначала остальные значения, затем `NaN`, затем `NULL`.
-   С модификатором `NULLS FIRST`: сначала `NULL`, затем `NaN`, затем другие значения.

### Пример {#example}

Для таблицы

``` text
┌─x─┬────y─┐
│ 1 │ ᴺᵁᴸᴸ │
│ 2 │    2 │
│ 1 │  nan │
│ 2 │    2 │
│ 3 │    4 │
│ 5 │    6 │
│ 6 │  nan │
│ 7 │ ᴺᵁᴸᴸ │
│ 6 │    7 │
│ 8 │    9 │
└───┴──────┘
```

Выполнение запроса `SELECT * FROM t_null_nan ORDER BY y NULLS FIRST` получить:

``` text
┌─x─┬────y─┐
│ 1 │ ᴺᵁᴸᴸ │
│ 7 │ ᴺᵁᴸᴸ │
│ 1 │  nan │
│ 6 │  nan │
│ 2 │    2 │
│ 2 │    2 │
│ 3 │    4 │
│ 5 │    6 │
│ 6 │    7 │
│ 8 │    9 │
└───┴──────┘
```

При сортировке чисел с плавающей запятой NaNs отделяются от других значений. Независимо от порядка сортировки, NaNs приходят в конце. Другими словами, при восходящей сортировке они помещаются так, как будто они больше всех остальных чисел, а при нисходящей сортировке они помещаются так, как будто они меньше остальных.

## Поддержка collation {#collation-support}

Для сортировки по значениям типа String есть возможность указать collation (сравнение). Пример: `ORDER BY SearchPhrase COLLATE 'tr'` - для сортировки по поисковой фразе, по возрастанию, с учётом турецкого алфавита, регистронезависимо, при допущении, что строки в кодировке UTF-8. `COLLATE` может быть указан или не указан для каждого выражения в ORDER BY независимо. Если есть `ASC` или `DESC`, то `COLLATE` указывается после них. При использовании `COLLATE` сортировка всегда регистронезависима.

Рекомендуется использовать `COLLATE` только для окончательной сортировки небольшого количества строк, так как производительность сортировки с указанием `COLLATE` меньше, чем обычной сортировки по байтам.

## Деталь реализации {#implementation-details}

Если кроме `ORDER BY` указан также не слишком большой [LIMIT](limit.md), то расходуется меньше оперативки. Иначе расходуется количество памяти, пропорциональное количеству данных для сортировки. При распределённой обработке запроса, если отсутствует [GROUP BY](group-by.md), сортировка частично делается на удалённых серверах, а на сервере-инициаторе запроса производится слияние результатов. Таким образом, при распределённой сортировке, может сортироваться объём данных, превышающий размер памяти на одном сервере.

Существует возможность выполнять сортировку во внешней памяти (с созданием временных файлов на диске), если оперативной памяти не хватает. Для этого предназначена настройка `max_bytes_before_external_sort`. Если она выставлена в 0 (по умолчанию), то внешняя сортировка выключена. Если она включена, то при достижении объёмом данных для сортировки указанного количества байт, накопленные данные будут отсортированы и сброшены во временный файл. После того, как все данные будут прочитаны, будет произведено слияние всех сортированных файлов и выдача результата. Файлы записываются в директорию `/var/lib/clickhouse/tmp/` (по умолчанию, может быть изменено с помощью параметра `tmp_path`) в конфиге.

На выполнение запроса может расходоваться больше памяти, чем `max_bytes_before_external_sort`. Поэтому, значение этой настройки должно быть существенно меньше, чем `max_memory_usage`. Для примера, если на вашем сервере 128 GB оперативки, и вам нужно выполнить один запрос, то выставите `max_memory_usage` в 100 GB, а `max_bytes_before_external_sort` в 80 GB.

Внешняя сортировка работает существенно менее эффективно, чем сортировка в оперативке.

