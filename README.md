# Data Ops Lottery Platform - SV

This project, as a solution to Zeal's Case Study, processes real-time user interactions with lottery games and aggregates into business metrics such as revenue, logins, payouts, and active users, on an hourly basis. The pipeline is Python-based, and includes data validation, Postgres DB for insights, and observability dashboards in Grafana, within a containerized, local setup.

## Flow Diagram
```
   +------------------+            +----------------+
   |  Event Producer  |  ------>   |     Kafka      |
   | (lottery events) |            | user-events    |
   +------------------+            +----------------+
                                         ||
                                         \/
                               +----------------------+
                               |    Kafka Consumer     |
                               |  (aggregator + writer)|
                               +----------------------+
                               | - Writes to Postgres |
                               | - Exposes metrics    |
                               +----------------------+
                                         ||
                                         \/
                     +----------------------------+
                     |     Prometheus + Grafana    |
                     | - Metrics, Alerts, Dashboards|
                     +----------------------------+
   
```

### Assumptions

*	Domain = lottery platform interactions (event_types = [ "login", "view_numbers", "ticket_purchased", "claim_reward", "logout" ])
*	One Kafka topic for all user events
*	Aggregation = hourly by event type, in Postgres
*	Consumer does both ingestion and aggregation


### Pre-requisites

*	Python 3.10
*	Docker & Docker Compose


### Project Structure
```
.
├── producer/                # Kafka producer (simulates streaming events from an event json file)
│   └── producer.py
├── consumer/                # Kafka consumer with core logic to process and validate events
│   └── consumer.py
├── database/                # PostgreSQL 14 (events_db)
│   └── schema.sql           # Schema for events, hourly_business_metrics, and some views
├── data/                    # Data Source and Error Output dir
│   └── events.jsonl         # Sample event data (125 events)
├── tests/                   # Unit tests for producer and consumer logic (with pytest and mocks)
│   └── test_consumer.py
├── monitoring/              # Monitoring and Observability
│   └── grafana              # Grafana dashboard config
│           └── dashboards
│                   └── dashboard.json      # Metrics dashboard config
│           └── provisioning
│   └── prometheus           # Prometheus config for logs and alerts
├── docker-compose.yml       # Fully-containerized setup and execution
├── Dockerfile               # Python app
├── requirements.txt         
└── README.md
```

## Execution
1. Setup:
```bash
git clone <your-repo-url>
cd data-ops-lottery-platform-sv

python -m venv .venv
source .venv/bin/activate
pip install -r requirements.txt
```

2. Run tests:
```bash
PYTHONPATH=producer pytest -sv tests/test_producer.py

PYTHONPATH=consumer pytest -sv tests/test_consumer.py
```

3. Start the services:
```bash
docker-compose up --build
```

4. Observability Dashboards:

*	Prometheus: http://localhost:9090
*	Grafana: http://localhost:3000 (Default login: admin / admin)

5. Business Metrics
* Total revenue from ticket purchases
* Total payouts from rewards
* Count of ticket purchases and rewards per hour
* Number of logins, unique users, and new sessions
  
```
psql -h localhost -p 5432 -U user -d events_db
[user: password]

select * from events;
select * from hourly_business_metrics;
select * from rolling_24h_metrics;
select * from daily_event_summary;
```

## Notes and Future Enhancements
* There might be a little lag in the consumer event count showing up on the Grafana dashboard, but eventually it tallies up (Producer Sent = Consumer Processed + Consumer Failed).
* CI workflows with unit tests can be integrated in later.
* Validation can be improved with tools like pydantic.
* More observability metrics like stream latency, eg. Prometheus gauge to show lag between event_timestamp and now() to measure freshness, can be introduced later.
