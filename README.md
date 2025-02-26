# Web Server Log Analysis with Apache Hive

## **Project Overview**
This project focuses on analyzing **web server logs using Apache Hive**, enabling structured querying and extracting valuable insights from website traffic data. The project aims to:

- **Count total web requests**
- **Analyze HTTP status codes**
- **Identify most visited pages**
- **Analyze user agents (browsers) used**
- **Detect suspicious IP addresses** (frequent failed requests)
- **Observe traffic trends over time**
- **Optimize performance using partitioning**
- **Export results for further analysis**

The web server log dataset is in CSV format and is processed using Hive, storing the results in **HDFS** before retrieving them to the local machine.

---

## **Implementation Approach**
### **1. Creating a Hive Table for Log Data**

Since `timestamp` is a **reserved keyword in Hive**, backticks (`) were used to escape it.

```sql
CREATE EXTERNAL TABLE IF NOT EXISTS web_server_logs (
    ip STRING,
    `timestamp` STRING,
    url STRING,
    status INT,
    user_agent STRING
)
ROW FORMAT DELIMITED
FIELDS TERMINATED BY ','
STORED AS TEXTFILE
TBLPROPERTIES ("skip.header.line.count"="1");
```

### **2. Loading Data into Hive from HDFS**
The web log file is first uploaded to **HDFS**:

```bash
docker cp web_server_logs.csv namenode:/web_server_logs.csv
docker exec -it namenode /bin/bash
hdfs dfs -mkdir -p /data
hdfs dfs -put /web_server_logs.csv /data/
```

Then, load the data into **Hive**:

```sql
LOAD DATA INPATH '/data/web_server_logs.csv' INTO TABLE web_server_logs;
```

### **3. Executing Hive Queries for Log Analysis**
#### 1️⃣ **Total Web Requests**
```sql
SELECT COUNT(*) AS total_requests FROM web_server_logs;
```

#### 2️⃣ **Status Code Analysis**
```sql
SELECT status, COUNT(*) AS count
FROM web_server_logs
GROUP BY status
ORDER BY count DESC;
```

#### 3️⃣ **Most Visited Pages**
```sql
CREATE TABLE most_visited_temp AS
SELECT url, COUNT(*) AS visit_count
FROM web_server_logs
GROUP BY url;

INSERT OVERWRITE DIRECTORY '/data/output_most_visited'
SELECT url, visit_count
FROM most_visited_temp
ORDER BY visit_count DESC
LIMIT 3;
```

#### 4️⃣ **Traffic Source Analysis (User Agents)**
```sql
CREATE TABLE traffic_sources_temp AS
SELECT user_agent, COUNT(*) AS request_count
FROM web_server_logs
GROUP BY user_agent;

INSERT OVERWRITE DIRECTORY '/data/output_traffic_sources'
SELECT user_agent, request_count
FROM traffic_sources_temp
ORDER BY request_count DESC
LIMIT 3;
```

#### 5️⃣ **Suspicious IP Detection**
```sql
SELECT ip, COUNT(*) AS failed_requests
FROM web_server_logs
WHERE status IN (404, 500)
GROUP BY ip
HAVING COUNT(*) > 3;
```

#### 6️⃣ **Traffic Trend Over Time**
```sql
SELECT SUBSTR(`timestamp`, 1, 16) AS time, COUNT(*) AS request_count
FROM web_server_logs
GROUP BY SUBSTR(`timestamp`, 1, 16)
ORDER BY time;
```

#### 7️⃣ **Optimizing Queries with Partitioning**
```sql
CREATE TABLE IF NOT EXISTS web_server_logs_partitioned (
    ip STRING,
    `timestamp` STRING,
    url STRING,
    user_agent STRING
)
PARTITIONED BY (status INT)
ROW FORMAT DELIMITED
FIELDS TERMINATED BY ','
STORED AS TEXTFILE;

SET hive.exec.dynamic.partition = true;
SET hive.exec.dynamic.partition.mode = nonstrict;

INSERT INTO TABLE web_server_logs_partitioned PARTITION(status)
SELECT ip, `timestamp`, url, user_agent, status FROM web_server_logs;
```

---

## **Execution Steps**
### **1️⃣ Start Hive Environment**
```bash
docker-compose up -d
docker exec -it hive-server /bin/bash
hive
```

### **2️⃣ Upload Data to HDFS**
```bash
docker cp web_server_logs.csv namenode:/web_server_logs.csv
docker exec -it namenode /bin/bash
hdfs dfs -mkdir -p /data
hdfs dfs -put /web_server_logs.csv /data/
```

### **3️⃣ Run Hive Queries**
- Execute the **queries inside Hive CLI** to generate insights.

### **4️⃣ Retrieve Results from HDFS**
```bash
hdfs dfs -get /data/output_* /root/
```

### **5️⃣ Copy Results from Namenode to Local Machine**
```bash
docker cp namenode:/root/output_total_requests.txt ./
docker cp namenode:/root/output_status_codes.txt ./
docker cp namenode:/root/output_most_visited.txt ./
docker cp namenode:/root/output_traffic_sources.txt ./
docker cp namenode:/root/output_suspicious_ips.txt ./
docker cp namenode:/root/output_traffic_trends.txt ./
```

---

## **Challenges Faced & Solutions**
### **1. Hive Partitioning Issues**
**Problem:** Dynamic partitioning was not enabled.
**Solution:**
```sql
SET hive.exec.dynamic.partition = true;
SET hive.exec.dynamic.partition.mode = nonstrict;
```

### **2. Aggregate Function Errors in `INSERT OVERWRITE DIRECTORY`**
**Problem:** Hive **does not support `COUNT(*)` inside `INSERT OVERWRITE DIRECTORY`**.
**Solution:** Store results in a **temporary table first**.

### **3. Copying Files from HDFS to Local Machine**
**Problem:** Hive stores results in `000000_0` inside directories.
**Solution:** Move and rename files before copying:
```bash
mv /root/output_most_visited/000000_0 /root/output_most_visited.txt
```

---

## **Sample Input & Expected Output**
### **Input Sample (web_server_logs.csv)**
```
ip,timestamp,url,status,user_agent
192.168.1.1,2024-02-01 10:15:00,/home,200,Mozilla/5.0
192.168.1.2,2024-02-01 10:16:00,/products,200,Chrome/90.0
192.168.1.3,2024-02-01 10:17:00,/checkout,404,Safari/13.1
```

### **Expected Output Example**
```
Total Requests: 100

Status Code Analysis:
200: 80
404: 10
500: 10

Most Visited Pages:
/home: 50
/products: 30
/checkout: 20

Suspicious IPs:
192.168.1.10: 5 failed requests
192.168.1.15: 4 failed requests
```

---

## **Conclusion**
This project successfully implements **Apache Hive for log analysis**, enabling efficient **data querying, partitioning for performance, and structured reporting**. The approach ensures **scalability and optimized query execution for large-scale logs**.

