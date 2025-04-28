# Bank-Marketing-english
In this project, I used the Bank Marketing dataset to conduct an exploratory analysis. Using SQL and Power BI, I answered key questions to better understand the strengths, weaknesses, and overall performance of the bank’s marketing efforts.

## Content Table

- [Data](#data)
- [Analysis](#analysis)
- [Suggestions](#suggestions)
- [Power Bi](#power-bi)

## Data 

📂 [bank-full.csv](./bank-full.csv): Original dataset in CSV format. From [https://archive.ics.uci.edu/dataset/222/bank+marketing](https://archive.ics.uci.edu/dataset/222/bank+marketing)

### Variables from the dataset

Input variables:
   - bank client data:

   1 - age (numeric)
     
   2 - job : type of job (categorical: "admin.","unknown","unemployed","management","housemaid","entrepreneur","student",
                                       "blue-collar","self-employed","retired","technician","services") 
                                       
   3 - marital : marital status (categorical: "married","divorced","single"; note: "divorced" means divorced or widowed)
   
   4 - education (categorical: "unknown","secondary","primary","tertiary")
   
   5 - default: has credit in default? (binary: "yes","no")
   
   6 - balance: average yearly balance, in euros (numeric) 
   
   7 - housing: has housing loan? (binary: "yes","no")
   
   8 - loan: has personal loan? (binary: "yes","no")
   
   - related with the last contact of the current campaign:

   9 - contact: contact communication type (categorical: "unknown","telephone","cellular")
     
  10 - day: last contact day of the month (numeric)
  
  11 - month: last contact month of year (categorical: "jan", "feb", "mar", ..., "nov", "dec")
  
  12 - duration: last contact duration, in seconds (numeric)
  
   - other attributes:

  13 - campaign: number of contacts performed during this campaign and for this client (numeric, includes last contact)
     
  14 - pdays: number of days that passed by after the client was last contacted from a previous campaign (numeric, -1 means client was not previously contacted)
  
  15 - previous: number of contacts performed before this campaign and for this client (numeric)
  
  16 - poutcome: outcome of the previous marketing campaign (categorical: "unknown","other","failure","success")

  - Output variable (desired target):
  
  17 - deposit - has the client subscribed a term deposit? (binary: "yes","no")

## Tools

- SQL (for data analisis and organization)
- Power BI (for data visualization)

## Data preparation

In order to analyze the data y used PostgreSQL, where I created a new dataset and a new table.I changed the original column days default, days and month, because those are reserved words in SQL.

 ```sql
CREATE TABLE bank_data (
    age INT,
    job VARCHAR(50),
    marital VARCHAR(50),
    education VARCHAR(50),
    default_account VARCHAR(3),
    balance DECIMAL(10, 2),
    housing VARCHAR(3),
    loan VARCHAR(3),
    contact VARCHAR(50),
    last_contact_day INT,
    last_contact_month VARCHAR(3),
    duration INT,
    campaign INT,
    pdays INT,
    previous INT,
    poutcome VARCHAR(50),
    deposit VARCHAR(3)
);
```
## Exploratory Data Analysis

I aimed to answer business-relevant questions such as:

- What is the total Conversion rate?

- What is the conversion rate by age job?

- What is the conversion rate by age group?

- What is the conversion rate by balance group?

- What is the Conversion rate by educational level?

- Does previous succesful campaigns have an impact?

- Which groups should be prioritized for marketing campaigns?

## Analysis

- Total Conversion rate.

```sql

SELECT 
    COUNT(*) AS total_clients,
    SUM(CASE WHEN deposit = 'yes' THEN 1 ELSE 0 END) AS clients_with_deposit,
    ROUND(100.0 * SUM(CASE WHEN deposit = 'yes' THEN 1 ELSE 0 END) / COUNT(*), 2) AS conversion_rate
FROM bank_data;
```
- Conversion rate by job
```sql
SELECT
	job,
	COUNT (*) AS total_clients,
	SUM (CASE WHEN deposit = 'yes' THEN 1 ELSE 0 END) AS clients_with_deposit,
	ROUND(100.0 * SUM(CASE WHEN deposit = 'yes' THEN 1 ELSE 0 END) / COUNT(*), 2) AS conversion_rate
FROM bank_data
GROUP BY job
ORDER BY conversion_rate DESC;
```

- Conversion rate by age group. Here I used CTE, so that in the result of the query I can see the results in numeric order.

```sql

WITH AgeGroupData AS (
    SELECT 
        CASE 
            WHEN age < 30 THEN 'Under 30'
            WHEN age BETWEEN 30 AND 39 THEN '30s'
            WHEN age BETWEEN 40 AND 49 THEN '40s'
            WHEN age BETWEEN 50 AND 59 THEN '50s'
            ELSE '60 and above'
        END AS age_group,
        COUNT(*) AS total_clients,
        SUM(CASE WHEN deposit = 'yes' THEN 1 ELSE 0 END) AS clients_with_deposit,
        ROUND(100.0 * SUM(CASE WHEN deposit = 'yes' THEN 1 ELSE 0 END) / COUNT(*), 2) AS conversion_rate
    FROM bank_data
    GROUP BY age_group
)
SELECT * 
FROM AgeGroupData
ORDER BY 
    CASE 
        WHEN age_group = 'Under 30' THEN 1
        WHEN age_group = '30s' THEN 2
        WHEN age_group = '40s' THEN 3
        WHEN age_group = '50s' THEN 4
        ELSE 5
    END;
```
- Conversion rate by balance group (quartiles). In this query I also implement a CTE to define the quartiles, looking cleaner and more efficient.

```sql
WITH Quartiles AS (
    SELECT 
        PERCENTILE_CONT(0.25) WITHIN GROUP (ORDER BY balance) AS first_quartile,
        PERCENTILE_CONT(0.75) WITHIN GROUP (ORDER BY balance) AS third_quartile
    FROM bank_data
)
SELECT 
    CASE 
        WHEN balance < (SELECT first_quartile FROM Quartiles) THEN '0-25%'
        WHEN balance BETWEEN (SELECT first_quartile FROM Quartiles) AND (SELECT third_quartile FROM Quartiles) THEN '25-75%'
        ELSE '>75%'
    END AS Quartile,
    COUNT(*) AS total_clients,
    ROUND(AVG(balance), 2) AS avg_balance,
    ROUND(100.0 * SUM(CASE WHEN deposit = 'yes' THEN 1 ELSE 0 END) / COUNT(*), 2) AS conversion_rate
FROM bank_data
CROSS JOIN Quartiles
GROUP BY Quartile
ORDER BY avg_balance DESC;
```
- Conversion rate by educational level.

```sql
SELECT 
    education,
    COUNT(*) AS total_clients,
    SUM(CASE WHEN deposit = 'yes' THEN 1 ELSE 0 END) AS clients_with_deposit,
    ROUND(100.0 * SUM(CASE WHEN deposit = 'yes' THEN 1 ELSE 0 END) / COUNT(*), 2) AS conversion_rate
FROM bank_data
GROUP BY education
ORDER BY conversion_rate DESC;
```
- Impact of previous campaigns.

```sql
SELECT 
    poutcome,
    COUNT(*) AS total_clients,
    ROUND(100.0 * SUM(CASE WHEN deposit = 'yes' THEN 1 ELSE 0 END)/COUNT(*), 2) AS conversion_rate
FROM bank_data
GROUP BY poutcome
ORDER BY conversion_rate DESC;
```
- Which groups should be prioritized for marketing campaigns? To response this I group characteristics of the clients, such as job, marital status and education. With this I can see which are the profiles with most Conversion rate. In addition added a HAVING clause, so my data is not biased by non-representative profiles.

```sql
SELECT 
    job, marital, education,
    COUNT(*) AS total_clients,
    ROUND(100.0 * SUM(CASE WHEN deposit = 'yes' THEN 1 ELSE 0 END)/COUNT(*), 2) AS conversion_rate
FROM bank_data
GROUP BY job, marital, education
HAVING COUNT (*) > 300
ORDER BY conversion_rate DESC
LIMIT 10;
```
## Results

1. The Conversion rate is 11.70%. This mean that every 100 clients contacted approximately 12 suscribe a term deposit.
2.  Students and retired people are the client groups with more Conversion rate by job.
3. The age groups with higher Convertion rate are the under 30 (17.60 %) and 60 and above (33.63%)
4. The clients with more balance on their account have a higher Conversion rate. Top 25% of clients by balance have a Conversion rate of 16.15%.
5. Higher education level correlates with higer Conversion rate.
6. The previous campaign outcome is strongly correlated with the chance of suscribing a term deposit. The conversion rate for clients with a successful previous campaign is 64.73%.

## Suggestions
- Prioritize retargeting clients from successful past campaigns.
- Students and retirees emerge as the job groups with the highest conversion rates, indicating that marketing efforts should be specifically designed for these profiles.
- Focus on older clients (60+) and younger adults (<30). In line with the above.
- Target high-balance clients aggressively prioritize high-balance accounts in outreach campaigns. Consider offering them premium or personalized products.
- Clients with higher education levels tend to convert more.  Use education as a segmentation factor when creating marketing lists.

## Power Bi

This Power BI dashboard was created to analyze the Bank Marketing dataset with the goal of identifying the client segments most likely to subscribe to a term deposit.The dashboard provides an interactive visualization of the main conversion drivers, including:

- Conversion Rate by Job: Highlights the top-performing job categories like students and retirees.

- Conversion Rate by Education: Shows a positive correlation between higher education levels and subscription rates.

- Conversion Rate by Balance Group: Clients with higher account balances (>75th percentile) show the highest conversion rate (16.2%).

- Conversion Rate by Age: Younger clients (under 30) and older clients (over 60) show higher engagement.

- Impact of Previous Campaign Outcome: Clients with successful previous campaigns have a notably higher conversion rate (62.73%).

- Top Client Profiles: Lists client profiles with the highest total count and highest conversion rates based on job, marital status, and education.

🔹 General Results:

- Overall conversion rate: 11.70% (5,289 clients out of 45,000).

- Previous campaign success and client balance are among the strongest predictors of conversion.

- Younger clients, retirees, and students are priority segments for future marketing efforts.

The dashboard allows dynamic filtering by age group, balance group, contact method, and education level, offering a flexible exploration of the data and supporting data-driven marketing strategies.

- [📄 Dashboard](./Bank%20Marketing%20dashboard.pdf) : PDF document for visualization.
- [📊 Bank Marketing Dashboard.pbix](./Bank%20Marketing%20dashboard.pbix): Power Bi document for editing.
