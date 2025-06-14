# CDC (Change Data Capture) with Kafka Demo

This project demonstrates Change Data Capture (CDC) using PostgreSQL, Debezium, and Kafka to capture and stream database changes in real-time.

## Overview

This demo sets up a complete CDC pipeline that:
- Uses PostgreSQL with logical replication enabled
- Deploys Debezium connector to capture database changes
- Streams changes to Kafka topics
- Monitors data changes in real-time

## Prerequisites

- Docker and Docker Compose
- curl (for API calls)
- Basic knowledge of PostgreSQL, Kafka, and CDC concepts

## Architecture

```
PostgreSQL (Logical Replication) → Debezium Connector → Kafka → Consumer Applications
```

## Quick Start

### 1. Create Docker Network

First, create a Docker network for the services to communicate:

```bash
docker network create cdc-net
```

### 2. Start PostgreSQL

Start PostgreSQL with logical replication enabled:

```bash
docker-compose -f postgres.yaml up -d
```

This will start PostgreSQL on port `5433` with the following configuration:
- **User**: `dev`
- **Password**: `mysecret`
- **Database**: `cdc`
- **WAL Level**: `logical` (required for CDC)
- **Max WAL Senders**: `10`
- **Max Replication Slots**: `10`

### 3. Setup Database Schema

Connect to PostgreSQL and create the demo table:

```bash
docker exec -it postgres_db psql -U dev -d cdc
```

Run the following SQL commands:

```sql
-- Enable UUID extension
CREATE EXTENSION IF NOT EXISTS "uuid-ossp";

-- Create demo table for CDC
CREATE TABLE public.demo_cdc (
    id           BIGINT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    uuid_col     UUID        NOT NULL DEFAULT uuid_generate_v4(),
    name         VARCHAR(100) NOT NULL,
    description  TEXT,
    active       BOOLEAN     NOT NULL DEFAULT true,
    price        NUMERIC(12,2)          CHECK (price >= 0),
    score        DOUBLE PRECISION,
    tags         TEXT[],
    metadata     JSONB,
    data_blob    BYTEA,
    birthdate    DATE,
    created_at   TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at   TIMESTAMPTZ NOT NULL DEFAULT now(),
    status       SMALLINT
);

-- Optional: Set REPLICA IDENTITY FULL for capturing BEFORE/AFTER images
-- ALTER TABLE public.demo_cdc REPLICA IDENTITY FULL;
```

### 4. Start Kafka and Debezium

Start the Kafka ecosystem with Debezium Connect:

```bash
# Add your Kafka and Debezium docker-compose configuration here
# This typically includes Zookeeper, Kafka, and Kafka Connect with Debezium
```

### 5. Register Debezium Connector

Register the PostgreSQL connector with Debezium:

```bash
curl -X POST http://localhost:8083/connectors \
-H 'Content-Type: application/json' \
-d '{
  "name": "pg-demo-cdc",
  "config": {
    "connector.class": "io.debezium.connector.postgresql.PostgresConnector",
    "plugin.name": "pgoutput",
    "slot.name": "demo_cdc_slot",
    "publication.autocreate.mode": "filtered",

    "database.hostname": "db",
    "database.port": "5432",
    "database.user": "dev",
    "database.password": "mysecret",
    "database.dbname": "cdc",

    "schema.include.list": "public",
    "table.include.list": "public.demo_cdc",

    "topic.prefix": "cdc_demo"
  }
}'
```

## Configuration Details

### PostgreSQL Configuration

The PostgreSQL instance is configured with specific parameters for logical replication:

- **wal_level=logical**: Enables logical decoding required by Debezium
- **max_wal_senders=10**: Maximum number of concurrent WAL sender processes
- **max_replication_slots=10**: Maximum number of replication slots

### Debezium Connector Configuration

- **Connector Class**: `io.debezium.connector.postgresql.PostgresConnector`
- **Plugin**: `pgoutput` (PostgreSQL's built-in logical replication)
- **Slot Name**: `demo_cdc_slot`
- **Publication Mode**: `filtered` (automatically creates publications)
- **Topic Prefix**: `cdc_demo`

## Testing the Setup

### 1. Insert Test Data

```sql
INSERT INTO public.demo_cdc (name, description, price, tags, metadata, birthdate) 
VALUES (
    'Sample Product', 
    'This is a test product for CDC demo', 
    29.99, 
    ARRAY['electronics', 'demo'], 
    '{"category": "test", "featured": true}',
    '1990-01-01'
);
```

### 2. Update Data

```sql
UPDATE public.demo_cdc 
SET price = 35.99, active = false 
WHERE name = 'Sample Product';
```

### 3. Delete Data

```sql
DELETE FROM public.demo_cdc WHERE name = 'Sample Product';
```

### 4. Monitor Kafka Topics

Check the Kafka topics to see the CDC events:

```bash
# List topics
docker exec -it <kafka-container> kafka-topics --list --bootstrap-server localhost:9092

# Consume messages from CDC topic
docker exec -it <kafka-container> kafka-console-consumer \
  --bootstrap-server localhost:9092 \
  --topic cdc_demo.public.demo_cdc \
  --from-beginning
```

## Data Types Demonstration

The `demo_cdc` table includes various PostgreSQL data types to demonstrate CDC capabilities:

- **BIGINT IDENTITY**: Auto-incrementing primary key
- **UUID**: Universally unique identifier
- **VARCHAR/TEXT**: String data types
- **BOOLEAN**: Boolean values
- **NUMERIC**: Precise decimal numbers
- **DOUBLE PRECISION**: Floating-point numbers
- **TEXT[]**: Array of text values
- **JSONB**: Binary JSON data
- **BYTEA**: Binary data
- **DATE/TIMESTAMPTZ**: Date and timestamp with timezone

## Monitoring and Management

### Check Connector Status

```bash
curl -X GET http://localhost:8083/connectors/pg-demo-cdc/status
```

### Delete Connector

```bash
curl -X DELETE http://localhost:8083/connectors/pg-demo-cdc
```

### View Replication Slots

```sql
SELECT * FROM pg_replication_slots;
```

## Troubleshooting

### Common Issues

1. **WAL Level Not Set**: Ensure `wal_level=logical` in PostgreSQL configuration
2. **Network Issues**: Verify all services are on the same Docker network (`cdc-net`)
3. **Permission Issues**: Check PostgreSQL user permissions for replication
4. **Port Conflicts**: Ensure ports 5433 (PostgreSQL) and 8083 (Kafka Connect) are available

### Logs

Check container logs for debugging:

```bash
# PostgreSQL logs
docker logs postgres_db

# Kafka Connect logs
docker logs <kafka-connect-container>
```

## Cleanup

To stop and remove all resources:

```bash
# Stop PostgreSQL
docker-compose -f postgres.yaml down -v

# Remove network
docker network rm cdc-net

# Remove Docker volumes (optional)
docker volume prune
```

## Next Steps

- Add Kafka consumer applications
- Implement data transformation
- Set up monitoring with Kafka Connect metrics
- Add schema registry for Avro serialization
- Implement error handling and dead letter queues

## Resources

- [Debezium PostgreSQL Connector Documentation](https://debezium.io/documentation/reference/connectors/postgresql.html)
- [PostgreSQL Logical Replication](https://www.postgresql.org/docs/current/logical-replication.html)
- [Kafka Connect Documentation](https://kafka.apache.org/documentation/#connect)
