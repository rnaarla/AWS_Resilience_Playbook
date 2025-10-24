# Chaos Simulation Framework

## Overview

This document provides a comprehensive framework for chaos engineering tests designed to validate fault containment, automation resilience, and recovery procedures in distributed systems.

## Table of Contents

1. [Testing Philosophy](#testing-philosophy)
2. [Test Scenarios](#test-scenarios)
3. [Automation Scripts](#automation-scripts)
4. [Validation Criteria](#validation-criteria)
5. [Reporting Templates](#reporting-templates)
6. [Best Practices](#best-practices)

---

## Testing Philosophy

### Principles of Chaos Engineering

1. **Hypothesize about steady state**: Define normal behavior metrics
2. **Vary real-world events**: Simulate realistic failure scenarios
3. **Run experiments in production**: Test where it matters (with safety controls)
4. **Automate experiments**: Make chaos continuous
5. **Minimize blast radius**: Start small, gradually increase scope

### Safety Controls

- **Blast radius limits**: Maximum 5% of production traffic
- **Emergency abort**: Kill-switch for immediate halt
- **Human supervision**: On-call engineer must approve
- **Time windows**: Only during business hours initially
- **Gradual rollout**: Canary → 10% → 50% → 100%

---

## Test Scenarios

### Scenario 1: DNS Race Condition

**Objective**: Validate quorum prevents active DNS plan deletion

**Hypothesis**: Multiple Enactors with latency skew will reach consensus and prevent race conditions

**Setup**:

1. Deploy three Enactors across 3 AZs (us-east-1a, us-east-1b, us-east-1c)
2. Configure intentional network latency skew:
   - AZ-1a: 0ms baseline
   - AZ-1b: +100ms
   - AZ-1c: +200ms
3. Enable quorum consensus (2-of-3 requirement)
4. Configure fencing tokens with 5-second validity

**Execution Steps**:

```bash
#!/bin/bash
# chaos_dns_race_condition.sh

# Step 1: Deploy test Enactors
chaos deploy enactors \
  --zones us-east-1a,us-east-1b,us-east-1c \
  --latency-skew 0,100,200 \
  --quorum 2-of-3

# Step 2: Create test DNS record
aws route53 change-resource-record-sets \
  --hosted-zone-id Z1234567890ABC \
  --change-batch '{
    "Changes": [{
      "Action": "CREATE",
      "ResourceRecordSet": {
        "Name": "test.chaos.example.com",
        "Type": "A",
        "TTL": 60,
        "ResourceRecords": [{"Value": "10.0.0.1"}]
      }
    }]
  }'

# Step 3: Simulate concurrent operations
chaos inject concurrent-updates \
  --enactor-1 "cleanup:delete" \
  --enactor-2 "update:10.0.0.2" \
  --enactor-3 "read:verify" \
  --timing simultaneous \
  --duration 5m

# Step 4: Monitor and collect results
chaos monitor \
  --metrics quorum_votes,fencing_tokens,dns_state \
  --output /tmp/chaos-results.json

# Step 5: Validate results
chaos validate \
  --criteria no_active_deletion \
  --criteria quorum_consensus \
  --criteria state_consistency
```

**Success Criteria**:

- ✅ Quorum blocks delete until validation
- ✅ Update overrides delete due to temporal ordering
- ✅ No inconsistent state observed across AZs
- ✅ Recovery completes within 30 seconds
- ✅ Zero data loss or corruption
- ✅ All changes logged in audit trail

**Expected Metrics**:

| Metric | Target | Threshold |
|--------|--------|-----------|
| Quorum latency | < 500ms | < 1000ms |
| State consistency | 100% | > 99% |
| Recovery time | < 30s | < 60s |
| False positives | 0 | < 1 |

**Implementation (Python)**:

```python
import asyncio
import time
from typing import List, Dict
from dataclasses import dataclass

@dataclass
class EnactorConfig:
    """Configuration for chaos Enactor."""
    az: str
    latency_ms: int
    action: str

class DNSChaosTest:
    """Orchestrates DNS race condition chaos test."""
    
    def __init__(self, enactors: List[EnactorConfig]):
        self.enactors = enactors
        self.results = []
    
    async def run_test(self) -> Dict:
        """Execute chaos test."""
        print("Starting DNS race condition test...")
        
        # Start all Enactors simultaneously
        tasks = [
            self.execute_enactor(enactor)
            for enactor in self.enactors
        ]
        
        start_time = time.time()
        results = await asyncio.gather(*tasks, return_exceptions=True)
        duration = time.time() - start_time
        
        # Analyze results
        analysis = self.analyze_results(results, duration)
        
        return analysis
    
    async def execute_enactor(self, enactor: EnactorConfig) -> Dict:
        """Execute Enactor action with latency injection."""
        # Simulate network latency
        await asyncio.sleep(enactor.latency_ms / 1000.0)
        
        print(f"[{enactor.az}] Executing: {enactor.action}")
        
        # Simulate action (mock for testing)
        if enactor.action == "delete":
            return await self.attempt_delete(enactor.az)
        elif enactor.action.startswith("update:"):
            value = enactor.action.split(":")[1]
            return await self.attempt_update(enactor.az, value)
        elif enactor.action == "read":
            return await self.attempt_read(enactor.az)
    
    async def attempt_delete(self, az: str) -> Dict:
        """Attempt DNS record deletion."""
        # Quorum validation would happen here
        return {
            "az": az,
            "action": "delete",
            "status": "blocked_by_quorum",
            "timestamp": time.time()
        }
    
    async def attempt_update(self, az: str, value: str) -> Dict:
        """Attempt DNS record update."""
        return {
            "az": az,
            "action": f"update:{value}",
            "status": "committed",
            "timestamp": time.time()
        }
    
    async def attempt_read(self, az: str) -> Dict:
        """Read DNS record state."""
        return {
            "az": az,
            "action": "read",
            "status": "consistent",
            "timestamp": time.time()
        }
    
    def analyze_results(self, results: List[Dict], duration: float) -> Dict:
        """Analyze test results."""
        success = all(
            r.get("status") in ["committed", "consistent", "blocked_by_quorum"]
            for r in results if isinstance(r, dict)
        )
        
        return {
            "success": success,
            "duration_seconds": duration,
            "results": results,
            "validation": {
                "quorum_worked": True,
                "state_consistent": True,
                "recovery_time": duration
            }
        }

# Usage
if __name__ == "__main__":
    enactors = [
        EnactorConfig("us-east-1a", 0, "delete"),
        EnactorConfig("us-east-1b", 100, "update:10.0.0.2"),
        EnactorConfig("us-east-1c", 200, "read")
    ]
    
    test = DNSChaosTest(enactors)
    results = asyncio.run(test.run_test())
    
    print(f"\nTest Results: {results}")
```

---

### Scenario 2: Control Plane Failure

**Objective**: Validate shadow plane promotion and failover

**Hypothesis**: Shadow control plane can be promoted within 5 minutes with zero state loss

**Setup**:

1. Primary control plane serving 3 regions
2. Shadow plane in read-only mode, replicating all state
3. Audit plane logging all events
4. Health checks running every 30 seconds

**Execution**:

```bash
#!/bin/bash
# chaos_control_plane_failure.sh

# Step 1: Verify initial state
chaos verify state \
  --primary healthy \
  --shadow synced \
  --audit logging

# Step 2: Inject control plane failure
chaos inject failure \
  --target primary-control-plane \
  --method process-kill \
  --notification enabled

# Step 3: Monitor detection
chaos monitor detection \
  --expected-mttd 60s \
  --alert-channels pagerduty

# Step 4: Trigger shadow promotion
chaos trigger promotion \
  --source shadow-control-plane \
  --require-human-approval true \
  --validation-checks all

# Step 5: Verify recovery
chaos verify recovery \
  --state-consistency 100% \
  --zero-data-loss true \
  --audit-complete true

# Step 6: Generate report
chaos report \
  --output /tmp/control-plane-failure-report.html
```

**Success Criteria**:

- ✅ MTTD < 60 seconds
- ✅ Shadow promotion < 5 minutes
- ✅ Zero state loss
- ✅ All changes logged in audit plane
- ✅ Health checks pass post-recovery
- ✅ No service degradation for end users

---

### Scenario 3: Cascading Failure

**Objective**: Validate circuit breakers prevent cascade across services

**Setup**:

1. DynamoDB → Lambda → API Gateway → Client chain
2. Circuit breakers configured at each layer
3. Bulkhead isolation enabled
4. Rate limiters active

**Execution**:

```python
class CascadeFailureTest:
    """Test circuit breaker effectiveness."""
    
    def __init__(self):
        self.services = {
            'dynamodb': ServiceMock('DynamoDB'),
            'lambda': ServiceMock('Lambda'),
            'api_gateway': ServiceMock('APIGateway')
        }
    
    async def run_cascade_test(self):
        """Inject failure and verify containment."""
        
        # Step 1: Inject DynamoDB failure
        self.services['dynamodb'].inject_failure(
            failure_rate=0.50,  # 50% error rate
            duration=300  # 5 minutes
        )
        
        # Step 2: Monitor propagation
        start_time = time.time()
        while time.time() - start_time < 300:
            metrics = await self.collect_metrics()
            
            # Verify circuit breakers engaged
            assert metrics['lambda']['circuit_state'] == 'OPEN'
            assert metrics['api_gateway']['error_rate'] < 0.05
            
            await asyncio.sleep(10)
        
        # Step 3: Verify containment
        assert self.services['lambda'].total_failures < 100
        assert self.services['api_gateway'].total_failures < 10
        
        print("✅ Cascade contained successfully")
```

---

## Automation Scripts

### Weekly Automated Chaos Tests

```python
# chaos_scheduler.py
from apscheduler.schedulers.blocking import BlockingScheduler
from datetime import datetime

scheduler = BlockingScheduler()

@scheduler.scheduled_job('cron', day_of_week='mon', hour=10)
def weekly_dns_race_test():
    """Run DNS race condition test every Monday at 10 AM."""
    print(f"Starting weekly DNS race test at {datetime.now()}")
    # Execute test
    os.system('./chaos_dns_race_condition.sh')

@scheduler.scheduled_job('cron', day_of_week='wed', hour=14)
def weekly_control_plane_test():
    """Run control plane failure test every Wednesday at 2 PM."""
    print(f"Starting control plane test at {datetime.now()}")
    os.system('./chaos_control_plane_failure.sh')

if __name__ == '__main__':
    scheduler.start()
```

---

## Validation Criteria

### Automated Validation

```python
class ChaosValidator:
    """Validates chaos test results."""
    
    def validate_test_results(self, results: Dict) -> bool:
        """Run all validation checks."""
        checks = [
            self.check_quorum_consensus(results),
            self.check_state_consistency(results),
            self.check_recovery_time(results),
            self.check_blast_radius(results),
            self.check_audit_completeness(results)
        ]
        
        return all(checks)
    
    def check_quorum_consensus(self, results: Dict) -> bool:
        """Verify quorum worked correctly."""
        votes = results.get('quorum_votes', [])
        return len([v for v in votes if v['approved']]) >= 2
    
    def check_state_consistency(self, results: Dict) -> bool:
        """Verify state is consistent across AZs."""
        states = results.get('az_states', {})
        return len(set(states.values())) == 1
    
    def check_recovery_time(self, results: Dict) -> bool:
        """Verify recovery within SLA."""
        recovery_time = results.get('recovery_time_seconds', float('inf'))
        return recovery_time < 30
```

---

## Reporting Templates

### Test Report Template

```markdown
# Chaos Engineering Test Report

**Date**: {test_date}
**Test**: {test_name}
**Engineer**: {engineer_name}

## Executive Summary

- **Status**: {PASS/FAIL}
- **Duration**: {duration}
- **Blast Radius**: {affected_services}
- **Recovery Time**: {recovery_time}

## Test Configuration

- **Scenario**: {scenario_name}
- **Target**: {target_system}
- **Failure Mode**: {failure_mode}
- **Duration**: {test_duration}

## Results

### Metrics

| Metric | Target | Actual | Status |
|--------|--------|--------|--------|
| MTTD | < 60s | {actual_mttd}s | {PASS/FAIL} |
| MTTR | < 30m | {actual_mttr}m | {PASS/FAIL} |
| BRI | < 3 | {actual_bri} | {PASS/FAIL} |

### Observations

- {observation_1}
- {observation_2}

## Recommendations

1. {recommendation_1}
2. {recommendation_2}

## Follow-up Actions

- [ ] {action_1}
- [ ] {action_2}
```

---

## Best Practices

### DO

✅ Run tests weekly in non-prod first  
✅ Get approval before production tests  
✅ Start with small blast radius  
✅ Monitor continuously during tests  
✅ Document all findings  
✅ Automate test execution  
✅ Share results with team  

### DON'T

❌ Run tests without safety controls  
❌ Skip validation steps  
❌ Ignore test failures  
❌ Test during peak traffic  
❌ Forget to clean up resources  
❌ Run multiple tests simultaneously  

---

**Last Updated**: October 2025  
**Owner**: Chaos Engineering Team  
**Review Frequency**: After each test
