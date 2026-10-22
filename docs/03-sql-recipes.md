# 03. SQL-рецепти / SQL recipes

Приклади написано в стандартному SQL і перевірені на схемі нашого [SQL-тренажера](https://antonemec.github.io/analyst-html-tools/sql-trainer.html) (SQLite). У PostgreSQL/BigQuery відрізняються лише функції дат.

## Схема / Schema

```sql
customers(customer_id, region, segment, age, income_uah, signup_date)
cards(card_id, customer_id, product, credit_limit, open_date, status)
transactions(txn_id, card_id, txn_date, amount, mcc, merchant_category, channel, status)
monthly_snapshots(card_id, month, balance, dpd, min_payment_due, paid_amount)
```

## 1. Утилізація та сегменти ризику

```sql
SELECT
  CASE WHEN s.balance*1.0/c.credit_limit > 0.9 THEN '90%+'
       WHEN s.balance*1.0/c.credit_limit > 0.5 THEN '50-90%'
       ELSE '<=50%' END AS util_bucket,
  COUNT(*) AS cards,
  ROUND(AVG(s.dpd > 0) * 100, 2) AS delinquent_pct
FROM monthly_snapshots s
JOIN cards c ON c.card_id = s.card_id
WHERE s.month = '2026-09-01' AND c.status = 'active'
GROUP BY util_bucket;
```

**Навіщо:** перевірити гіпотезу «висока утилізація → вища прострочка».

## 2. Кошики прострочки

```sql
SELECT
  CASE WHEN dpd = 0 THEN '0'
       WHEN dpd <= 30 THEN '1-30'
       WHEN dpd <= 60 THEN '31-60'
       WHEN dpd <= 90 THEN '61-90'
       ELSE '90+' END AS bucket,
  COUNT(*) AS cards,
  ROUND(SUM(balance), 2) AS balance
FROM monthly_snapshots
WHERE month = '2026-09-01'
GROUP BY bucket;
```

## 3. Матриця переходів (roll-rate matrix)

```sql
WITH b AS (
  SELECT card_id, month,
    CASE WHEN dpd = 0 THEN '0' WHEN dpd <= 30 THEN '1-30'
         WHEN dpd <= 60 THEN '31-60' WHEN dpd <= 90 THEN '61-90' ELSE '90+' END AS bucket
  FROM monthly_snapshots
)
SELECT a.bucket AS from_bucket, n.bucket AS to_bucket,
       COUNT(*) AS cards,
       ROUND(100.0 * COUNT(*) / SUM(COUNT(*)) OVER (PARTITION BY a.bucket), 2) AS pct_of_from
FROM b a
JOIN b n ON n.card_id = a.card_id AND n.month = '2026-06-01'
WHERE a.month = '2026-05-01'
GROUP BY a.bucket, n.bucket
ORDER BY a.bucket, n.bucket;
```

Результат — повна матриця переходів між двома місяцями; діагональ — «залишились у кошику».

## 4. Vintage-аналіз

```sql
SELECT substr(c.open_date, 1, 4) AS vintage_year,
       COUNT(*) AS cards,
       ROUND(100.0 * SUM(s.dpd > 30) / COUNT(*), 2) AS dpd30_pct
FROM monthly_snapshots s
JOIN cards c ON c.card_id = s.card_id
WHERE s.month = '2026-09-01'
GROUP BY vintage_year
ORDER BY vintage_year;
```

**Для коректного порівняння** вінтажів додайте вік картки в місяцях (`months_on_book`) і порівнюйте когорти на однаковому віці.

## 5. Transactor / Revolver

```sql
SELECT month,
  SUM(paid_amount >= balance AND balance > 0) AS transactors,
  SUM(paid_amount <  balance AND balance > 0 AND dpd = 0) AS revolvers,
  SUM(dpd > 0) AS delinquent
FROM monthly_snapshots
GROUP BY month
ORDER BY month;
```

## 6. Відтік: неактивні N місяців

```sql
WITH last_txn AS (
  SELECT card_id, MAX(txn_date) AS last_date
  FROM transactions WHERE status = 'approved' GROUP BY card_id
)
SELECT COUNT(*) AS dormant_cards
FROM cards c
LEFT JOIN last_txn l ON l.card_id = c.card_id
WHERE c.status = 'active'
  AND (l.last_date IS NULL OR l.last_date < '2026-07-01');
```

## 7. Ковзне середнє витрат

```sql
WITH m AS (
  SELECT substr(txn_date, 1, 7) AS month, SUM(amount) AS spend
  FROM transactions WHERE status = 'approved' GROUP BY 1
)
SELECT month, ROUND(spend, 2) AS spend,
       ROUND(AVG(spend) OVER (ORDER BY month ROWS BETWEEN 2 PRECEDING AND CURRENT ROW), 2) AS ma3
FROM m ORDER BY month;
```

## 8. Топ-N категорій витрат на клієнта

```sql
WITH s AS (
  SELECT c.customer_id, t.merchant_category, SUM(t.amount) AS spend
  FROM transactions t JOIN cards c ON c.card_id = t.card_id
  WHERE t.status = 'approved'
  GROUP BY 1, 2
), r AS (
  SELECT *, ROW_NUMBER() OVER (PARTITION BY customer_id ORDER BY spend DESC) AS rn FROM s
)
SELECT customer_id, merchant_category, ROUND(spend, 2) AS spend FROM r WHERE rn <= 3;
```

## Правила, які економлять години

1. Перед JOIN перевірте унікальність ключа: `SELECT key, COUNT(*) ... HAVING COUNT(*) > 1`.
2. Порівнюйте кількість рядків до та після JOIN.
3. Фільтрувати за датою, а не обертати колонку у функцію без потреби — інакше не спрацює індекс/партиція.
4. Використовуйте CTE для читабельності; називайте їх за змістом.
5. Для відсотків множте на `100.0`, а не на `100`, щоб уникнути цілочисельного ділення.
