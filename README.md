# Customer Profile & Financial Risk Analysis
*Mini-Project #2 — SQL, Window Functions & Data Integrity (Banking)*

## Business Context

Ikaw ang Data Analyst ng isang bangko. Bago ang **monthly Risk Committee meeting**, hiniling sa'yo ng **Head of Retail Banking**:

> *"Gusto kong makita ang buong profile ng bawat customer namin — magkano ang balance nila ngayon, paano tumaas o bumaba ang balance nila sa paglipas ng panahon, ilang loans meron sila, at kung may senyales ba ng financial trouble. Gusto kong malaman kung sino ang dapat bigyan ng priority para sa retention, at sino naman ang dapat bantayan."*

## Business Questions

Bago pumunta sa SQL, kinailangan munang linawin ang pinaka-malabong term sa request: **"financial trouble."**

**Mga posibleng signs na na-consider:**
- `loans.status` = Overdue/Default — malinaw na sign
- Mababang balance sa `accounts` — parang walang "cushion"
- Madalas na withdrawal kaysa deposit sa `transactions` — parang "umuubos" ang pera

**Bakit hindi "overdue lang"?** Isang customer na may overdue loan pero may ₱2,000,000 na balance ay hindi kasing-urgent ng isang customer na may overdue loan at ₱20,000 lang ang balance. Mas pipiliin natin yung may overdue na mababa ang balance, kesa sa may overdue pero may mataas na balance.

**Final na depinisyon:** *Financial Trouble* = may overdue loan **AT** nasa **bottom 20%** (NTILE quintile 1) ng total balance compare sa lahat ng customers.

**Mga tanong na sinagot:**
1. Magkano ang current total balance ng bawat customer? — `SUM()`, table `accounts`
2. Paano tumaas o bumaba ang balance nila sa paglipas ng panahon? — Running total, table `transactions`
3. Ilang loans meron sila? — `COUNT()`, table `loans`
4. May senyales ba ng financial trouble? — Kombinasyon ng `loans` at `accounts`

## Data Exploration

| Tanong | Table(s) | Columns |
|---|---|---|
| Total Balance | `accounts` | customer_id, balance |
| Balance Trend | `transactions` | account_id, transaction_date, transaction_type, amount |
| Loan Count | `loans` | customer_id |
| Financial Trouble | `loans` + `accounts` | status, balance |

## SQL Query

```sql
WITH customer_balance AS (
    SELECT
        customer_id,
        SUM(balance) AS total_balance
    FROM accounts
    GROUP BY customer_id
),
transaction_running AS (
    SELECT
        account_id,
        transaction_date,
        SUM(
            CASE WHEN transaction_type = 'Withdrawal' THEN -amount ELSE amount END
        ) OVER (PARTITION BY account_id ORDER BY transaction_date) AS running_total,
        ROW_NUMBER() OVER (PARTITION BY account_id ORDER BY transaction_date ASC) AS rn_asc,
        ROW_NUMBER() OVER (PARTITION BY account_id ORDER BY transaction_date DESC) AS rn_desc
    FROM transactions
),
account_trend AS (
    SELECT
        account_id,
        MAX(CASE WHEN rn_asc = 1 THEN running_total END) AS first_balance,
        MAX(CASE WHEN rn_desc = 1 THEN running_total END) AS last_balance
    FROM transaction_running
    GROUP BY account_id
),
customer_trend AS (
    SELECT
        a.customer_id,
        CASE 
            WHEN SUM(last_balance) > SUM(first_balance) THEN 'Increased' 
            WHEN SUM(last_balance) < SUM(first_balance) THEN 'Decreased'
            WHEN SUM(last_balance) = SUM(first_balance) THEN 'No Change'
            ELSE 'No Transaction History'
        END AS trends
    FROM accounts a
    LEFT JOIN account_trend ct ON ct.account_id = a.account_id
    GROUP BY a.customer_id
),
customer_loan_count AS (
    SELECT
        customer_id,
        COUNT(*) AS loan_count,
        MAX(CASE WHEN status = 'Overdue' THEN 'Yes' ELSE 'No' END) AS loan_status
    FROM loans
    GROUP BY customer_id
),
customer_financial_trouble AS (
    SELECT
        customer_id,
        NTILE(5) OVER (ORDER BY total_balance ASC) AS balance_rank
    FROM customer_balance
)
SELECT
    c.customer_id,
    c.customer_name,
    cb.total_balance,
    clc.loan_count AS total_loans,
    tr.trends,
    CASE
        WHEN cft.balance_rank = 1 AND clc.loan_status = 'Yes' THEN 'Yes'
        ELSE 'No'
    END AS financial_trouble_flag
FROM customers c
LEFT JOIN customer_balance cb ON c.customer_id = cb.customer_id
LEFT JOIN customer_loan_count clc ON cb.customer_id = clc.customer_id
LEFT JOIN customer_financial_trouble cft ON clc.customer_id = cft.customer_id
LEFT JOIN customer_trend tr ON cft.customer_id = tr.customer_id;
```

## Mga Desisyon Sa Likod Ng Query

- **`SUM()` ginamit, hindi `MAX()`/`MIN()`, para sa trend**, kasi gusto malaman ang total balance ng customer (lahat ng accounts niya sabay-sabay), hindi isang account lang.
- **`LEFT JOIN` ginamit, hindi regular JOIN**, importante ito. Nadicover habang gumagawa na si Carla Santos ay may account na walang recorded transactions. Kung regular JOIN lang, mawawala siya sa buong report nang walang error o alert. Kaya ginamit ang `LEFT JOIN`, simula sa `accounts` table (dahil kumpleto ito, lahat ng customers naka-list doon).
- **`NTILE(5)` walang `PARTITION BY`** — dahil ang layunin ay i-rank ang lahat ng customers laban sa isa't isa, hindi bawat customer nang hiwalay.

## Insight

Si Carla Santos ang dapat bigyan ng pinaka-mataas na atensyon at bantayan. Siya ang may pinakamababang total balance (₱20,000, kabilang sa bottom 20% ng lahat ng customers) at may overdue Auto Loan. Wala rin siyang transaction history, na nagpapahiwatig na bago pa lamang siya o hindi gaanong aktibo ang kanyang account. Ang combination ng mababa ang balance, may overdue loan, at walang transaction activity ay may mataas na sign ng financial trouble

## Recommendation

Kailangan nating i-reach out agad si Carla Santos para alamin ang dahilan ng overdue payment, at alukin siya ng flexible o customized na payment plan. Ito ay dahil siya ang may pinakamababang balance (₱20,000) sa lahat ng customers, may overdue na Auto Loan, at walang recorded transaction activity, kombinasyon na nagpapakita ng malakas na financial risk. Sa pamamagitan ng maagang pakikipag-ugnayan, maiiwasan nating tuluyang ma-default ang loan, at mas maprotektahan ang relasyon ng bangko sa customer.

---

## Skills na Ginamit

- **SQL:** Multiple CTEs, window functions (`SUM() OVER`, `ROW_NUMBER()`, `NTILE()`), conditional aggregation (`MAX(CASE WHEN...)`)
- **Data Integrity:** Natukoy at naayos ang isang silent data-loss bug (INNER JOIN vs LEFT JOIN) na hindi nagpapakita ng error message
- **Business Thinking:** Pag-define ng malabong termino ("financial trouble") gamit ang kumbinasyon ng mga signal, hindi isang sukatan lang
- **Business Communication:** "Answer First" na recommendation format
