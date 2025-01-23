```sql
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
