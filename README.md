# Olist Brazilian E-Commerce — Product Metrics Analysis

Сквозная продуктовая аналитика на **реальном** датасете [Olist Brazilian E-Commerce](https://www.kaggle.com/datasets/olistbr/brazilian-ecommerce):
AOV, LTV, когортный Retention — всё через **SQL** (sqlite3 in-memory) с CTE,
JOIN и оконными функциями. Плюс EDA-ноутбук с графиками на тех же данных.

## Данные

Датасет — 99 441 заказ и 96 095 клиентов Olist (маркетплейс, Бразилия,
сентябрь 2016 — октябрь 2018). CSV в git **не хранятся** — скачиваются одной командой:

```bash
pip install kagglehub
python download_data.py    # ~50 МБ, БЕЗ Kaggle-аккаунта и API-ключа (публичный датасет)
python olist_analysis.py   # сам обнаружит реальные файлы и пересчитает метрики
```

Альтернатива: скачать файлы вручную со [страницы датасета](https://www.kaggle.com/datasets/olistbr/brazilian-ecommerce)
и положить `olist_orders_dataset.csv`, `olist_order_payments_dataset.csv`,
`olist_customers_dataset.csv` (+ опционально `olist_order_items_dataset.csv`,
`olist_products_dataset.csv`) в папку `data/`.

Скрипт определяет режим автоматически: если реальных файлов в `data/` нет —
генерируется синтетика (`numpy`, seed=42), чтобы CI и запуск «из коробки» работали
без скачивания. Группировка метрик — по `customer_unique_id` (настоящий уникальный
клиент Olist; `customer_id` уникален для каждого заказа).

## Что считается

1. **AOV** (Average Order Value) — средний чек через SQL:
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

## Результаты (реальный датасет, пересчитаны на месте)

| Метрика | Значение |
|---|---|
| Заказов (с оплатой) | 99 440 |
| Уникальных клиентов | 96 095 |
| Выручка | R$ 16 008 872 |
| **AOV (средний чек)** | **R$ 160.99** |
| Медианный чек | R$ 105.29 |
| LTV средний / медиана | R$ 166.59 / **R$ 108.00** |
| Repeat rate (>1 заказа) | 3.12% (2 997 клиентов) |
| Период | 2016-09 → 2018-10 |
| Топ-3 категории по выручке | beleza_saude, relogios_presentes, cama_mesa_banho |

> Repeat rate ~3% — характеристика Olist: подавляющее большинство клиентов делает
> ровно один заказ, поэтому медианный LTV (R$ 108) ниже среднего (R$ 166.59).
> Средний чек выше медианного — распределение стоимости заказа лог-нормальное.

Полные результаты — в `results/olist-real-results.json` (флаг `is_synthetic: false`).

## Ноутбук с графиками

[`olist_eda.ipynb`](olist_eda.ipynb) — EDA на тех же данных: помесячная выручка
(рост 10+ раз за 2 года), топ-10 категорий, время доставки (в среднем 12.1 дней,
медиана 10; 8.1% доставленных заказов приехали позже обещанного), распределение
отзывов (средняя 4.09 из 5, 11.5% единиц) + демо SQL-запроса с CTE.
Перед запуском — `python download_data.py`.

## Как запустить

```bash
python -m venv .venv
source .venv/bin/activate    # Windows: .venv\Scripts\activate
pip install -r requirements.txt kagglehub
python download_data.py      # реальные данные (рекомендуется)
python olist_analysis.py
```

Скрипт пишет результаты в `results/olist-real-results.json` (путь относительный).

## Стек

- Python 3.10+
- pandas
- sqlite3 (стандартная библиотека Python)
- numpy (только для fallback-синтетики)
- kagglehub (загрузка датасета без API-ключа)

## Структура

```
olist-analysis/
├── README.md
├── requirements.txt
├── LICENSE
├── .gitignore                     # data/*.csv не коммитится
├── olist_analysis.py              # SQL-анализ (AOV, LTV, retention)
├── download_data.py               # загрузка реального Olist через kagglehub
├── olist_eda.ipynb                # EDA-ноутбук с графиками
├── .github/workflows/ci.yml       # CI: синтетический режим (без скачивания)
├── data/                          # CSV-файлы (не в git, см. download_data.py)
└── results/
    └── olist-real-results.json    # перегенерируется запуском
```

## Что я бы сделал дальше

- RFM-сегментация клиентов поверх LTV
- Связь опозданий доставки с оценкой отзыва (гипотеза: опоздание → 1-звёздочный отзыв)
- Сезонность AOV по месяцам и дням недели
- Прогноз месячной выручки (тренд + сезонность)
