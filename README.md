# goit-rdb-hw-05 — Узагальнені табличні вирази (CTE), Materialized View

Домашнє завдання 5 (Тема 5) курсу реляційних БД: customer-level feature store для ML на e-commerce-датасеті Olist у PostgreSQL —
raw- і typed-шари, feature engineering pipeline через CTE-chain, recursive CTE на synthetic referral-графі,
Materialized View з `REFRESH ... CONCURRENTLY` і порівняння вартості через `EXPLAIN (ANALYZE, BUFFERS)`.

## Джерело даних

**Olist Brazilian E-commerce Public Dataset** — Kaggle:
[olistbr/brazilian-ecommerce](https://www.kaggle.com/datasets/olistbr/brazilian-ecommerce)
(≈100 тис. замовлень бразильського маркетплейсу Olist, 2016–2018, ліцензія CC BY-NC-SA 4.0).

Використано **повний датасет без семплювання**, 5 таблиць:

| CSV | Raw-таблиця | Typed-таблиця | Ключ | Рядків |
|---|---|---|---|---:|
| `olist_customers_dataset.csv` | `olist_customers_raw` | `olist_customers` | `customer_id` | 99 441 |
| `olist_orders_dataset.csv` | `olist_orders_raw` | `olist_orders` | `order_id` | 99 441 |
| `olist_order_items_dataset.csv` | `olist_order_items_raw` | `olist_order_items` | `(order_id, order_item_id)` | 112 650 |
| `olist_products_dataset.csv` | `olist_products_raw` | `olist_products` | `product_id` | 32 951 |
| `olist_sellers_dataset.csv` | `olist_sellers_raw` | `olist_sellers` | `seller_id` | 3 095 |

Службові об'єкти ДЗ: `hw5_referrals` (synthetic referral-граф), `mv_olist_customer_features` (Materialized View з ознаками).

## Отримання файлів

Notebook знаходить CSV автоматично, без ручних дій:

1. **З репозиторію** — файли лежать у [`data/olist/`](data/olist) (≈45 МБ, кожен файл < 100 МБ, тож вкладаються в ліміт GitHub).
   Використовуються, якщо notebook запущено з кореня репозиторію або репозиторій склоновано в Colab у `/content/goit-rdb-hw-05`.
2. **Через KaggleHub** — інакше виконується `kagglehub.dataset_download('olistbr/brazilian-ecommerce')`
   (датасет публічний, Kaggle credentials не потрібні).
3. **Stub-schema fallback** — якщо обидва варіанти недоступні (або задано `HW5_USE_STUB=1`), створюється детермінована
   stub-схема із завдання (30 customers / 50 orders / 100 order_items); усі наступні клітинки працюють без змін.

## Raw- і typed-шар

- **Raw** (`olist_*_raw`) — CSV «як є» через pandas `to_sql(if_exists='replace')`, без обмежень; лише проміжний шар.
- **Typed** (`olist_*`) — явні типи (`TIMESTAMP`, `NUMERIC(12,2)`, `CHAR(2)`), PK, FK (`orders → customers`,
  `order_items → orders / products / sellers`), `CHECK` на статуси й невід'ємні значення, `NOT NULL`. Наповнюється через
  `INSERT ... SELECT` з кастами; помилкові назви колонок CSV (`product_name_lenght`) нормалізовано до `*_length`.
  Smoke-test: кількість рядків typed = raw для всіх 5 таблиць.

## CTE-stages feature pipeline

В Olist `customer_id` створюється на кожне замовлення, а людину ідентифікує `customer_unique_id`. Тому ознаки рахуються на рівні
`customer_unique_id` (інакше `frequency` завжди = 1), а ключем feature store лишається `customer_id` (один рядок на `customer_id`).

| Stage | CTE | Що робить | Ознаки |
|---|---|---|---|
| 0 | `reference_date` | стабільна дата відліку `MAX(order_purchase_timestamp)::date + 1` (2018-10-18), не `CURRENT_DATE` | — |
| 1 | `stage1_orders_total` | order-level: статус, timestamp, `order_total = Σ(price + freight)` | — |
| 2 | `stage2_rfm` | агрегація на людину | `recency_days`, `frequency`, `monetary` |
| 3 | `stage3_cohort` | квартал першої покупки | `cohort_quarter` |
| 4 | `stage4_rolling_aov` | `AVG(...) OVER (RANGE 30 days)` + `ROW_NUMBER` для останнього замовлення | `avg_order_value_30d` |
| 5 | `stage5_cancel_rate` | частка `canceled` замовлень | `cancellation_rate` |
| 6 | `stage6_category_revenue` | revenue за категоріями (item-level) | — |
| 7 | `stage7_preferred_category` | top-1 категорія через `ROW_NUMBER` | `preferred_category` |

Pipeline зберігається в одній змінній `FEATURES_SQL` — з неї ж створюється MV і будуються `EXPLAIN`-плани, тож MV гарантовано
містить повний pipeline. Кожен stage можна перевірити окремо (`stage_sql('stage4_rolling_aov')`), у notebook є набір інваріантів.

## Структура репозиторію

```
.
├── hw5_olist_cte_mv.ipynb   # notebook з output усіх клітинок
├── README.md
└── data/
    └── olist/               # 5 CSV Olist (копія з Kaggle)
```

## Вміст notebook

1. **Setup і typed-схема** — `pgserver` (fallback: `apt-get postgresql`), версія сервера, raw-шар, typed-таблиці з constraints,
   `INSERT ... SELECT`, stub-schema fallback, індекси під JOIN-и, smoke-test raw vs typed.
2. **CTE-chain pipeline** — діагностика гранулярності, 8 named stages, перевірка окремих stages та інваріантів, профіль ознак.
3. **Recursive CTE** — `hw5_referrals` (детерміноване бінарне дерево з 31 клієнта), anchor через `NOT EXISTS`, depth/path,
   розмір піддерева; експеримент із циклом `A → B → A`: depth guard vs path guard vs `CYCLE`-clause.
4. **Materialized View** — MV з повного pipeline, `UNIQUE INDEX` на `customer_id`, звірка MV з живим pipeline,
   `REFRESH MATERIALIZED VIEW CONCURRENTLY`, демонстрація staleness/freshness, latency pipeline vs MV.
5. **EXPLAIN + Reflection** — `EXPLAIN (ANALYZE, BUFFERS)` для повного pipeline, вартість кожного stage, MV lookup,
   top-N з MV без/з індексом на `monetary`, розмір MV, Reflection.

## Запуск

### Google Colab

1. Відкрити `hw5_olist_cte_mv.ipynb` у Colab.
2. (Опційно, щоб узяти CSV з репозиторію замість KaggleHub) першою клітинкою виконати:
   `!git clone https://github.com/sofiiadorosh/goit-rdb-hw-05.git /content/goit-rdb-hw-05`
3. `Runtime → Restart session and run all`.

### Локально

Для `pgserver` потрібен Python 3.10–3.12:

```bash
python3.12 -m venv .venv && source .venv/bin/activate
pip install pgserver psycopg2-binary sqlalchemy pandas kagglehub jupyter
jupyter nbconvert --to notebook --execute --inplace hw5_olist_cte_mv.ipynb
```

Якщо PostgreSQL уже запущено, можна не встановлювати `pgserver`, а передати адресу сервера через змінну середовища
(так notebook проганявся для збереження output, PostgreSQL 16):

```bash
HW5_DB_URI=postgresql://postgres@127.0.0.1:5432/postgres jupyter nbconvert --to notebook --execute --inplace hw5_olist_cte_mv.ipynb
```

Перевірити stub-schema fallback: додати `HW5_USE_STUB=1`.
