# RFM Analysis
I performed RFM analysis using the dataset below. When calculating recency, I used the latest order date, not today's date.

The dataset is taken from this link: https://www.kaggle.com/datasets/carrie1/ecommerce-data

### RECENCY:
Last purchase date for each customer:

```sql
SELECT 
    customer_id,
    MAX(invoicedate)
FROM rfm
WHERE customer_id IS NOT NULL
GROUP BY 1;
```

#### (Current_date) - (max_invoice_date):

```sql
WITH day_difference AS (
    SELECT 
        customer_id,
        MAX(invoicedate) AS max_invoice_date
    FROM rfm
    GROUP BY 1
)
SELECT 
    customer_id,
    max_invoice_date,
    current_date - max_invoice_date::date
FROM day_difference;
```

#### To get the last invoice date instead of current_date:

```sql
WITH last_invoice_date AS (
    SELECT 
        customer_id,
        MAX(invoicedate)::date AS max_invoice_date
    FROM rfm
    GROUP BY 1
)
SELECT 
    customer_id,
    max_invoice_date,
    '2011-12-09' - max_invoice_date AS recency
FROM last_invoice_date;
```

### FREQUENCY:
For each customer's number of orders to date:

```sql
SELECT 
    customer_id,
    COUNT(DISTINCT invoiceno)
FROM rfm
WHERE invoiceno NOT LIKE 'C%' 
    AND customer_id IS NOT NULL
GROUP BY 1;
```

### MONETARY:
For total spending by customers:

```sql
SELECT 
    customer_id,
    SUM(ROUND((quantity * unitprice)::integer, 0)) AS total_spent
FROM rfm
WHERE invoiceno NOT LIKE 'C%' 
    AND customer_id IS NOT NULL
GROUP BY 1;
```

To merge all three tables:
```sql
WITH recency AS (
    WITH tablo AS (
        SELECT 
            customer_id,
            MAX(invoicedate)::date AS max_invoice_date
        FROM rfm
        WHERE invoiceno NOT LIKE 'C%' 
            AND customer_id IS NOT NULL
        GROUP BY 1
    )
    SELECT 
        customer_id,
        max_invoice_date,
        '2011-12-09' - max_invoice_date AS recency
    FROM tablo
),
frequency AS (
    SELECT 
        customer_id,
        COUNT(DISTINCT invoiceno) AS frequency
    FROM rfm
    WHERE invoiceno NOT LIKE 'C%' 
        AND customer_id IS NOT NULL
    GROUP BY 1
),
monetary AS (
    SELECT 
        customer_id,
        SUM(ROUND((quantity * unitprice)::integer, 0)) AS monetary
    FROM rfm
    WHERE invoiceno NOT LIKE 'C%' 
        AND customer_id IS NOT NULL
    GROUP BY 1
)
SELECT  
    r.customer_id,
    recency,
    frequency,
    monetary
FROM recency r
JOIN frequency f ON r.customer_id = f.customer_id
JOIN monetary m ON r.customer_id = m.customer_id;
```
#### SQL OUTPUT
![alt text](https://github.com/hilalguleryuz/RFM_analysis/blob/main/rfm_screenchots/rfm_1.png)

### FREQUENCY REVIEW:
How many people shopped with a certain frequency?

```sql
WITH tablo AS (
    SELECT 
        customer_id,
        COUNT(DISTINCT invoiceno) AS frequency
    FROM rfm
    WHERE invoiceno NOT LIKE 'C%' 
        AND customer_id IS NOT NULL
    GROUP BY 1
)
SELECT 
    frequency,
    COUNT(1)
FROM tablo
GROUP BY 1
ORDER BY 1;
```
#### SQL OUTPUT
![alt text](https://github.com/hilalguleryuz/RFM_analysis/blob/main/rfm_screenchots/rfm_2.png)
![alt text](https://github.com/hilalguleryuz/RFM_analysis/blob/main/rfm_screenchots/rfm_3.png)
![alt text](https://github.com/hilalguleryuz/RFM_analysis/blob/main/rfm_screenchots/rfm_4.png)

As can be understood from the table and graph above, the number of people who shopped only once is 1494, which makes up 34% of the total number. Therefore, I gave a frequency_score of 1 to those who shopped once. Again, since the number of people who shopped twice was quite high, I gave them a score of 2. Since the percentage of people who ordered 3 and 4 times was close to the percentage of people who ordered twice, I gave them a score of 3. As a result, as can be understood from the excel table above, I grouped the shopping frequencies so that their percentages were close to each other and gave each of them a score between 1-5 accordingly. I did this grouping with CASE WHEN in the query below.

### Customer segment with Ntile & Case when:

```sql
WITH recency AS (
    WITH tablo AS (
        SELECT 
            customer_id,
            MAX(invoicedate)::date AS max_invoice_date
        FROM rfm
        WHERE invoiceno NOT LIKE 'C%' 
            AND customer_id IS NOT NULL
        GROUP BY 1
    )
    SELECT 
        customer_id,
        max_invoice_date,
        '2011-12-09' - max_invoice_date AS recency
    FROM tablo
),
frequency AS (
    SELECT 
        customer_id,
        COUNT(DISTINCT invoiceno) AS frequency
    FROM rfm
    WHERE invoiceno NOT LIKE 'C%' 
        AND customer_id IS NOT NULL
    GROUP BY 1
),
monetary AS (
    SELECT 
        customer_id,
        SUM(ROUND((quantity * unitprice)::integer, 0)) AS monetary
    FROM rfm
    WHERE invoiceno NOT LIKE 'C%' 
        AND customer_id IS NOT NULL
    GROUP BY 1
)
SELECT  
    r.customer_id,
    recency,
    NTILE(5) OVER (ORDER BY recency DESC) AS recency_score,
    frequency,
    CASE 
        WHEN frequency = 1 THEN 1
        WHEN frequency = 2 THEN 2
        WHEN frequency = 3 OR frequency = 4 THEN 3
        WHEN frequency BETWEEN 5 AND 13 THEN 4
        ELSE 5 
    END AS frequency_score,
    monetary,
    NTILE(5) OVER (ORDER BY monetary) AS monetary_score
FROM recency r
JOIN frequency f ON r.customer_id = f.customer_id
JOIN monetary m ON r.customer_id = m.customer_id;
```

### Final stage:

```sql
WITH recency AS (
    WITH tablo AS (
        SELECT 
            customer_id,
            MAX(invoicedate)::date AS max_invoice_date
        FROM rfm
        WHERE invoiceno NOT LIKE 'C%' 
            AND customer_id IS NOT NULL
        GROUP BY 1
    )
    SELECT 
        customer_id,
        max_invoice_date,
        '2011-12-09' - max_invoice_date AS recency
    FROM tablo
),
frequency AS (
    SELECT 
        customer_id,
        COUNT(DISTINCT invoiceno) AS frequency
    FROM rfm
    WHERE invoiceno NOT LIKE 'C%' 
        AND customer_id IS NOT NULL
    GROUP BY 1
),
monetary AS (
    SELECT 
        customer_id,
        SUM(ROUND((quantity * unitprice)::integer, 0)) AS monetary
    FROM rfm
    WHERE invoiceno NOT LIKE 'C%' 
        AND customer_id IS NOT NULL
    GROUP BY 1
),
scores AS (
    SELECT 
        r.customer_id,
        recency,
        NTILE(5) OVER (ORDER BY recency DESC) AS recency_score,
        frequency,
        CASE 
            WHEN frequency = 1 THEN 1
            WHEN frequency = 2 THEN 2
            WHEN frequency = 3 OR frequency = 4 THEN 3
            WHEN frequency BETWEEN 5 AND 13 THEN 4
            ELSE 5 
        END AS frequency_score,
        monetary,
        NTILE(5) OVER (ORDER BY monetary) AS monetary_score
    FROM recency r
    JOIN frequency f ON r.customer_id = f.customer_id
    JOIN monetary m ON r.customer_id = m.customer_id
)
SELECT 
    customer_id,
    recency_score || '-' || frequency_score || '-' || monetary_score AS rfm_Score
FROM scores;
```

#### SQL OUTPUT
![alt text](https://github.com/hilalguleryuz/RFM_analysis/blob/main/rfm_screenchots/rfm_5.png)





