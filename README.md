```sql
CREATE TABLE hsa13.users (
    id BIGINT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
    name VARCHAR(255) NOT NULL,
    date_of_birth DATE NOT NULL,
    email VARCHAR(255),
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
) ENGINE=InnoDB;

DELIMITER $$

CREATE PROCEDURE hsa13.insert_users_batch()
BEGIN
    DECLARE batch_start INT DEFAULT 0;
    DECLARE batch_size INT DEFAULT 1000; -- Менший розмір партії

    WHILE batch_start < 40000000 DO
        INSERT INTO users (name, date_of_birth, email)
        SELECT 
            CONCAT('User', batch_start + seq) AS name,
            DATE_ADD('1970-01-01', INTERVAL FLOOR(RAND() * 20000) DAY) AS date_of_birth,
            CONCAT('user', batch_start + seq, '@example.com') AS email
        FROM (
            SELECT @row := @row + 1 AS seq
            FROM (SELECT 1 UNION ALL SELECT 2 UNION ALL SELECT 3 UNION ALL SELECT 4 UNION ALL SELECT 5 UNION ALL
                  SELECT 6 UNION ALL SELECT 7 UNION ALL SELECT 8 UNION ALL SELECT 9 UNION ALL SELECT 10) t1,
                 (SELECT 1 UNION ALL SELECT 2 UNION ALL SELECT 3 UNION ALL SELECT 4 UNION ALL SELECT 5 UNION ALL
                  SELECT 6 UNION ALL SELECT 7 UNION ALL SELECT 8 UNION ALL SELECT 9 UNION ALL SELECT 10) t2,
                 (SELECT @row := 0) r
            LIMIT batch_size
        ) AS seqs;

        COMMIT; -- Завершення транзакції після кожного батчу
        SET batch_start = batch_start + batch_size;
    END WHILE;
END$$

DELIMITER ;


SET GLOBAL wait_timeout = 600;
--    Визначає, скільки секунд сервер буде чекати на активність з'єднання до його закриття.

SET GLOBAL interactive_timeout = 600;
-- Таймаут для інтерактивних з'єднань (наприклад, через MySQL CLI).
-- Те ж значення, що й для wait_timeout, дозволяє уникнути раптового розриву підключення.

SET GLOBAL net_read_timeout = 600;
-- Час (у секундах), протягом якого сервер чекатиме на отримання пакета даних від клієнта.

SET GLOBAL net_write_timeout = 600;
-- Час, протягом якого сервер чекатиме, перш ніж припинити відправлення пакета клієнту.

SET GLOBAL max_allowed_packet = 268435456; -- 256M;
-- Визначає максимальний розмір окремого пакета даних (наприклад, для BLOB, TEXT).
-- Якщо значення занадто низьке, можуть виникати помилки типу "Packet too large".

SET GLOBAL innodb_log_buffer_size=67108864; -- 64M;
-- Визначає розмір буфера для журналу транзакцій.
-- 64M підходить для навантажених систем, допомагаючи зменшити кількість операцій запису на диск.
-- Якщо буфер занадто малий, може спостерігатись частий запис на диск, що знижує продуктивність.

-- SET GLOBAL innodb_buffer_pool_size=4G;
-- Основний кеш для збереження таблиць та індексів InnoDB.
--    Для виділеного MySQL-сервера встановлюйте до 60-70% від усієї доступної пам'яті.
--    Якщо таблиці великі, рекомендується збільшити значення.
-- 2G підходить для середніх навантажень.

SET GLOBAL innodb_redo_log_capacity=536870912; -- 512M
-- Визначає обсяг redo-журналу, який використовується для відновлення при збоях.


SET GLOBAL innodb_flush_log_at_trx_commit=2;
-- Дозволяє InnoDB записувати зміни в журнал одразу після коміту, але записувати на диск кожну секунду.

SET autocommit = 0;

CALL hsa13.insert_users_batch();
-- 2.172 sec
-- 2.156 sec


SET autocommit = 1;



SET GLOBAL innodb_flush_log_at_trx_commit=1;
-- При кожному COMMIT, InnoDB:
--    Записує (write) дані в журнал транзакцій (redo log).
--    Виконує примусовий запис на диск (fsync).

SET autocommit = 0;

CALL hsa13.insert_users_batch();
-- 2.516 sec
-- 2.578 sec

SET autocommit = 1;



SET GLOBAL innodb_flush_log_at_trx_commit=0;
-- При коміті транзакції:
--    Лог записується лише у буфер.
--    Фізичний запис у лог-файл та синхронізація з диском відбувається раз на секунду.

SET autocommit = 0;

CALL hsa13.insert_users_batch();
-- 1.891 sec
-- 1.875 sec

SET autocommit = 1;

# HSA13_hw8_SQL_Database

-- SELECT * FROM hsa13.users
-- ORDER BY id desc
-- limit 100;

 -- SELECT COUNT(*) FROM hsa13.users;

-- DELETE FROM hsa13.users

SELECT * FROM hsa13.users WHERE date_of_birth = '1990-01-01';
-- 2238 row(s) returned	2.672 sec / 14.656 sec

EXPLAIN SELECT * FROM hsa13.users WHERE date_of_birth = '1990-01-01';
-- id, select_type, table, partitions, type, possible_keys, key, key_len, ref, rows, filtered, Extra
-- '1', 'SIMPLE', 'users', NULL, 'ALL', NULL, NULL, NULL, NULL, '43074311', '10.00', 'Using where'







CREATE INDEX idx_dob_btree ON hsa13.users (date_of_birth);
-- 0 row(s) affected Records: 0  Duplicates: 0  Warnings: 0 returned 104.828 sec

SELECT * FROM hsa13.users WHERE date_of_birth = '1990-01-01';
-- 2238 row(s) returned  0.094 sec / 0.531 sec

EXPLAIN SELECT * FROM hsa13.users WHERE date_of_birth = '1990-01-01';
-- id, select_type, table, partitions, type, possible_keys, key, key_len, ref, rows, filtered, Extra
-- '1', 'SIMPLE', 'users', NULL, 'ref', 'idx_dob_btree', 'idx_dob_btree', '3', 'const', '2238', '100.00', NULL






ALTER TABLE hsa13.users 
ADD dob_hash BINARY(32) GENERATED ALWAYS AS (UNHEX(SHA2(date_of_birth, 256))) STORED;
-- 43496031 row(s) affected Records: 43496031  Duplicates: 0  Warnings: 0 1214.313 sec
CREATE INDEX idx_dob_hash ON hsa13.users (dob_hash);
-- 0 row(s) affected Records: 0  Duplicates: 0  Warnings: 0 450.797 sec


SELECT * FROM hsa13.users WHERE dob_hash = UNHEX(SHA2('1990-01-01', 256));
-- 2238 row(s) returned 0.032 sec / 0.031 sec

EXPLAIN SELECT * FROM hsa13.users WHERE dob_hash = UNHEX(SHA2('1990-01-01', 256));
-- id, select_type, table, partitions, type, possible_keys, key, key_len, ref, rows, filtered, Extra
-- '1', 'SIMPLE', 'users', NULL, 'ref', 'idx_dob_hash', 'idx_dob_hash', '33', 'const', '2238', '100.00', 'Using index condition'






ALTER TABLE hsa13.users 
ADD dob_hash_64 BINARY(64) GENERATED ALWAYS AS (SHA2(date_of_birth, 256)) STORED;
-- 43496031 row(s) affected Records: 43496031  Duplicates: 0  Warnings: 0 3190.797 sec

CREATE INDEX idx_dob_hash_64 ON hsa13.users (dob_hash_64);
-- 0 row(s) affected Records: 0  Duplicates: 0  Warnings: 0 1652.828 sec


SELECT * FROM hsa13.users WHERE dob_hash_64 = SHA2('1990-01-01', 256);
-- 2238 row(s) returned 0.047 sec / 0.610 sec
-- 2238 row(s) returned 0.000 sec / 0.016 sec

EXPLAIN SELECT * FROM hsa13.users WHERE dob_hash_64 = SHA2('1990-01-01', 256);
-- id, select_type, table, partitions, type, possible_keys, key, key_len, ref, rows, filtered, Extra
-- '1', 'SIMPLE', 'users', NULL, 'ref', 'idx_dob_hash_64', 'idx_dob_hash_64', '65', 'const', '4060', '100.00', 'Using index condition'






SELECT ROUND(SUM(LENGTH(date_of_birth))/ 1024 / 1024, 2) AS date_of_birth_size_in_Mbytes,
ROUND(SUM(LENGTH(dob_hash))/ 1024 / 1024, 2) AS dob_hash_size_in_Mbytes,
ROUND(SUM(LENGTH(dob_hash_64))/ 1024 / 1024, 2) AS dob_hash64_size_in_Mbytes
FROM hsa13.users;
-- date_of_birth_size_in_Mbytes, dob_hash_size_in_Mbytes, dob_hash64_size_in_Mbytes
-- '418.24', '1338.38', '2676.76'
