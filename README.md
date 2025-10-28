# AWS Stock Trend Pipeline — Near Real-Time Analytics (Kinesis → Lambda → DynamoDB/S3 → Athena → SNS)

![Architecture](screenshots/final%20architechture.png)

A near real-time, event-driven stock analytics pipeline on AWS. It ingests live prices (AAPL via `yfinance`) into **Kinesis**, processes records with **Lambda**, persists **processed** data in **DynamoDB**, archives **raw** JSON in **S3**, runs **SQL** analytics with **Athena/Glue**, and sends **trend alerts** via **SNS**. Designed to be **serverless**, **low-cost**, and easy to extend.

> ⚠️ This is “near real-time” (≈30s producer delay + Kinesis batch size) for cost control.

---

## ✨ Highlights
- **Streaming ingestion**: Kinesis Data Streams
- **Serverless processing**: Lambda (Kinesis trigger)
- **Low-latency store**: DynamoDB (`symbol` + `timestamp`)
- **Cold storage + analytics**: S3 (raw JSON) + Glue + Athena
- **Alerts**: SNS (email/SMS) using SMA(5)/SMA(20) crossover
- **Cost awareness**: on-demand stream, serverless, minimal retention

---

## 🧱 Architecture & Services

- **Kinesis Data Streams** – `stock-market-stream` for live events  
  ![Kinesis](screenshots/kinesis%20data%20stream%20console.png)

- **Lambda (Processor)** – decodes Kinesis, saves raw → **S3**, computes metrics, writes processed → **DynamoDB**  
  ![Lambda Code/Config 1](screenshots/lambda%20function%201.png)
  ![Lambda Code/Config 2](screenshots/lambda%20function%202.png)
  ![Lambda Code/Config 3](screenshots/lamda%20function%203%20-code.png)

- **DynamoDB** – table `stock-market-data` (PK: `symbol` [S], SK: `timestamp` [S])  
  ![DynamoDB Items](screenshots/Dynamodb%20items%20in%20the%20stock%20table.png)

- **S3 (raw)** – `raw-data/<symbol>/<timestamp>.json` for Athena/ML later  
  ![S3 Raw](screenshots/s3%20raw%20data%20.png)

- **Glue + Athena** – external table on S3 JSON to query history  
  ![Athena 1](screenshots/athena%20queries%201.png)
  ![Athena 2](screenshots/athena%20queries%202.png)

- **SNS** – topic `Stock_Trend_Alerts` for BUY/SELL crossovers  
  ![SNS](screenshots/sns.png)

- **Lambda (Trend Analysis)** – reads latest window from DynamoDB, computes SMA5/SMA20, publishes to SNS  
  ![Trend Lambda 1](screenshots/lambda%20stock%20trend%20analysis.png)
  ![Trend Lambda 2](screenshots/lambda%20stock%20trend%20analysis%202.png)
  ![Trend Lambda 3](screenshots/lambda%20stock%20trend%20analysis%203.png)

---
# 📊 Athena Query Examples & Troubleshooting Notes

This companion file contains the advanced SQL queries and real-world debugging notes from the **AWS Stock Trend Pipeline** project.

---

## ⚙️ Amazon Athena — Example Queries

### 🔹 1. Top 5 Stocks with the Highest Price Change
```sql
SELECT symbol, price, previous_close,
       (price - previous_close) AS price_change
FROM stock_data_table
ORDER BY price_change DESC
LIMIT 5;
🔹 2. Average Trading Volume per Stock
SELECT symbol, AVG(volume) AS avg_volume
FROM stock_data_table
GROUP BY symbol;
🔹 3. Identify Anomalous Stocks (Price Change > 5%)
SELECT symbol, price, previous_close,
       ROUND(((price - previous_close) / previous_close) * 100, 2) AS change_percent
FROM stock_data_table
WHERE ABS(((price - previous_close) / previous_close) * 100) > 5;

🧩 Troubleshooting Notes

This section highlights specific problems encountered during implementation — and the exact fixes applied.
These are the moments that built deep AWS fluency.

🪛 1. pip Not Recognized on Windows

Issue: PowerShell didn’t recognize pip after installing Python.
Fix:

Disabled App Execution Aliases for python.exe.

Reinstalled Python with “Add to PATH” enabled.

Confirmed via:
python --version
pip --version

🪛 2. Script Failing to Run (can't open file ...)

Issue: Running from the wrong working directory.
Fix:
Navigated to the correct project folder:
cd Downloads
python stream_stock_data.py

🪛 3. Kinesis Stream Validation Error

Issue: ValidationException ... Value 'stock-market-stream ' (extra space).
Fix: Removed trailing space in:
STREAM_NAME = "stock-market-stream"

🪛 4. Region Mismatch

Issue: ResourceNotFoundException ... Stream not found
Fix: Ensured region alignment:
kinesis_client = boto3.client("kinesis", region_name="us-east-2")
🪛 5. Lambda Not Writing to DynamoDB or S3

Issue: Hidden Unicode characters (U+200B) broke parsing.
Fix: Re-typed suspect lines in AWS console editor and redeployed.

🪛 6. S3 NoSuchBucket Error

Issue: Bucket name typo or region mismatch.
Fix: Used exact bucket name from AWS Console and same region as Lambda:
S3_BUCKET = "stock-market-data-bucket-33454-slade"

🪛 7. Empty CloudWatch Metrics

Issue: Data appeared missing right after deployment.
Fix: Waited 2–5 minutes for metrics to propagate and verified records directly in DynamoDB and S3 instead.

🎯 Key Takeaways

Small errors break big systems: even a hidden space or wrong region will stall an entire data flow.

Logs over guessing: CloudWatch Logs are the fastest way to debug AWS pipelines.

Automate sanity checks: always validate resources (names, regions, roles) before deploying Lambda.

Keep cost in mind: using Kinesis On-Demand + 30s delay achieves real-time feel with near-zero spend.

Practice > perfection: each issue reinforced hands-on fluency with IAM, Lambda permissions, and Python AWS SDK behavior.





