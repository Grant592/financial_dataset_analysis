# Querying PKDD99 Financial Dataset

## Query 1
Showing each individual account with total of credits and debits
```sql
SELECT 
   ID1, CREDIT, DEBIT, CREDIT-DEBIT net   
FROM  
  (SELECT 
     account_id ID1, 
     SUM(amount) CREDIT 
   FROM 
      trans  
   WHERE 
      type LIKE 'PRIJEM' 
   GROUP BY ID1) a 
LEFT OUTER JOIN 
   (SELECT
      account_id ID2, 
      SUM(amount) DEBIT 
    FROM 
      trans  
    WHERE 
      type LIKE 'VYDAJ' 
GROUP BY ID2) b  
ON a.ID1 = b.ID2;
```
## Query 2
Add column showing status of any loans taken out. Queried loan database to check no more than 1 loan per account_id
```sql
SELECT 
   a.*, 
   CASE WHEN loan_status IS NULL THEN 'E' ELSE loan_status END loan_status  
FROM 
   (SELECT 
       * 
    FROM 
       account_in_out) a 
LEFT OUTER JOIN 
   (SELECT 
       account_id ID2, 
       status loan_status 
    FROM 
       loan) b 
ON a.ID1 = b.ID2;
```
 ## Query 3 - Combines and replaces Query 1 and 2
```sql
CREATE VIEW account_in_out AS 
SELECT 
   ID1, 
   CREDIT, 
   DEBIT, 
   CREDIT-DEBIT net, 
   CASE WHEN loan_status is NULL THEN  'E'ELSE loan_status END loan_status 
FROM  
   (SELECT 
       account_id ID1, 
       SUM(amount) CREDIT 
   FROM 
       trans  
   WHERE 
       type LIKE 'PRIJEM' 
   GROUP BY ID1) a 
LEFT OUTER JOIN 
   (SELECT 
       account_id ID2, 
       SUM(amount) DEBIT 
   FROM 
       trans  
   WHERE 
       type LIKE 'VYDAJ' 
   GROUP BY ID2) b  
ON 
   a.ID1 = b.ID2 
LEFT OUTER JOIN 
   (SELECT 
       account_id ID3, 
       status loan_status 
   FROM 
       loan 
   GROUP BY 
       ID3) c 
ON a.ID1 = c.ID3; 
```

## Query 4 - 
Looking at relative proportion of each 'k_symbol' payment type
```sql
SELECT 
   account_id, 
   SUM(amount) OVER(PARTITION BY account_id) account_spend,  
   SUM(amount) OVER(partition by account_id, k_symbol) k_spend, 
   SUM(amount) OVER(PARTITION BY account_id, k_symbol)/SUM(amount) OVER(PARTITION BY account_id)  k_proportion,  
   CASE WHEN k_symbol LIKE '' THEN 'OTHER' ELSE k_symbol END transaction_type
FROM 
   `order`; 
```


## Query 5
Looking at loan category by district - are some districts more likely to defualt
```sql
SELECT 
   l.account_id, 
   COUNT(*) count_values,
   l.status, 
   a.district_id 
FROM 
   loan l, 
   account a 
WHERE 
   l.account_id = a.account_id 
GROUP BY 
   a.district_id, l.status 
ORDER BY count_values DESC; 
```

```sql
SELECT 
   l.account_id, 
   COUNT(*),
   l.status, 
   l.amount,
   l.duration,
   l.payments,
   a.district_id, 
   d.A4 pop, 
   d.A11 salary, 
   d.A12 unemp_95, 
   d.A13 unemp_96, 
   d.A15/d.A4 crime_rate_95, 
   d.A16/d.A4 crime_rate_96 
FROM 
   loan l, 
   account a, 
   district d 
WHERE 
   l.account_id = a.account_id 
AND 
   a.district_id = d.district_id 
GROUP BY 
   a.district_id, 
   l.status 
ORDER BY 
   status, COUNT(*); 
```


## Query 6
Looking at transaction table. Can we assess the amount spent from each account on the various k_symbols. As a porportion of total expenditure. Average count of transaction type per month e.g. cash withdrawal, credit card withdrawal etc.
```sql
-- Simple groupby for account, operation and k_symbol
SELECT account_id, date, amount, operation, k_symbol, 
SUM(amount) from trans 
GROUP BY account_id, operation, k_symbol; 
```

```sql
-- Average number of transactions by type per account
 SELECT account_id, operation, AVG(num) FROM 
(SELECT account_id, EXTRACT(YEAR FROM date) Y, EXTRACT(MONTH FROM date) M,  COUNT(*) num, SUM(amo
unt), operation from trans 
GROUP BY account_id, Y, M, operation) sub 
GROUP BY account_id, operation;
```

```sql
-- Count of sanction interest if negative balance
SELECT 
   account_id, 
   COUNT(*) sanction_count, 
   SUM(amount) sanction_amount 
FROM 
   trans 
WHERE 
   k_symbol 
LIKE 
   'SANKC. UROK' 
GROUP BY 
   account_id;
```


## Query 7 - Querying details of individuals and linking to account details
```sql
SELECT 
   c.client_id client_id, 
   d.account_id, 
   d.type, 
   c.gender gender, 
   c.birth_date birthdate, 
   c.district_id district_id 
FROM 
   client c 
INNER JOIN 
   disp d 
ON 
   c.client_id = d.client_id; 
```
Adding age at loan 
```sql
SELECT 
   c.client_id client_id, 
   d.account_id, 
   d.type, 
   c.gender gender, 
   c.birth_date birthdate, 
   c.district_id district_id, 
   l.loan_id, 
   l.date loan_date, 
   EXTRACT(YEAR from l.date) - EXTRACT(YEAR from c.birth_date) age_at_loan 
FROM 
   client c 
INNER JOIN 
   disp d 
ON 
   c.client_id = d.client_id 
INNER JOIN 
   loan l 
ON 
   d.account_id = l.account_id 
WHERE type = 'OWNER';      
```

Combining above queries with account information from view account_in_out
```sql
SELECT 
   a.*, 
   e.* 
FROM 
   account_in_out a 
INNER JOIN 
   (SELECT 
      c.client_id client_id, 
      d.account_id, 
      d.type, 
      c.gender gender, 
      c.birth_date birthdate, 
      c.district_id district_id, 
      l.loan_id, 
      l.date loan_date, 
      EXTRACT(YEAR from l.date) - EXTRACT(YEAR from c.birth_date) age_at_loan 
      CASE 
         WHEN 
            ca.type LIKE 'junior' THEN 0 
         WHEN 
            ca.type LIKE 'classic' THEN 1 
         WHEN ca.type LIKE 'gold' THEN 2 
         ELSE 
            3 
         END 
            card_type
   FROM 
      client c 
   INNER JOIN 
      disp d 
   ON 
      c.client_id = d.client_id 
   INNER JOIN 
      loan l 
   ON 
      d.account_id = l.account_id 
   INNER JOIN
      card ca
   ON
      ca.disp_id = d.disp_id
   WHERE 
      d.type = 'OWNER') e 
ON a.ID1 = e.account_id;                    
```

## Query 8
Combine district data with individual data for accounts granted a loan
```sql
 SELECT  
   l.account_id,  
   l.status loan_status,  
   l.amount loan_amount, 
   l.duration loan_duration, 
   l.payments loan_payments, 
   a.district_id,  
   d.A4 population,  
   d.A11 distrist_salary,  
   d.A12 unemp_95,  
   d.A13 unemp_96,  
   d.A15/d.A4 crime_rate_95,  
   d.A16/d.A4 crime_rate_96  
FROM  
   loan l,  
   account a,  
   district d  
WHERE  
   l.account_id = a.account_id  
AND  
   a.district_id = d.district_id  
```

## Query 9
Transaction amount in months prior to loan being granted
```sql
SELECT 
   t.account_id, 
   t.date trans_date, 
   SUM(t.amount * CASE WHEN t.type LIKE 'PRIJEM' THEN 1 WHEN t.type LIKE 'VYDAJ' OR t.type LIKE 'VYBER' THEN -1 END), 
   l.date loan_date, 
   DATE_SUB(l.date, INTERVAL 1 MONTH) loan_lag 
FROM 
   trans t, 
   loan l 
WHERE 
   t.account_id = l.account_id 
AND 
   t.date BETWEEN DATE_SUB(l.date, INTERVAL 1 MONTH) 
AND 
   l.date 
GROUP BY 
   t.account_id; 
```












