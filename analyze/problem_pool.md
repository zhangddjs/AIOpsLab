# AIOpsLab Problem Pool Analysis

## Overview

AIOpsLab's Problem Pool is a comprehensive collection of standardized AIOps problems designed to evaluate autonomous agents across four core tasks: Detection, Localization, Analysis, and Mitigation. The pool contains 70+ problems spanning multiple applications and fault categories.

## Architecture

### Problem Registry (`aiopslab/orchestrator/problems/registry.py`)

The `ProblemRegistry` class serves as the central hub for all problems in AIOpsLab:

```python
class ProblemRegistry:
    def __init__(self):
        self.PROBLEM_REGISTRY = {
            # Problem ID -> Lambda function returning problem instance
            "k8s_target_port-misconfig-detection-1": lambda: K8STargetPortMisconfigDetection(
                faulty_service="user-service"
            ),
            # ... 70+ more problems
        }
        self.DOCKER_REGISTRY = [...]  # Docker-based problems
```

**Key Features:**
- **Lazy Loading**: Problems use lambda functions for efficient memory usage
- **Naming Convention**: `{fault_category}-{task_type}-{variant}`
- **Deployment Separation**: K8s vs Docker deployment modes
- **Dynamic Filtering**: Search by task type, count problems, etc.

### Problem Structure

Each problem follows a consistent 5-component architecture:

1. **Application**: Target microservice system (SocialNetwork, HotelReservation, etc.)
2. **Task**: AIOps task type (Detection/Localization/Analysis/Mitigation)
3. **Fault**: Specific failure mode injected
4. **Workload**: Traffic pattern generated during testing
5. **Evaluator**: Metrics and success criteria

## Problem Categories

### 1. Kubernetes Configuration Issues
- **Target Port Misconfigurations** (`k8s_target_port-misconfig-*`)
  - Services: user-service, text-service, post-storage-service
  - Fault: Service targetPort changed from 9090 to 9999
  - Tasks: All 4 task types supported

### 2. Database Authentication Problems
- **MongoDB Auth Missing** (`auth_miss_mongodb-*`)
  - Fault: Enable TLS without proper certificates
  - Recovery: Disable TLS mode
  
- **MongoDB Auth Revoked** (`revoke_auth_mongodb-*`)
  - Services: mongodb-geo, mongodb-rate
  - Fault: Remove admin privileges via shell scripts
  - Recovery: Restore admin access

- **MongoDB User Unregistered** (`user_unregistered_mongodb-*`)
  - Fault: Create database user registration issues

### 3. Application-Level Failures
- **Application Misconfiguration** (`misconfig_app_hotel_res-*`)
  - Target: HotelReservation application
  - Configuration-based failures

### 4. Resource Management Issues
- **Pod Scaling Problems** (`scale_pod_zero_social_net-*`)
  - Fault: Scale deployment replicas to 0
  - Recovery: Scale back to 1 replica

- **Node Assignment Issues** (`assign_to_non_existent_node_social_net-*`)
  - Fault: Assign pods to non-existent nodes via nodeSelector
  - Recovery: Remove nodeSelector constraint

### 5. Chaos Engineering Faults
- **Container Kill** (`container_kill-*`)
- **Pod Failure** (`pod_failure_hotel_res-*`)
- **Pod Kill** (`pod_kill_hotel_res-*`)
- **Network Issues**
  - Network Loss (`network_loss_hotel_res-*`)
  - Network Delay (`network_delay_hotel_res-*`)

### 6. OpenTelemetry Demo (Astronomy Shop) Problems
Feature flag and service-specific failures:
- Ad Service Failure/High CPU/Manual GC
- Cart Service Failure
- Image Slow Load
- Kafka Queue Problems
- Payment Service Issues
- Product Catalog Failures
- Recommendation Service Cache Issues

### 7. Operational Errors
- **Wrong Binary Usage** (`wrong_bin_usage-*`)
  - Fault: Use 'geo' binary instead of 'profile'
  - Recovery: Restore correct binary

- **Redeploy Without PV** (`redeploy_without_PV-*`)
  - Fault: Delete namespace without cleaning Persistent Volumes
  - Creates storage conflicts on redeployment

### 8. Machine Learning Workloads
- **Flower Framework Issues**
  - Node Stop (`flower_node_stop-detection`)
  - Model Misconfiguration (`flower_model_misconfig-detection`)
  - Deployed via Docker instead of Kubernetes

## Problem Implementation Pattern

### Base Task Structure
```python
class K8STargetPortMisconfigBaseTask:
    def __init__(self, faulty_service: str = "user-service"):
        self.app = SocialNetwork()
        self.kubectl = KubeCtl()
        self.namespace = self.app.namespace
        self.faulty_service = faulty_service

    def start_workload(self):
        # Generate traffic using wrk2
        wrk = Wrk(rate=10, dist="exp", connections=2, duration=10, threads=2)
        wrk.start_workload(payload_script=self.payload_script, url=frontend_url)

    def inject_fault(self):
        # Use fault injector to introduce specific failure
        injector = VirtualizationFaultInjector(namespace=self.namespace)
        injector._inject("misconfig_k8s", [self.faulty_service])

    def recover_fault(self):
        # Restore system to healthy state
        injector.recover("misconfig_k8s", [self.faulty_service])
```

### Task Inheritance Pattern
```python
class K8STargetPortMisconfigDetection(K8STargetPortMisconfigBaseTask, DetectionTask):
    def eval(self, soln: Any, trace: list[SessionItem], duration: float):
        expected_solution = "Yes"
        if soln.strip().lower() == expected_solution.lower():
            self.add_result("Detection Accuracy", "Correct")
        else:
            self.add_result("Detection Accuracy", "Incorrect")
        return super().eval(soln, trace, duration)
```

## Task-Specific Evaluation

### Detection Tasks
- **Input**: Agent observes system state
- **Output**: Binary answer (Yes/No for anomaly presence)
- **Evaluation**: Exact match comparison

### Localization Tasks  
- **Input**: System telemetry and observations
- **Output**: List of faulty components
- **Evaluation**: 
  - Exact match (100% accuracy)
  - Subset match (proportional accuracy)
  - No match (0% accuracy)

### Analysis Tasks
- **Input**: Localized fault information
- **Output**: Structured analysis with system_level and fault_type
- **Evaluation**: Boolean correctness for each field
- **Example**: `{"system_level": "Virtualization", "fault_type": "Misconfiguration"}`

### Mitigation Tasks
- **Input**: Fault analysis results
- **Output**: Actions to fix the system
- **Evaluation**: System health verification after agent actions
- **Success Criteria**: All pods running, no CrashLoopBackOff states, correct configurations restored

## Registry Operations

### Core Methods
```python
# Get problem instance (creates new instance)
problem = registry.get_problem_instance("k8s_target_port-misconfig-detection-1")

# Get problem factory function
factory = registry.get_problem("k8s_target_port-misconfig-detection-1")

# Filter problems by task type
detection_problems = registry.get_problem_ids(task_type="detection")

# Count problems
total_count = registry.get_problem_count()
detection_count = registry.get_problem_count(task_type="detection")

# Check deployment type
deployment = registry.get_problem_deployment("flower_node_stop-detection")  # Returns "docker"
```

### Problem Distribution
- **Total Problems**: 70+
- **Detection**: ~25 problems
- **Localization**: ~20 problems  
- **Analysis**: ~15 problems
- **Mitigation**: ~15 problems
- **Docker-based**: 2 problems (Flower)
- **K8s-based**: 68+ problems

## Extensibility

### Adding New Problems
1. **Create Problem Class**: Inherit from appropriate task type
2. **Implement Required Methods**: `start_workload()`, `inject_fault()`, `eval()`
3. **Register in Registry**: Add entry to `PROBLEM_REGISTRY` dictionary
4. **Update Docker Registry**: If Docker-based, add to `DOCKER_REGISTRY`

### Problem Naming Convention
Format: `{fault_category}_{application}_{task_type}_{variant}`
- `fault_category`: Type of fault (misconfig, auth_miss, scale_pod, etc.)
- `application`: Target app (hotel_res, social_net, astronomy_shop)
- `task_type`: detection, localization, analysis, mitigation
- `variant`: Numeric identifier for variations (1, 2, 3...)

## Quality Assurance

### Problem Validation
- Each problem includes fault injection AND recovery mechanisms
- Automatic cleanup via `atexit` handlers
- Critical section protection for fault injection atomicity
- System health verification after recovery

### Consistency Checks
- Standardized evaluation metrics across similar problems
- Consistent workload generation patterns
- Uniform error handling and logging
- Reproducible fault injection procedures

The Problem Pool serves as the foundation for systematic AIOps agent evaluation, providing diverse, realistic scenarios that test agent capabilities across the full incident response lifecycle.