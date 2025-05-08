# Практическое задание 3. Контрольное задание «АТГЦ-1984»
Проектирование базы данных ДНК

1. Модель данных
Для хранения информации о последовательностях ДНК миллиарда людей предлагается следующая структура:

Основные таблицы:
person - информация о людях
 
```sql
CREATE TABLE person (
  person_id BIGSERIAL PRIMARY KEY,
  full_name TEXT NOT NULL,
  created_at TIMESTAMPTZ DEFAULT now(),
  metadata JSONB
);
```
dna_sequence - последовательности ДНК


```sql
CREATE TABLE dna_sequence (
  sequence_id BIGSERIAL PRIMARY KEY,
  person_id BIGINT REFERENCES person(person_id),
  sequence TEXT NOT NULL,
  length INT GENERATED ALWAYS AS (length(sequence)) STORED,
  created_at TIMESTAMPTZ DEFAULT now()
);
```
sequence_fingerprint - оптимизированные данные для поиска

```sql
CREATE TABLE sequence_fingerprint (
  fingerprint_id BIGSERIAL PRIMARY KEY,
  sequence_id BIGINT REFERENCES dna_sequence(sequence_id),
  kmer_hash BIGINT NOT NULL,
  kmer TEXT NOT NULL
);
```
2. Методы доступа и индексы
Для ускорения поиска ближайших родственников предлагается:

GiST-индекс для полнотекстового поиска по последовательностям:

```sql
CREATE INDEX idx_dna_sequence_gin ON dna_sequence USING gin(sequence gin_trgm_ops);
```
RUM-индекс (расширение rum) для эффективного поиска похожих последовательностей:

```sql
CREATE EXTENSION rum;
CREATE INDEX idx_dna_sequence_rum ON dna_sequence USING rum(sequence rum_tsvector_ops);
```
B-дерево для быстрого доступа по person_id:

```sql
CREATE INDEX idx_dna_sequence_person ON dna_sequence(person_id);
```
Хеш-индекс для быстрого поиска по k-mer хэшам:

```sql
CREATE INDEX idx_fingerprint_hash ON sequence_fingerprint USING hash(kmer_hash);
```
3. Алгоритм поиска родственников
Препроцессинг данных:

Разбиваем все последовательности на k-mers (короткие подпоследовательности длиной k)

Вычисляем хэши для каждого k-mer

Сохраняем в таблицу sequence_fingerprint

Поисковой процесс:

Для входного биоматериала (несколько последовательностей) выполняем тот же препроцессинг

Находим совпадения k-mer хэшей с базой данных

Вычисляем меру сходства (Jaccard index или MinHash)

Ранжируем результаты по степени сходства

Оптимизация:

Используем вероятностные структуры данных (Bloom filter) для предварительной фильтрации

Применяем алгоритмы локального выравнивания (Smith-Waterman) для точного сравнения

4. Оптимизации для больших данных
Шардирование:

Разделение данных по хэшу от person_id

Горизонтальное масштабирование

Кэширование:

Redis для хранения популярных запросов

Memcached для промежуточных результатов

Компрессия:

Использование алгоритмов сжатия для последовательностей

Хранение в бинарном формате (2 бита на нуклеотид)

Columnar storage:

Для аналитических запросов можно использовать ClickHouse

Предварительные агрегации по k-mer частотам
