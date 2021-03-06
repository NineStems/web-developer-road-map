# SQL Basic
## SQL JOIN  
### Типы соединений  
**Пример соединения**  
```
select 
* 
from a
join b on a.f1 = b.f1

```
- a – outer-таблица, по которой мы производим чтение
- b – inner-таблица, в которой мы ищем соответствующие записи
- on a.f1 = b.f1 - условие соединения таблицы  
**Пример соединения**  
- Inner join: соединение таблиц происходит по тем строкам, для которых полностью выполняется условие соединения
-  Outer joins:
   -  left outer join – тип соединения, при котором выводятся все строки из outer-таблицы и соответствующие условию соединения строки из inner-таблицы
   -  right outer join – тип соединения, при котором выводятся все строки из inner-таблицы и соответствующие условию соединения строки из outer-таблицы
   -  full outer join - тип соединения, который представляет собой объединение left и right outer joins

- cross join – тип соединения, в котором результатом выборки будет являться декартовое произведение строк
аналог cross join:
```
select *
from tableA, tableB;

select *
from tableA
cross join tableB
```

### Методы соединений
**Nested loop**  
Для каждой строки outer-таблицы по условию соединения происходит последовательный поиск строк из inner-таблицы. Применяется в том случае, когда в условии соединения участвуют проиндексированные столбцы. В таком случае, основной таблицей будет считаться та, у которой столбец, по которому идет соединение, не проиндексирован. В случае, если столбцы в обеих таблицах имеют индексы, основной будет считаться та, у которой размер индекса больше.
- Плюсы: быстрая выборка в том случае, когда соединение идет по индексированным столбцам
- Минусы: многократное чтение неосновной таблицы  

**Hash join**  
Применяется в том случае, когда соединяется большой набор данных. В данном случае на основании наименьшей таблицы во внутренней памяти строится хэш-таблица. Хэш-таблица представляет собой ассоциативный массив, в котором индексом служат значения ключа соединения (набор значений столбцов второй таблицы, по которым идет соединение), а значением элемента массива является результат хэш-функции. После построения хэш-таблицы производится сканирование основной таблицы, в рамках которого ищется соответствующее значение в хэш-таблице.
- Плюсы: быстрая выборка по сравнению с NL в том случае:
    - когда размер таблиц большой (или одна большая и одна маленькая)
    - когда соединение идет не по индексу
- Минусы:
    - для выполнения данного метода требуется много внутренней памяти. Есть риск того, что она закончится, в таком случае запрос не выполнится.

**Sort merge join**  
Вариация NL, при которой таблицы сначала сортируются по столбцам, входящим в условие соединения.
   - Плюсы: если таблицы заранее отсортированы, то соединение будет выполняться быстрее, чем NL за счет того, что данные из второй таблицы будут перечитываться не с начала, а от последнего выбранного значения.
   - Минусы: если таблицы не отсортированы, будет тратиться дополнительное время на сортировку

**Cartesian join (merge join cartesian)**  
Используется в том случае, когда нет условий соединения хотя бы с одной из таблиц в запросе. Типичные примеры таблиц, получаемых в результате данного вида соединения: таблица умножения, ежедневник с почасовой разбивкой.

### Порядок соединения
Порядок основывается на сравнении следующих параметров: 
- Размер таблиц
- Индексы таблицах
- Возможность использования индексов в текущем запросе
- Количество строк, которые нужно просканировать для каждой таблицы в различных вариантах соединения.
   - В большинстве случаев порядок соединения таблиц определяется количеством строк, которые предполагается выбрать (на основе cardinality) — та таблица, в которой будет меньше строк, будет основной.
   - Исключением является тот случай, когда в условии соединения участвуют проиндексированные столбцы. В таком случае основной таблицей будет считаться та, у которой вернется больше строк. Если оба столбца проиндексированы, то в данном случае основной будет считаться та таблица, размер которой меньше.

Оптимизатор не может менять порядок соединения в том случае, если используются outer-join’ы (left,right,full). В данном случае порядок соединения будет именно такой, какой указан в запросе.

### Access path
[Ссылка на документацию](https://docs.oracle.com/database/121/TGSQL/tgsql_optop.htm#TGSQL95143)  
ROWID – уникальный идентификатор строки, указывающий на ее физическое расположение.

**Heap-Organized Tables**  
- Full Table Scan применяется в том случае, когда:
   - у таблицы нет индексов
   - по условию запроса к индексированному столбцу применяется функция
   - если в запросе применяется функция count(<индексированный столбец>), и в данном столбце содержатся пустые значения
   - в случае составного индекса на таблице, в условии запроса не используются первые столбцы, входящие в индекс (может применяться также index skip scan)
   - запрос является неселективным (когда оптимизатор понимает, что для выполнения запроса необходимо прочитать бОльшую часть таблицы)
   - устаревшая статистика
   - маленький размер таблицы (тот случай, когда всю таблицу прочитать дешевле, чем ее индекс и потом по нему вытягивать значения других столбцов)
   - таблица имеет высокую степень параллелизма
   - применяется хинт /*+ full(<синоним таблицы>)*/  
- Table Access by Rowid применяется в том случае, когда:
   - результат запроса можно получить при использовании индекса. В случае, если все требуемые столбцы входят в индекс, будет использоваться Index Fast Full Scan
- Sample Table Scan  
   - Index-organized tables:  
   - Index unique Scan применяется в том случае, если условие запроса можно выполнить после сканирования уникального индекса  
   - Index range scan применяется в том случае, когда:  
     - один или более столбцов индекса (ведущих) используются в условии запроса
     - по условию запроса индекс может вернуть 0, 1 и более значений
   - index full scan применяется в том случае, когда:
     - в условии запроса есть сортировка на проиндексированные столбцы, не содержащие пустых значений
   - index fast full scan применяется в том случае, когда в запросе участвуют только столбцы, входящие в индекс, и при этом сортировка не важна.
   - index skip scan применяется в том случае, когда:
     - ведущий столбец индекса не участвует в запросе
     - ведущий столбец индекса имеет всего несколько уникальных значений, в то время когда ведомые столбцы индекса содержат множество различных значений.
   - index join scan применяется в том случае, когда:
     - hash-join нескольких индексов может вернуть все столбцы, указанные в запросе, без чтения таблиц
     - стоимость чтения строк таблицы будет больше, чем чтение индексов 

### HINTS
Инструкции, которые указывают оптимизатору, как нужно построить план запроса.

select :
- /*+ */
- /*+ ordered(<синоним таблицы>)*/ - указывает, что порядок соединения таблиц нельзя менять: он должен остаться таким, как в выражении.
- /*+ leading(<синоним таблицы1>, <синоним таблицы2>..)*/ - хинт указывает, в каком порядке нужно соединять таблицы:
```
select /*+ use_hash(o c)*/
*
from whs.tmp_op o
join whs.contractor c on o.id_contr = c.id_contr
```

- /*+ use_nl(<таблицы через ,>)*/ --для указанных таблиц будет использован вид соединения nested loop
- /*+ use_hash(<таблицы через ,>)*/ --для указанных таблиц будет использован вид соединения hash join
- /*+ use_merge(<таблицы через ,>)*/ --для указанных таблиц будет использован вид соединения sort merge join

- /*+ no_Use_nl*/
- /*+ no_Use_merge*/
- /*+ no_Use_hash*/

- /*+ (HASH|NL|MERGE)_(SJ|AJ)*/ - указывает, какой тип соединения необходимо использовать для подзапросов в конструкциях NOT IN (AJ) и EXISTS(SJ)

- /*+ use_concat*/ - указывает, что если в запросе используется условный оператор OR, то его нужно разбить на несколько запросов, объединенных UNION ALL.
- /*+ merge*/ - указывает, что нужно соединить результат запроса и подзапроса
- /*+ no_merge*/ - указывает что нужно сначала получить результат из основного запроса, а потом по ним искать результат из подзапроса.

- /*+ rule*/ --указывает, что нужно построить план запроса без учета собранной статистики
- /*+ all_rows*/ --для наиболее быстрого получения всех строк
- /*+ first_rows(N)*/ --для наиболее быстрого получения первых N строк
- /*+ choose*/ - указывает, что нужно построить план запроса, предварительно выбрав наиболее оптимальный способ оптимизации.

- /*+ index(<таблица> <наименование индекса>)*/
- /*+ index_ffs(<таблица> <наименование индекс>)*/ -- index fast full scan
- /*+ full*/ - указывает, что нужно использовать TABLE ACCESS FULL
- /*+ rowid*/ -указывает, что нужно использовать TABLE ACCESS BY ROWID
- /*+ index_asc*/ - в случае range scan’а, сканирование идет в восходящем порядке проиндексированных значений
- /*+ index_combine*/ - указывает, что нужно использовать комбинацию индексов для доступа к таблице
- /*+ index_join*/ -указывает, что необходимо использовать index join scan
- /*+ index_desc*/ - в случае range scan’а, сканирование идет в нисходящем порядке проиндексированных значений
- /*+ no_index*/ - указывает, что доступ к таблице нужно осуществить без использования индекса

- /*+ parallel(<таблица> <количество потоков>)*/
- /*+ no_parallel*/
- /*+ parallel_index(<таблица> <индекс> <количество потоков>)*/
- /*+ noparallel_index*/

- /*+ cardinality(<таблица> <количество строк>)*/
- /*+ append*/ - указывает, что нужно вставлять данные в конец таблицы.
- /*+ unnest*/ - указывает, что нужно выполнять подзапрос совместно с запросом
- /*+ no_unnest*/ - указывает что нужно сначала выполнить основной запрос, а потом для полученного результата выполнить подзапрос
- /*+ cache*/
- /*+ nocache*/

Хинты можно использовать также при обращении к представлениям. В таком случае, необходимо указать полный синоним таблицы:
```
CREATE OR REPLACE VIEW v1 AS
SELECT *
FROM employeesw
WHERE employee_id < 150;

CREATE OR REPLACE VIEW v2 AS
SELECT v1.employee_id employee_id, departments.department_id department_id
FROM v1, departments
WHERE v1.department_id = departments.department_id;

SELECT /*+ INDEX( v2.v1.employees emp_emp_id_pk ) FULL(v2.departments) */ *
FROM v2
WHERE department_id = 30; 
```  

### Parallel execution TODO  
- DDL: создание индексов/таблиц с указанной степенью параллелизма  
- DML: вставка/удаление данных из таблиц с указанной степенью параллелизма  
- SELECT: выборка из таблиц с ненулевой степенью параллелизма. Ручное указание количества потоков с помощью хинтов /*+ parallel*/, /*+ parallel_index*/  
- alter session enable parallel dml — включить параллельное выполнение команд DML (insert,delete,update,merge)  

### Group/Analytic functions
**count**  
Возвращает количество ненулевых значений столбца (внутри можно использовать distinct, в таком случае функция вернет количество уникальных значений). Всегда возвращает значение.

**min/max/avg/sum**  
Вычисление минимального/максимального/среднего/суммы значений в столбце  

**Аналитические функции** — это не скалярные функции (за исключением аналитических статистических, скалярных по результату), которые в отличие от стандартных агрегатных могут употребляться только во фразах SELECT и ORDER BY, так как применяются к уже отобранному результату.  
Свое название получили по той причине, что позволяют средствами SQL (в Oracle) строить запросы, анализирующие данные в БД. Являются вариацией "оконных функций", вошедших в SQL:2003.
Функции этой категории иногда называют "функциями OLAP" ввиду того, что они хорошо подходят для систем типа OLAP (On-Line Analytical Processing), аналитических систем и "аналитических баз данных".

В Oracle они могут быть следующих видов:
- функции ранжирования;
- статистические функции для плавающего интервала;
- функции подсчета долей;
- статистические функции LAG/LEAD с запаздывающим/опережающим аргументом;
- статистические функции (линейная регрессия и т. д.).  
Техническая цель введения аналитических функций — дать лаконичную формулировку и увеличить скорость выполнения "аналитических запросов" к БД, то есть запросов, имеющих смыслом выявление внутренних соотношений и зависимостей в данных. Более точно, пользование аналитическими функциями может дать следующие выгоды перед обычными SQL-операторами:
- Лаконичную и простую формулировку. Многие аналитические запросы к БД традиционными средствами сложно формулируются, а потому с трудом осмысливаются и плохо отлаживаются.
- Снижение нагрузки на сеть. То, что раньше могло формулироваться только серией запросов, сворачивается в один запрос. По сети только отправляется запрос и получается окончательный результат.
- Перенос вычислений на сервер. С использованием аналитических функций нет нужды организовывать расчеты на клиенте; они полностью проводятся на сервере, ресурсы которого могут быть более подходящи для быстрой обработки больших объемов данных.
- Лучшую эффективность обработки запросов. Аналитические функции имеют алгоритмы вычисления, неразрывно связанные со специальными планами обработки запросов, оптимизированными для большей скорости получения результата.

**Особенности обработки**  
Разбиение данных на группы для вычислений
Аналитические функции агрегируют данные порциями (partitions; группами), количество и размер которых можно регулировать специальной синтаксической конструкцией. Ниже она указана на примере агрегирующей функции SUM:   
```SUM(выражение 1) OVER([PARTITION BY выражение 2 [, выражение 3 [, …]]])```  
Если PARTITION BY не указано, то в качестве единственной группы для вычислений будет взят полный набор строк:

**Упорядочение в границах отдельной группы**  
С помощью синтаксической конструкции ORDER BY строки в группах вычислений можно упорядочивать. Синтаксис иллюстрируется на примере агрегирующей функции SUM:
```
SUM(выражение 1) OVER([PARTITION …]
ORDER BY выражение 2 [,…] [{ASC|DESC}] [{NULLS FIRST|NULLS LAST}])
```

**Выполнение вычислений для строк в группе по плавающему окну (интервалу)**  
Для некоторых аналитических функций, например, агрегирующих, можно дополнительно указать объем строк, участвующих в вычислении, выполняемом для каждой строки в группе. Этот объем, своего рода контекст строки, называется "окном", а границы окна могут задаваться различными способами.
```{ROWS | RANGE} {{UNBOUNDED | выражение} PRECEDING | CURRENT ROW }
{ROWS | RANGE}
BETWEEN
{{UNBOUNDED PRECEDING | CURRENT ROW |
{UNBOUNDED | выражение 1}{PRECEDING | FOLLOWING}}
AND
{{UNBOUNDED FOLLOWING | CURRENT ROW |
{UNBOUNDED | выражение 2}{PRECEDING | FOLLOWING}}
```


Фразы PRECEDING и FOLLOWING задают верхнюю и нижнюю границы агрегирования (то есть интервал строк, "окно" для агрегирования).  
Формирование интервалов агрегирования "по строкам" и "по значениям" 
- ROWS и RANGE определяют, как говорится в документации, "физические" и "логические" интервалы-окна
- Функции FIRST_VALUE и LAST_VALUE для интервалов агрегирования
Эти функции позволяют для каждой строки выдать первое значение ее окна и последнее  
**Интервалы времени**  
Для интервалов (окон), упорядоченных внутри по значению ("логическом", RANGE) в случае, если это значение имеет тип "дата", границы интервала можно указывать выражением над датой, а не конкретными значениями из строк. Примеры таких выражений:
- INTERVAL число {YEAR | MONTH | DAY | HOUR | MINUTE | SECOND}
- NUMTODSINTERVAL(число, '{DAY | HOUR | MINUTE | SECOND}')
- NUMTOYMINTERVAL(число, '{YEAR | MONTH}'  

**Функции ранжирования**  
Функции ранжирования позволяют "раздать" строкам "места" в зависимости от имеющихся в них значениях. Некоторые примеры:
```SELECT ename, sal,
ROW_NUMBER () OVER (ORDER BY sal DESC) AS salbacknumber,
ROW_NUMBER () OVER (ORDER BY sal) AS salnumber,
RANK() OVER (ORDER BY sal) AS salrank,
DENSE_RANK() OVER (ORDER BY sal) AS saldenserank
FROM emp;
```
(раздать сотрудникам места в порядке убывания/возрастания зарплат)

**Аналитические функции, применяемые по группам значений**:  
Конструкция `KEEP FIRST/LAST` используется в SQL Oracle для вычисления значения, первой или последней записи в заданной подгруппе, отcортированной по некоторому признаку
она так же позволяет найти результат агрегатной функции по сгруппированным данным, если таких значений несколько
  
**LAG/LEAD ФУНКЦИЯ**  
Аналитическая функция Oracle / PLSQL LAG, которая позволяет запрашивать более одной строки в таблице, в то время, не имея присоединенной к себе таблицы. Она возвращает значения из предыдущей строки в таблице. Для возврата значения из следующего ряда, попробуйте использовать функцию LEAD.  
Синтаксис функции Oracle / PLSQL LAG:  
```LAG ( expression [, offset [, default] ] ) over ( [ query_partition_clause ] order_by_clause )```