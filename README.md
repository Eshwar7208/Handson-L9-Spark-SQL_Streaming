# Ride Sharing Analytics Using Spark Streaming and Spark SQL

A comprehensive hands-on assignment for learning Apache Spark Structured Streaming and real-time data processing. This project implements a real-time analytics pipeline for a ride-sharing platform, processing streaming data to compute driver metrics and time-windowed aggregations.

---

## **Table of Contents**
- [Overview](#overview)
- [Prerequisites](#prerequisites)
- [Project Structure](#project-structure)
- [Setup Instructions](#setup-instructions)
- [Architecture](#architecture)
- [Tasks Overview](#tasks-overview)
- [Detailed Task Instructions](#detailed-task-instructions)
- [Running the Pipeline](#running-the-pipeline)
- [Output Verification](#output-verification)
- [Key Concepts](#key-concepts)
- [Troubleshooting](#troubleshooting)
- [Submission Checklist](#submission-checklist)

---

## **Overview**

This assignment teaches you how to build a real-time analytics pipeline for a ride-sharing platform using **Apache Spark Structured Streaming**. You will:

- Ingest and parse streaming JSON data from a socket source
- Perform real-time aggregations on driver earnings and trip statistics
- Implement sliding time-window analytics to track trends over time
- Write results to CSV files for persistent storage

**Real-time data processing** is crucial for modern applications that need to respond quickly to incoming data streams. This project demonstrates core concepts used in production systems like Uber, Lyft, and DoorDash.

---

## **Prerequisites**

Before starting, ensure you have the following software installed and properly configured:

### **1. Python 3.7+**
   - [Download and Install Python](https://www.python.org/downloads/)
   - Verify installation:
     ```bash
     python3 --version
     ```
   - Ensure `pip` is installed:
     ```bash
     pip --version
     ```

### **2. Apache Spark / PySpark**
   - Install via pip:
     ```bash
     pip install pyspark==3.3.0
     ```
   - Verify installation:
     ```bash
     python -c "import pyspark; print(pyspark.__version__)"
     ```
   - **Note**: This project requires Java 8 or later. PySpark will attempt to find Java automatically.

### **3. Faker Library**
   - Install for generating test data:
     ```bash
     pip install faker==8.10.0
     ```
   - Verify installation:
     ```bash
     python -c "from faker import Faker; print('Faker installed successfully')"
     ```

### **4. Java Runtime Environment (JRE)**
   - Spark requires Java. Download from [Java SE Downloads](https://www.oracle.com/java/technologies/javase-downloads.html)
   - Verify Java installation:
     ```bash
     java -version
     ```

---

## **Project Structure**

```
Handson-L9-Spark-SQL_Streaming/
│
├── data_generator.py              # Generates streaming ride data on localhost:9999
├── task1.py                       # Task 1: Stream ingestion and parsing
├── task2.py                       # Task 2: Driver-level aggregations
├── task3.py                       # Task 3: Windowed time-based analytics
│
├── outputs/
│   ├── task_1/                    # CSV output files from Task 1
│   │   └── part-00000-*.csv       # Parsed streaming data
│   ├── task_2/
│   │   └── batch_0/               # Driver aggregation results
│   │       └── part-00000-*.csv   # Driver metrics by batch
│   ├── task_3/
│   │   └── batch_0/               # Windowed aggregation results
│   │       └── part-00000-*.csv   # Time-windowed fare sums
│   └── checkpoints_task1/         # Spark checkpoints for fault tolerance
│       ├── offsets/               # Stream offset tracking
│       └── commits/               # Commit metadata
│
├── README.md                       # This file - complete documentation
└── .gitignore                      # Git ignore file
```

### **File Descriptions**

- **data_generator.py**: Continuously generates synthetic ride-sharing data in JSON format, streaming it to a socket server on `localhost:9999`
- **task1.py**: Reads streaming data, parses JSON, and writes to CSV
- **task2.py**: Computes real-time driver metrics (total fare, average distance)
- **task3.py**: Implements 5-minute sliding windows with 1-minute slide and 1-minute watermark
- **outputs/**: Directory containing all output CSV files organized by task
- **README.md**: Complete project documentation and instructions

---

## **Setup Instructions**

### **Step 1: Install Dependencies**

```bash
# Clone or download the project
cd Handson-L9-Spark-SQL_Streaming

# Install required Python packages
pip install -r requirements.txt
# OR install manually
pip install pyspark==3.3.0 faker==8.10.0
```

### **Step 2: Verify Directory Structure**

Ensure the `outputs/` directory exists:

```bash
# Create outputs directory if it doesn't exist
mkdir -p outputs/task_1
mkdir -p outputs/task_2/batch_0
mkdir -p outputs/task_3/batch_0
```

### **Step 3: Understand the Data Flow**

```
data_generator.py (Producer)
        ↓
   localhost:9999 (Socket Stream)
        ↓
   task1.py (Consumer: Parse & Store)
        ↓
   CSV Files (Persistent Storage)
        ↓
   task2.py & task3.py (Aggregations)
```

---

## **Architecture**

### **System Design**

The pipeline uses a **producer-consumer** model:

1. **Producer** (`data_generator.py`): Continuously generates ride data and streams to a local socket
2. **Consumer** (`task1.py`, `task2.py`, `task3.py`): Read from the stream, process in real-time, and output results

### **Data Schema**

Each ride record contains:
```json
{
  "trip_id": "uuid-string",
  "driver_id": "integer (1-100)",
  "distance_km": "float (0.5-50)",
  "fare_amount": "float (5-500)",
  "timestamp": "ISO 8601 string (e.g., 2024-01-15T10:30:45.123Z)"
}
```

### **Stream Processing Architecture**

```
Raw Stream Data
    ↓
[Spark Structured Streaming]
    ↓
[Parsing & Schema Application]
    ↓
[Transformations & Aggregations]
    ├─→ Task 1: Passthrough to CSV
    ├─→ Task 2: Group by driver_id → Aggregations
    └─→ Task 3: Window by time → Windowed Aggregations
    ↓
[Output Sinks (CSV)]
```

---

## **Tasks Overview**

| Task | Objective | Input | Output | Key Concepts |
|------|-----------|-------|--------|--------------|
| **Task 1** | Ingest and parse streaming data | Socket (localhost:9999) | Parsed records (CSV) | Stream ingestion, JSON parsing, schema |
| **Task 2** | Real-time driver-level metrics | Parsed data (Task 1) | Driver stats (CSV) | Aggregations, groupBy, stateful operations |
| **Task 3** | Time-windowed analytics | Parsed data with timestamps | Windowed sums (CSV) | Tumbling windows, watermarking, event time |

---

## **Detailed Task Instructions**

### **Task 1: Basic Streaming Ingestion and Parsing**

**Objective**: Ingest streaming data from a socket and parse JSON into a structured DataFrame.

**What You'll Learn**:
- Creating a Spark session for streaming
- Reading from a socket source with Spark Structured Streaming
- Parsing JSON with schema inference or explicit schema definition
- Writing streaming data to console and CSV

**Instructions:**

1. **Create a Spark Session** with streaming configuration:
   ```python
   from pyspark.sql import SparkSession
   
   spark = SparkSession.builder \
       .appName("Task1_Streaming") \
       .config("spark.sql.streaming.checkpointLocation", "outputs/checkpoints_task1") \
       .getOrCreate()
   ```

2. **Read from Socket**:
   ```python
   df = spark.readStream \
       .format("socket") \
       .option("host", "localhost") \
       .option("port", 9999) \
       .load()
   ```

3. **Parse JSON Payload** into structured columns:
   ```python
   from pyspark.sql.functions import from_json, col, schema_of_json
   from pyspark.sql.types import *
   
   schema = StructType([
       StructField("trip_id", StringType()),
       StructField("driver_id", IntegerType()),
       StructField("distance_km", DoubleType()),
       StructField("fare_amount", DoubleType()),
       StructField("timestamp", StringType())
   ])
   
   parsed_df = df.select(
       from_json(col("value"), schema).alias("data")
   ).select("data.*")
   ```

4. **Write to Console** (for testing):
   ```python
   query = parsed_df.writeStream \
       .format("console") \
       .option("truncate", False) \
       .start()
   query.awaitTermination()
   ```

5. **Write to CSV** (for persistence):
   ```python
   query = parsed_df.writeStream \
       .format("csv") \
       .option("path", "outputs/task_1") \
       .option("checkpointLocation", "outputs/checkpoints_task1") \
       .start()
   ```

**Expected Output**: CSV file(s) in `outputs/task_1/` with columns: `trip_id`, `driver_id`, `distance_km`, `fare_amount`, `timestamp`

---

### **Task 2: Real-Time Aggregations (Driver-Level)**

**Objective**: Aggregate stream data in real-time to calculate driver metrics.

**What You'll Learn**:
- Grouping streaming data with `groupBy()`
- Computing aggregations (SUM, AVG) on streams
- Handling stateful operations in Spark Structured Streaming
- Writing aggregation results to CSV

**Questions to Answer**:
- What is the total fare earned by each driver?
- What is the average distance driven by each driver?

**Instructions:**

1. **Start with Task 1 data** (parsed DataFrame):
   ```python
   # Reuse parsed_df from Task 1
   ```

2. **Group by driver_id** and compute aggregations:
   ```python
   from pyspark.sql.functions import sum, avg
   
   aggregated_df = parsed_df.groupBy("driver_id").agg(
       sum("fare_amount").alias("total_fare"),
       avg("distance_km").alias("avg_distance")
   )
   ```

3. **Write Results** to CSV:
   ```python
   query = aggregated_df.writeStream \
       .format("csv") \
       .option("path", "outputs/task_2/batch_0") \
       .option("checkpointLocation", "outputs/checkpoints_task2") \
       .start()
   ```

**Expected Output**: CSV in `outputs/task_2/batch_0/` with columns: `driver_id`, `total_fare`, `avg_distance`

**Example Result**:
```
driver_id,total_fare,avg_distance
1,1250.50,15.3
2,980.75,12.8
3,2100.25,18.5
```

---

### **Task 3: Windowed Time-Based Analytics**

**Objective**: Perform time-windowed aggregations to analyze trends over time.

**What You'll Learn**:
- Converting string timestamps to TimestampType
- Using Spark's `window()` function for time-based grouping
- Implementing sliding windows
- Watermarking for late data handling

**Specifications**:
- **Window Duration**: 5 minutes
- **Slide Duration**: 1 minute (sliding window)
- **Watermark**: 1 minute (tolerance for late-arriving data)

**Instructions:**

1. **Convert Timestamp** to proper TimestampType:
   ```python
   from pyspark.sql.functions import to_timestamp
   
   parsed_df = parsed_df.withColumn(
       "event_time",
       to_timestamp("timestamp", "yyyy-MM-dd'T'HH:mm:ss.SSS'Z'")
   )
   ```

2. **Apply Watermark** to handle late data:
   ```python
   windowed_df = parsed_df.withWatermark("event_time", "1 minute")
   ```

3. **Apply Window Function** and aggregate:
   ```python
   from pyspark.sql.functions import window, sum
   
   windowed_aggregation = windowed_df.groupBy(
       window("event_time", "5 minutes", "1 minute")
   ).agg(
       sum("fare_amount").alias("total_fare_in_window")
   )
   ```

4. **Write to CSV**:
   ```python
   query = windowed_aggregation.writeStream \
       .format("csv") \
       .option("path", "outputs/task_3/batch_0") \
       .option("checkpointLocation", "outputs/checkpoints_task3") \
       .start()
   ```

**Expected Output**: CSV in `outputs/task_3/batch_0/` with columns: `window`, `total_fare_in_window`

**Example Result**:
```
window,total_fare_in_window
[2024-01-15T10:00:00Z, 2024-01-15T10:05:00Z],5250.75
[2024-01-15T10:01:00Z, 2024-01-15T10:06:00Z],4980.50
```

---

## **Running the Pipeline**

### **Complete Workflow**

Follow these steps to run the entire pipeline:

#### **Terminal 1: Start the Data Generator**
```bash
cd /path/to/project
python data_generator.py
```

**Expected Output**:
```
Generated ride data: {"trip_id": "uuid", "driver_id": 45, ...}
Streaming to localhost:9999...
Sending data every 0.5 seconds...
```

#### **Terminal 2: Run Task 1 (Parsing)**
```bash
python task1.py
```

**Expected Output**:
```
Spark Session started
Reading from socket: localhost:9999
-------------------------------------------
Batch: 0
-------------------------------------------
+-------+--------+----------+---------------+-------------------+
|trip_id|driver_id|distance_km|fare_amount|timestamp          |
+-------+--------+----------+---------------+-------------------+
|uuid-1 |45      |15.5      |125.50        |2024-01-15T10:30:00|
...
```

#### **Terminal 3: Run Task 2 (Driver Aggregations)**
```bash
python task2.py
```

#### **Terminal 4: Run Task 3 (Windowed Analytics)**
```bash
python task3.py
```

**Keep all terminals running** to allow continuous data ingestion and processing.

### **Running Sequentially (for Testing)**

If you want to test one task at a time:

```bash
# Terminal 1
python data_generator.py

# Wait 10 seconds, then in Terminal 2
python task1.py

# Let Task 1 run for 20 seconds, then stop it (Ctrl+C)
# The data will be saved in outputs/task_1/

# In Terminal 2
python task2.py

# In Terminal 3
python task3.py
```

---

## **Output Verification**

### **Checking Task 1 Output**

```bash
# List all CSV files generated
ls -lh outputs/task_1/

# View the first few rows of a CSV file
head -10 outputs/task_1/part-00000*.csv
```

**Expected format**:
```
trip_id,driver_id,distance_km,fare_amount,timestamp
uuid-abc,42,12.3,98.50,2024-01-15T10:30:45.123Z
uuid-def,67,25.8,156.75,2024-01-15T10:30:46.456Z
```

### **Checking Task 2 Output**

```bash
ls -lh outputs/task_2/batch_0/
cat outputs/task_2/batch_0/part-00000*.csv
```

**Expected format**:
```
driver_id,total_fare,avg_distance
42,1250.50,15.3
67,980.75,18.5
```

### **Checking Task 3 Output**

```bash
ls -lh outputs/task_3/batch_0/
cat outputs/task_3/batch_0/part-00000*.csv
```

**Expected format**:
```
window,total_fare_in_window
[2024-01-15T10:00:00Z, 2024-01-15T10:05:00Z],5250.75
```

### **Automated Verification Script**

Create `verify.py`:
```python
import os
import pandas as pd

tasks = ['task_1', 'task_2', 'task_3']
for task in tasks:
    path = f'outputs/{task}/'
    if os.path.exists(path):
        csv_files = [f for f in os.listdir(path) if f.endswith('.csv')]
        if csv_files:
            print(f"\n✓ {task} output found: {csv_files[0]}")
            # Read and display sample
            df = pd.read_csv(f"{path}{csv_files[0]}")
            print(df.head(3))
        else:
            print(f"\n✗ {task}: No CSV files found")
    else:
        print(f"\n✗ {task}: Directory not found")
```

Run: `python verify.py`

---

## **Key Concepts**

### **1. Spark Structured Streaming**

Structured Streaming treats streaming data as an unbounded table that continuously grows:

```
Time 0: | record1 |
Time 1: | record1 | record2 |
Time 2: | record1 | record2 | record3 |
...
```

**Key APIs**:
- `readStream`: Read from streaming source
- `writeStream`: Write to streaming sink
- `groupBy()`: Stateful grouping
- `window()`: Time-based grouping

### **2. Event Time vs. Processing Time**

- **Event Time**: When the event actually occurred (in the data)
- **Processing Time**: When Spark processes the event

For windowing, always use event time for real-world accuracy.

### **3. Watermarking**

Watermarks allow Spark to handle late-arriving data gracefully:

```python
.withWatermark("event_time", "1 minute")
```

This means: "Drop data that arrives more than 1 minute late."

### **4. Checkpointing**

Spark saves progress to disk for fault tolerance:

```python
.option("checkpointLocation", "path/to/checkpoint")
```

Checkpoints store:
- Stream offsets (where we left off)
- Aggregation state
- Metadata for recovery

### **5. Output Modes**

- **Append**: Only new rows since last batch
- **Complete**: Entire result set (for aggregations)
- **Update**: Only changed rows

---

## **Troubleshooting**

### **Problem: "Address already in use" error**

```
Error: java.net.BindException: Address already in use
```

**Solution**:
```bash
# Find process using port 9999
lsof -i :9999

# Kill the process
kill -9 <PID>

# Or change port in data_generator.py and task*.py:
PORT = 9998  # Use different port
```

### **Problem: Java not found**

```
Error: Could not find java
```

**Solution**:
```bash
# Install Java
# On macOS: brew install java
# On Ubuntu: sudo apt-get install default-jdk
# On Windows: Download from Oracle

# Set JAVA_HOME environment variable
export JAVA_HOME=/path/to/java
```

### **Problem: "No such file or directory: outputs/"**

**Solution**:
```bash
mkdir -p outputs/{task_1,task_2/batch_0,task_3/batch_0}
mkdir -p outputs/checkpoints_task{1,2,3}
```

### **Problem: "Socket connection refused"**

```
Error: Connection refused
```

**Solution**:
1. Ensure `data_generator.py` is running: `python data_generator.py`
2. Check port is correct (default: 9999)
3. Run all scripts in the correct order (generator first!)

### **Problem: No output files created**

**Check**:
1. Is `data_generator.py` still running?
2. Are all three tasks running simultaneously?
3. Have you waited at least 20-30 seconds?

**Debug**:
```python
# Add to task1.py:
query.awaitTermination(timeout=60000)  # Wait 60 seconds
```

### **Problem: Out of Memory**

```
Error: Java heap space
```

**Solution**:
```bash
# Increase Spark memory
export SPARK_LOCAL_IP=127.0.0.1
export PYSPARK_SUBMIT_ARGS="--driver-memory 4g --executor-memory 4g pyspark-shell"
python task1.py
```

### **Problem: Slow Performance**

**Optimization Tips**:
- Reduce data generation rate in `data_generator.py`
- Use `coalesce()` to reduce partition count
- Monitor with Spark UI: http://localhost:4040

---

## **Performance Considerations**

### **Optimization Strategies**

1. **Batch Size**: Adjust in `data_generator.py`:
   ```python
   time.sleep(0.5)  # Reduce for faster generation
   ```

2. **Parallelism**: Spark defaults to number of cores; adjust with:
   ```python
   spark.sparkContext.defaultParallelism = 4
   ```

3. **Memory**: For large datasets:
   ```bash
   export PYSPARK_SUBMIT_ARGS="--driver-memory 8g pyspark-shell"
   ```

### **Monitoring**

Access Spark UI (real-time metrics):
- Open browser: `http://localhost:4040`
- View tasks, stages, storage, and executors

---

## **Submission Checklist**

Before submission, verify:

- [ ] All dependencies installed (`pip list | grep pyspark`)
- [ ] Output directories exist: `outputs/{task_1,task_2/batch_0,task_3/batch_0}`
- [ ] All Python scripts present: `task1.py`, `task2.py`, `task3.py`, `data_generator.py`
- [ ] All tasks run without errors (sequentially or parallel)
- [ ] CSV output files generated in respective `outputs/` folders
- [ ] README.md is complete and detailed
- [ ] Code is pushed to GitHub: `https://github.com/Eshwar7208/Handson-L9-Spark-SQL_Streaming`
- [ ] GitHub link submitted on Canvas

---

## **Next Steps & Extensions**

After completing this assignment, consider these enhancements:

1. **Add More Metrics**: Peak hours, surge pricing alerts, driver ratings
2. **Use External Sources**: Read from Kafka instead of socket
3. **Machine Learning**: Predict fare amounts using historical data
4. **Visualization**: Create dashboards with the output data
5. **Scaling**: Deploy to a Spark cluster (Standalone, YARN, or Kubernetes)

---

## **References**

- [Apache Spark Structured Streaming Documentation](https://spark.apache.org/docs/latest/structured-streaming-programming-guide.html)
- [PySpark API Documentation](https://spark.apache.org/docs/latest/api/python/)
- [Faker Documentation](https://faker.readthedocs.io/)
- [Spark SQL Window Functions](https://spark.apache.org/docs/latest/sql-ref-window-frame.html)

---

**Last Updated**: March 2024  
**Course**: Hands-On L9 - Spark SQL Streaming  
**Author**: Data Engineering Team

