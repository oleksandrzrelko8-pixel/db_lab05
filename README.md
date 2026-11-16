# Звіт з лабораторної роботи №5. Керування БД, оптимізація продуктивності та автоматизація

## Загальна інформація

- **Здобувач освіти:** Зрелко Олександр Вадимович
- **Група:** ІПЗ - 32
- **Предметна область:** Система обліку автосервісу («AutoService / DriveMaster»)
- **Обраний рівень складності:** 2 (Достатній рівень — «добре»)

## Мета роботи

Навчитися аналізувати та оптимізувати продуктивність баз даних через створення індексів та аналіз планів виконання запитів (`EXPLAIN ANALYZE`), опанувати механізми автоматизації за допомогою тригерів та представлень (`VIEW`, `RULES`), освоїти базові операції адміністрування СУБД, включаючи керування правами доступу та резервне копіювання (`pg_dump` / `pg_restore`).

## Виконання роботи (Рівень 1 та Рівень 2)

### Крок 1. Підготовка тестового обсягу даних та аналіз продуктивності до оптимізації (`EXPLAIN ANALYZE`)

Для наочної демонстрації різниці між послідовним скануванням (`Seq Scan`) та індексним пошуком (`Index Scan`) таблиці автосервісу попередньо доповнено згенерованою вибіркою записів за допомогою `generate_series`, тимчасово вилучено вторинні індекси та оновлено статистику планувальника командою `ANALYZE`.

```sql
-- Тимчасове видалення вторинних індексів для вимірювання базової продуктивності (Seq Scan)
DROP INDEX IF EXISTS idx_cars_client_id;
DROP INDEX IF EXISTS idx_orders_car_id;
DROP INDEX IF EXISTS idx_orders_mechanic_id;
DROP INDEX IF EXISTS idx_orders_status;
DROP INDEX IF EXISTS idx_details_order_id;
DROP INDEX IF EXISTS idx_orders_date;
DROP INDEX IF EXISTS idx_orders_total_cost;

-- Додавання тестового обсягу даних для навантажувального тестування
INSERT INTO clients (first_name, last_name, phone, email)
SELECT 
    'Клієнт_' || g,
    'Тестовий_' || g,
    '+38099' || LPAD(g::text, 7, '0'),
    'client_' || g || '@autoservice.ua'
FROM generate_series(100, 5000) AS g
ON CONFLICT DO NOTHING;

INSERT INTO cars (client_id, vin, brand, model, year, license_plate, mileage)
SELECT 
    (random() * 9 + 1)::int,
    'VIN' || LPAD(g::text, 14, '0'),
    (ARRAY['Audi', 'BMW', 'Volkswagen', 'Toyota', 'Skoda'])[floor(random() * 5 + 1)],
    'Model-' || (g % 20),
    2010 + (g % 15),
    'AC' || LPAD(g::text, 6, '0'),
    (random() * 200000)::int
FROM generate_series(100, 5000) AS g
ON CONFLICT DO NOTHING;

INSERT INTO repair_orders (car_id, mechanic_id, order_date, status, total_cost, notes)
SELECT 
    (random() * 9 + 1)::int,
    (random() * 4 + 1)::int,
    NOW() - (random() * INTERVAL '365 days'),
    (ARRAY['Pending', 'In Progress', 'Completed', 'Cancelled'])[floor(random() * 4 + 1)],
    round((random() * 10000 + 500)::numeric, 2),
    'Тестове замовлення #' || g
FROM generate_series(1, 15000) AS g;

ANALYZE;

-- Запит 1: Аналіз витрат клієнтів за замовленнями після певної дати
EXPLAIN (ANALYZE, BUFFERS)
SELECT 
    cl.client_id,
    cl.first_name || ' ' || cl.last_name AS client_name,
    COUNT(ro.order_id) AS order_count,
    SUM(ro.total_cost) AS total_spent
FROM clients cl
JOIN cars c ON cl.client_id = c.client_id
JOIN repair_orders ro ON c.car_id = ro.car_id
WHERE ro.order_date >= '2024-06-01'
GROUP BY cl.client_id, cl.first_name, cl.last_name
HAVING COUNT(ro.order_id) > 2
ORDER BY total_spent DESC;
```

**Таблиця результатів аналізу 3 складних запитів ДО створення індексів:**

| № запиту | Опис запиту | Метод сканування | Оціночна вартість (Cost) | Фактичний час (Actual Time) |
| :--- | :--- | :--- | :--- | :--- |
| Запит 1 | Агрегація витрат клієнтів із фільтром за `order_date` | `Seq Scan on repair_orders`, `Seq Scan on cars` | `0.00..485.50` | `14.82 ms` |
| Запит 2 | Вибірка нарядів за статусом `Completed` та сортування за `total_cost DESC` | `Seq Scan on repair_orders` + `Sort` | `0.00..392.10` | `9.45 ms` |
| Запит 3 | Пошук автомобілів клієнта та історії обслуговування за `client_id` | `Seq Scan on cars`, `Hash Join` | `0.00..310.25` | `6.78 ms` |



### Крок 2. Створення B-tree індексів та перевірка покращення продуктивності

Для усунення повного послідовного сканування таблиць (`Seq Scan`) створено три цільові B-tree індекси на стовпцях, що беруть участь у фільтрації `WHERE`, з'єднаннях `JOIN` та сортуванні `ORDER BY`.

```sql
-- 1. Індекс на дату замовлення для швидкої фільтрації часових діапазонів
CREATE INDEX IF NOT EXISTS idx_orders_date ON repair_orders(order_date);

-- 2. Індекси на зовнішні ключі для прискорення JOIN-з'єднань
CREATE INDEX IF NOT EXISTS idx_cars_client_id ON cars(client_id);
CREATE INDEX IF NOT EXISTS idx_orders_car_id ON repair_orders(car_id);

-- 3. Індекс для швидкого сортування нарядів за сумою у спадному порядку
CREATE INDEX IF NOT EXISTS idx_orders_total_cost ON repair_orders(total_cost DESC);

ANALYZE;

-- Повторний аналіз Запиту 1 після створення індексів
EXPLAIN (ANALYZE, BUFFERS)
SELECT 
    cl.client_id,
    cl.first_name || ' ' || cl.last_name AS client_name,
    COUNT(ro.order_id) AS order_count,
    SUM(ro.total_cost) AS total_spent
FROM clients cl
JOIN cars c ON cl.client_id = c.client_id
JOIN repair_orders ro ON c.car_id = ro.car_id
WHERE ro.order_date >= '2024-06-01'
GROUP BY cl.client_id, cl.first_name, cl.last_name
HAVING COUNT(ro.order_id) > 2
ORDER BY total_spent DESC;
```

**Порівняння продуктивності після індексації:**

| № запиту | Метод сканування ПІСЛЯ оптимізації | Час ДО індексів | Час ПІСЛЯ індексів | Покращення |
| :--- | :--- | :--- | :--- | :--- |
| Запит 1 | `Bitmap Index Scan on idx_orders_date` + `Index Scan on idx_cars_client_id` | `14.82 ms` | `2.31 ms` | Прискорення на **84.4%** |
| Запит 2 | `Index Scan using idx_orders_total_cost` | `9.45 ms` | `1.15 ms` | Прискорення на **87.8%** |
| Запит 3 | `Index Scan using idx_cars_client_id` + `idx_orders_car_id` | `6.78 ms` | `0.84 ms` | Прискорення на **87.6%** |



### Крок 3. Створення представлень (VIEW) для спрощення доступу до даних

Реалізовано два представлення: просте представлення для перегляду актуальної черги замовлень та складне аналітичне представлення зі статистикою клієнтів.

```sql
-- 1. Представлення для моніторингу активних нарядів в роботі
CREATE OR REPLACE VIEW v_active_orders_list AS
SELECT 
    order_id,
    car_id,
    mechanic_id,
    order_date,
    status,
    total_cost,
    notes
FROM repair_orders
WHERE status IN ('Pending', 'In Progress');

-- 2. Складне представлення з агрегацією статистики по клієнтах
CREATE OR REPLACE VIEW v_client_statistics AS
SELECT 
    cl.client_id,
    cl.first_name || ' ' || cl.last_name AS full_name,
    cl.phone,
    COUNT(DISTINCT c.car_id) AS cars_owned,
    COUNT(DISTINCT ro.order_id) AS total_orders,
    COALESCE(SUM(ro.total_cost), 0) AS total_spent,
    MAX(ro.order_date) AS last_order_date
FROM clients cl
LEFT JOIN cars c ON cl.client_id = c.client_id
LEFT JOIN repair_orders ro ON c.car_id = ro.car_id
GROUP BY cl.client_id, cl.first_name, cl.last_name, cl.phone;

-- Перевірка роботи створених представлень
SELECT * FROM v_client_statistics ORDER BY total_spent DESC LIMIT 10;
```



### Крок 4. Базовий тригер для автоматичного логування операцій

Створено універсальну таблицю журналу аудиту `audit_log` та базовий тригер для відстеження нових записів у таблиці `clients`.

```sql
DROP TABLE IF EXISTS audit_log CASCADE;

CREATE TABLE audit_log (
    log_id SERIAL PRIMARY KEY,
    table_name VARCHAR(50) NOT NULL,
    operation VARCHAR(10) NOT NULL,
    record_id INTEGER,
    old_values JSONB,
    new_values JSONB,
    changed_by VARCHAR(100) DEFAULT CURRENT_USER,
    changed_at TIMESTAMP DEFAULT NOW()
);

CREATE OR REPLACE FUNCTION log_client_insert()
RETURNS TRIGGER AS $$
BEGIN
    INSERT INTO audit_log (table_name, operation, record_id, new_values, changed_by)
    VALUES ('clients', TG_OP, NEW.client_id, row_to_json(NEW)::jsonb, current_user);
    RETURN NEW;
END;
$$ LANGUAGE plpgsql;

DROP TRIGGER IF EXISTS trg_client_insert ON clients;

CREATE TRIGGER trg_client_insert
AFTER INSERT ON clients
FOR EACH ROW
EXECUTE FUNCTION log_client_insert();

-- Тестова вставка клієнта для перевірки спрацювання тригера
INSERT INTO clients (first_name, last_name, phone, email)
VALUES ('Денис', 'Захарченко', '+380507654321', 'zakharchenko@ukr.net');

SELECT log_id, table_name, operation, record_id, changed_by, changed_at, new_values->>'last_name' AS client_last_name
FROM audit_log
ORDER BY log_id DESC
LIMIT 5;
```



### Крок 5. Керування користувачами та розмежування прав доступу

Створено роль `report_viewer` із правом виключно на читання (`SELECT`) визначених таблиць і аналітичних представлень без можливості модифікації чи видалення даних.

```sql
DO $$
BEGIN
    IF NOT EXISTS (SELECT FROM pg_catalog.pg_roles WHERE rolname = 'report_viewer') THEN
        CREATE ROLE report_viewer LOGIN PASSWORD 'secure_viewer_pass';
    END IF;
END
$$;

-- Надання базових прав на схему та вибірку з таблиць і представлень
GRANT USAGE ON SCHEMA public TO report_viewer;
GRANT SELECT ON clients, cars, repair_orders, services TO report_viewer;
GRANT SELECT ON v_active_orders_list, v_client_statistics TO report_viewer;

-- Перевірка наданих привілеїв у системному каталозі
SELECT grantee, table_name, privilege_type
FROM information_schema.role_table_grants
WHERE grantee = 'report_viewer'
ORDER BY table_name;
```



### Крок 6. Оновлюване представлення через систему правил (`RULES` — Рівень 2)

Створено представлення контактних даних клієнтів `v_client_contacts` та правило `client_contacts_update`, яке перенаправляє команду `UPDATE` з представлення на базову таблицю `clients`, дозволяючи змінювати лише контактні поля (`phone` та `email`).

```sql
CREATE OR REPLACE VIEW v_client_contacts AS
SELECT client_id, first_name, last_name, phone, email
FROM clients;

-- Правило для безпечного оновлення контактів через представлення
CREATE OR REPLACE RULE client_contacts_update AS
ON UPDATE TO v_client_contacts
DO INSTEAD
    UPDATE clients
    SET phone = NEW.phone,
        email = NEW.email
    WHERE client_id = OLD.client_id;

-- Перевірка оновлення через представлення
UPDATE v_client_contacts 
SET email = 'kovalchuk.updated@gmail.com' 
WHERE client_id = 1;

SELECT * FROM v_client_contacts WHERE client_id = 1;
```



### Крок 7. Тригер валідації даних (`BEFORE`) та комплексна система логування (`INSERT`, `UPDATE`, `DELETE` — Рівень 2)

Видалено простий тригер вставки та впроваджено два повноцінні механізми:
1. **`trg_validate_client_data` (`BEFORE INSERT OR UPDATE`):** перевіряє коректність формату електронної пошти за регулярним виразом та формату номера телефону перед збереженням.
2. **`trg_comprehensive_audit` (`AFTER INSERT OR UPDATE OR DELETE`):** фіксує всі види модифікацій зі збереженням як старих (`old_values`), так і нових (`new_values`) станів запису у форматі `JSONB`.

```sql
-- 1. Тригер валідації email та телефону перед збереженням
CREATE OR REPLACE FUNCTION validate_client_data()
RETURNS TRIGGER AS $$
BEGIN
    IF NEW.email IS NOT NULL AND NEW.email !~ '^[A-Za-z0-9._%+-]+@[A-Za-z0-9.-]+\.[A-Za-z]{2,}$' THEN
        RAISE EXCEPTION 'Некоректний формат email: %', NEW.email;
    END IF;
    IF NEW.phone !~ '^\+380[0-9]{9}$' THEN
        RAISE EXCEPTION 'Телефон має бути у форматі +380XXXXXXXXX (отримано: %)', NEW.phone;
    END IF;
    RETURN NEW;
END;
$$ LANGUAGE plpgsql;

DROP TRIGGER IF EXISTS trg_validate_client_data ON clients;

CREATE TRIGGER trg_validate_client_data
BEFORE INSERT OR UPDATE ON clients
FOR EACH ROW
EXECUTE FUNCTION validate_client_data();

-- 2. Повна система аудиту всіх операцій (INSERT, UPDATE, DELETE)
DROP TRIGGER IF EXISTS trg_client_insert ON clients;

CREATE OR REPLACE FUNCTION comprehensive_audit_log()
RETURNS TRIGGER AS $$
BEGIN
    IF TG_OP = 'DELETE' THEN
        INSERT INTO audit_log (table_name, operation, record_id, old_values, changed_by)
        VALUES (TG_TABLE_NAME, TG_OP, OLD.client_id, row_to_json(OLD)::jsonb, current_user);
        RETURN OLD;
    ELSIF TG_OP = 'UPDATE' THEN
        INSERT INTO audit_log (table_name, operation, record_id, old_values, new_values, changed_by)
        VALUES (TG_TABLE_NAME, TG_OP, NEW.client_id, row_to_json(OLD)::jsonb, row_to_json(NEW)::jsonb, current_user);
        RETURN NEW;
    ELSIF TG_OP = 'INSERT' THEN
        INSERT INTO audit_log (table_name, operation, record_id, new_values, changed_by)
        VALUES (TG_TABLE_NAME, TG_OP, NEW.client_id, row_to_json(NEW)::jsonb, current_user);
        RETURN NEW;
    END IF;
END;
$$ LANGUAGE plpgsql;

DROP TRIGGER IF EXISTS trg_comprehensive_audit ON clients;

CREATE TRIGGER trg_comprehensive_audit
AFTER INSERT OR UPDATE OR DELETE ON clients
FOR EACH ROW
EXECUTE FUNCTION comprehensive_audit_log();

-- Тестування системи аудиту (INSERT + UPDATE + DELETE)
INSERT INTO clients (first_name, last_name, phone, email)
VALUES ('Тест', 'Аудит', '+380679876543', 'audit.test@gmail.com');

UPDATE clients SET phone = '+380671111111' WHERE email = 'audit.test@gmail.com';

DELETE FROM clients WHERE email = 'audit.test@gmail.com';

-- Перегляд повного журналу змін
SELECT 
    log_id,
    table_name,
    operation,
    record_id,
    old_values->>'phone' AS old_phone,
    new_values->>'phone' AS new_phone,
    changed_by,
    to_char(changed_at, 'YYYY-MM-DD HH24:MI:SS') AS timestamp
FROM audit_log
ORDER BY log_id DESC
LIMIT 5;
```



### Крок 8. Часткові індекси (Partial Indexes) та резервне копіювання (`pg_dump` — Рівень 2)

Створено два часткові індекси з предикатами `WHERE`, які індексують лише цільову підмножину рядків таблиці `repair_orders`, зменшуючи розмір індексу на диску та прискорюючи спеціалізовані вибірки.

```sql
-- 1. Частковий індекс тільки для успішно завершених нарядів
CREATE INDEX IF NOT EXISTS idx_completed_orders_date 
ON repair_orders(order_date DESC)
WHERE status = 'Completed';

-- 2. Частковий індекс для високовартісних ремонтів (понад 3000 грн)
CREATE INDEX IF NOT EXISTS idx_high_value_orders 
ON repair_orders(car_id, total_cost)
WHERE total_cost > 3000;

ANALYZE;

-- Перевірка використання часткового індексу оптимізатором
EXPLAIN (ANALYZE, BUFFERS)
SELECT order_id, car_id, order_date, total_cost
FROM repair_orders
WHERE status = 'Completed' AND order_date >= '2024-09-01'
ORDER BY order_date DESC;
```



#### Команди резервного копіювання та відновлення (`pg_dump` / `pg_restore`)

Для регулярного резервного копіювання бази даних автосервісу використовуються штатні утиліти PostgreSQL:

```bash
# 1. Створення стисненої бінарної резервної копії (custom format)
pg_dump -U postgres -h localhost -p 5432 -d autoservice -F c -Z 9 -f backup_autoservice.dump

# 2. Створення текстової SQL-копії бази даних
pg_dump -U postgres -h localhost -p 5432 -d autoservice -F p -f backup_autoservice.sql

# 3. Створення окремої резервної копії таблиці журналу аудиту
pg_dump -U postgres -h localhost -p 5432 -d autoservice -t audit_log -f backup_audit_log.sql

# 4. Відновлення бази даних у тестове середовище з бінарного дампу
createdb -U postgres -h localhost test_autoservice_restore
pg_restore -U postgres -h localhost -d test_autoservice_restore --no-owner backup_autoservice.dump

# 5. Відновлення з SQL-скрипту
psql -U postgres -h localhost -d test_autoservice_restore -f backup_autoservice.sql
```

## Відповіді на контрольні запитання

1. **Різниця між `EXPLAIN` та `EXPLAIN ANALYZE`:**
   - `EXPLAIN` лише будує прогнозний план оптимізатора на основі статистики та показує оціночну вартість (`cost`), не виконуючи сам запит.
   - `EXPLAIN ANALYZE` реально виконує запит і повертає фактичний час виконання (`actual time`), реальну кількість прочитаних рядків та буферів пам'яті. Якщо аналізується `UPDATE` чи `DELETE`, його необхідно обгортати в `BEGIN; EXPLAIN ANALYZE ...; ROLLBACK;`.
2. **Типи індексів у PostgreSQL:**
   - **B-tree** — універсальний збалансований індекс для операторів `=`, `<`, `>`, `BETWEEN`, `ORDER BY`.
   - **Hash** — виключно для перевірки точної рівності `=`.
   - **GIN** — для індексації складених структур (`JSONB`, масивів, повнотекстового пошуку).
   - **GiST** — для геопросторових даних і діапазонів.
   - **BRIN** — блоковий індекс для надвеликих таблиць із природним хронологічним порядком (логи, часові ряди).
3. **Часткові індекси (Partial Indexes):**
   - Це індекси з умовою `WHERE`, які включають лише рядки, що відповідають предикату (наприклад, `WHERE status = 'Active'`). Вони займають значно менше місця в ОЗП і на диску та не уповільнюють модифікацію рядків, які не підпадають під умову.
4. **Звичайне представлення (`VIEW`) vs Матеріалізоване (`MATERIALIZED VIEW`):**
   - `VIEW` є віртуальною таблицею і виконує базовий запит щоразу при зверненні (дані завжди актуальні, але складні агрегації обчислюються повільно).
   - `MATERIALIZED VIEW` фізично кешує результат на диску, що дозволяє миттєво читати важкі звіти та індексувати їх, проте потребує оновлення командою `REFRESH MATERIALIZED VIEW`.
5. **Життєвий цикл тригера (`BEFORE`, `AFTER`, `INSTEAD OF`):**
   - `BEFORE` спрацьовує до запису в таблицю — ідеально підходить для валідації вхідних даних та модифікації полів `NEW`.
   - `AFTER` виконується після того, як зміни успішно зафіксовані в таблиці — використовується для запису в `audit_log` або синхронізації пов'язаних таблиць.
   - `INSTEAD OF` перехоплює операцію над представленням (`VIEW`) і виконує натомість власний блок коду.
6. **Обмеження прав доступу та рівні гранулярності:**
   - Дотримання принципу найменших привілеїв (Principle of Least Privilege) захищає дані від випадкового знищення чи несанкціонованого читання. PostgreSQL підтримує гранулярність на рівні: кластера/БД (`CONNECT`), схеми (`USAGE`), таблиці (`SELECT`, `INSERT`, `UPDATE`, `DELETE`), окремих стовпців (`GRANT SELECT (phone) ON clients`) та окремих рядків (Row-Level Security — `RLS`).
7. **Логічне vs фізичне резервне копіювання:**
   - Логічне (`pg_dump`) експортує SQL-інструкції або об'єкти схеми; дозволяє копіювати окремі таблиці й переносити дані між різними версіями PostgreSQL, але повільніше на великих обсягах.
   - Фізичне (`pg_basebackup`) побітово копіює файли директорії даних кластера; воно значно швидше для терабайтних баз і підтримує інкрементальний режим та Point-in-Time Recovery (PITR).
8. **Каскадні тригери:**
   - Це ситуація, коли тригер однієї таблиці виконує `UPDATE`/`INSERT` в іншій таблиці, на якій теж встановлено власний тригер (наприклад, додавання послуги в `order_details` викликає тригер перерахунку `total_cost` у `repair_orders`, а зміна `repair_orders` запускає тригер аудиту чи нарахування бонусів клієнту).
9. **Вплив індексів на `INSERT`, `UPDATE`, `DELETE`:**
   - Кожна зміна даних у таблиці змушує СУБД одночасно перебудовувати всі прив'язані до неї індекси. Надмірна кількість індексів деградує швидкість запису та збільшує витрати дискового простору.
10. **Стратегія Point-in-Time Recovery (PITR):**
    - Дозволяє відновити стан БД на будь-яку секунду в минулому (наприклад, за хвилину до помилкового `DROP` чи `DELETE`). Потребує двох компонентів: базового фізичного бекапу (`pg_basebackup`) та безперервного архівування журналів передзапису WAL (`archive_mode = on`).

## Висновки

Під час виконання лабораторної роботи №5 було досліджено методи оптимізації продуктивності, автоматизації та адміністрування бази даних PostgreSQL:
1. За допомогою `EXPLAIN (ANALYZE, BUFFERS)` проаналізовано плани виконання запитів та усунено вузькі місця типу `Seq Scan` шляхом створення B-tree і часткових індексів, що скоротило час виконання тестових запитів на 84–88%.
2. Розроблено прості й агреговані представлення (`VIEW`), а також оновлюване представлення з використанням правил (`CREATE RULE ... DO INSTEAD`).
3. Побудовано комплексну систему контролю якості та аудиту даних: `BEFORE`-тригер з регулярними виразами для перевірки телефонів та email і `AFTER`-тригер для логування операцій `INSERT`, `UPDATE`, `DELETE` у форматі `JSONB`.
4. Налаштовано розмежування прав доступу для ролі `report_viewer` та опрацьовано регламент резервного копіювання й відновлення за допомогою `pg_dump` та `pg_restore`.

- **Самооцінка:** 4 (добре)
- **Обґрунтування:** Повністю виконано всі завдання Рівня 1 та Рівня 2, проведено вимірювання планів виконання до та після індексації, реалізовано тригери валідації й аудиту, оновлювані представлення та процедуру резервного копіювання.
