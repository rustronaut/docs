# Multi-Gateway Orchestration

## Overview

The Multi-Gateway Orchestration module enables Rustronaut to effectively manage, monitor, and coordinate multiple gateway instances across your network. This provides enhanced reliability, scalability, and performance through intelligent load balancing and centralized configuration management.

## Key Features

### 1. Gateway Health Monitoring

The orchestrator continually monitors all registered gateways to track their health status, including:

- CPU usage
- Memory consumption
- Active connections
- Response latency
- Consecutive failures

This data drives intelligent decision-making for load balancing and failure recovery.

### 2. Load Balancing

Multiple strategies for distributing client connections across gateways:

- **Round Robin**: Sequential gateway selection for even distribution
- **Least Connections**: Routes to the gateway with fewest active connections
- **Weighted Random**: Probabilistic selection based on gateway health metrics
- **Latency-Based**: Prioritizes gateways with lowest response times
- **Region-Aware**: Directs clients to geographically closer gateways

Health thresholds automatically exclude degraded gateways from the selection pool.

### 3. Configuration Distribution

Centralized management of configuration across the gateway fleet:

- **Batch Processing**: Efficiently distribute changes to multiple gateways
- **Retry Mechanism**: Automatically retry failed configuration updates
- **Status Tracking**: Monitor the progress and success of configuration changes
- **Targeted Updates**: Apply changes to specific gateways or the entire fleet

### 4. High Availability

Automatic failover capabilities ensure service continuity:

- Detect gateway failures in real-time
- Redirect traffic from failing gateways
- Maintain configuration consistency across the fleet
- Support for primary/backup gateway relationships

## Architecture

The orchestrator consists of several key components:

- **Orchestrator**: Main coordination component that integrates with GatewayManager
- **LoadBalancer**: Handles gateway selection using configurable strategies
- **ConfigDistributor**: Manages configuration distribution across gateways
- **Background Tasks**: Health monitoring and maintenance processes

## API Endpoints

| Endpoint | Method | Description |
|----------|--------|-------------|
| `/api/v1/orchestration/status` | GET | Get orchestration system status |
| `/api/v1/orchestration/load-balancing` | GET | Get current load balancing configuration |
| `/api/v1/orchestration/load-balancing` | POST | Update load balancing strategy |
| `/api/v1/orchestration/select-gateway` | POST | Find optimal gateway based on strategy |
| `/api/v1/orchestration/distribute` | POST | Distribute configuration to gateways |
| `/api/v1/orchestration/distributions/:id` | GET | Get status of a specific distribution |
| `/api/v1/orchestration/distributions` | GET | List all distributions |

## Configuration Options

The orchestrator can be configured in `config/default.toml`:

```toml
[orchestrator]
enabled = true
distribution_batch_size = 5
distribution_retry_count = 3
distribution_retry_delay_ms = 1000
load_balancing_strategy = "RoundRobin"  # RoundRobin, LeastConnections, WeightedRandom, LatencyBased, RegionAware
health_threshold_percent = 70.0
ha_enabled = true
```

## Usage Examples

### Select Optimal Gateway

```rust
// Find the best gateway based on the current load balancing strategy
let gateway_id = orchestrator.select_optimal_gateway(Some("us-west")).await?;

// Use the selected gateway for a client connection
if let Some(id) = gateway_id {
    // Connect to this gateway
}
```

### Distribute Configuration

```rust
// Distribute a new configuration to all gateways
let distribution_id = orchestrator.distribute_configuration(
    ConfigType::Route,
    serde_json::json!({
        "destination": "10.0.0.0/24",
        "gateway": "192.168.1.1"
    }),
    None, // All gateways
).await?;

// Check the status of the distribution
let status = orchestrator.get_distribution_status(&distribution_id).await;
```

### Change Load Balancing Strategy

```rust
// Update to use weighted random selection based on health metrics
orchestrator.update_load_balancing_config(
    LoadBalancingStrategy::WeightedRandom,
    80.0 // health threshold
).await?;
```

## Implementation Details

### Health Monitoring

The health monitoring system runs in the background, periodically checking the status of all gateways. It updates the load balancer with the latest health metrics, enabling it to make informed decisions about gateway selection.

### Configuration Distribution

When distributing configurations, the system follows these steps:

1. Create a distribution record with a unique ID
2. Identify target gateways (specific gateways or all active ones)
3. Send configuration updates in batches
4. Track success/failure status for each gateway
5. Retry failed updates according to configured retry policy
6. Mark distribution as completed when all gateways are updated

### Gateway Selection

The load balancer uses these metrics to determine gateway weights:
- CPU usage: Lower is better
- Memory usage: Lower is better
- Active connections: Fewer is better
- Response time: Lower is better
- Health status: Only gateways above the health threshold are considered
- Region: Closer regions are preferred in region-aware mode

## Best Practices

1. **Gateway Diversity**: Deploy gateways across different regions/datacenters for better resilience
2. **Load Balancing Strategy**: Choose appropriate strategy based on workload patterns
   - Use Region-Aware for geographically distributed clients
   - Use Least Connections for balanced resource utilization
   - Use Weighted Random for general-purpose deployment
3. **Configuration Management**: Test configuration changes on a subset of gateways before full deployment
4. **Monitoring**: Regularly check orchestration status for any distribution failures
5. **Health Thresholds**: Adjust thresholds based on your infrastructure capabilities

## Future Enhancements

1. Advanced gateway role support (primary/secondary/specialized)
2. Canary deployment pattern for configuration changes
3. Predictive scaling based on traffic patterns
4. Custom load balancing strategies for specialized use cases
5. Advanced analytics and recommendations for optimal gateway distribution
