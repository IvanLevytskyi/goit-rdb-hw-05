# Olist SQL: DML, CTE pipeline, Recursive CTE, Materialized View

Домашнє завдання з PostgreSQL на датасеті Olist Brazilian E-Commerce. Ноутбук виконується в Google Colab, а SQL запускається через SQLAlchemy.

## Джерело даних

| | |
|---|---|
| Датасет | **Brazilian E-Commerce Public Dataset by Olist** |
| Стабільне посилання | <https://www.kaggle.com/datasets/olistbr/brazilian-ecommerce> |
| Kaggle handle | `olistbr/brazilian-ecommerce` |
| Ліцензія | CC BY-NC-SA 4.0 (див. сторінку датасету на Kaggle) |
| Період даних | замовлення 2016–2018 |

Використовуються 5 CSV-файлів:

| Raw-таблиця | CSV-файл |
|---|---|
| `olist_customers_raw` | `olist_customers_dataset.csv` |
| `olist_orders_raw` | `olist_orders_dataset.csv` |
| `olist_order_items_raw` | `olist_order_items_dataset.csv` |
| `olist_products_raw` | `olist_products_dataset.csv` |
| `olist_sellers_raw` | `olist_sellers_dataset.csv` |

> У CSV колонки `product_name_lenght` і `product_description_lenght` написані з помилкою.
> Raw-таблиці зберігають оригінальні назви, а typed-таблиця `olist_products` використовує
> нормалізовані `product_name_length` і `product_description_length`.

## Отримання файлів

**`kagglehub` (використовується в ноутбуці)**

```python
import kagglehub
dataset_dir = kagglehub.dataset_download('olistbr/brazilian-ecommerce')
print(dataset_dir)   # шлях до папки з CSV
```

Для публічного датасету автентифікація зазвичай не потрібна. Якщо Kaggle її вимагає, задайте
`KAGGLE_USERNAME` і `KAGGLE_KEY` (Kaggle → Settings → API → Create New Token).

## Вимоги

- PostgreSQL **14+**, бо `CYCLE`-clause у recursive CTE з'явився в 14 версії; решта працює з 11+
- Python 3.10+, `sqlalchemy`, `psycopg2-binary`, `pandas`, `kagglehub`

## Запуск

1. Клонуйте репозиторій або завантажте `.ipynb` файл.
2. Запустіть notebook у Google Colab. Для Colab завантажте файл через **File → Upload notebook**.
5. Виконуйте клітинки **послідовно зверху вниз** (Run all). Порядок важливий: typed-таблиці створюються після raw.

## Структура завдань

| Завдання | Що робиться | Ключові об'єкти |
|---|---|---|
| 2.1 DML | 4 ідемпотентні DML з `RETURNING`: `UPDATE ... IS DISTINCT FROM`, upsert `ON CONFLICT DO UPDATE`, `ON CONFLICT DO NOTHING`, data-modifying CTE `DELETE ... RETURNING → INSERT` | `olist_orders.is_late`, `seller_score`, `seller_alerts`, `seller_alerts_archive` |
| 2.2 CTE pipeline | Feature engineering per customer, 12 named stages | RFM, `cohort_quarter`, `avg_order_value_30d`, `cancellation_rate`, `preferred_category` |
| 3 Recursive CTE | Синтетичне referral-дерево, `depth` і `path`, захист від циклів | `hw5_referrals`, `hw5_referral_tree` |
| 4 Materialized View | Повний pipeline у MV, `UNIQUE INDEX`, `REFRESH ... CONCURRENTLY` | `mv_customer_features` |
| Reflection | Аналіз `EXPLAIN ANALYZE`, вибір refresh-cycle, trade-offs | — |

## CTE pipeline: stages

Клієнт ідентифікується за `customer_unique_id`. В Olist `customer_id` створюється заново для кожного
замовлення, тому з ним `frequency` завжди дорівнював би 1. `monetary`, rolling AOV і category revenue
рахуються лише по реалізованих замовленнях, тобто без статусів `canceled` і `unavailable`.

| Stage | CTE | Відповідальність |
|:---:|---|---|
| 0 | `reference_date` | Фіксована `as_of_date = MAX(order_purchase_timestamp)::date + 1` з даних, щоб результат був відтворюваним |
| 1 | `stage1_orders` | Grain «одне замовлення»: прив'язка до `customer_unique_id`, прапорці статусу |
| 2 | `stage2_items_total` | Сума позицій по замовленню (price + freight) |
| 3 | `stage3_orders_total` | Приєднання `order_total` до замовлень; замовлення без items отримують 0 |
| 4 | `stage4_rfm` | RFM: `recency_days`, `frequency`, `monetary` |
| 5 | `stage5_cohort` | `first_order_date` → `cohort_quarter` |
| 6 | `stage6_rolling_aov` | Rolling 30-day AVG по кожному замовленню (window function, `RANGE ... INTERVAL '30 days'`) |
| 7 | `stage7_latest_aov` | Значення rolling AOV на останнє замовлення клієнта |
| 8 | `stage8_cancel_rate` | `cancellation_rate` = частка скасованих серед усіх замовлень |
| 9 | `stage9_category_revenue` | Виручка в розрізі клієнт × категорія |
| 10 | `stage10_preferred_category` | Top-1 категорія з детермінованим tie-break за назвою |
| 11 | `stage11_customer_dim` | Вимір клієнта: один рядок на клієнта, штат з останнього замовлення |

Вихід містить один рядок на `customer_unique_id`, тобто 96 096 клієнтів.

## Відтворюваність

- `reference_date` береться з даних, а не з `CURRENT_DATE`, тому повторний запуск дає той самий результат.
- Усі DML безпечні для повторного запуску: повторне виконання змінює 0 рядків.
- Тестовий цикл A → B → A і демо-зміна статусу замовлення для перевірки refresh після запуску відкочуються окремими комірками.
