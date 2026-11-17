# Olist Brazilian E-Commerce — Product Metrics Analysis

Сквозная продуктовая аналитика на датасете **Olist Brazilian E-Commerce**:
AOV, LTV, когортный Retention — всё через **SQL** (sqlite3 in-memory) с CTE,
JOIN и оконными функциями.

## Датасет

Скрипт **автоматически определяет**, какие данные использовать:

1. **Реальные данные** (если в `data/` лежат файлы с Kaggle):
   - `olist_orders_dataset.csv`, `olist_order_payments_dataset.csv`, `olist_customers_dataset.csv`
   - (опционально `olist_order_items_dataset.csv`, `olist_products_dataset.csv` — для категорий)
   - Группировка по `customer_unique_id` — настоящему уникальному клиенту Olist
     (в Olist `customer_id` уникален для каждого заказа).

2. **Синтетические данные** (по умолчанию, если реальных файлов нет):
   - Генерируются через `numpy` с `seed=42` (воспроизводимо)
   - Совпадают по характеристикам с публичной статистикой оригинала:
     99 441 заказ, 96 095 клиентов, AOV ~R$ 152

### Как получить реальные данные (бесплатно)

```bash
pip install kaggle
# Положите kaggle.json (kaggle.com → My Account → Create New API Token) в ~/.kaggle/
kaggle datasets download -d olistbr/brazilian-ecommerce -p data --unzip
python olist_analysis.py   # автоматически подхватит реальные файлы
```

## Что считается

1. **AOV** (Average Order Value) — средний чек:
   ```sql
   SELECT AVG(order_total) FROM (
     SELECT order_id, SUM(payment_value) AS order_total FROM payments GROUP BY order_id
   )
   ```
2. **LTV** (Lifetime Value) на клиента — через CTE:
   ```sql
   WITH customer_revenue AS (
     SELECT customer_id, COUNT(*) AS orders_count, SUM(order_total) AS total_revenue
     FROM orders GROUP BY customer_id
   )
   SELECT AVG(total_revenue), ... FROM customer_revenue
   ```
3. **Когортный Retention** — через 4 CTE:
   - `first_orders` — `MIN(order_month)` по `customer_id` (когорта клиента)
   - `customer_orders_with_cohort` — JOIN заказов с когортой
   - `cohort_sizes` — `COUNT DISTINCT` новых клиентов в месяц
   - `cohort_active` — `COUNT DISTINCT` активных клиентов по месяцам и когортам

## Результаты (синтетический режим, seed=42)

| Метрика | Значение |
|---|---|
| Заказов | 99 441 |
| Уникальных клиентов | 96 095 |
| AOV (средний чек) | R$ 112.67 |
| LTV (медиана на клиента) | R$ 92.40 |
| Repeat rate (>1 заказа) | 2.98% |
| Период | 2016–2018 |
| Топ-3 категории по выручке | cama_mesa_banho, beleza_saude, esporte_lazer |

> Repeat rate ~3% — характеристика Olist: подавляющее большинство клиентов делает
> ровно один заказ. Медианный LTV < AOV именно потому, что большинство клиентов
> сделали один заказ на сумму ниже среднего.

Полные результаты — в `results/olist-real-results.json`. При переключении на реальные
данные метрики пересчитаются автоматически и флаг `is_synthetic` в JSON станет `false`.

## Как запустить

```bash
python -m venv .venv
source .venv/bin/activate    # Windows: .venv\Scripts\activate
pip install -r requirements.txt
python olist_analysis.py
```

Скрипт пишет результаты в `results/olist-real-results.json` (путь относительный).

## Стек

- Python 3.10+
- pandas
- sqlite3 (стандартная библиотека Python)
- numpy (для генерации синтетики)

## Структура

```
olist-analysis/
├── README.md
├── requirements.txt
├── LICENSE
├── .gitignore
├── olist_analysis.py
├── .github/workflows/ci.yml
├── data/
│   ├── olist_synthetic_orders.csv   # генерируется при первом запуске (если нет реальных)
│   └── olist_*_dataset.csv          # положите сюда реальные файлы с Kaggle
└── results/
    └── olist-real-results.json      # перегенерируется запуском
```

## Что я бы сделал дальше

- Перейти на реальные данные (одна команда `kaggle datasets download …`)
- Добавить RFM-сегментацию клиентов поверх LTV
- Сезонность AOV по месяцам и дням недели
- Анализ времени доставки (order_delivered_customer_date − order_purchase_timestamp)
