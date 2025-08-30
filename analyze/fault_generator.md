# AIOpsLab Fault Generator Analysis

## Overview

AIOpsLab's Fault Generator system provides a comprehensive fault injection framework that can introduce failures at multiple system layers. The system is designed to create realistic failure scenarios for evaluating AIOps agents, with automatic recovery mechanisms to ensure clean test environments.

## Architecture

### Base Fault Injector (`aiopslab/generators/fault/base.py`)

The foundation of all fault injection is the `FaultInjector` base class:

```python
class FaultInjector:
    def __init__(self, testbed):
        self.testbed = testbed  # Usually namespace or environment identifier
    
    def _inject(self, fault_type: str, microservices: list[str] = None, duration: str = None):
        # Dynamic method invocation: inject_{fault_type}
        self._invoke_method("inject", fault_type, microservices, duration)
    
    def _recover(self, fault_type: str, microservices: list[str] = None):
        # Dynamic method invocation: recover_{fault_type}
        self._invoke_method("recover", fault_type, microservices)
    
    def _invoke_method(self, action_prefix, *args):
        # Calls inject_{fault_type} or recover_{fault_type} methods
        method_name = f"{action_prefix}_{args[0]}"
        method = getattr(self, method_name, None)
        if method:
            method(*args[1:])
```

**Key Design Principles:**
- **Dynamic Method Dispatch**: Uses naming convention `{action}_{fault_type}` for extensibility
- **Paired Operations**: Every inject operation has corresponding recovery
- **Layer Abstraction**: Base class handles common patterns, subclasses implement specifics
- **Sleep Delays**: Built-in timing controls for fault propagation

## Fault Injection Layers

### 1. Virtualization Layer (`inject_virtual.py`)

**Scope**: Kubernetes, Docker, container orchestration failures

**Class**: `VirtualizationFaultInjector`

#### V.1 - Kubernetes Service Misconfiguration
```python
def inject_misconfig_k8s(self, microservices: list[str]):
    """Misconfigure service's targetPort in Kubernetes."""
    for service in microservices:
        service_config = self._modify_target_port_config(
            from_port=9090, to_port=9999,
            configs=self.kubectl.get_service_json(service, self.testbed)
        )
        self.kubectl.patch_service(service, self.testbed, service_config)

def recover_misconfig_k8s(self, microservices: list[str]):
    """Restore correct targetPort configuration."""
    # Reverse the port change: 9999 -> 9090
```

**Impact**: Breaks service communication, causes connection failures
**Detection**: Service connectivity issues, connection refused errors
**Mitigation**: Correct service port configuration

#### V.2 - MongoDB TLS Authentication Issues
```python
def inject_auth_miss_mongodb(self, microservices: list[str]):
    """Enable TLS for MongoDB without proper certificates."""
    set_values = {
        "url-shorten-mongodb.tls.mode": "requireTLS",
        "url-shorten-mongodb.tls.certificateKeyFile": "/etc/tls/tls.pem",
        "url-shorten-mongodb.tls.CAFile": "/etc/tls/ca.crt",
    }
    Helm.upgrade(release_name="social-network", set_values=set_values)
    # Force pod restart to apply changes
    self.delete_service_pods(target_service_pods)
```

**Impact**: MongoDB connections fail due to TLS requirement without valid certificates
**Detection**: Database connection errors, TLS handshake failures
**Mitigation**: Disable TLS or provide proper certificates

#### V.3 - Pod Scaling to Zero
```python
def inject_scale_pods_to_zero(self, microservices: list[str]):
    """Scale deployment replicas to zero."""
    for service in microservices:
        self.kubectl.exec_command(
            f"kubectl scale deployment {service} --replicas=0 -n {self.namespace}"
        )
```

**Impact**: Complete service unavailability
**Detection**: Service discovery failures, no available endpoints
**Mitigation**: Scale replicas back to desired count

#### V.4 - Node Assignment Issues
```python
def inject_assign_to_non_existent_node(self, microservices: list[str]):
    """Assign pods to non-existent nodes."""
    deployment_yaml["spec"]["template"]["spec"]["nodeSelector"] = {
        "kubernetes.io/hostname": "extra-node"  # Non-existent node
    }
    # Delete and recreate deployment with nodeSelector
```

**Impact**: Pods stuck in Pending state, cannot be scheduled
**Detection**: Pod scheduling failures, Pending status
**Mitigation**: Remove nodeSelector constraint

#### V.5 - Persistent Volume Issues
```python
def inject_redeploy_without_pv(self, app: Application):
    """Delete namespace without cleaning Persistent Volumes."""
    self.kubectl.delete_namespace(self.namespace)
    # Redeploy application - PVs create conflicts
    app.deploy_without_wait()
```

**Impact**: Storage conflicts, mounting failures on redeploy
**Detection**: PVC binding issues, volume mount errors
**Mitigation**: Clean up orphaned PVs before redeployment

#### V.6 - Wrong Binary Usage
```python
def inject_wrong_bin_usage(self, microservices: list[str]):
    """Use incorrect binary for service."""
    containers = deployment_yaml["spec"]["template"]["spec"]["containers"]
    for container in containers:
        if "profile" in container["command"]:
            container["command"] = ["geo"]  # Wrong binary
```

**Impact**: Service functionality broken, wrong behavior
**Detection**: Application logic failures, unexpected responses
**Mitigation**: Restore correct binary/command

#### V.7 - Container Management (Docker)
```python
def inject_container_stop(self, microservices: list[str]):
    """Stop Docker containers."""
    for service in microservices:
        self.docker.get_container(service).stop()

def inject_model_misconfig(self, microservices: list[str]):
    """Misconfigure ML model parameters."""
    command = f"""docker exec -it {service} sh -c "sed -i '24s/84/80/' /app/.flwr/apps/*/task.py" """
    self.docker.exec_command(command)
```

### 2. Application Layer (`inject_app.py`)

**Scope**: Database, application-specific failures

**Class**: `ApplicationFaultInjector`

#### A.1 - MongoDB Admin Privilege Revocation
```python
def inject_revoke_auth(self, microservices: list[str]):
    """Revoke MongoDB admin privileges via shell scripts."""
    for pod in target_mongo_pods:
        if service == "mongodb-rate":
            revoke_command = f"kubectl exec -it {pod} -n {self.namespace} -- /bin/bash /scripts/revoke-admin-rate-mongo.sh"
        elif service == "mongodb-geo":
            revoke_command = f"kubectl exec -it {pod} -n {self.namespace} -- /bin/bash /scripts/revoke-admin-geo-mongo.sh"
        self.kubectl.exec_command(revoke_command)
```

**Impact**: Database access denied, authentication failures
**Recovery**: Execute mitigation scripts to restore admin access
**Scripts**: Located in `generators/fault/script/` directory

#### A.2 - Storage User Registration Issues
```python
def inject_storage_user_unregistered(self, microservices: list[str]):
    """Create unregistered user scenarios in MongoDB."""
    # Manipulates user registration state
```

### 3. OS/Hardware Layer (`inject_os.py`, `inject_hw.py`)

**Scope**: System-level resource failures

- **OS Faults**: Memory pressure, CPU exhaustion, disk I/O issues
- **Hardware Faults**: Simulated hardware failures, resource constraints

### 4. Observability Layer (`inject_otel.py`)

**Scope**: Telemetry and monitoring system failures

- **Metrics Collection Issues**: Prometheus scraping failures
- **Log Processing Problems**: Log shipping interruptions  
- **Trace Collection Faults**: Distributed tracing issues

### 5. Chaos Engineering Integration

#### Chaos Mesh YAML Templates (`chaos-yaml/`)

Pre-defined Chaos Mesh configurations:
- `container-kill.yaml`: Kill specific containers
- `network-delay.yaml`: Introduce network latency
- `network-loss.yaml`: Simulate packet loss
- `pod-failure.yaml`: Pod-level failures
- `pod-kill.yaml`: Terminate pods
- `kernel-faults.yaml`: Kernel-level system faults

**Usage Pattern**:
```python
def inject_network_delay(self, microservices: list[str], duration: str):
    """Apply network delay using Chaos Mesh."""
    chaos_config = self._load_chaos_yaml("network-delay.yaml")
    chaos_config["spec"]["selector"]["labelSelectors"] = self._build_selectors(microservices)
    self.kubectl.apply_yaml(chaos_config)
```

### 6. eBPF-based Fault Injection (`bpf_injector/`)

**Advanced System-Level Faults**:
- **err_inject.bpf.c**: eBPF program for system call interception
- **err_inject.c**: User-space controller
- **Makefile**: Build system for eBPF components

**Capabilities**:
- System call failure injection
- Kernel-level error simulation
- Fine-grained fault targeting

## Fault Injection Patterns

### 1. Configuration Manipulation
```python
def _modify_target_port_config(self, from_port: int, to_port: int, configs: dict):
    """Helper to modify Kubernetes service configurations."""
    for port in configs["spec"]["ports"]:
        if port.get("targetPort") == from_port:
            port["targetPort"] = to_port
    return configs
```

### 2. Helm Value Manipulation
```python
set_values = {
    "service.tls.mode": "requireTLS",
    "service.tls.certificateKeyFile": "/etc/tls/tls.pem",
}
Helm.upgrade(release_name="app", set_values=set_values)
```

### 3. YAML Deployment Modification
```python
def _get_deployment_yaml(self, service_name: str):
    """Fetch current deployment configuration."""
    deployment_yaml = self.kubectl.exec_command(
        f"kubectl get deployment {service_name} -n {self.namespace} -o yaml"
    )
    return yaml.safe_load(deployment_yaml)

def _write_yaml_to_file(self, service_name: str, yaml_content: dict):
    """Write modified YAML for redeployment."""
    file_path = f"/tmp/{service_name}_modified.yaml"
    with open(file_path, "w") as file:
        yaml.dump(yaml_content, file)
    return file_path
```

### 4. Pod Lifecycle Management
```python
def delete_service_pods(self, target_service_pods: list[str]):
    """Force pod restart to apply fault changes."""
    for pod in target_service_pods:
        delete_pod_command = f"kubectl delete pod {pod} -n {self.namespace}"
        self.kubectl.exec_command(delete_pod_command)
```

## Fault Lifecycle Management

### 1. Fault Injection Phase
```python
with CriticalSection():  # Atomic operation protection
    prob.inject_fault()
    atexit.register(exit_cleanup_fault, prob=prob)  # Ensure cleanup
```

### 2. Fault Propagation
```python
time.sleep(6)  # Allow fault effects to propagate
```

### 3. Workload Generation
```python
wrk = Wrk(rate=10, dist="exp", connections=2, duration=10, threads=2)
wrk.start_workload(payload_script=script_path, url=target_url)
```

### 4. Agent Interaction
- Agent observes fault effects through telemetry APIs
- Agent takes corrective actions
- System monitors agent effectiveness

### 5. Fault Recovery
```python
def recover_fault(self):
    """Restore system to healthy state."""
    injector = VirtualizationFaultInjector(namespace=self.namespace)
    injector._recover("misconfig_k8s", [self.faulty_service])
```

### 6. Health Verification
```python
def sys_status_after_recovery(self) -> bool:
    """Verify all pods are healthy after recovery."""
    pod_list = self.kubectl.list_pods(self.namespace)
    for pod in pod_list.items:
        # Check for CrashLoopBackOff, terminated containers, etc.
    return all_normal
```

## Integration with Problem Pool

### Fault Injection in Problems
```python
class K8STargetPortMisconfigBaseTask:
    def inject_fault(self):
        injector = VirtualizationFaultInjector(namespace=self.namespace)
        injector._inject("misconfig_k8s", [self.faulty_service])
    
    def recover_fault(self):
        injector = VirtualizationFaultInjector(namespace=self.namespace)
        injector._recover("misconfig_k8s", [self.faulty_service])
```

### Fault Categories by Layer
- **Virtualization (V)**: 7 fault types
- **Application (A)**: 2+ fault types  
- **OS/Hardware (O/H)**: System resource faults
- **Observability (T)**: Telemetry faults
- **Chaos (C)**: Chaos engineering faults

## Helper Systems

### 1. Script Repository (`script/`)
Pre-written shell scripts for complex fault scenarios:
- MongoDB admin manipulation scripts
- Database initialization scripts
- Service recovery procedures

### 2. YAML Templates
Reusable Chaos Mesh and Kubernetes configurations for common fault patterns

### 3. eBPF Tooling
Advanced system-level fault injection capabilities for kernel-level testing

## Quality Assurance

### Fault Validation
- Every fault injection includes recovery mechanism
- Automatic cleanup via exit handlers
- Health verification after recovery
- Critical section protection for atomicity

### Error Handling
- Graceful degradation when faults cannot be applied
- Detailed logging of fault injection steps
- Validation of system state before/after faults

### Reproducibility
- Deterministic fault injection procedures
- Consistent timing and propagation delays
- Standardized recovery verification

The Fault Generator system provides the foundation for creating realistic failure scenarios that test AIOps agents across the full spectrum of cloud system failures, from configuration issues to hardware faults.