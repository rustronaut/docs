# ⚖️ Load Balancing

## Overview

Rustronaut's load balancing system provides intelligent traffic distribution across multiple gateways, optimizing performance, reliability, and resource utilization.

**Status**: ✅ **Production Ready**

## 📋 Documentation

- **[Strategies](strategies.md)** - Comprehensive comparison of all load balancing strategies
  - Round Robin: Simple sequential distribution
  - Least Connections: Resource-aware balancing
  - Weighted Random: Health-based probabilistic selection
  - Latency-Based: Performance-optimized routing
  - Region-Aware: Geographic proximity routing

## 🔧 Available Strategies

### 🔄 Round Robin
**Best For**: Simple deployments with similar gateway capacities
- Even distribution across all gateways
- Predictable behavior and easy troubleshooting
- No special configuration required

### 📊 Least Connections
**Best For**: Balanced resource utilization
- Routes to gateway with fewest active connections
- Requires accurate connection tracking
- Optimal for varying connection durations

### 🎲 Weighted Random
**Best For**: General-purpose with varied gateway capacities
- Probabilistic selection based on health metrics
- Adapts to gateway capacity and performance
- Good balance of simplicity and intelligence

### ⚡ Latency-Based
**Best For**: Performance-critical applications
- Prioritizes gateways with lowest response times
- Requires active latency measurements
- Optimal for real-time applications

### 🌍 Region-Aware
**Best For**: Geographically distributed deployments
- Routes clients to geographically closer gateways
- Reduces network latency and improves performance
- Requires proper region configuration

## ⚙️ Configuration

Load balancing strategy is configured in `config/local.toml`:

```toml
[orchestrator]
load_balancing_strategy = "RoundRobin"  # or LeastConnections, WeightedRandom, LatencyBased, RegionAware
health_threshold_percent = 70.0
```

## 📊 Performance Characteristics

| Strategy | Complexity | Adaptivity | Performance | Use Case |
|----------|------------|------------|-------------|----------|
| **Round Robin** | Low | None | Good | Simple deployments |
| **Least Connections** | Medium | Medium | Very Good | Balanced loads |
| **Weighted Random** | Medium | High | Excellent | General purpose |
| **Latency-Based** | High | High | Excellent | Performance critical |
| **Region-Aware** | High | Medium | Good | Global deployments |

## 🧭 Navigation

- **Parent**: [🎯 Features](../README.md)
- **Related**: [🌐 Multi-Gateway](../multi-gateway/README.md) | [🛠️ Implementation](../../implementation/README.md)
