
# Table of Contents

1.  [Описать реализацию Hash Join в PostgreSQL](#org0bd6a07)
2.  [Описание](#org896592d)
    1.  [Описать алгоритм соединения таблиц, соответствующий приведенному выше плану запроса](#org4081197)
    2.  [Описать существующую реализацию hash-таблицы, используемой при соединении](#orgd1efb4a)
    3.  [Описать, как обрабатываются значения Null](#org425ca59)
    4.  [Описать логику обработки ситуации, когда hash-таблица не помещается в память](#org0f02b41)
        1.  [Основной алгоритм увеличения количества партий (batches):](#orge5d9143)
        2.  [Шаги выполнения](#orgaae5ecf)
            1.  [Проверка возможности увеличения:](#org38f0afa)
            2.  [Расширение массивов файлов:](#orgd0a0aff)
            3.  [Оптимизация количества корзин:](#orgefec82f)
        3.  [Логика разделения кортежей](#org5f5dac7)
        4.  [Неразделяемые группы кортежей: to overflow spaceAllowed.](#orgc2c4d22)
        5.  [Управление памятью:](#org14e46f1)
        6.  [Результат выполнения](#orgcdada89)



<a id="org0bd6a07"></a>

# Описать реализацию Hash Join в PostgreSQL

    postgres=# create table t1 as select gen a, gen % 100 b from generate_series(1, 10000) gen;
    SELECT 10000
    postgres=# create table t2 as select gen a, gen % 100 b from generate_series(1, 10000) gen;
    SELECT 10000
    postgres=# analyze t1;
    ANALYZE
    postgres=# analyze t2;
    ANALYZE
    postgres=# explain (analyze, costs off) select t1.*, t2.* from t1 join t2 using(a);
    QUERY PLAN
    --------------------------------------------------------------------------
    Hash Join (actual time=2.217..5.807 rows=10000 loops=1)
    Hash Cond: (t1.a = t2.a)
    -> Seq Scan on t1 (actual time=0.062..0.798 rows=10000 loops=1)
    -> Hash (actual time=2.102..2.103 rows=10000 loops=1)
    Buckets: 16384 Batches: 1 Memory Usage: 519kB
    -> Seq Scan on t2 (actual time=0.020..0.818 rows=10000 loops=1)
    Planning Time: 1.056 ms
    Execution Time: 6.245 ms
    (8 rows)

1.  Описать алгоритм соединения таблиц, соответствующий приведенному выше плану запроса;
2.  Описать существующую реализацию hash-таблицы, используемой при соединении;
3.  Описать, как обрабатываются значения Null;
4.  Описать логику обработки ситуации, когда hash-таблица не помещается в память

Описание необходимо сопроводить ссылками на код на зеркале репозитория на GitHub
(<https://github.com/postgres/postgres>). За основу взять тег, соответствующий версии 15.2.


<a id="org896592d"></a>

# Описание

<https://github.com/postgres/postgres/blob/REL_15_2/src/backend/executor/execProcnode.c>


<a id="org4081197"></a>

## Описать алгоритм соединения таблиц, соответствующий приведенному выше плану запроса

1.  Инициализация (ExecutorStart): [ExecutorStart(QueryDesc \*queryDesc, int eflags)](https://github.com/postgres/postgres/blob/REL_15_2/src/backend/executor/execMain.c#L131)
    -   Вызывается InitPlan() [InitPlan(QueryDesc \*queryDesc, int eflags)](https://github.com/postgres/postgres/blob/REL_15_2/src/backend/executor/execMain.c#L805),
    -   Вызывается инициализация верхнего узла [planstate = ExecInitNode(plan, estate, eflags);](https://github.com/postgres/postgres/blob/REL_15_2/src/backend/executor/execMain.c#L938)  [ExecInitNode(Plan \*node, EState \*estate, int eflags)](https://github.com/postgres/postgres/blob/REL_15_2/src/backend/executor/execProcnode.c#L142)
    -   создаётся состояние узла HashJoin [case T\_HashJoin:](https://github.com/postgres/postgres/blob/REL_15_2/src/backend/executor/execProcnode.c#L307)
    -   для Hash Join вызывает специализированную функцию ExecInitHashJoin(), которая:
        -   Создает состояние узла соединения (HashJoinState).
        -   Рекурсивно инициализирует левое и правое поддеревья:
            -   Левое поддерево: Seq Scan on t1.
            -   Правое поддерево: Hash (который строит хэш-таблицу на основе Seq Scan on t2).
2.  Выполнение ([ExecutorRun(QueryDesc \*queryDesc,](https://github.com/postgres/postgres/blob/REL_15_2/src/backend/executor/execMain.c#L300)):
    -   [ExecutePlan](https://github.com/postgres/postgres/blob/REL_15_2/src/backend/executor/execMain.c#L363) циклически вызывает  [ExecProcNode](https://github.com/postgres/postgres/blob/REL_15_2/src/backend/executor/execMain.c#L1636)()  на корневом узле Hash Join.
    -   ExecHashJoinImpl() работает как конечный автомат со следующими состояниями:
        -   [HJ\_BUILD\_HASHTABLE](https://github.com/postgres/postgres/blob/REL_15_2/src/backend/executor/nodeHashjoin.c#L211):
            -   Создается хэш-таблица для правой таблицы (t2). [hashtable = ExecHashTableCreate(hashNode,](https://github.com/postgres/postgres/blob/REL_15_2/src/backend/executor/nodeHashjoin.c#L279)
            -   Полное сканирование t2, все строки хэшируются по ключу a и помещаются в хэш-таблицу.
        
        -   [HJ\_NEED\_NEW\_OUTER](https://github.com/postgres/postgres/blob/REL_15_2/src/backend/executor/nodeHashjoin.c#L345):
            -   Получается следующая строка из левой таблицы (t1).
            -   Вычисляется хэш-значение для атрибута a.
            -   Определяется номер корзины в хэш-таблице.
        
        -   [HJ\_SCAN\_BUCKET](https://github.com/postgres/postgres/blob/REL_15_2/src/backend/executor/nodeHashjoin.c#L418):
            [ExecScanHashBucket(node, econtext)](https://github.com/postgres/postgres/blob/REL_15_2/src/backend/executor/nodeHashjoin.c#L434)
            -   Сканируется найденная корзина хэш-таблицы для поиска совпадений.
            -   Проверяется условие соединения t1.a = t2.a.
            -   При совпадении строки объединяются и возвращаются как результат.
        
        -   [HJ\_NEED\_NEW\_BATCH](https://github.com/postgres/postgres/blob/REL_15_2/src/backend/executor/nodeHashjoin.c#L553) / [HJ\_FILL\_OUTER\_TUPLE](https://github.com/postgres/postgres/blob/REL_15_2/src/backend/executor/nodeHashjoin.c#L502) / [HJ\_FILL\_INNER\_TUPLES](https://github.com/postgres/postgres/blob/REL_15_2/src/backend/executor/nodeHashjoin.c#L527):
            -   Обработка особых случаев (multiple batches, outer joins), которые в данном запросе не используются.
3.  Завершение (ExecutorEnd):
    -   ExecEndNode() вызывает [ExecEndHashJoin((HashJoinState \*) node);](https://github.com/postgres/postgres/blob/REL_15_2/src/backend/executor/execProcnode.c#L706), который освобождает ресурсы и рекурсивно завершает работу подпланов (Seq Scan на t1 и t2).


<a id="orgd1efb4a"></a>

## Описать существующую реализацию hash-таблицы, используемой при соединении

[typedef struct HashJoinTableData](https://github.com/postgres/postgres/blob/REL_15_2/src/include/executor/hashjoin.h#L284)
Основная структура данных

    typedef struct HashJoinTableData {
        int         nbuckets;           // количество корзин в памяти
        int         log2_nbuckets;      // log2(nbuckets) для быстрого вычисления
        int         nbuckets_optimal;   // Оптимальное количество корзин на партию

Организация корзин (buckets)

-   buckets: массив указателей на списки кортежей
    -   unshared - для непараллельного выполнения (локальная память)
    -   shared - для параллельного выполнения (разделяемая память DSA)
-   Каждая корзина содержит связанный список кортежей HashJoinTuple

Оптимизация для skewed-данных

    bool        skewEnabled;        // Использование оптимизации для частых значений
    HashSkewBucket **skewBucket;   // Дополнительная хэш-таблица для skewed-корзин
    int         nSkewBuckets;      // Количество активных skewed-корзин

Пакетная обработка (batching)

    int         nbatch;            // Общее количество партий (batches)
    int         curbatch;          // Текущая обрабатываемая партия
    BufFile   **innerBatchFile;    // Временные файлы для внутренних кортежей
    BufFile   **outerBatchFile;    // Временные файлы для внешних кортежей

При нехватке памяти данные разбиваются на партии
Только текущая партия хранится в памяти, остальные - в временных файлах

Управление памятью

    Size        spaceUsed;         // Текущее использование памяти
    Size        spaceAllowed;      // Лимит памяти
    MemoryContext hashCxt;         // Контекст для всей хэш-таблицы
    MemoryContext batchCxt;        // Контекст для текущей партии
    HashMemoryChunk chunks;        // Список блоков памяти для кортежей

Хэш-функции и сравнение

    FmgrInfo   *outer_hashfunctions; // Хэш-функции для внешней таблицы
    FmgrInfo   *inner_hashfunctions; // Хэш-функции для внутренней таблицы
    bool       *hashStrict;         // Флаги строгости хэш-функций
    Oid        *collations;         // Правила сортировки

Параллельное выполнение

    ParallelHashJoinState *parallel_state; // Состояние параллельного соединения
    dsa_area   *area;              // Область разделяемой памяти

Особенности реализации

1.  Skew-оптимизация: Частые значения выделяются в отдельные корзины
2.  Пакетная обработка: Поддержка данных, не помещающихся в память
3.  Плотное размещение: Кортежи размещаются в связанных блоках памяти
4.  Поддержка NULL: Флаг keepNulls управляет обработкой NULL-значений


<a id="org425ca59"></a>

## Описать, как обрабатываются значения Null

Обработка NULL-значений происходит в [ExecHashGetHashValue](https://github.com/postgres/postgres/blob/REL_15_2/src/backend/executor/nodeHash.c#L1814)

1.  Строгие хэш-функции (hashStrict): [if (hashtable->hashStrict[i] && !keep\_nulls)](https://github.com/postgres/postgres/blob/REL_15_2/src/backend/executor/nodeHash.c#L1869)
    -   Если хэш-функция строгая и встречается NULL:
        -   При keep\_nulls = false: кортеж немедленно отбрасывается (return false)
        -   При keep\_nulls = true: NULL обрабатывается с хэш-кодом 0 - но это не работает [Note: currently, all hashjoinable operators must be strict](https://github.com/postgres/postgres/blob/REL_15_2/src/backend/executor/nodeHash.c#L1863)

2.  Параметр keep\_nulls: [hashtable->keepNulls = keepNulls;](https://github.com/postgres/postgres/blob/REL_15_2/src/backend/executor/nodeHash.c#L489)
    -   false: стандартное поведение для INNER JOIN - отбрасывать NULL
    -   true: для OUTER JOIN - сохранять кортежи с NULL


<a id="org0f02b41"></a>

## Описать логику обработки ситуации, когда hash-таблица не помещается в память

Ситуация, когда hash-таблица не помещается в память:
[while (hashtable->spaceUsedSkew > hashtable->spaceAllowedSkew)](https://github.com/postgres/postgres/blob/REL_15_2/src/backend/executor/nodeHash.c#L2463)

    while (hashtable->spaceUsedSkew > hashtable->spaceAllowedSkew)
    		ExecHashRemoveNextSkewBucket(hashtable);
    
    	/* Check we are not over the total spaceAllowed, either */
    	if (hashtable->spaceUsed > hashtable->spaceAllowed)
    		ExecHashIncreaseNumBatches(hashtable);


<a id="orge5d9143"></a>

### Основной алгоритм увеличения количества партий (batches):

[ExecHashIncreaseNumBatches(HashJoinTable hashtable)](https://github.com/postgres/postgres/blob/REL_15_2/src/backend/executor/nodeHash.c#L909)

    ExecHashIncreaseNumBatches(HashJoinTable hashtable)
    {
        int oldnbatch = hashtable->nbatch;
        int nbatch = oldnbatch * 2;  // Удваиваем количество партий


<a id="orgaae5ecf"></a>

### Шаги выполнения


<a id="org38f0afa"></a>

#### Проверка возможности увеличения:

-   Проверка флага growEnabled [do nothing if we&rsquo;ve decided to shut off growth](https://github.com/postgres/postgres/blob/REL_15_2/src/backend/executor/nodeHash.c#L919)
-   Защита от переполнения при удвоении [oldnbatch > Min(INT\_MAX / 2, MaxAllocSize / (sizeof(void \*) \* 2))](https://github.com/postgres/postgres/blob/REL_15_2/src/backend/executor/nodeHash.c#L924)


<a id="orgd0a0aff"></a>

#### Расширение массивов файлов:

[if (hashtable->innerBatchFile == NULL)](https://github.com/postgres/postgres/blob/REL_15_2/src/backend/executor/nodeHash.c#L937)
[else](https://github.com/postgres/postgres/blob/REL_15_2/src/backend/executor/nodeHash.c#L947)

    // Создание или расширение массивов для хранения временных файлов
    hashtable->innerBatchFile = repalloc(...);  // Для внутренней таблицы
    hashtable->outerBatchFile = repalloc(...);  // Для внешней таблицы

[palloc0(Size size)](https://github.com/postgres/postgres/blob/REL_15_2/src/common/fe_memutils.c#L122)

[repalloc(void \*pointer, Size size)](https://github.com/postgres/postgres/blob/REL_15_2/src/common/fe_memutils.c#L173)

[pg\_malloc\_internal(size\_t size, int flags)](https://github.com/postgres/postgres/blob/REL_15_2/src/common/fe_memutils.c#L23)

    static inline void *
    pg_malloc_internal(size_t size, int flags)
    {
    	void	   *tmp;
    
    	/* Avoid unportable behavior of malloc(0) */
    	if (size == 0)
    		size = 1;
    	tmp = malloc(size);
    	if (tmp == NULL)
    	{
    		if ((flags & MCXT_ALLOC_NO_OOM) == 0)
    		{
    			fprintf(stderr, _("out of memory\n"));
    			exit(EXIT_FAILURE);
    		}
    		return NULL;
    	}
    
    	if ((flags & MCXT_ALLOC_ZERO) != 0)
    		MemSet(tmp, 0, size);
    	return tmp;
    }


<a id="orgefec82f"></a>

#### Оптимизация количества корзин:

-   При необходимости увеличивается количество корзин до оптимального значения
-   Перераспределение памяти для массива корзин
-   Перераспределение кортежей:
    [so, let&rsquo;s scan through the old chunks, and all tuples in each chunk](https://github.com/postgres/postgres/blob/REL_15_2/src/backend/executor/nodeHash.c#L994)
    [process all tuples stored in this chunk (and then free it)](https://github.com/postgres/postgres/blob/REL_15_2/src/backend/executor/nodeHash.c#L1002)
    
        while (oldchunks != NULL) {
            // Обработка каждого chunk памяти
            while (idx < oldchunks->used) {
                HashJoinTuple hashTuple = ...;
                ExecHashGetBucketAndBatch(hashtable, hashTuple->hashvalue,
                                          &bucketno, &batchno);


<a id="org5f5dac7"></a>

### Логика разделения кортежей

-   Кортежи остаются в памяти ([batchno == curbatch](https://github.com/postgres/postgres/blob/REL_15_2/src/backend/executor/nodeHash.c#L1015)):

    // Копирование кортежа в новую память
    copyTuple = (HashJoinTuple) dense_alloc(hashtable, hashTupleSize);
    memcpy(copyTuple, hashTuple, hashTupleSize);
    // Добавление в соответствующую корзину

-   Кортежи выгружаются в файл ([dump it out](https://github.com/postgres/postgres/blob/REL_15_2/src/backend/executor/nodeHash.c#L1029)):

    // Сохранение в файл соответствующей партии
    ExecHashJoinSaveTuple(HJTUPLE_MINTUPLE(hashTuple),
                          hashTuple->hashvalue,
                          &hashtable->innerBatchFile[batchno]);
    // Освобождение памяти
    hashtable->spaceUsed -= hashTupleSize;


<a id="orgc2c4d22"></a>

### Неразделяемые группы кортежей: [to overflow spaceAllowed.](https://github.com/postgres/postgres/blob/REL_15_2/src/backend/executor/nodeHash.c#L1059)

    
    	/*
    	 * If we dumped out either all or none of the tuples in the table, disable
    	 * further expansion of nbatch.  This situation implies that we have
    	 * enough tuples of identical hashvalues to overflow spaceAllowed.
    	 * Increasing nbatch will not fix it since there's no way to subdivide the
    	 * group any more finely. We have to just gut it out and hope the server
    	 * has enough RAM.
    	 */
    	if (nfreed == 0 || nfreed == ninmemory)
    	{
    		hashtable->growEnabled = false;
    #ifdef HJDEBUG
    		printf("Hashjoin %p: disabling further increase of nbatch\n",
    			   hashtable);
    #endif
    	}


<a id="org14e46f1"></a>

### Управление памятью:

-   Освобождение старых chunks по мере обработки
-   Использование dense\_alloc для эффективного размещения в новой памяти


<a id="orgcdada89"></a>

### Результат выполнения

-   Количество партий удваивается
-   Память освобождается за счет выгрузки ненужных кортежей
-   Только кортежи текущей партии остаются в памяти
-   Остальные кортежи сохраняются во временных файлах для последующей обработки

