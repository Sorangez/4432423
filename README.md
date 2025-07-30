```markdown
# Assignment 1: Data Pipeline with Docker

This repository contains three separate tasks, each of which you can run independently.

Tutorial Video Link: https://youtu.be/mttadZUKRcY

## Table of Contents
1. [Prerequisites](#prerequisites)  
2. [Task 1: OpenWeatherMap Data Pipeline (12 pts)](#task-1-openweathermap-data-pipeline-12-pts)  
3. [Task 2: Faker API Data Pipeline (12 pts)](#task-2-faker-api-data-pipeline-12-pts)  
4. [Task 3: CoinGecko Crypto Prices Pipeline (12 pts)](#task-3-coingecko-crypto-prices-pipeline-12-pts)  
5. [License](#license)  

---

## Prerequisites

- Docker & Docker Compose installed  
- Python 3.8 or higher available on your PATH  
- Ensure the following ports are free:  
  - `9000` for Kafka UI  
  - `8083` for Kafka Connect  
  - `9042` for Cassandra CQLSH  
  - `8888` for Jupyter Notebook  
- Repository layout as follows:

```
```
.
├── cassandra/
│   └── docker-compose.yml
├── kafka/
│   └── docker-compose.yml
├── owm-producer/
│   ├── docker-compose.yml
│   └── openweathermap\_producer.py
├── faker-producer/
│   ├── docker-compose.yml
│   ├── Dockerfile
│   ├── requirements.txt
│   └── faker\_producer.py
├── consumers/
│   └── docker-compose.yml
├── data-vis/
│   ├── docker-compose.yml
│   └── python/
│       └── blog-visuals.ipynb
├── cassandra/schema-faker.cql
└── kafka/connect/create-cassandra-sink.sh

```

---

## Task 1: OpenWeatherMap Data Pipeline (12 pts)

Build a Kafka→Cassandra pipeline to ingest live weather data for two new cities.

1. **Create Docker Networks**  
 ```bash
 docker network create kafka-network
 docker network create cassandra-network
````

2. **Launch Cassandra & Kafka**

   ```bash
   docker-compose -f cassandra/docker-compose.yml up -d
   docker-compose -f kafka/docker-compose.yml up -d
   ```

3. **Configure Kafka Cluster via UI**

   * Open `http://localhost:9000`
   * Log in with `admin` / `bigbang`
   * Click **Add Cluster** and configure:

     * **Cluster name**: your choice
     * **Zookeeper host**: `zookeeper:2181`
     * Enable JMX polling (ensure `JMX_PORT` set in Kafka server env)
     * Optionally enable Active Offset Cache

4. **Start Kafka Connect**

   ```bash
   # inside kafka/connect container
   ./start-and-wait.sh
   ```

5. **Configure & Run OpenWeatherMap Producer**

   1. Edit `owm-producer/openweathermap_producer.py`
   2. On line 34, replace the city list with two new city IDs (find IDs here:
      [https://openweathermap.org/weathermap](https://openweathermap.org/weathermap)?...)
   3. Launch the producer:

      ```bash
      docker-compose -f owm-producer/docker-compose.yml up -d
      ```

6. **Launch Consumers & Visualizer**

   ```bash
   docker-compose -f consumers/docker-compose.yml up -d
   docker-compose -f data-vis/docker-compose.yml up -d
   ```

7. **Verify Data in Cassandra**

   ```bash
   cqlsh --cqlversion=3.4.4 127.0.0.1
   ```

   Within CQL shell:

   ```sql
   USE kafkapipeline;
   SELECT * FROM weatherreport;
   ```

8. **Run the Data Visualization Notebook**

   * Open `http://localhost:8888`
   * Navigate to `data-vis/python/blog-visuals.ipynb`
   * Execute all cells to generate and view visualizations

---

## Task 2: Faker API Data Pipeline (12 pts)

Repeat tutorial 4, ingesting Faker-generated user data with at least **10 fields**.

1. **Define Cassandra Schema**

   * Create `cassandra/schema-faker.cql` with:

     ```sql
     USE kafkapipeline;
     CREATE TABLE kafkapipeline.fakerdata (
         id UUID,
         full_name TEXT,
         street_address TEXT,
         city TEXT,
         email TEXT,
         phone TEXT,
         company TEXT,
         job_title TEXT,
         year INT,
         website TEXT,
         PRIMARY KEY (id)
     );
     ```
   * Start (or rebuild) Cassandra and apply schema:

     ```bash
     docker-compose -f cassandra/docker-compose.yml up -d --build
     cqlsh --cqlversion=3.4.4 127.0.0.1 -f cassandra/schema-faker.cql
     ```

2. **Verify Table Creation**

   ```bash
   cqlsh --cqlversion=3.4.4 127.0.0.1
   ```

   Within CQL shell:

   ```sql
   USE kafkapipeline;
   SELECT * FROM fakerdata;
   ```

3. **Configure Cassandra Sink Connector**

   * Edit `kafka/connect/create-cassandra-sink.sh` to include:

     ```bash
     echo "Starting Faker Sink"
     curl -s -X POST http://localhost:8083/connectors \
       -H "Content-Type: application/json" \
       -d '{
         "name": "fakersink",
         "config": {
           "connector.class":"com.datastax.oss.kafka.sink.CassandraSinkConnector",
           "value.converter":"org.apache.kafka.connect.json.JsonConverter",
           "value.converter.schemas.enable":"false",
           "key.converter":"org.apache.kafka.connect.json.JsonConverter",
           "key.converter.schemas.enable":"false",
           "tasks.max":"10",
           "topics":"faker",
           "contactPoints":"cassandradb",
           "loadBalancing.localDc":"datacenter1",
           "topic.faker.kafkapipeline.fakerdata.mapping":
             "id=value.id, full_name=value.full_name, street_address=value.street_address, city=value.city, email=value.email, phone=value.phone, company=value.company, job_title=value.job_title, year=value.year, website=value.website",
           "topic.faker.kafkapipeline.fakerdata.consistencyLevel":"LOCAL_QUORUM"
         }
       }'
     ```
   * Start (or rebuild) Kafka to pick up connector:

     ```bash
     docker-compose -f kafka/docker-compose.yml up -d --build
     ```

4. **Set Up Faker Producer**

   1. Duplicate `owm-producer/` → `faker-producer/`
   2. Remove `openweathermap_service.cfg`
   3. Rename Python file to `faker_producer.py` and replace contents with:

      ```python
      import time, os, json
      from uuid import uuid4
      from kafka import KafkaProducer
      from faker import Faker

      fake = Faker()
      KAFKA_BROKER_URL = os.environ.get("KAFKA_BROKER_URL")
      TOPIC_NAME = os.environ.get("TOPIC_NAME")
      SLEEP_TIME = int(os.environ.get("SLEEP_TIME", 5))

      def get_registered_user():
          return {
              "id": str(uuid4()),
              "full_name": fake.name(),
              "street_address": fake.street_address(),
              "city": fake.city(),
              "email": fake.email(),
              "phone": fake.phone_number(),
              "company": fake.company(),
              "job_title": fake.job(),
              "year": int(fake.year()),
              "website": fake.url()
          }

      def run():
          producer = KafkaProducer(
              bootstrap_servers=[KAFKA_BROKER_URL],
              value_serializer=lambda x: json.dumps(x).encode('utf-8')
          )
          while True:
              record = get_registered_user()
              producer.send(TOPIC_NAME, value=record)
              time.sleep(SLEEP_TIME)

      if __name__ == "__main__":
          run()
      ```
   4. Update `requirements.txt` to include `faker` instead of `dataprep`
   5. Update `Dockerfile` to run `faker_producer.py`
   6. In `faker-producer/docker-compose.yml`, set:

      * Service name/image to `faker-producer`
      * Environment `TOPIC_NAME=faker`, `SLEEP_TIME=5`
   7. Launch producer:

      ```bash
      docker-compose -f faker-producer/docker-compose.yml up -d
      ```

5. **Set Up Faker Consumer**

   1. Duplicate `consumers/python/weather_consumer.py` → `consumers/python/faker_consumer.py`
   2. Replace contents with:

      ```python
      import os, json
      from kafka import KafkaConsumer

      if __name__ == "__main__":
          TOPIC_NAME = os.environ.get("FAKER_TOPIC_NAME", "faker")
          KAFKA_BROKER_URL = os.environ.get("KAFKA_BROKER_URL", "localhost:9092")
          consumer = KafkaConsumer(TOPIC_NAME, bootstrap_servers=[KAFKA_BROKER_URL])
          for msg in consumer:
              print(json.loads(msg.value.decode('utf-8')))
      ```
   3. In `consumers/docker-compose.yml`, add:

      ```yaml
      fakerconsumer:
        container_name: fakerconsumer
        image: twitterconsumer
        environment:
          KAFKA_BROKER_URL: broker:9092
          TOPIC_NAME: faker
          CASSANDRA_HOST: cassandradb
          CASSANDRA_KEYSPACE: kafkapipeline
        command: ["python", "-u", "python/faker_consumer.py"]
      ```
   4. Build and run:

      ```bash
      docker-compose -f consumers/docker-compose.yml up --build
      ```

6. **Verify Data in Cassandra**

   ```bash
   cqlsh --cqlversion=3.4.4 127.0.0.1
   ```

   Within CQL shell:

   ```sql
   USE kafkapipeline;
   SELECT * FROM fakerdata;
   ```


## Task 3: CoinGecko Crypto Prices Pipeline (12 pts)

Ingest hourly crypto price data from the CoinGecko API into Cassandra via Kafka.

1. **Define Cassandra Schema**  
 - Create `cassandra/schema-coinGecko.cql` with:
   ```sql
   USE kafkapipeline;
   CREATE TABLE crypto_prices (
       coin_id         text,
       utc_hour_bucket timestamp,
       ts_utc          timestamp,
       price_usd       decimal,
       market_cap_usd  bigint,
       volume_24h_usd  bigint,
       change_1h_pct   double,
       change_24h_pct  double,
       change_7d_pct   double,
       PRIMARY KEY ((coin_id, utc_hour_bucket), ts_utc)
   )
   WITH CLUSTERING ORDER BY (ts_utc DESC)
     AND default_time_to_live = 604800;
   ```
 - Rebuild and start Cassandra, then apply schema:
   ```bash
   docker-compose -f cassandra/docker-compose.yml up -d --build
   cqlsh --cqlversion=3.4.4 127.0.0.1 -f cassandra/schema-coinGecko.cql
   ```

2. **Include Schema in Docker Build**  
 - In your Dockerfile(s), add:
   ```dockerfile
   COPY [ "schema.cql", "schema-faker.cql", "schema-coinGecko.cql", "keyspace.cql", "bootstrap.sh", "wait-for-it.sh", "./" ]
   ```
 - Rebuild all relevant images:
   ```bash
   docker-compose -f cassandra/docker-compose.yml build
   docker-compose -f kafka/docker-compose.yml build
   ```

3. **Run Schema in Cassandra**  
 ```bash
 cqlsh --cqlversion=3.4.4 127.0.0.1 -f cassandra/schema-coinGecko.cql
````

4. **Configure Cassandra Sink Connector**

   * Append to `kafka/connect/create-cassandra-sink.sh`:

     ```bash
     echo "Starting crypto_prices Sink"
     curl -s -X POST http://localhost:8083/connectors \
       -H "Content-Type: application/json" \
       -d '{
         "name": "crypto_pricessink",
         "config": {
           "connector.class":"com.datastax.oss.kafka.sink.CassandraSinkConnector",
           "value.converter":"org.apache.kafka.connect.json.JsonConverter",
           "value.converter.schemas.enable":"false",
           "key.converter":"org.apache.kafka.connect.storage.StringConverter",
           "key.converter.schemas.enable":"false",
           "tasks.max":"10",
           "topics":"crypto_prices",
           "contactPoints":"cassandradb",
           "loadBalancing.localDc":"datacenter1",
           "topic.crypto_prices.kafkapipeline.crypto_prices.mapping":
             "coin_id=value.coin_id, utc_hour_bucket=value.utc_hour_bucket, ts_utc=value.ts_utc, price_usd=value.price_usd, market_cap_usd=value.market_cap_usd, volume_24h_usd=value.volume_24h_usd, change_1h_pct=value.change_1h_pct, change_24h_pct=value.change_24h_pct, change_7d_pct=value.change_7d_pct",
           "topic.crypto_prices.kafkapipeline.crypto_prices.consistencyLevel":"LOCAL_QUORUM",
           "errors.tolerance":"all",
           "errors.log.enable":"true",
           "errors.log.include.messages":"true"
         }
       }'
     echo "Done."
     ```
   * Ensure Kafka is set to create the topic:

     ```yaml
     environment:
       KAFKA_CREATE_TOPICS: "twitter:1:1,weather:1:1,twittersink:1:1,faker:1:1,crypto_prices:1:1"
     ```
   * Rebuild and start Kafka:

     ```bash
     docker-compose -f kafka/docker-compose.yml up -d --build
     ```

5. **Launch CoinGecko Producer**

   * Use provided `cg-producer/` folder (contains script, cfg, Dockerfile, compose)
   * Configure your API key in `cg-producer/config.cfg` or environment
   * Build & run:

     ```bash
     docker-compose -f cg-producer/docker-compose.yml up -d
     ```

6. **Set Up Crypto Prices Consumer**

   * Create `consumers/python/crypto_prices_consumer.py`:

     ```python
     import os, json
     from kafka import KafkaConsumer

     if __name__ == "__main__":
         TOPIC_NAME = os.getenv("TOPIC_NAME", "crypto_prices")
         KAFKA_BROKER_URL = os.getenv("KAFKA_BROKER_URL", "localhost:9092")
         consumer = KafkaConsumer(
             TOPIC_NAME,
             bootstrap_servers=[KAFKA_BROKER_URL],
             auto_offset_reset="latest",
             value_deserializer=lambda v: v.decode("ascii"),
             group_id="crypto_prices_viewer"
         )
         for msg in consumer:
             try:
                 print(json.loads(msg.value))
             except json.JSONDecodeError:
                 print("Skipping malformed record")
     ```
   * Add to `consumers/docker-compose.yml`:

     ```yaml
     crypto_pricesconsumer:
       container_name: crypto_pricesconsumer
       image: twitterconsumer
       environment:
         KAFKA_BROKER_URL: broker:9092
         TOPIC_NAME: crypto_prices
         CASSANDRA_HOST: cassandradb
         CASSANDRA_KEYSPACE: kafkapipeline
       command: ["python", "-u", "python/crypto_prices_consumer.py"]
     ```
   * Build & start:

     ```bash
     docker-compose -f consumers/docker-compose.yml up --build
     ```

7. **Verify Data in Cassandra**

   ```bash
   cqlsh --cqlversion=3.4.4 127.0.0.1
   ```

   Within CQL shell:

   ```sql
   USE kafkapipeline;
   SELECT * FROM crypto_prices;
   ```


## License

This project is licensed under Nguyen Thanh Tung - s3979425.


## Acknowledgements

- Readme and debugging assistance generated with AI tools.  
- Main help from [docker-kafka-cassandra GitHub repository](https://github.com/vnyennhi/docker-kafka-cassandra).

```
```
