# Multi-Gateway Orchestration Setup Guide

This guide walks you through setting up and configuring Rustronaut's Multi-Gateway Orchestration system to effectively manage multiple gateway instances.

## Prerequisites

- Rustronaut Console setup with database
- At least two gateway instances deployed and registered with the console
- Admin access to the console server

## 1. Enable Orchestration

By default, the Multi-Gateway Orchestration module is enabled in the configuration. If you need to modify settings, edit your `config/local.toml` file with the following:

```toml
[orchestrator]
enabled = true
distribution_batch_size = 5
distribution_retry_count = 3
distribution_retry_delay_ms = 1000
load_balancing_strategy = "RoundRobin"
health_threshold_percent = 70.0
ha_enabled = true
```

The settings control:
- `enabled`: Turn orchestration on or off
- `distribution_batch_size`: Number of gateways to update in a single batch
- `distribution_retry_count`: How many times to retry failed updates
- `distribution_retry_delay_ms`: Delay between retry attempts
- `load_balancing_strategy`: Strategy for gateway selection (options: RoundRobin, LeastConnections, WeightedRandom, LatencyBased, RegionAware)
- `health_threshold_percent`: Minimum health percentage for gateway selection
- `ha_enabled`: Enable high availability features

## 2. Start the Console with Orchestration

Restart your console service to activate the orchestration settings:

```bash
cd /Volumes/EXT/repos/rustronaut/rustronaut/console
cargo run
```

You should see a log message confirming that orchestration has been initialized:

```
INFO: Multi-gateway orchestration initialized successfully
```

## 3. Verify Gateway Registration

Ensure your gateways are properly registered with the console:

```bash
curl -X GET http://localhost:8080/api/v1/gateways -H "Authorization: Bearer YOUR_TOKEN"
```

You should see at least two gateways with `"status": "online"`.

## 4. Check Orchestration Status

Verify that the orchestration system is working properly:

```bash
curl -X GET http://localhost:8080/api/v1/orchestration/status -H "Authorization: Bearer YOUR_TOKEN"
```

You should see output similar to:

```json
{
  "status": "ok",
  "features": {
    "load_balancing": "enabled",
    "configuration_distribution": "enabled",
    "health_monitoring": "enabled"
  },
  "active_distributions": 0,
  "successfully_completed_distributions": 0,
  "failed_distributions": 0,
  "gateways_managed": 2,
  "load_balancing_strategy": "RoundRobin",
  "health_check_threshold": 70.0
}
```

## 5. Configure Load Balancing Strategy

Choose the most appropriate load balancing strategy for your environment:

```bash
curl -X POST http://localhost:8080/api/v1/orchestration/load-balancing \
  -H "Authorization: Bearer YOUR_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"strategy": "WeightedRandom", "health_threshold": 75.0}'
```

Available strategies:
- `RoundRobin`: Sequential gateway selection
- `LeastConnections`: Select gateway with fewest active connections
- `WeightedRandom`: Probabilistic selection based on health metrics
- `LatencyBased`: Prioritize gateways with lowest response times
- `RegionAware`: Direct clients to geographically closer gateways

## 6. Test Gateway Selection

Test the load balancer by requesting an optimal gateway:

```bash
curl -X POST http://localhost:8080/api/v1/orchestration/select-gateway \
  -H "Authorization: Bearer YOUR_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"region": "us-west"}'
```

The response will include the UUID of the selected gateway:

```json
"550e8400-e29b-41d4-a716-446655440000"
```

## 7. Distribute Configuration to Multiple Gateways

Push a configuration change to all gateways:

```bash
curl -X POST http://localhost:8080/api/v1/orchestration/distribute \
  -H "Authorization: Bearer YOUR_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "config_type": "Route",
    "config_data": {
      "destination": "10.0.0.0/24",
      "gateway": "192.168.1.1"
    },
    "target_gateways": null
  }'
```

This will return a distribution ID:

```json
{
  "distribution_id": "dist-550e8400-e29b-41d4-a716-446655440000",
  "status": "pending"
}
```

## 8. Check Distribution Status

Monitor the status of your configuration distribution:

```bash
curl -X GET http://localhost:8080/api/v1/orchestration/distributions/dist-550e8400-e29b-41d4-a716-446655440000 \
  -H "Authorization: Bearer YOUR_TOKEN"
```

The response shows the progress of the distribution:

```json
{
  "id": "dist-550e8400-e29b-41d4-a716-446655440000",
  "status": "Completed",
  "target_gateways": ["gateway-id-1", "gateway-id-2"],
  "successful_gateways": ["gateway-id-1", "gateway-id-2"],
  "failed_gateways": [],
  "config_type": "Route",
  "created_at": "2023-05-27T16:30:00.000Z",
  "completed_at": "2023-05-27T16:30:05.000Z"
}
```

## 9. View All Distributions

List all configuration distributions:

```bash
curl -X GET "http://localhost:8080/api/v1/orchestration/distributions?include_completed=true" \
  -H "Authorization: Bearer YOUR_TOKEN"
```

## 10. Verify Gateway Health

The orchestration system automatically monitors gateway health. To verify:

1. Check gateway metrics from the console API
2. Monitor for any gateways being removed from the load balancing pool due to health issues
3. Observe automatic failover when a gateway becomes unhealthy

## Troubleshooting

### Distribution Failures

If a configuration distribution fails:

1. Check the individual gateway logs for specific errors
2. Verify network connectivity between console and gateways
3. Ensure gateway management API endpoints are accessible
4. Check for configuration conflicts or validation errors

### Load Balancing Issues

If load balancing is not selecting the appropriate gateways:

1. Verify gateway health metrics are being collected correctly
2. Check that gateways meet the minimum health threshold
3. Adjust the health threshold if necessary
4. Try a different load balancing strategy

### Gateway Registration Problems

If gateways are not appearing in the orchestration system:

1. Verify gateways are registered with the console
2. Check gateway status is "Online"
3. Ensure gateways are sending regular heartbeats
4. Restart the gateway registration process if needed

## Next Steps

- Configure region information for region-aware load balancing
- Set up gateway roles for specialized traffic handling
- Implement automatic scaling based on load metrics
- Create scheduled configuration distribution for maintenance windows
