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
- # ⚙️ Troubleshooting & Athena Query Reference

This companion document expands on the **AWS Stock Trend Pipeline** project.  
It includes the Athena SQL queries used for analytics, along with detailed notes on real-world troubleshooting and debugging throughout the build process.

---

## 🧠 Athena Query Library

Below are the most useful queries executed through **Amazon Athena** after creating the Glue Catalog table (`stock_data_table`) from the raw JSON files in S3.

---

### 🔹 1. Top 5 Stocks with the Highest Price Change
SELECT symbol, price, previous_close,
       (price - previous_close) AS price_change
FROM stock_data_table
ORDER BY price_change DESC
LIMIT 5;

---

### 🔹 2. Average Trading Volume Per Stock
SELECT symbol, AVG(volume) AS avg_volume
FROM stock_data_table
GROUP BY symbol;

---

### 🔹 3. Identify Anomalous Stocks ( Price Change > 5% )
SELECT symbol, price, previous_close,
       ROUND(((price - previous_close) / previous_close) * 100, 2) AS change_percent
FROM stock_data_table
WHERE ABS(((price - previous_close) / previous_close) * 100) > 5;

---

## 🧩 Troubleshooting Notes

This section documents the real debugging path during the project—what went wrong, why, and how it was fixed.

---

### 🪛 1. `pip` Not Recognized on Windows
**Problem:** PowerShell didn’t recognize `pip`.  
**Fix:** Disabled App Execution Aliases for `python.exe`, then reinstalled Python with “Add to PATH” enabled. Verified installation with:
python --version  
pip --version  

---

### 🪛 2. Script Failed to Run (`can't open file ...`)
**Problem:** Attempted to run the Python script from the wrong directory.  
**Fix:** Navigated to the correct project folder before execution:  
cd Downloads  
python stream_stock_data.py  

---

### 🪛 3. Kinesis Stream Validation Error
**Problem:** `ValidationException ... Value 'stock-market-stream '` — extra space at the end of the stream name.  
**Fix:** Removed the trailing space in the configuration file:  
STREAM_NAME = "stock-market-stream"  

---

### 🪛 4. Region Mismatch
**Problem:** `ResourceNotFoundException ... Stream not found` due to region mismatch.  
**Fix:** Ensured region alignment across services and code:  
kinesis_client = boto3.client("kinesis", region_name="us-east-2")  

---

### 🪛 5. Lambda Not Writing to DynamoDB or S3
**Problem:** Lambda function failed silently because of a hidden Unicode character (U+200B).  
**Fix:** Re-typed affected lines in the AWS Lambda editor and re-deployed. Verified via CloudWatch logs.

---

### 🪛 6. S3 `NoSuchBucket` Error
**Problem:** Incorrect bucket name or region in Lambda code.  
**Fix:** Used the exact bucket name and region from the AWS console:  
S3_BUCKET = "stock-market-data-bucket-33454-slade"  

---

### 🪛 7. Empty CloudWatch Metrics
**Problem:** No visible data in CloudWatch immediately after deployment.  
**Fix:** Waited 2–5 minutes for metric propagation and validated records in DynamoDB and S3 instead.

---

## 🎯 Key Takeaways

- **Tiny details matter:** Even a space or wrong region can break an entire pipeline.  
- **Logs first, guess later:** CloudWatch logs were the fastest way to pinpoint every failure.  
- **Consistency is king:** Match regions, resource names, and IAM roles across every AWS service.  
- **Cost-smart design:** Using Kinesis On-Demand with a 30-second interval achieved real-time performance on a budget.  
- **Repetition builds intuition:** Each fix reinforced hands-on AWS fluency with Lambda, IAM, and serverless event flows.

---

✅ **Next Steps:** Integrate visual analytics via Amazon QuickSight and experiment with OpenSearch for true real-time streaming dashboards.

