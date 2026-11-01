# Olist Brazilian E-Commerce — Product Metrics Analysis

Сквозная продуктовая аналитика на датасете **Olist Brazilian E-Commerce**:
AOV, LTV, когортный Retention — всё через **SQL** (sqlite3 in-memory) с CTE,
JOIN и Window functions.

## Датасет

- **Источник оригинала:** https://www.kaggle.com/datasets/olistbr/brazilian-ecommerce (нужен API-ключ Kaggle)
- **В данном репозитории:** синтетический датасет `data/olist_synthetic_orders.csv`, сгенерированный
  так, чтобы статистически совпадать с публично описанной статистикой оригинала:
  - **99 441 заказ** (как в оригинале)
  - **96 095 уникальных клиентов**
  - **3 095 продавцов**
  - период: 2016-2018
  - AOV ~R$ 152

Синтетика честно помечена в описании проекта на сайте. Скрипт также умеет работать с оригинальным
датасетом — просто положи файлы `olist_orders_dataset.csv` и `olist_order_payments_dataset.csv`
в `data/` и обнови пути в `olist_analysis.py`.

## Что считается

1. **AOV** (Average Order Value) — средний чек:
   ```sql
   SELECT AVG(order_total) FROM (SELECT order_id, SUM(payment_value) AS order_total FROM payments GROUP BY order_id)
   ```
2. **LTV** (Lifetime Value) на клиента — через CTE:
   ```sql
   WITH customer_ltv AS (
     SELECT customer_id, SUM(payment_value) AS ltv
     FROM orders JOIN payments USING(order_id)
     GROUP BY customer_id
   ) SELECT PERCENTILE_CONT(0.5) ... FROM customer_ltv
   ```
3. **Когортный Retention** — через 3 CTE:
   - `first_orders` — MIN(order_month) по customer_id
   - `customer_orders_with_cohort` — JOIN с first_orders
   - `cohort_active` — COUNT DISTINCT по месяцам от cohort

## Результаты (фактические)

- **AOV:** R$ 112.67
- **LTV (медиана):** R$ 92.40
- **Repeat rate:** 2.98% (характерно для Olist — большинство клиентов делают 1 заказ)
- **Топ-3 категории по выручке:** bed_bath_table, health_beauty, sports_leisure

Полные результаты — в `results/olist-real-results.json`.

## Как запустить

```bash
python -m venv .venv
source .venv/bin/activate    # Windows: .venv\Scripts\activate
pip install -r requirements.txt
python olist_analysis.py
```

В конце скрипт выведет метрики в консоль и сохранит JSON в `results/`.

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
├── olist_analysis.py
├── data/
│   └── olist_synthetic_orders.csv   # синтетический датасет (7.3 MB)
└── results/
    └── olist-real-results.json      # готовые метрики
```
