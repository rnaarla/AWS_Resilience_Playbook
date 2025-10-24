# Fault Containment Zones (FCZ)

## Overview

Fault Containment Zones (FCZ) are logical and physical boundaries that limit the propagation of failures in distributed systems. By segmenting services into independent domains, we prevent cascading failures and contain blast radius to acceptable levels.

## Table of Contents

1. [Containment Principles](#containment-principles)
2. [Zone Hierarchy](#zone-hierarchy)
3. [Implementation Patterns](#implementation-patterns)
4. [Monitoring and Detection](#monitoring-and-detection)
5. [Recovery Procedures](#recovery-procedures)
6. [Testing Strategies](#testing-strategies)

---

## Containment Principles

### 1. Blast Radius Limitation

**Objective**: Limit failure impact to smallest possible scope

**Metrics**:

- Maximum 1 dependent service impacted
- Maximum 1 availability zone affected
- Maximum 5% of total capacity at risk

### 2. Failure Isolation

**Objective**: Prevent failure propagation across boundaries

**Mechanisms**:

- Circuit breakers
- Bulkheads
- Rate limiters
- Timeout enforcement

### 3. Independent Recovery

**Objective**: Each zone recovers without external dependencies

**Requirements**:

- Self-contained health checks
- Local state management
- Independent control planes
- Autonomous remediation

---

## Zone Hierarchy

### Level 1: AZ-Level Containment

Each AWS-style service must be segmented at the Availability Zone level, with automation operating independently within its AZ.

**Isolation Period**: 60 minutes  
**Cross-AZ Communication**: Limited to read-only during recovery  
**Failover**: Automatic within 5 minutes

**Architecture**:

```text
┌─────────────────────────────────────────────────┐
│                  Region: us-east-1               │
│                                                  │
│  ┌─────────────┐  ┌─────────────┐  ┌──────────┐│
│  │  AZ-1a      │  │  AZ-1b      │  │  AZ-1c   ││
│  │             │  │             │  │          ││
│  │ ┌─────────┐ │  │ ┌─────────┐ │  │┌────────┐││
│  │ │ Service │ │  │ │ Service │ │  ││Service │││
│  │ │ Instance│ │  │ │ Instance│ │  ││Instance│││
│  │ └─────────┘ │  │ └─────────┘ │  │└────────┘││
│  │             │  │             │  │          ││
│  │ ┌─────────┐ │  │ ┌─────────┐ │  │┌────────┐││
│  │ │  Local  │ │  │ │  Local  │ │  ││ Local  │││
│  │ │ Control │ │  │ │ Control │ │  ││Control │││
│  │ └─────────┘ │  │ └─────────┘ │  │└────────┘││
│  └─────────────┘  └─────────────┘  └──────────┘│
│         │                 │                │     │
│         └─────────────────┼────────────────┘     │
│                           │                      │
│                    Regional Router               │
└─────────────────────────────────────────────────┘
```

**Implementation Example (Terraform)**:

```hcl
# AZ-level fault containment
resource "aws_lb_target_group" "service_az1a" {
  name     = "service-az1a-tg"
  port     = 8080
  protocol = "HTTP"
  vpc_id   = aws_vpc.main.id
  
  health_check {
    enabled             = true
    healthy_threshold   = 2
    unhealthy_threshold = 2
    timeout             = 5
    interval            = 30
    path                = "/health"
  }
  
  deregistration_delay = 30
  
  # Sticky sessions for AZ affinity
  stickiness {
    type            = "lb_cookie"
    cookie_duration = 86400
    enabled         = true
  }
  
  tags = {
    ContainmentZone = "az1a"
    FailoverTime    = "300s"
  }
}

# Circuit breaker implementation
resource "aws_cloudwatch_metric_alarm" "az1a_failure_rate" {
  alarm_name          = "service-az1a-high-failure-rate"
  comparison_operator = "GreaterThanThreshold"
  evaluation_periods  = 2
  metric_name         = "FailureRate"
  namespace           = "Service/AZ1A"
  period              = 60
  statistic           = "Average"
  threshold           = 0.05
  alarm_description   = "Trigger circuit breaker when failure rate exceeds 5%"
  
  alarm_actions = [
    aws_sns_topic.circuit_breaker.arn
  ]
}
```

### Level 2: Regional Partitioning

Control planes are decoupled by region, preventing global recursion.

**Isolation Period**: 4 hours  
**Cross-Region Communication**: Asynchronous replication only  
**Failover**: Manual promotion with human approval

**Regional Autonomy Requirements**:

- Independent configuration store per region
- Separate IAM control planes
- Regional DNS authority
- Autonomous incident response

**Implementation Pattern**:

```python
from enum import Enum
from typing import Optional

class RegionStatus(Enum):
    HEALTHY = "healthy"
    DEGRADED = "degraded"
    ISOLATED = "isolated"
    RECOVERING = "recovering"

class RegionalControlPlane:
    """Regional control plane with fault isolation."""
    
    def __init__(self, region: str):
        self.region = region
        self.status = RegionStatus.HEALTHY
        self.cross_region_enabled = True
        self.isolation_start: Optional[float] = None
    
    def enter_isolation(self):
        """Isolate region from cross-region dependencies."""
        print(f"[{self.region}] Entering isolation mode")
        self.status = RegionStatus.ISOLATED
        self.cross_region_enabled = False
        self.isolation_start = time.time()
        
        # Disable cross-region replication
        self._disable_replication()
        
        # Switch to local-only mode
        self._enable_local_mode()
        
        # Alert operators
        self._send_alert("Region isolated for fault containment")
    
    def can_communicate_cross_region(self) -> bool:
        """Check if cross-region communication is allowed."""
        if not self.cross_region_enabled:
            return False
        
        # Only allow read operations during isolation
        return self.status == RegionStatus.HEALTHY
    
    def exit_isolation(self, human_approval: bool = False):
        """Exit isolation mode after validation."""
        if not human_approval:
            print(f"[{self.region}] Cannot exit isolation without approval")
            return False
        
        print(f"[{self.region}] Exiting isolation mode")
        self.status = RegionStatus.RECOVERING
        
        # Gradual re-enable
        self._validate_health()
        self._sync_state()
        self.cross_region_enabled = True
        self.status = RegionStatus.HEALTHY
        
        return True
```

### Level 3: Cross-Service Containment

Lambda, EC2, and DynamoDB share no real-time dependency loops.

**Principles**:

- **Service Mesh Boundaries**: Each service has independent control plane
- **API Rate Limiting**: Prevent resource exhaustion cascades
- **Timeout Enforcement**: Fail fast, don't wait indefinitely
- **Fallback Strategies**: Graceful degradation without propagation

**Dependency Graph Example**:

```text
┌──────────────────────────────────────────────────┐
│          Service Dependency Graph                 │
│                                                   │
│   ┌────────┐         ┌────────┐                 │
│   │   EC2  │────X────│DynamoDB│                 │
│   └────────┘         └────────┘                 │
│        │                  │                      │
│        │                  │                      │
│        ▼                  ▼                      │
│   ┌────────┐         ┌────────┐                 │
│   │  ELB   │         │ Lambda │                 │
│   └────────┘         └────────┘                 │
│        │                  │                      │
│        └──────────┬───────┘                      │
│                   │                              │
│                   ▼                              │
│            ┌────────────┐                        │
│            │   Client   │                        │
│            └────────────┘                        │
│                                                   │
│  X = Isolation Boundary (no sync dependencies)   │
└──────────────────────────────────────────────────┘
```

**Circuit Breaker Implementation (Go)**:

```go
package faultisolation

import (
    "context"
    "errors"
    "sync"
    "time"
)

type CircuitState int

const (
    StateClosed CircuitState = iota
    StateOpen
    StateHalfOpen
)

type CircuitBreaker struct {
    maxFailures    int
    resetTimeout   time.Duration
    halfOpenMax    int
    state          CircuitState
    failures       int
    lastFailTime   time.Time
    halfOpenCalls  int
    mu             sync.RWMutex
}

func NewCircuitBreaker(maxFailures int, resetTimeout time.Duration) *CircuitBreaker {
    return &CircuitBreaker{
        maxFailures:  maxFailures,
        resetTimeout: resetTimeout,
        halfOpenMax:  1,
        state:        StateClosed,
    }
}

func (cb *CircuitBreaker) Call(ctx context.Context, fn func() error) error {
    cb.mu.Lock()
    
    // Check if circuit should reset
    if cb.state == StateOpen && time.Since(cb.lastFailTime) > cb.resetTimeout {
        cb.state = StateHalfOpen
        cb.halfOpenCalls = 0
    }
    
    currentState := cb.state
    cb.mu.Unlock()
    
    // Circuit is open, reject immediately
    if currentState == StateOpen {
        return errors.New("circuit breaker open: service unavailable")
    }
    
    // Circuit is half-open, limit concurrent calls
    if currentState == StateHalfOpen {
        cb.mu.Lock()
        if cb.halfOpenCalls >= cb.halfOpenMax {
            cb.mu.Unlock()
            return errors.New("circuit breaker half-open: max calls reached")
        }
        cb.halfOpenCalls++
        cb.mu.Unlock()
    }
    
    // Execute function
    err := fn()
    
    cb.mu.Lock()
    defer cb.mu.Unlock()
    
    if err != nil {
        cb.failures++
        cb.lastFailTime = time.Now()
        
        if cb.failures >= cb.maxFailures {
            cb.state = StateOpen
        }
        return err
    }
    
    // Success - reset circuit
    if cb.state == StateHalfOpen {
        cb.state = StateClosed
    }
    cb.failures = 0
    return nil
}

func (cb *CircuitBreaker) GetState() CircuitState {
    cb.mu.RLock()
    defer cb.mu.RUnlock()
    return cb.state
}
```

---

## Implementation Patterns

### Pattern 1: Bulkhead Pattern

Isolate resources to prevent resource exhaustion:

```python
from concurrent.futures import ThreadPoolExecutor
from typing import Callable, Any

class BulkheadExecutor:
    """Isolates execution pools to prevent resource exhaustion."""
    
    def __init__(self, max_workers: int, queue_size: int):
        self.executor = ThreadPoolExecutor(
            max_workers=max_workers,
            thread_name_prefix="bulkhead"
        )
        self.max_queue_size = queue_size
        self.current_queue_size = 0
    
    def submit(self, fn: Callable, *args, **kwargs) -> Any:
        """Submit task with queue size limit."""
        if self.current_queue_size >= self.max_queue_size:
            raise RuntimeError("Bulkhead queue full - rejecting request")
        
        self.current_queue_size += 1
        future = self.executor.submit(fn, *args, **kwargs)
        future.add_done_callback(lambda _: self._decrement_queue())
        
        return future
    
    def _decrement_queue(self):
        """Decrement queue counter on completion."""
        self.current_queue_size -= 1
```

### Pattern 2: Timeout Enforcement

Prevent indefinite waiting:

```python
import signal
from contextlib import contextmanager

@contextmanager
def timeout(seconds: int):
    """Context manager for timeout enforcement."""
    
    def timeout_handler(signum, frame):
        raise TimeoutError(f"Operation exceeded {seconds}s timeout")
    
    # Set up signal handler
    signal.signal(signal.SIGALRM, timeout_handler)
    signal.alarm(seconds)
    
    try:
        yield
    finally:
        signal.alarm(0)  # Disable alarm

# Usage
try:
    with timeout(5):
        result = slow_external_call()
except TimeoutError:
    # Fallback to cached value or default
    result = get_cached_value()
```

### Pattern 3: Graceful Degradation

Maintain partial functionality during failures:

```python
from typing import Optional, TypeVar, Callable

T = TypeVar('T')

class GracefulService:
    """Service with fallback capabilities."""
    
    def __init__(self):
        self.cache = {}
        self.circuit_breaker = CircuitBreaker(max_failures=5, reset_timeout=60)
    
    def get_data(self, key: str) -> Optional[T]:
        """Get data with graceful degradation."""
        
        # Try primary source
        try:
            return self.circuit_breaker.call(
                lambda: self._fetch_from_primary(key)
            )
        except Exception as e:
            print(f"Primary failed: {e}")
        
        # Fallback to cache
        if key in self.cache:
            print(f"Using cached value for {key}")
            return self.cache[key]
        
        # Fallback to default
        print(f"Using default value for {key}")
        return self._get_default_value(key)
    
    def _fetch_from_primary(self, key: str) -> T:
        """Fetch from primary data source."""
        # Implementation
        pass
    
    def _get_default_value(self, key: str) -> T:
        """Return safe default value."""
        # Implementation
        pass
```

---

## Monitoring and Detection

### Key Metrics

| Metric | Description | Alert Threshold |
|--------|-------------|-----------------|
| **Failure Rate** | % of failed requests per zone | > 5% |
| **Error Propagation** | # of zones affected by single failure | > 1 |
| **Recovery Time** | Time to restore zone health | > 5 min |
| **Cross-Zone Traffic** | % of requests crossing zone boundaries | > 20% |

### Detection Mechanisms

```python
class FaultDetector:
    """Monitors fault containment effectiveness."""
    
    def __init__(self):
        self.zone_health = {}
        self.propagation_graph = {}
    
    def check_containment_breach(self, failed_zone: str) -> bool:
        """Detect if failure is propagating beyond zone."""
        
        affected_zones = self._get_affected_zones(failed_zone)
        
        if len(affected_zones) > 1:
            self._trigger_alert(
                f"Containment breach: {failed_zone} → {affected_zones}"
            )
            return True
        
        return False
    
    def _get_affected_zones(self, origin_zone: str) -> list:
        """Identify zones affected by origin failure."""
        affected = [origin_zone]
        
        for zone, health in self.zone_health.items():
            if zone != origin_zone and health['status'] == 'degraded':
                # Check if degradation started after origin failure
                if self._is_causally_related(origin_zone, zone):
                    affected.append(zone)
        
        return affected
```

---

## Recovery Procedures

### Automated Recovery

1. **Detect**: Health check failure or metric degradation
2. **Isolate**: Remove zone from load balancer rotation
3. **Diagnose**: Run automated diagnostics
4. **Remediate**: Apply known fixes (restart, scale, rollback)
5. **Validate**: Confirm health restoration
6. **Restore**: Gradually restore traffic

### Manual Recovery

1. **Human Override Protocol (HOP)**
2. **Manual inspection and diagnostics**
3. **Targeted remediation**
4. **Gradual traffic restoration**
5. **Post-incident review**

---

## Testing Strategies

### Chaos Engineering Tests

```bash
# Test AZ isolation
chaos-test az-failure \
  --zone us-east-1a \
  --duration 30m \
  --verify-containment

# Test region isolation
chaos-test region-failure \
  --region us-east-1 \
  --verify-cross-region-isolation

# Test service isolation
chaos-test service-cascade \
  --source dynamodb \
  --verify-no-propagation
```

### Gameday Scenarios

1. **Single AZ failure**: Verify other AZs unaffected
2. **Regional failure**: Verify other regions operational
3. **Cascading failure**: Verify circuit breakers engage
4. **Recovery drill**: Verify isolation and restore procedures

---

## Best Practices

### DO

✅ Design for failure containment from day one  
✅ Test isolation boundaries regularly  
✅ Monitor cross-zone dependencies  
✅ Implement circuit breakers everywhere  
✅ Document recovery procedures  
✅ Practice gameday scenarios quarterly  

### DON'T

❌ Create tight coupling between zones  
❌ Allow synchronous cross-region calls  
❌ Skip circuit breaker implementation  
❌ Ignore blast radius metrics  
❌ Wait for failures to test isolation  

---

**Last Updated**: October 2025  
**Owner**: Resilience Engineering Team  
**Review Frequency**: Quarterly
