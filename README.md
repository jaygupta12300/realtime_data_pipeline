# Real-Time Interaction Analytics Pipeline

## Overview
This project implements an end-to-end real-time data pipeline that simulates user interaction events, processes them using Kafka, performs real-time aggregations, stores results in MongoDB, and visualizes insights using a Streamlit dashboard.

The solution is designed to be scalable, modular, and suitable for high-throughput data streams.

---

## Architecture

Producer → Kafka → Consumer → MongoDB → Dashboard

1. **Producer** generates interaction events.
2. **Kafka** acts as the streaming backbone.
3. **Consumer** performs windowed aggregations.
4. **MongoDB** stores aggregated results.
5. **Streamlit Dashboard** visualizes metrics in near real-time.

---

## Components

### 1. Producer (`producer/producer.py`)
- Generates random interaction events.
- Fields:
  - `user_id`
  - `item_id`
  - `interaction_type`
  - `timestamp`
- Publishes messages to Kafka.
- Supports configurable data generation rate.
- Includes connection retry logic with exponential backoff.

---

### 2. Consumer (`consumer/consumer.py`)
- Consumes messages from Kafka.
- Performs real-time windowed aggregations using Spark Structured Streaming.
- Aggregates stored in MongoDB:
  - `user_interactions`
  - `item_interactions`
- Uses tumbling/sliding windows (configurable).
- Ensures continuous updates as new data arrives.
- Includes lag monitoring and progress logging.

---

### 3. Storage (MongoDB)

**Database:** `kpi_db`

**Collections:**

#### user_interactions
| Field | Type |
|-----|------|
| user_id | String |
| total_interactions | Int |
| window_start | ISODate |
| window_end | ISODate |

#### item_interactions
| Field | Type |
|------|------|
| item_id | String |
| total_interactions | Int |
| window_start | ISODate |
| window_end | ISODate |

Pre-aggregated schema ensures fast reads for dashboards.

---

### 4. Dashboard (`reporting/reporting.py`)
- Built using Streamlit.
- Auto-refresh enabled.
- Displays:
  - Average interactions per user
  - Maximum interactions per item
  - Minimum interactions per item
  - Top-N users by interactions
  - Top-N items for latest window
- Includes expandable raw aggregated data tables.

---

### 5. Checkpoint Cleanup (`clean_checkpoint/`)
Utilities for managing Spark checkpoint files:

- **`cleanup_checkpoints.py`**: Python script to safely clean old checkpoint files while preserving recent commits, offsets, and state batches for recovery.
- **`cleanup_checkpoint.sh`**: Shell script for basic checkpoint cleanup.

**⚠️ Important**: Always stop Spark streaming jobs before running cleanup scripts.

---

## How to Run

### Prerequisites
- Python 3.9+
- Apache Kafka (local or Docker)
- MongoDB (local or Docker)
- Apache Spark 4.1.0+ (for the consumer)
- Java 8+ (required for Spark)

### Installation

1. **Install Python dependencies:**
   ```bash
   pip install streamlit pymongo plotly streamlit-autorefresh kafka-python pyspark
   ```

2. **Set up Kafka:**
   - Option 1: Using Docker Compose (recommended)
     ```bash
     # Create a docker-compose-kafka.yml with Kafka and Zookeeper
     docker-compose up -d
     ```
   - Option 2: Install Kafka locally
     - Download from https://kafka.apache.org/downloads
     - Start Zookeeper and Kafka broker

3. **Set up MongoDB:**
   - Option 1: Using Docker
     ```bash
     docker run -d -p 27017:27017 \
       -e MONGO_INITDB_ROOT_USERNAME=admin \
       -e MONGO_INITDB_ROOT_PASSWORD=password \
       mongo:latest
     ```
   - Option 2: Install MongoDB locally
     - Download from https://www.mongodb.com/try/download/community
     - Start MongoDB service

4. **Install Apache Spark:**
   - Download Spark from https://spark.apache.org/downloads.html
   - Extract and set `SPARK_HOME` environment variable
   - Ensure Java is installed and `JAVA_HOME` is set

---

## Usage

### 1. Start the Producer

The producer generates interaction events and sends them to Kafka.

```bash
python producer/producer.py \
  --broker localhost:9092 \
  --topic test-topic \
  --rate 1000 \
  --batch 50
```

**Arguments:**
- `--broker`: Kafka bootstrap servers (default: `localhost:9092`)
- `--topic`: Kafka topic name (default: `test-topic`)
- `--rate`: Messages per second (default: `1000`)
- `--batch`: Messages per batch (default: `10`)

**Example:**
```bash
# Generate 500 events per second in batches of 25
python producer/producer.py --broker localhost:9092 --topic test-topic --rate 500 --batch 25
```

---

### 2. Start the Consumer

The consumer reads from Kafka, performs windowed aggregations using Spark Structured Streaming, and writes results to MongoDB.

```bash
python consumer/consumer.py \
  --brokers localhost:9092 \
  --topic test-topic \
  --checkpoint-base ./storage/checkpoint \
  --starting-offsets earliest \
  --mongo-uri mongodb://admin:password@localhost:27017/?authSource=admin
```

**Arguments:**
- `--brokers`: Kafka bootstrap servers (default: `localhost:9092`)
- `--topic`: Kafka topic to subscribe (default: `test-topic`)
- `--checkpoint-base`: Base directory for Spark checkpointing (default: absolute path to `./storage/checkpoint` in project root)
- `--starting-offsets`: Starting offsets for Kafka - `earliest` or `latest` (default: `earliest`)
- `--mongo-uri`: MongoDB connection URI (default: `mongodb://admin:password@localhost:27017/?authSource=admin`)
- `--kafka-package`: Kafka connector package (default: `org.apache.spark:spark-sql-kafka-0-10_2.13:4.1.0`)
- `--mongo-package`: MongoDB connector package (default: `org.mongodb.spark:mongo-spark-connector_2.13:10.4.0`)

**Aggregation Details:**
- **Window Size**: 10 minutes
- **Slide Interval**: 5 minutes (sliding window)
- **Watermark**: 10 minutes (handles late-arriving data)
- **Output Collections**:
  - `user_interactions`: Aggregated interactions per user per window
  - `item_interactions`: Aggregated interactions per item per window

---

### 3. Start the Dashboard

The Streamlit dashboard visualizes real-time metrics from MongoDB.

```bash
streamlit run reporting/reporting.py
```

The dashboard will be available at `http://localhost:8501`

**Features:**
- Auto-refresh (configurable interval: 2-30 seconds)
- Real-time KPIs:
  - Average interactions per user
  - Maximum interactions per item
  - Minimum interactions per item
- Interactive charts:
  - Top-N users by total interactions
  - Top-N items for the latest time window
- Raw data tables (expandable)

**Dashboard Controls:**
- Auto-refresh rate slider
- Top users/items count sliders
- Toggle for raw data display

Image :

<img width="1715" height="876" alt="image" src="https://github.com/user-attachments/assets/6a1394e0-472d-4db1-87aa-b5d97d8843f1" />

<img width="1713" height="870" alt="image" src="https://github.com/user-attachments/assets/1c9c3020-5675-4b5d-a0b4-d569688235bf" />

<img width="1714" height="874" alt="image" src="https://github.com/user-attachments/assets/d3b3712e-14cf-4fee-89c7-cd2340e13451" />



---

### 4. Cleanup Checkpoints (Optional)

To manage disk space used by Spark checkpoints, use the cleanup utilities:

```bash
# Using Python script (recommended)
python clean_checkpoint/cleanup_checkpoints.py

# Or using shell script
bash clean_checkpoint/cleanup_checkpoint.sh
```

**⚠️ Warning**: Always stop the Spark streaming consumer before running cleanup scripts to avoid corruption.

---

## Project Structure

```
AvriocProject/
├── producer/
│   └── producer.py          # Kafka producer for generating events
├── consumer/
│   └── consumer.py          # Spark streaming consumer with aggregations
├── reporting/
│   └── reporting.py         # Streamlit dashboard
├── storage/
│   └── checkpoint/          # Spark checkpoint data
│       ├── user_interactions_checkpoint/
│       └── item_interactions_checkpoint/
├── clean_checkpoint/
│   ├── cleanup_checkpoints.py  # Python checkpoint cleanup utility
│   └── cleanup_checkpoint.sh   # Shell checkpoint cleanup utility
└── README.md                # This file
```

---

## Event Schema

Each interaction event has the following structure:

```json
{
  "user_id": "user_12345",
  "item_id": "item_6789",
  "interaction_type": "click",
  "timestamp": "2025-12-22T10:30:00.000Z"
}
```

**Interaction Types:**
- `click`
- `view`
- `purchase`
- `like`
- `add_to_cart`

---

## Configuration

### MongoDB Collections Schema

#### `user_interactions`
```json
{
  "window_start": "2025-12-22T10:00:00Z",
  "window_end": "2025-12-22T10:10:00Z",
  "user_id": "user_12345",
  "total_interactions": 42
}
```

#### `item_interactions`
```json
{
  "window_start": "2025-12-22T10:00:00Z",
  "window_end": "2025-12-22T10:10:00Z",
  "item_id": "item_6789",
  "total_interactions": 156
}
```

### Kafka Configuration

- **Default Topic**: `test-topic` (configurable)
- **Partitions**: Default Kafka configuration
- **Replication**: Default Kafka configuration

### Spark Configuration

- **Spark Version**: 4.1.0+
- **Checkpointing**: Enabled for fault tolerance
- **Watermark**: 10 minutes for late data handling
- **Window**: 10-minute windows sliding every 5 minutes

---

## Troubleshooting

### Producer Issues

**Problem**: Cannot connect to Kafka broker
- **Solution**: Verify Kafka is running and the broker address is correct
- Check `KAFKA_ADVERTISED_LISTENERS` if using Docker
- Ensure the port (default: 9092) is accessible

**Problem**: `KafkaTimeoutError`
- **Solution**: Check network connectivity and broker configuration
- Verify topic exists or auto-creation is enabled

### Consumer Issues

**Problem**: Spark cannot find Kafka connector
- **Solution**: Ensure the correct `--kafka-package` is specified
- Check Spark version compatibility with the connector

**Problem**: MongoDB connection fails
- **Solution**: Verify MongoDB is running and credentials are correct
- Check connection URI format
- Ensure MongoDB connector package is correct

**Problem**: Checkpoint errors
- **Solution**: Clear checkpoint directory if starting fresh (be careful - this resets offsets)
- Ensure checkpoint directory has write permissions
- Use cleanup scripts to manage checkpoint disk usage

**Problem**: Consumer lag increasing
- **Solution**: Monitor lag in consumer logs
- Adjust Spark resources (executors, cores, memory) if needed
- Consider increasing Kafka partitions for better parallelism

### Dashboard Issues

**Problem**: No data displayed
- **Solution**: Verify MongoDB collections have data
- Check MongoDB connection URI in `reporting.py`
- Ensure consumer is running and processing events

**Problem**: Dashboard not updating
- **Solution**: Check auto-refresh interval
- Verify MongoDB connection is active
- Check browser console for errors

---

## Performance Considerations

- **Producer Rate**: Adjust `--rate` and `--batch` based on your system capacity
- **Consumer Lag**: Monitor lag in consumer logs; adjust Spark resources if needed
- **MongoDB**: Index `window_start`, `window_end`, `user_id`, and `item_id` for better query performance
- **Checkpointing**: Ensure sufficient disk space for checkpoint data; use cleanup scripts periodically
- **Spark Resources**: Configure executor memory and cores based on data volume

---

## Development

### Adding New Aggregations

To add new aggregations in the consumer:

1. Create a new aggregation DataFrame in `consumer/consumer.py`
2. Start a new streaming query writing to a new MongoDB collection
3. Update the dashboard to display the new metrics

### Extending Event Schema

To add new fields to events:

1. Update `gen_event()` in `producer/producer.py`
2. Update the schema in `consumer/consumer.py`
3. Update MongoDB collections and dashboard as needed

### Managing Checkpoints

The checkpoint cleanup utilities help manage disk space:

- **`cleanup_checkpoints.py`**: Safely removes old checkpoint files while preserving recent data for recovery
- **`cleanup_checkpoint.sh`**: Basic shell script for checkpoint cleanup
- Always stop streaming jobs before cleanup to prevent corruption

---

## License

[Add your license information here]

---

## Contributing

[Add contribution guidelines here]

## Extra Commands
Mongo DB Setup using Docker:
1. Download and Install Docker from https://www.docker.com/get-started
2. Pull the MongoDB Docker Image: https://hub.docker.com/_/mongo
   ```bash
   docker pull mongo:latest
   ```
3. Run MongoDB Container:
   ```bash
   docker run -d --name local-mongo -p 27017:27017 -e MONGO_INITDB_ROOT_USERNAME=admin -e MONGO_INITDB_ROOT_PASSWORD=password mongo:latest
   ```
Kafka Setup using Docker Compose:
1. Use file Docker/docker-compose-kafka.yml to create Kafka and Zookeeper services.
2. Start Kafka and Zookeeper:
   ```bash
   docker-compose -f Docker/docker-compose-kafka.yml up -d
   ```
3. Run below command to check if Kafka and Zookeeper are running:
   ```bash
   docker ps
   ```
4. Access Kafka Broker:
   ```bash
   docker exec -it <kafka_container_id> bash
   ```
5. Create Kafka Topic:
   ```bash
   kafka-topics --create --topic test-topic --partitions 3 --replication-factor 2 --add-config retention.ms=1000,cleanup.policy=delete --bootstrap-server localhost:29092
   ```
6. List Kafka Topics:
   ```bash
   kafka-topics --list --bootstrap-server localhost:29092
   ```
7. Describe Kafka Topic:
   ```bash
   kafka-topics --describe --topic test-topic --bootstrap-server localhost:29092
   ```
8. Produce Sample Messages to Topic:
   ```bash
   kafka-console-producer --topic test-topic --bootstrap-server localhost:29092
   ```
9. Consume Sample Messages from Topic:
   ```bash
   kafka-console-consumer --topic test-topic --from-beginning --bootstrap-server localhost:29092
   ```

