# TEST 1

## Task

```sql
PACKAGE BODY contract_consent IS

  PROCEDURE show_coordination(p_plnid IN NUMBER,
                              p_stage IN NUMBER,
                              p_obj   IN NUMBER) IS
  
    l_plnname        contract_out.cssouts_work%TYPE;
  
    CURSOR get_crd(c_scntrid NUMBER) IS
      SELECT contrc_id,
             scntr_id,
             scntst_id,
             scntst_name,
             empl_id,
             empl_name,
             contrc_date,
             contrc_reply,
             is_buildstatus,
             LEVEL AS c_level
        FROM (SELECT c.contrc_id,
                     c.contrc_parent_id,
                     s.scntr_id,
                     s.scntst_id,
                     s.scntst_name,
                     em.empl_id,
                     em.empl_lastname || ' ' || em.empl_firstname || ' ' ||
                     em.empl_middlename AS empl_name,
                     c.contrc_date,
                     c.contrc_reply,
                     decode(c.contrc_parent_id,
                            -1,
                            1,
                            (SELECT ss.scntst_issuccess
                               FROM contract_out_consent cc, sconsentstatus ss
                              WHERE cc.contrc_id = c.contrc_parent_id
                                AND cc.contr_stage = p_stage
                                AND cc.scntst_id = ss.scntst_id
                                AND cc.is_type_budget = 0
                                AND ss.status = 1)) AS is_buildstatus
                FROM contract_out_consent c,
                     sconsentstatus       s,
                     ais_res.semployee    em
               WHERE c.contr_id = p_plnid
                 AND c.contr_stage = p_stage
                 AND c.scntst_id = s.scntst_id
                 AND c.empl_id = em.empl_id
                 AND c.is_type_budget = 0
                 AND c.status = 1)
      CONNECT BY PRIOR contrc_id = contrc_parent_id
       START WITH scntr_id = c_scntrid
              AND contrc_parent_id = -1
       ORDER SIBLINGS BY contrc_islocal DESC;
    TYPE crdtype IS TABLE OF get_crd%ROWTYPE INDEX BY BINARY_INTEGER;
    crd_arr crdtype;
  BEGIN
  ...
```
---
## Questions
---
### Question 1
Что означает конструкция `%TYPE` в блоке объявления переменных и в каких случаях её следует использовать?
### Answer
`%TYPE` при определении переменной используется для динамического копирования 
типа данных из, например, другой переменной.

Это позволяет:
1. Писать меньше кода, сделать программу более читаемой, 
т.к. нет необходимости прописывать такие атрибуты, как 
точность, размер или длина.
1. Динамическая типизация даёт бОльшую свободу действий 
и некоторые возможности.

### Example
```sql
DECLARE
    name VARCHAR2(50) NOT NULL;
    -- surname and lastname is VARCHAR(50) NOT NULL;
    surname name%TYPE;
    lastname name%TYPE;

    birsday DATE NOT NULL;
    death birsday%TYPE;
BEGIN
    null;
END;
```
```sql
CREATE TABLE z#CLIENT (
    ID              INTEGER(10)    NOT NULL 
    , BIRSDAY       DATE           NOT NULL
    
    , CONSTRAINT indx PRIMARY KEY (ID)
);
-----
DECLARE
    -- if BIRSDAY will varchar then won't errors
    bday     z#CLIENT.BIRSDAY%TYPE;
BEGIN
    bday := '01-01-2010';
END;
```

---
### Question 2
Что означает конструкция, начинающаяся с `CURSOR`
### Answer
Курсор - тип данных, представляющий собой "Указатель" на некоторый `SELECT` запрос.

Переменные этого типа могут итерироваться по запросу, переходить в его начало, конец, сдвигаться относительно текущей позиции или начала/конца запроса.

Использование курсоров позволяет свободно доставать кортежи из запроса без использования дополнительных запросов => сэкономить огромное количество ресурсов.

### Example
```sql
DECLARE
    TYPE DateTab IS TABLE OF z#CLIENT.birsday%TYPE;

    -- Cursor declaration
    CURSOR cl is 
    SELECT 
        id
        , birsday as C_DATE
    FROM z#CLIENT
    WHERE birsday BETWEEN '01-01-2010' AND '01-01-2011'
    ORDER BY birsday;
    -- End Cursor declaration

    dt z#CLIENT.birsday%TYPE;
    ind z#CLIENT.id%TYPE;

    tempTable DateTab;
    i INTEGER := 1;
BEGIN
    OPEN cl;
    FETCH cl INTO ind, dt;

    -- Insert into temp table clients birsdays:
    -- Over 2 on birsdays in range ('01-01-2010'..'01-05-2010')
    -- Over 3 on birsdays in range ('01-05-2010'..'01-01-2011')
    while not cl%notfound loop
        tempTable(i) := dt;
        i := i + 1;
        if dt > '01-05-2010' then
            FETCH RELATIVE 2 cl INTO ind, dt;
        else
            FETCH RELATIVE 3 cl INTO ind, dt;
        end if;
    end loop;

    CLOSE cl;
END;
```

---
### Question 3
В чём специфика запроса, приведённого в примере?
### Answer
В запросе мы находим все контракты с определенным идентификатором на определенном этапе и информацию по сотруднику, прикрепленному к нему.

У контракта есть *родительский* контракт (если это поле задано как -1, то контракту присвоен флаг `is_buildstatus`).

*Результатом запроса будут все контракты, начиная с владельца флага `is_buildstatus`, у которого стоит статус, определенный параметром.*

Не понятно, что за что отвечает, т.к. много полей, имена которых отличаются на одну букву.

По поводу синтаксиса:

* Используется `SELECT` запрос тройной вложенности.
* Вместо `inner join` использована фильтрация `where`
* `connect by` используется для указания родителя в иерархии.
* `start with` используется для указания корневого(ых) элемента(ов) иерархии. Элементы, не попавшие в дерево будут исключены из выборки.
* `level` используется для получения глубины иерархии для данного элемента.
* `order siblings` используется для сортировки элементов внутри текущей глубины иерархии.

---
### Question 4
Как можно иначе определить условия для связи между таблицами в приведённом запросе?

### Answer
Как уже было сказано выше, для такой связи таблиц может быть использован `inner join`.

### Example
```sql
SELECT *
FROM    contract_out_consent c 
        INNER JOIN  sconsentstatus s
        ON c.scntst_id = s.scntst_id
        INNER JOIN ais_res.semployee em
        ON c.empl_id = em.empl_id
WHERE c.contr_id = p_plnid
AND c.contr_stage = p_stage
AND c.is_type_budget = 0
AND c.status = 1;
```

---
# Test 2

## Task

```sql
PACKAGE utl_api_types AS

  TYPE numType IS TABLE OF NUMBER INDEX BY BINARY_INTEGER;

END;
```

```sql
PACKAGE BODY "FUNC_CONSENT" IS

  FUNCTION find_route_contract(p_contrid   IN NUMBER,
                               p_sum_contr IN NUMBER,
                               p_initca    IN NUMBER) RETURN NUMBER IS
    l_routeid  NUMBER;
    sdv_arr    ais_sys.utl_api_types.numtype;
    l_category NUMBER;
  BEGIN
  
    l_category := f_get_category(p_id => p_contrid, p_type => 1);
  
    CASE l_category
      WHEN 1 THEN
        WITH m AS
         (SELECT sr.route_id,
                 sr.ccalc_id,
                 sr.scasp_ca_id
            FROM contract_out cc,
                 sroute       sr
           WHERE cc.cssouts_id = p_contrid
             AND cc.cgroup_id = sr.cgroup_id
             AND cc.category_id = sr.category_id
             AND (cc.ccalc_id = sr.ccalc_id OR sr.ccalc_id = -1)
             AND cc.frame_id = sr.frame_id
             AND cc.contest = sr.contest
             AND cc.korp_order = sr.korp_order
             AND (sr.scasp_ca_id =
                 (SELECT sp.scasp_ca_id
                     FROM sperson_css sp
                    WHERE cc.orgzn_id_custom = sp.person_id) OR
                 sr.scasp_ca_id = 0
                 )
             AND cc.casp_id IN
                 (SELECT sc.casp_id
                    FROM sroute_casp sc
                   WHERE sr.route_id = sc.route_id)
             AND p_sum_contr BETWEEN sr.sum_without_nds_min AND
                 sr.sum_without_nds_max
             AND sr.status = 1
             AND sr.is_route_actual = 1 -- актуальность
             AND sr.is_belong = decode(p_initca, -1, 0, 1)
             AND sr.is_agree = 0
             AND sr.is_default = 0),
        c1 AS
         (SELECT COUNT(*) AS cnt FROM m WHERE m.ccalc_id > 0),
        c2 AS
         (SELECT COUNT(*) AS cnt FROM m WHERE m.scasp_ca_id != 0),
        c3 AS
         (SELECT COUNT(*) AS cnt
            FROM m
           WHERE m.ccalc_id > 0
             AND m.scasp_ca_id != 0)
        SELECT CASE
                 WHEN c3.cnt > 0 THEN
                  (SELECT m.route_id
                     FROM m
                    WHERE m.ccalc_id > 0
                      AND m.scasp_ca_id != 0
                      AND rownum = 1)
                 WHEN c1.cnt > 0 THEN
                  (SELECT m.route_id
                     FROM m
                    WHERE m.ccalc_id > 0
                      AND rownum = 1)
                 WHEN c2.cnt > 0 THEN
                  (SELECT m.route_id
                     FROM m
                    WHERE m.scasp_ca_id != 0
                      AND rownum = 1)
                 ELSE
                  (SELECT m.route_id FROM m WHERE rownum = 1)
               END BULK COLLECT
          INTO sdv_arr
          FROM c1,
               c2,
               c3;
      
      WHEN 3 THEN
        --- наряд-заказ
        WITH m AS
         (SELECT sr.route_id,
                 sr.scasp_ca_id
            FROM contract_out cc,
                 sroute       sr
           WHERE cc.cssouts_id = p_contrid
             AND cc.cgroup_id = sr.cgroup_id
             AND cc.category_id = sr.category_id
             AND cc.frame_id = sr.frame_id
             AND cc.contest = sr.contest
             AND sr.korp_order =
                 get_korp_order_parent(p_contrid => p_contrid)
             AND cc.casp_id IN
                 (SELECT sc.casp_id
                    FROM sroute_casp sc
                   WHERE sr.route_id = sc.route_id)
             AND (sr.scasp_ca_id = get_scasp_parent(p_contrid => p_contrid) OR
                 sr.scasp_ca_id = 0)
             AND p_sum_contr BETWEEN sr.sum_without_nds_min AND
                 sr.sum_without_nds_max
             AND sr.status = 1
             AND sr.is_route_actual = 1 -- актуальность
             AND sr.is_belong = decode(p_initca, -1, 0, 1)
             AND sr.is_agree = 0
             AND sr.is_default = 0),
        k AS
         (SELECT COUNT(*) AS cnt FROM m WHERE m.scasp_ca_id != 0)
        SELECT CASE
                 WHEN k.cnt > 0 THEN
                  (SELECT m.route_id
                     FROM m
                    WHERE m.scasp_ca_id != 0
                      AND rownum = 1)
                 ELSE
                  (SELECT m.route_id FROM m WHERE rownum = 1)
               END BULK COLLECT
          INTO sdv_arr
          FROM k;
      
      WHEN 4 THEN
        WITH m AS
         (SELECT sr.route_id,
                 sr.ccalc_id
            FROM contract_out cc,
                 sroute       sr
           WHERE cc.cssouts_id = p_contrid
             AND cc.cgroup_id = sr.cgroup_id
             AND cc.category_id = sr.category_id
             AND (cc.ccalc_id = sr.ccalc_id OR sr.ccalc_id = -1)
             AND cc.frame_id = sr.frame_id
             AND cc.contest = sr.contest
             AND sr.korp_order =
                 get_korp_order_parent(p_contrid => p_contrid)
             AND cc.casp_id IN
                 (SELECT sc.casp_id
                    FROM sroute_casp sc
                   WHERE sr.route_id = sc.route_id)
             AND p_sum_contr BETWEEN sr.sum_without_nds_min AND
                 sr.sum_without_nds_max
             AND sr.status = 1
             AND sr.is_route_actual = 1 -- актуальность
             AND sr.is_belong = decode(p_initca, -1, 0, 1)
             AND sr.central = cc.central
             AND sr.is_agree = 0
             AND sr.is_default = 0),
        k AS
         (SELECT m.route_id,
                 MAX(m.ccalc_id) AS ccalc_id
            FROM m
           GROUP BY m.route_id)
        SELECT k.route_id BULK COLLECT
          INTO sdv_arr
          FROM k
         WHERE rownum = 1;
      
      WHEN 40 THEN
        WITH m AS
         (SELECT sr.route_id,
                 sr.ccalc_id
            FROM contract_out cc,
                 sroute       sr
           WHERE cc.cssouts_id = p_contrid
             AND cc.cgroup_id = sr.cgroup_id
             AND cc.category_id = sr.category_id
             AND (cc.ccalc_id = sr.ccalc_id OR sr.ccalc_id = -1)
             AND cc.frame_id = sr.frame_id
             AND cc.contest = sr.contest
             AND sr.korp_order =
                 get_korp_order_parent(p_contrid => p_contrid)
             AND cc.casp_id IN
                 (SELECT sc.casp_id
                    FROM sroute_casp sc
                   WHERE sr.route_id = sc.route_id)
             AND p_sum_contr BETWEEN sr.sum_without_nds_min AND
                 sr.sum_without_nds_max
             AND sr.status = 1
             AND sr.is_route_actual = 1 -- актуальность
             AND sr.is_belong = decode(p_initca, -1, 0, 1)
             AND sr.central = cc.central
             AND sr.is_agree = 0
             AND sr.is_default = 0),
        k AS
         (SELECT m.route_id,
                 MAX(m.ccalc_id) AS ccalc_id
            FROM m
           GROUP BY m.route_id)
        SELECT k.route_id BULK COLLECT
          INTO sdv_arr
          FROM k
         WHERE rownum = 1;
      
      ELSE
        -- расходные
        WITH m AS
         (SELECT sr.route_id,
                 sr.ccalc_id
            FROM contract_out cc,
                 sroute       sr
           WHERE cc.cssouts_id = p_contrid
             AND cc.cgroup_id = sr.cgroup_id
             AND cc.category_id = sr.category_id
             AND (cc.ccalc_id = sr.ccalc_id OR sr.ccalc_id = -1)
             AND cc.frame_id = sr.frame_id
             AND cc.contest = sr.contest
             AND sr.korp_order = nvl((SELECT cntr.korp_order
                                       FROM contract_out cntr
                                      WHERE cntr.cssouts_id = cc.parent_contr_id
                                        AND cntr.status = 1),
                                     0)
             AND cc.casp_id IN
                 (SELECT sc.casp_id
                    FROM sroute_casp sc
                   WHERE sr.route_id = sc.route_id)
             AND p_sum_contr BETWEEN sr.sum_without_nds_min AND
                 sr.sum_without_nds_max
             AND sr.status = 1
             AND sr.is_route_actual = 1 -- актуальность
             AND sr.is_belong = decode(p_initca, -1, 0, 1)
             AND sr.central = cc.central -- #23193
             AND sr.is_agree = 0
             AND sr.is_default = 0),
        k AS
         (SELECT COUNT(*) AS cnt FROM m WHERE m.ccalc_id > 0)
        SELECT CASE
                 WHEN k.cnt > 0 THEN
                  (SELECT m.route_id
                     FROM m
                    WHERE m.ccalc_id > 0
                      AND rownum = 1)
                 ELSE
                  (SELECT m.route_id FROM m WHERE rownum = 1)
               END BULK COLLECT
          INTO sdv_arr
          FROM k; -- #35160
    
    END CASE;
  
    IF NOT sdv_arr.exists(1) THEN
      RETURN 1; -- маршрут по умолчанию
    END IF;
  
    l_routeid := sdv_arr(1);
  
    RETURN l_routeid;
  EXCEPTION
    WHEN OTHERS THEN
      ais_sys.dyn_js.prn_script('alert("find_route_contract: SQLERRM= ' ||
                                SQLERRM || '")');
      RETURN - 1;
  END;
```
---
## Questions
---
### Question 1
Объясните, что означает и с какой целью используется условие: **rownum = 1**
и насколько это уместно в данном случае?

### Answer
`rownum` немного похож на `level`. Это системная переменная, которая вводится во время исполнения `SELECT` запроса. В ней хранится номер (отсчет от 1) кортежа запроса.

Условие **rownum = 1** используется в случае, если нам нужен только один (любой или первый) кортеж в запросе.

Чаще всего эту конструкцию можно увидеть во вложенных запросах, которые должны возвращать скалярную величину, либо при использовании `INTO` для записи результата запроса в скалярные переменные.

### Example
```sql
SELECT 
    (SELECT cl.id 
        FROM z#CLIENT cl 
        WHERE cl.NAME like 'IVANOV%' 
        AND ROWNUM = 1
    ) as C_CLIENT
FROM dual;
```
```sql
DECLARE
    ind z#CLIENT.id%TYPE;
    name z#CLIENT.name%TYPE;
BEGIN
    -- if query return range will exception
    SELECT cl.id, cl.name
        FROM z#CLIENT cl 
        WHERE cl.NAME like 'IVANOV%' 
        AND ROWNUM = 1
        INTO ind, name;
END;
```
---
### Question 2
Как вы считаете, в данном случае есть ли смысл использования группировки в подзапросах?

### Answer
```sql
---
SELECT  m.route_id,
        MAX(m.ccalc_id) AS ccalc_id
FROM m
GROUP BY m.route_id
---
SELECT k.route_id BULK COLLECT
    INTO sdv_arr
    FROM k
    WHERE rownum = 1;
```

1. В запросе нет сортировки, поэтому не имеет значения, какой `rout_id` выбрать (выбирается первый из запроса).
1. В случае, если для `route_id` важно, чтобы был выбран "последний" `ccalc_id`, то группировка имела бы смысл.
1. В дальнейшем не используется ничего, кроме `rout_id`, в коллекцию добавляется только одно число, поэтому лишние вычисления не имеют смысла.

*Итог:*

В данной реализации нет необходимости использовать `group by`.


# Test 3

## Task
```sql
SELECT REGEXP_SUBSTR(str, '[^,]', 1, LEVEL)
    FROM (SELECT '9,8,7,6,5' str FROM DUAL)
    CONNECT BY REGEXP_SUBSTR(str, '[^,]', 1, LEVEL) IS NOT NULL;
```
---
## Questions
---
### Question 1
Каким будет результат выполнения данного запроса?

### Answer
Результат запроса:
1. 9
1. 8
1. 7
1. 6
1. 5

Пояснение:

Данная конструкция работает как `while`.

До тех пор, пока уровень вложенности не достигнет количества символов в строке без запятых, будет выводиться следующий такой символ.

---
### Question 2
Предложите структуру таблицы и придумайте пример запроса к ней, в котором была бы использована подобная конструкция?

### Answer
Предлагаю реально существующую таблицу.

### Example
```sql
CREATE TABLE z#PARAMS (
    id                  INTEGER NOT NULL
    , COLLECTION_ID     INTEGER
    , C_VALUE           VARCHAR2(4000)
);

-- Values of params are keeped as varchar^
-- <parametr name> ~ <parametr value> # ......

INSERT INTO z#PARAMS 
    (COLLECTION_ID, C_VALUE)
VALUES
    (0, 'par1~val1#par2~val2#')
    , (0, 'part1~vaal1#')
    , (1, 'ppar1~vaal1#pars2~vaaal2#psdars2~vaaal2#')
    , (2, 'pssar1~vaal1#pddar2~vaaal2#')
;
```

```sql
SELECT 
    p.id                as C_PARAMS_ID
    , p.COLLECTION_ID   as C_COLLECTION
    , sub.pname         as C_PARAM_NAME
    , sub.pval          as C_PARAM_VALUE
FROM z#PARAMS p,
    (SELECT 
        --             str, pattern, start, nth_app, match_pr, sub_exp
        REGEXP_SUBSTR(p.C_VALUE, '(^|#)(.*)~', 1, LEVEL, '', 1) as pname
        , REGEXP_SUBSTR(p.C_VALUE, '~(.*)(#|$)', 1, LEVEL, '', 0) as pval
    FROM DUAL
    CONNECT BY REGEXP_INSTR(p.C_VALUE, '~.*#', 1, LEVEL) > 0;
    ) sub;
```