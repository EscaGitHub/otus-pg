## Хранимые функции и процедуры часть.
Домашнее задание 3 месяц 24 занятие

### Работа с БД
- Создаем удаленное подключение к Postgres в IDE Rider
```bash
jdbc:postgresql://158.160.21.77:5432/postgres -- строка подключения
```
### Подготовка БД
- Выполняем скрипты из приложенного файла
```postgresql
DROP SCHEMA IF EXISTS pract_functions CASCADE;
CREATE SCHEMA pract_functions;

SET search_path = pract_functions, publ

-- Товары:
CREATE TABLE goods
(
    goods_id    integer PRIMARY KEY,
    good_name   varchar(63) NOT NULL,
    good_price  numeric(12, 2) NOT NULL CHECK (good_price > 0.0)
);
INSERT INTO goods (goods_id, good_name, good_price)
VALUES 	(1, 'Спички хозяйственные', .50),
		(2, 'Автомобиль Ferrari FXX K', 185000000.01);

-- Продажи
CREATE TABLE sales
(
    sales_id    integer GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    good_id     integer REFERENCES goods (goods_id),
    sales_time  timestamp with time zone DEFAULT now(),
    sales_qty   integer CHECK (sales_qty > 0)
);

INSERT INTO sales (good_id, sales_qty) VALUES (1, 10), (1, 1), (1, 120), (2, 1);

-- Отчет:
SELECT G.good_name, sum(G.good_price * S.sales_qty)
FROM goods G
INNER JOIN sales S ON S.good_id = G.goods_id
GROUP BY G.good_name;

-- С увеличением объёма данных отчет стал создаваться медленно
-- Принято решение денормализовать БД, создать таблицу
CREATE TABLE good_sum_mart
(
	good_name   varchar(63) NOT NULL,
	sum_sale	numeric(16, 2)NOT NULL
);
```
- Вывод текущего отчета

| good\_name | sum |
| :--- | :--- |
| Автомобиль Ferrari FXX K | 185000000.01 |
| Спички хозяйственные | 65.5 |

### Создание триггера
Создать триггер на таблице продаж, для поддержки данных в витрине в актуальном состоянии (вычисляющий при 
каждой продаже сумму и записывающий её в витрину)

- Функция триггера
```postgresql
SET search_path = pract_functions, publ;

CREATE OR REPLACE FUNCTION update_good_sum_mart()
    RETURNS TRIGGER
AS $$
DECLARE
    v_good_price goods.good_price%TYPE;
    v_good_name goods.good_name%TYPE;
BEGIN
    -- Получаем цену и название продукта:
    SELECT good_price, good_name INTO v_good_price, v_good_name
    FROM goods WHERE goods_id = COALESCE(NEW.good_id, OLD.good_id); 
    
    IF tg_level = 'ROW' THEN
        CASE tg_op
            WHEN 'DELETE' 
                THEN
                    UPDATE good_sum_mart
                    SET sum_sale = sum_sale - (OLD.sales_qty * v_good_price)
                    WHERE good_name = v_good_name;
            WHEN 'UPDATE'
                THEN
                    UPDATE good_sum_mart
                    -- Уменьшаем со старой ценой и увеличиваем с новой:
                    SET sum_sale = sum_sale - (OLD.sales_qty * v_good_price) + (NEW.sales_qty * v_good_price)
                    WHERE good_name = v_good_name;
            WHEN 'INSERT'
                THEN
                    -- Может не быть в таблице нового продукта:
                    IF (SELECT COUNT(*) FROM good_sum_mart WHERE good_name = v_good_name) = 0 THEN
                        -- Вставляем новую строку, если нет:
                        INSERT INTO good_sum_mart (good_name, sum_sale)
                        VALUES (v_good_name, NEW.sales_qty * v_good_price);
                    ELSE
                        -- Или добавляем цену к существующей записи:
                        UPDATE good_sum_mart SET sum_sale = sum_sale + (NEW.sales_qty * v_good_price)
                        WHERE good_name = v_good_name;
                    END IF;
        END CASE;
    END IF;

    RAISE NOTICE E'\nG_TABLE_NAME = %\nTG_WHEN = %\nTG_OP = %\nTG_LEVEL = %\n -------------', TG_TABLE_NAME, TG_WHEN, TG_OP, TG_LEVEL;
   
    RETURN NULL; 
END;
$$ LANGUAGE plpgsql;
```
- Триггер, делаем два для оптимизации выполнения update
  https://postgrespro.ru/docs/postgrespro/16/functions-comparison
```text
тип_данных IS DISTINCT FROM тип_данных → boolean
Не равно, при этом NULL воспринимается как обычное значение.
1 IS DISTINCT FROM NULL → t (а не NULL)
NULL IS DISTINCT FROM NULL → f (а не NULL)
```
```postgresql
CREATE TRIGGER sales_after_insert_delete
    AFTER INSERT OR DELETE 
    ON sales
    FOR EACH ROW
EXECUTE FUNCTION update_good_sum_mart();

CREATE TRIGGER sales_after_update
    AFTER UPDATE 
    ON sales
    FOR EACH ROW
    WHEN (OLD.* IS DISTINCT FROM NEW.*) -- сравниваем строки, что бы лишний раз не вызывать фунуцмю, если равны
EXECUTE FUNCTION update_good_sum_mart();
```
### Проверка триггера
- Заливаем из текущих данных, для синхронизации view и текущей новой таблицы
```postgresql
INSERT INTO pract_functions.good_sum_mart(good_name, sum_sale) 
(SELECT G.good_name, sum(G.good_price * S.sales_qty)
      FROM goods G
               INNER JOIN sales S ON S.good_id = G.goods_id
      GROUP BY G.good_name);
```
| good\_name | sum\_sale |
| :--- | :--- |
| Автомобиль Ferrari FXX K | 555000000.03 |
| Спички хозяйственные | 67.50 |
- Добавляем новую продажу
```postgresql
INSERT INTO pract_functions.sales (good_id, sales_time, sales_qty) 
VALUES (2, now(), 2);
```
| good\_name | sum\_sale |
| :--- | :--- |
| Спички хозяйственные | 67.50 |
| Автомобиль Ferrari FXX K | 925000000.05 |
- Добавляем новый вид продукции и продажу по нему
```postgresql
-- Продукт
INSERT INTO pract_functions.goods (goods_id, good_name, good_price)
VALUES (3, 'Пружина', 50.02);
-- Продажа
INSERT INTO pract_functions.sales (good_id, sales_time, sales_qty)
VALUES (3, now(), 2);
```
| good\_name | sum\_sale |
| :--- | :--- |
| Спички хозяйственные | 67.50 |
| Автомобиль Ferrari FXX K | 925000000.05 |
| Пружина | 100.04 |
- Обновляем продажу спичек с 10 штук на 5
```postgresql
UPDATE pract_functions.sales SET good_id = 1, sales_time = '2024-07-07 05:46:47.705825 +00:00', sales_qty = 5 WHERE sales_id = 1;

G_TABLE_NAME = sales
TG_WHEN = AFTER
TG_OP = UPDATE
TG_LEVEL = ROW
```
| good\_name | sum\_sale |
| :--- | :--- |
| Автомобиль Ferrari FXX K | 925000000.05 |
| Пружина | 100.04 |
| Спички хозяйственные | 65.00 |
- Удаляем продажу первой Ferrari
```postgresql
DELETE FROM pract_functions.sales  where sales_id = 4;
```

| good\_name | sum\_sale |
| :--- | :--- |
| Пружина | 100.04 |
| Спички хозяйственные | 65.00 |
| Автомобиль Ferrari FXX K | 740000000.04 |
- Задание * - Чем такая схема (витрина+триггер) предпочтительнее отчета, создаваемого "по требованию" (кроме производительности)?
Схема с денормализованной таблицей отчета предпочтительнее, т.к. она хранит исторические расчеты на основе цен в момент добавления
продаж. Если же вызывать view она будет пересчитывать все продажи на цены в текущий момент времени.

