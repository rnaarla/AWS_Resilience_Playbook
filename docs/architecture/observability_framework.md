# Observability Framework

## Overview

The observability layer uses a **Causal Event Graph (CEG)** approach to provide deep insights into system behavior, enabling rapid detection, diagnosis, and remediation of issues in autonomous control planes.

## Table of Contents

1. [Architecture Principles](#architecture-principles)
2. [Core Components](#core-components)
3. [Telemetry Stack](#telemetry-stack)
4. [Dashboard Templates](#dashboard-templates)
5. [Alert Configuration](#alert-configuration)
6. [Integration Patterns](#integration-patterns)

---

## Architecture Principles

### 1. Causal Awareness

Every event must be tagged with causal metadata:

- **Trace ID**: Distributed trace identifier
- **Span ID**: Individual operation identifier  
- **Parent ID**: Causal parent span
- **Timestamp**: High-resolution timestamp (microseconds)
- **Vector Clock**: Distributed causality tracking

### 2. Three Pillars of Observability

**Metrics**: Quantitative time-series data  
**Logs**: Discrete event records  
**Traces**: Request flow visualization

### 3. Observability Driven Development (ODD)

- Instrumentation first, code second
- Every critical path instrumented
- SLIs defined before deployment
- Dashboards deployed with code

---

## Core Components

### 1. Causal Graph Engine (CGE)

**Purpose**: Real-time correlation of cross-service telemetry

**Capabilities**:

- Build dependency graphs in real-time
- Detect anomalous patterns
- Identify root cause candidates
- Predict failure propagation

**Implementation Architecture**:

```text
┌──────────────────────────────────────────────────┐
│          Causal Graph Engine (CGE)                │
│                                                   │
│  ┌─────────────┐    ┌──────────────┐            │
│  │  Ingestion  │───▶│   Graph DB   │            │
│  │   Pipeline  │    │  (Neo4j/JG)  │            │
│  └─────────────┘    └──────────────┘            │
│         │                    │                    │
│         │                    │                    │
│         ▼                    ▼                    │
│  ┌─────────────┐    ┌──────────────┐            │
│  │  Causality  │    │   Anomaly    │            │
│  │   Analyzer  │    │   Detector   │            │
│  └─────────────┘    └──────────────┘            │
│         │                    │                    │
│         └────────┬───────────┘                    │
│                  │                                │
│                  ▼                                │
│          ┌──────────────┐                        │
│          │  RCA Engine  │                        │
│          └──────────────┘                        │
└──────────────────────────────────────────────────┘
```

**Graph Query Example (Cypher)**:

```cypher
// Find root cause of service degradation
MATCH path = (root:Service {name: 'DynamoDB'})
  -[:DEPENDS_ON*1..5]->(dep:Service)
WHERE dep.health_score < 0.7
  AND root.failure_time < dep.failure_time
RETURN path
ORDER BY length(path) DESC
LIMIT 1
```

**Python Integration**:

```python
from neo4j import GraphDatabase
from datetime import datetime
from typing import List, Dict

class CausalGraphEngine:
    """Causal event graph for RCA."""
    
    def __init__(self, uri: str, user: str, password: str):
        self.driver = GraphDatabase.driver(uri, auth=(user, password))
    
    def add_event(self, event: Dict):
        """Add event to causal graph."""
        with self.driver.session() as session:
            session.execute_write(self._create_event, event)
    
    @staticmethod
    def _create_event(tx, event: Dict):
        """Create event node with relationships."""
        query = """
        MERGE (e:Event {id: $event_id})
        SET e.timestamp = $timestamp,
            e.service = $service,
            e.type = $type,
            e.metadata = $metadata
        WITH e
        MATCH (parent:Event {id: $parent_id})
        MERGE (e)-[:CAUSED_BY]->(parent)
        """
        tx.run(query, **event)
    
    def find_root_cause(self, failure_event_id: str) -> List[Dict]:
        """Trace back to root cause."""
        with self.driver.session() as session:
            result = session.execute_read(
                self._find_root_cause_query,
                failure_event_id
            )
            return result
    
    @staticmethod
    def _find_root_cause_query(tx, event_id: str):
        """Query to find root cause chain."""
        query = """
        MATCH path = (failure:Event {id: $event_id})
          -[:CAUSED_BY*]->(root:Event)
        WHERE NOT (root)-[:CAUSED_BY]->()
        RETURN path, length(path) as depth
        ORDER BY depth DESC
        LIMIT 1
        """
        result = tx.run(query, event_id=event_id)
        return [dict(record) for record in result]
```

### 2. Anomaly Detector

**Purpose**: ML-based pattern recognition for automation drift

**Models Deployed**:

| Model | Use Case | Algorithm |
|-------|----------|-----------|
| **Time-Series Anomaly** | Metric drift detection | LSTM, Prophet |
| **Log Anomaly** | Unusual log patterns | BERT, TF-IDF |
| **Trace Anomaly** | Abnormal request flows | Graph Neural Network |
| **Causal Anomaly** | Broken dependencies | Granger Causality |

**Implementation Example**:

```python
import numpy as np
from sklearn.ensemble import IsolationForest
from prophet import Prophet
import pandas as pd

class MultiModalAnomalyDetector:
    """Detects anomalies across metrics, logs, and traces."""
    
    def __init__(self):
        self.metric_model = IsolationForest(contamination=0.1)
        self.log_model = None  # BERTopic or similar
        self.trace_model = None  # GNN model
        self.is_trained = False
    
    def train_metric_model(self, historical_metrics: np.ndarray):
        """Train on historical metric data."""
        self.metric_model.fit(historical_metrics)
        self.is_trained = True
    
    def detect_metric_anomaly(
        self,
        current_metrics: np.ndarray
    ) -> tuple[bool, float]:
        """Detect anomalous metrics."""
        if not self.is_trained:
            raise ValueError("Model not trained")
        
        prediction = self.metric_model.predict(
            current_metrics.reshape(1, -1)
        )
        score = self.metric_model.score_samples(
            current_metrics.reshape(1, -1)
        )[0]
        
        is_anomaly = prediction[0] == -1
        confidence = abs(score)
        
        return is_anomaly, confidence
    
    def forecast_with_prophet(
        self,
        time_series: pd.DataFrame
    ) -> pd.DataFrame:
        """Forecast future values and detect anomalies."""
        model = Prophet(
            changepoint_prior_scale=0.05,
            interval_width=0.95
        )
        model.fit(time_series)
        
        # Forecast next 24 hours
        future = model.make_future_dataframe(periods=24, freq='H')
        forecast = model.predict(future)
        
        return forecast
```

### 3. Metric Federation Layer

**Purpose**: Aggregates MTTD, MTTR, CFI, and ADI into unified dashboards

**Data Sources**:

- CloudWatch / Datadog / Prometheus
- Application logs (CloudWatch Logs, Splunk)
- Distributed traces (X-Ray, Jaeger, Zipkin)
- Custom metrics (StatsD, Telegraf)

**Federation Architecture**:

```yaml
# Prometheus federation config
global:
  scrape_interval: 15s
  evaluation_interval: 15s

scrape_configs:
  - job_name: 'federate'
    scrape_interval: 30s
    honor_labels: true
    metrics_path: '/federate'
    
    params:
      'match[]':
        - '{job="resilience_metrics"}'
        - '{__name__=~"MTTD|MTTR|CFI|BRI|ADI|CIS"}'
    
    static_configs:
      - targets:
          - 'prometheus-us-east-1:9090'
          - 'prometheus-us-west-2:9090'
          - 'prometheus-eu-west-1:9090'
```

**Grafana Dashboard as Code**:

```json
{
  "dashboard": {
    "title": "Resilience Metrics Dashboard",
    "panels": [
      {
        "id": 1,
        "title": "MTTD (Mean Time to Detect)",
        "targets": [
          {
            "expr": "avg(MTTD_seconds)",
            "legendFormat": "MTTD"
          }
        ],
        "alert": {
          "conditions": [
            {
              "evaluator": {
                "params": [60],
                "type": "gt"
              },
              "operator": {
                "type": "and"
              },
              "query": {
                "params": ["A", "5m", "now"]
              },
              "reducer": {
                "params": [],
                "type": "avg"
              },
              "type": "query"
            }
          ],
          "frequency": "60s",
          "handler": 1,
          "name": "MTTD Alert",
          "noDataState": "no_data",
          "notifications": [
            {
              "uid": "pagerduty"
            }
          ]
        }
      }
    ]
  }
}
```

---

## Telemetry Stack

### Recommended Stack

**Metrics**: Prometheus + Grafana  
**Logs**: Loki or ELK Stack  
**Traces**: Jaeger or AWS X-Ray  
**APM**: Datadog or New Relic  
**Causality**: Neo4j + Custom CGE

### Instrumentation Example (OpenTelemetry)

```python
from opentelemetry import trace, metrics
from opentelemetry.exporter.otlp.proto.grpc.trace_exporter import OTLPSpanExporter
from opentelemetry.sdk.trace import TracerProvider
from opentelemetry.sdk.trace.export import BatchSpanProcessor
from opentelemetry.sdk.metrics import MeterProvider
from opentelemetry.sdk.metrics.export import PeriodicExportingMetricReader

# Configure tracing
trace.set_tracer_provider(TracerProvider())
otlp_exporter = OTLPSpanExporter(endpoint="otel-collector:4317")
trace.get_tracer_provider().add_span_processor(
    BatchSpanProcessor(otlp_exporter)
)

# Configure metrics
meter_provider = MeterProvider(
    metric_readers=[
        PeriodicExportingMetricReader(otlp_exporter, export_interval_millis=60000)
    ]
)
metrics.set_meter_provider(meter_provider)

# Create tracer and meter
tracer = trace.get_tracer(__name__)
meter = metrics.get_meter(__name__)

# Create custom metrics
mttd_histogram = meter.create_histogram(
    name="MTTD_seconds",
    description="Mean Time to Detect",
    unit="s"
)

mttr_histogram = meter.create_histogram(
    name="MTTR_seconds",
    description="Mean Time to Recover",
    unit="s"
)

# Instrument function
@tracer.start_as_current_span("apply_dns_change")
def apply_dns_change(record: str, value: str):
    """Apply DNS change with full instrumentation."""
    span = trace.get_current_span()
    span.set_attribute("dns.record", record)
    span.set_attribute("dns.value", value)
    
    # Your logic here
    try:
        result = perform_dns_update(record, value)
        span.set_attribute("dns.success", True)
        return result
    except Exception as e:
        span.set_attribute("dns.success", False)
        span.set_attribute("dns.error", str(e))
        span.record_exception(e)
        raise
```

---

## Dashboard Templates

### Executive Dashboard

**Purpose**: High-level resilience metrics for leadership

**Metrics Displayed**:

- MTTD, MTTR trend (last 30 days)
- CFI trend
- RMI (Resilience Maturity Index)
- Incident count by severity
- Top 5 failure causes

**Implementation**:

```python
# Grafana dashboard provisioning
dashboard_config = {
    "title": "Executive Resilience Dashboard",
    "refresh": "5m",
    "panels": [
        {
            "title": "MTTD Trend (30d)",
            "type": "graph",
            "datasource": "Prometheus",
            "targets": [{"expr": "avg_over_time(MTTD_seconds[30d])"}]
        },
        {
            "title": "Incident Heatmap",
            "type": "heatmap",
            "datasource": "Prometheus",
            "targets": [{"expr": "rate(incidents_total[1h])"}]
        }
    ]
}
```

### SRE Operations Dashboard

**Purpose**: Real-time operational metrics

**Metrics Displayed**:

- Current MTTD, MTTR, MTTM, MTTHI
- Active incidents
- Blast radius index
- Fault isolation ratio
- Automation drift

### Engineering Quality Dashboard

**Purpose**: Code quality and change management

**Metrics Displayed**:

- CFI (Change Failure Index)
- ADI (Automation Drift Index)
- CIS (Causal Integrity Score)
- Deployment frequency
- Lead time for changes

---

## Alert Configuration

### Alert Severity Levels

| Level | Response Time | Escalation | Notification |
|-------|---------------|------------|--------------|
| **P1 (Critical)** | Immediate | Page on-call | PagerDuty, Slack, Email |
| **P2 (High)** | 15 minutes | Slack notification | Slack, Email |
| **P3 (Medium)** | 1 hour | Email only | Email |
| **P4 (Low)** | Next business day | Dashboard only | None |

### Alert Rules (Prometheus)

```yaml
groups:
  - name: resilience_alerts
    interval: 30s
    rules:
      - alert: HighMTTD
        expr: MTTD_seconds > 60
        for: 5m
        labels:
          severity: critical
          team: sre
        annotations:
          summary: "MTTD exceeds target ({{ $value }}s > 60s)"
          description: "Detection latency is {{ $value }}s, exceeding 60s target"
          runbook: "https://runbooks.company.com/high-mttd"
      
      - alert: HighChangeFailureIndex
        expr: CFI_ratio > 0.05
        for: 10m
        labels:
          severity: high
          team: platform
        annotations:
          summary: "Change failure rate is {{ $value }}"
          description: "CFI of {{ $value }} exceeds 0.05 threshold"
          runbook: "https://runbooks.company.com/high-cfi"
      
      - alert: BlastRadiusBreach
        expr: BRI_count > 3
        for: 2m
        labels:
          severity: critical
          team: architecture
        annotations:
          summary: "Failure affecting {{ $value }} services"
          description: "Containment breach - {{ $value }} services impacted"
          runbook: "https://runbooks.company.com/containment-breach"
```

---

## Integration Patterns

### CloudWatch Integration

```python
import boto3
from datetime import datetime, timedelta

class CloudWatchMetrics:
    """Publish resilience metrics to CloudWatch."""
    
    def __init__(self):
        self.cloudwatch = boto3.client('cloudwatch')
        self.namespace = 'Resilience/Metrics'
    
    def publish_mttd(self, value_seconds: float):
        """Publish MTTD metric."""
        self.cloudwatch.put_metric_data(
            Namespace=self.namespace,
            MetricData=[
                {
                    'MetricName': 'MTTD',
                    'Value': value_seconds,
                    'Unit': 'Seconds',
                    'Timestamp': datetime.utcnow(),
                    'StorageResolution': 1  # High-resolution (1-second)
                }
            ]
        )
    
    def get_mttd_stats(self, hours: int = 24) -> dict:
        """Retrieve MTTD statistics."""
        response = self.cloudwatch.get_metric_statistics(
            Namespace=self.namespace,
            MetricName='MTTD',
            StartTime=datetime.utcnow() - timedelta(hours=hours),
            EndTime=datetime.utcnow(),
            Period=3600,  # 1-hour periods
            Statistics=['Average', 'Maximum', 'Minimum']
        )
        return response['Datapoints']
```

### Datadog Integration

```python
from datadog import initialize, api

options = {
    'api_key': 'YOUR_API_KEY',
    'app_key': 'YOUR_APP_KEY'
}

initialize(**options)

# Send custom metric
api.Metric.send(
    metric='resilience.mttd',
    points=[(datetime.now(), 45.0)],
    tags=['env:prod', 'team:sre']
)

# Create monitor
api.Monitor.create(
    type='metric alert',
    query='avg(last_5m):avg:resilience.mttd{*} > 60',
    name='High MTTD Alert',
    message='@pagerduty-sre MTTD exceeds 60s threshold',
    tags=['team:sre'],
    options={
        'notify_no_data': True,
        'no_data_timeframe': 10
    }
)
```

---

## Best Practices

### DO

✅ Instrument all critical paths  
✅ Use structured logging with correlation IDs  
✅ Implement distributed tracing  
✅ Define SLIs before deployment  
✅ Create runbooks for all alerts  
✅ Test alerting pipelines regularly  
✅ Maintain dashboard-as-code  

### DON'T

❌ Log sensitive data (PII, credentials)  
❌ Create alerts without runbooks  
❌ Ignore alert fatigue signals  
❌ Over-instrument non-critical paths  
❌ Forget to version dashboards  
❌ Skip observability in testing  

---

**Last Updated**: October 2025  
**Owner**: Observability Team  
**Review Frequency**: Quarterly
