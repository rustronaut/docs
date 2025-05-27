# SurrealDB Migration Plan for Rustronaut Console

## Executive Summary

This document outlines the migration strategy from PostgreSQL/SQLite to SurrealDB for the Rustronaut Console project. SurrealDB offers better performance, simplified architecture, and advanced features that align with our zero-trust networking requirements.

## Current State Analysis

### Current Database Usage
- **Primary Database**: PostgreSQL (production) + SQLite (development/testing)
- **Schema**: See `console/migrations/` for current structure
- **Key Tables**:
  - Users and authentication
  - Gateways and network configuration
  - Policies and access control
  - Monitoring and analytics data
  - Session management

### Current Tech Stack
- **Backend**: Rust with SQLx
- **Migrations**: SQLx migrations
- **Configuration**: TOML-based config
- **Connection Pooling**: SQLx connection pools

## Why SurrealDB?

### Advantages
1. **Multi-Model Database**: Document, Graph, and Relational in one
2. **Built-in Authentication**: Reduces auth complexity
3. **Real-time Subscriptions**: Perfect for live monitoring
4. **Rust-Native**: Better integration with our Rust backend
5. **Schema-less with Schema**: Flexible yet structured
6. **Built-in Security**: Row-level security, permissions
7. **Distributed**: Better scaling for multi-gateway deployments

### Alignment with Rustronaut
- **Zero-Trust**: Built-in access control and permissions
- **Real-time Monitoring**: Live queries for dashboard updates
- **Multi-tenancy**: Namespace isolation for different organizations
- **Performance**: Better for time-series data (monitoring metrics)

## Migration Strategy

### Phase 1: Preparation (Week 1-2)
1. **Environment Setup**
   - Install SurrealDB locally and in staging
   - Set up SurrealDB Rust SDK
   - Create development docker-compose with SurrealDB

2. **Schema Design**
   - Convert existing SQL schema to SurrealQL
   - Design new schema leveraging SurrealDB features
   - Plan data relationships and security permissions

3. **Code Structure**
   - Create new database module for SurrealDB
   - Implement dual-database support (gradual migration)
   - Set up configuration for both databases

### Phase 2: Core Migration (Week 3-4)
1. **Database Layer Refactoring**
   - Replace SQLx with SurrealDB SDK
   - Implement new query patterns
   - Update connection management

2. **Feature-by-Feature Migration**
   - Start with authentication (simplest)
   - Move gateway management
   - Migrate policy engine data
   - Update monitoring/analytics

3. **Testing**
   - Unit tests for new database layer
   - Integration tests with SurrealDB
   - Performance benchmarking

### Phase 3: Real-time Features (Week 5-6)
1. **Live Queries Implementation**
   - Real-time dashboard updates
   - Live monitoring streams
   - Event-driven notifications

2. **Advanced Features**
   - Implement graph relationships for network topology
   - Use document features for flexible policy storage
   - Leverage built-in analytics

### Phase 4: Production Deployment (Week 7-8)
1. **Data Migration**
   - Export from PostgreSQL
   - Transform and import to SurrealDB
   - Verify data integrity

2. **Deployment**
   - Blue-green deployment strategy
   - Rollback procedures
   - Monitoring and alerting

## Technical Implementation

### New Dependencies
```toml
# Cargo.toml additions
[dependencies]
surrealdb = "2.0"
serde = { version = "1.0", features = ["derive"] }
tokio = { version = "1.0", features = ["full"] }
uuid = { version = "1.0", features = ["v4", "serde"] }
```

### Database Configuration
```rust
// New config structure
#[derive(Debug, Deserialize, Clone)]
pub struct SurrealConfig {
    pub endpoint: String,
    pub namespace: String,
    pub database: String,
    pub username: Option<String>,
    pub password: Option<String>,
    pub connection_pool_size: usize,
}
```

### Schema Migration Examples

#### From SQL to SurrealQL
```sql
-- Current SQL
CREATE TABLE gateways (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name VARCHAR NOT NULL,
    region VARCHAR NOT NULL,
    status VARCHAR NOT NULL,
    config JSONB,
    created_at TIMESTAMP DEFAULT NOW()
);
```

```surql
-- New SurrealQL
DEFINE TABLE gateway SCHEMAFULL;
DEFINE FIELD id ON gateway TYPE record<gateway> VALUE $value OR rand::uuid();
DEFINE FIELD name ON gateway TYPE string;
DEFINE FIELD region ON gateway TYPE string;
DEFINE FIELD status ON gateway TYPE string ASSERT $value IN ['active', 'inactive', 'maintenance'];
DEFINE FIELD config ON gateway TYPE object;
DEFINE FIELD created_at ON gateway TYPE datetime VALUE $value OR time::now();
DEFINE FIELD updated_at ON gateway TYPE datetime VALUE time::now();

-- Permissions
DEFINE SCOPE gateway_scope SESSION 1h;
DEFINE TOKEN gateway_token ON SCOPE gateway_scope TYPE HS512 VALUE "your-secret-key";
```

## Directory Structure Changes

```
console/src/
├── database/
│   ├── mod.rs              # Database abstraction
│   ├── surreal/            # SurrealDB implementation
│   │   ├── mod.rs
│   │   ├── connection.rs   # Connection management
│   │   ├── queries.rs      # Query implementations
│   │   ├── schema.rs       # Schema definitions
│   │   └── migrations.rs   # Migration utilities
│   ├── models/             # Data models
│   └── traits.rs           # Database traits
├── config/
│   ├── surreal.toml        # SurrealDB config
│   └── ...
└── ...
```

## Risk Assessment and Mitigation

### High Risks
1. **Data Loss During Migration**
   - Mitigation: Comprehensive backup strategy, staged migration
   
2. **Performance Degradation**
   - Mitigation: Extensive performance testing, gradual rollout

3. **Learning Curve**
   - Mitigation: Training, proof-of-concept, documentation

### Medium Risks
1. **Integration Issues**
   - Mitigation: Extensive testing, dual-database support period

2. **Third-party Tool Compatibility**
   - Mitigation: Research alternatives, custom tooling if needed

## Timeline and Milestones

### Week 1-2: Setup and Planning
- [ ] SurrealDB environment setup
- [ ] Schema design completion
- [ ] Development environment ready

### Week 3-4: Core Implementation
- [ ] Database layer refactored
- [ ] Authentication migrated
- [ ] Gateway management migrated
- [ ] Basic tests passing

### Week 5-6: Advanced Features
- [ ] Real-time features implemented
- [ ] Policy engine migrated
- [ ] Monitoring system updated
- [ ] Performance optimized

### Week 7-8: Production
- [ ] Data migration completed
- [ ] Production deployment
- [ ] Monitoring and alerting active
- [ ] Team trained on new system

## Rollback Strategy

1. **Immediate Rollback** (< 1 hour)
   - Switch DNS/load balancer back to old system
   - Existing PostgreSQL remains untouched during migration

2. **Data Rollback** (< 4 hours)
   - Restore from PostgreSQL backup
   - Replay transactions from backup logs

3. **Code Rollback** (< 30 minutes)
   - Git revert to previous stable version
   - Automated deployment pipeline

## Success Metrics

### Performance
- Query response time: < 50ms for 95th percentile
- Concurrent connections: Support 1000+ simultaneous users
- Real-time updates: < 100ms latency

### Reliability
- Uptime: 99.9%
- Data consistency: 100%
- Zero data loss during migration

### Development Experience
- Reduced code complexity by 30%
- Faster development cycles
- Better real-time capabilities

## Next Steps

1. **Immediate Actions**
   - [ ] Set up SurrealDB development environment
   - [ ] Create proof-of-concept for core operations
   - [ ] Design new schema structure

2. **Week 1 Deliverables**
   - [ ] Complete environment setup
   - [ ] Finalize schema design
   - [ ] Create migration plan details
   - [ ] Set up testing framework

This migration will position Rustronaut for better scalability, real-time capabilities, and simplified architecture while maintaining our zero-trust security principles.
