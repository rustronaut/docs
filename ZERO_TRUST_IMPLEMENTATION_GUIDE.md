# Zero-Trust Implementation Guide: From Foundation to Advanced Features

## 🎯 Implementation Priorities

This guide provides a practical roadmap for implementing the enhanced zero-trust architecture, building incrementally on the existing WireGuard foundation.

## 📋 Phase 3A: Behavioral Analysis Foundation (Immediate - Week 3)

### Step 1: Basic Metrics Collection Infrastructure

First, let's enhance the existing gateway to collect behavioral metrics:

```rust
// gateway/src/behavior_analyzer.rs
use std::collections::HashMap;
use std::time::{SystemTime, UNIX_EPOCH};
use serde::{Deserialize, Serialize};
use tokio::sync::RwLock;
use std::sync::Arc;

#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct BehaviorMetrics {
    pub user_id: String,
    pub device_id: String,
    pub connection_count: u64,
    pub data_transferred: u64,
    pub connection_duration: u64,
    pub unusual_hours: bool,
    pub geographic_anomaly: bool,
    pub access_patterns: Vec<AccessPattern>,
    pub timestamp: u64,
}

#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct AccessPattern {
    pub resource: String,
    pub frequency: u32,
    pub typical_time_ranges: Vec<TimeRange>,
    pub data_volume: u64,
}

#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct TimeRange {
    pub start_hour: u8,
    pub end_hour: u8,
    pub days_of_week: Vec<u8>, // 0-6, Monday-Sunday
}

pub struct BehaviorAnalyzer {
    pub user_baselines: Arc<RwLock<HashMap<String, UserBaseline>>>,
    pub device_baselines: Arc<RwLock<HashMap<String, DeviceBaseline>>>,
    pub anomaly_threshold: f64,
    pub learning_window_days: u8,
}

#[derive(Debug, Clone)]
pub struct UserBaseline {
    pub user_id: String,
    pub typical_connection_hours: Vec<TimeRange>,
    pub typical_locations: Vec<String>,
    pub typical_data_volume: DataVolumeRange,
    pub typical_session_duration: DurationRange,
    pub typical_resources: Vec<String>,
    pub confidence_score: f64,
    pub last_updated: SystemTime,
}

#[derive(Debug, Clone)]
pub struct DataVolumeRange {
    pub min_mb: u64,
    pub max_mb: u64,
    pub average_mb: u64,
}

#[derive(Debug, Clone)]
pub struct DurationRange {
    pub min_minutes: u64,
    pub max_minutes: u64,
    pub average_minutes: u64,
}

impl BehaviorAnalyzer {
    pub fn new() -> Self {
        Self {
            user_baselines: Arc::new(RwLock::new(HashMap::new())),
            device_baselines: Arc::new(RwLock::new(HashMap::new())),
            anomaly_threshold: 0.7, // 70% confidence for anomaly detection
            learning_window_days: 30,
        }
    }
    
    pub async fn record_connection_event(&self, event: ConnectionEvent) -> Result<(), Box<dyn std::error::Error>> {
        // Record the event for baseline learning
        self.update_user_baseline(&event).await?;
        self.update_device_baseline(&event).await?;
        
        // Check for anomalies
        let anomaly_score = self.calculate_anomaly_score(&event).await?;
        
        if anomaly_score > self.anomaly_threshold {
            self.trigger_anomaly_alert(&event, anomaly_score).await?;
        }
        
        Ok(())
    }
    
    async fn calculate_anomaly_score(&self, event: &ConnectionEvent) -> Result<f64, Box<dyn std::error::Error>> {
        let user_baselines = self.user_baselines.read().await;
        let device_baselines = self.device_baselines.read().await;
        
        let mut anomaly_factors = Vec::new();
        
        // Check user behavior anomalies
        if let Some(user_baseline) = user_baselines.get(&event.user_id) {
            anomaly_factors.push(self.check_time_anomaly(event, user_baseline));
            anomaly_factors.push(self.check_location_anomaly(event, user_baseline));
            anomaly_factors.push(self.check_data_volume_anomaly(event, user_baseline));
        }
        
        // Check device behavior anomalies
        if let Some(device_baseline) = device_baselines.get(&event.device_id) {
            anomaly_factors.push(self.check_device_behavior_anomaly(event, device_baseline));
        }
        
        // Calculate combined anomaly score
        let total_score: f64 = anomaly_factors.iter().sum();
        let average_score = total_score / anomaly_factors.len() as f64;
        
        Ok(average_score)
    }
    
    fn check_time_anomaly(&self, event: &ConnectionEvent, baseline: &UserBaseline) -> f64 {
        let current_hour = event.timestamp.hour();
        let current_day = event.timestamp.weekday().num_days_from_monday() as u8;
        
        for time_range in &baseline.typical_connection_hours {
            if time_range.days_of_week.contains(&current_day) &&
               current_hour >= time_range.start_hour &&
               current_hour <= time_range.end_hour {
                return 0.0; // Normal time
            }
        }
        
        0.8 // High anomaly for unusual time
    }
    
    async fn trigger_anomaly_alert(&self, event: &ConnectionEvent, score: f64) -> Result<(), Box<dyn std::error::Error>> {
        let alert = AnomalyAlert {
            alert_id: uuid::Uuid::new_v4().to_string(),
            user_id: event.user_id.clone(),
            device_id: event.device_id.clone(),
            anomaly_score: score,
            detected_anomalies: self.describe_anomalies(event, score).await,
            timestamp: SystemTime::now(),
            severity: self.calculate_severity(score),
            recommended_actions: self.generate_recommendations(score),
        };
        
        // Log the alert
        tracing::warn!("Behavioral anomaly detected: {:?}", alert);
        
        // Send to security monitoring system
        self.send_to_security_system(&alert).await?;
        
        Ok(())
    }
}

#[derive(Debug, Clone)]
pub struct ConnectionEvent {
    pub user_id: String,
    pub device_id: String,
    pub source_ip: String,
    pub timestamp: chrono::DateTime<chrono::Utc>,
    pub session_duration: Option<u64>,
    pub data_transferred: Option<u64>,
    pub resources_accessed: Vec<String>,
    pub connection_type: String,
}

#[derive(Debug, Clone)]
pub struct AnomalyAlert {
    pub alert_id: String,
    pub user_id: String,
    pub device_id: String,
    pub anomaly_score: f64,
    pub detected_anomalies: Vec<String>,
    pub timestamp: SystemTime,
    pub severity: AlertSeverity,
    pub recommended_actions: Vec<String>,
}

#[derive(Debug, Clone)]
pub enum AlertSeverity {
    Low,
    Medium,
    High,
    Critical,
}
```

### Step 2: Integration with Existing Gateway

```rust
// gateway/src/wireguard.rs - Enhanced with behavioral analysis
use crate::behavior_analyzer::{BehaviorAnalyzer, ConnectionEvent};

impl WireGuardManager {
    // Add behavior analyzer to existing manager
    pub fn new_with_behavior_analysis(config: WireGuardConfig) -> Result<Self, WireGuardError> {
        let mut manager = Self::new(config)?;
        manager.behavior_analyzer = Some(Arc::new(BehaviorAnalyzer::new()));
        Ok(manager)
    }
    
    // Enhanced peer registration with behavioral tracking
    pub async fn register_peer_with_behavior_tracking(
        &mut self,
        registration: PeerRegistration,
    ) -> Result<PeerResponse, WireGuardError> {
        // Record connection event for behavioral analysis
        if let Some(analyzer) = &self.behavior_analyzer {
            let connection_event = ConnectionEvent {
                user_id: registration.user_id.clone(),
                device_id: registration.device_id.clone(),
                source_ip: registration.endpoint.to_string(),
                timestamp: chrono::Utc::now(),
                session_duration: None,
                data_transferred: None,
                resources_accessed: vec!["vpn_connection".to_string()],
                connection_type: "wireguard".to_string(),
            };
            
            if let Err(e) = analyzer.record_connection_event(connection_event).await {
                tracing::warn!("Failed to record behavioral event: {}", e);
            }
        }
        
        // Continue with normal peer registration
        self.register_peer(registration).await
    }
}
```

### Step 3: Basic API Endpoints for Behavioral Data

```rust
// gateway/src/api/behavior.rs
use axum::{
    extract::{Path, State},
    http::StatusCode,
    Json,
    response::Json as ResponseJson,
};
use crate::behavior_analyzer::{BehaviorAnalyzer, BehaviorMetrics};
use std::sync::Arc;

pub async fn get_user_behavior_baseline(
    State(analyzer): State<Arc<BehaviorAnalyzer>>,
    Path(user_id): Path<String>,
) -> Result<ResponseJson<UserBaseline>, StatusCode> {
    let baselines = analyzer.user_baselines.read().await;
    
    match baselines.get(&user_id) {
        Some(baseline) => Ok(ResponseJson(baseline.clone())),
        None => Err(StatusCode::NOT_FOUND),
    }
}

pub async fn get_anomaly_alerts(
    State(analyzer): State<Arc<BehaviorAnalyzer>>,
) -> Result<ResponseJson<Vec<AnomalyAlert>>, StatusCode> {
    // In a real implementation, this would query a database
    // For now, return empty alerts
    Ok(ResponseJson(vec![]))
}

pub async fn get_behavior_metrics(
    State(analyzer): State<Arc<BehaviorAnalyzer>>,
    Path(user_id): Path<String>,
) -> Result<ResponseJson<BehaviorMetrics>, StatusCode> {
    // Generate current behavior metrics for user
    let metrics = analyzer.generate_current_metrics(&user_id).await
        .map_err(|_| StatusCode::INTERNAL_SERVER_ERROR)?;
    
    Ok(ResponseJson(metrics))
}

// Add these routes to the main router
pub fn behavior_routes() -> axum::Router<Arc<BehaviorAnalyzer>> {
    axum::Router::new()
        .route("/behavior/user/:user_id/baseline", axum::routing::get(get_user_behavior_baseline))
        .route("/behavior/alerts", axum::routing::get(get_anomaly_alerts))
        .route("/behavior/user/:user_id/metrics", axum::routing::get(get_behavior_metrics))
}
```

## 📋 Phase 3B: Policy Engine Foundation (Week 3-4)

### Step 1: Basic Policy Framework

```rust
// gateway/src/policy_engine.rs
use serde::{Deserialize, Serialize};
use std::collections::HashMap;
use std::sync::Arc;
use tokio::sync::RwLock;

#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct AccessPolicy {
    pub policy_id: String,
    pub name: String,
    pub description: String,
    pub source_criteria: Vec<Criterion>,
    pub destination_criteria: Vec<Criterion>,
    pub action: PolicyAction,
    pub conditions: Vec<Condition>,
    pub priority: u32,
    pub enabled: bool,
}

#[derive(Debug, Clone, Serialize, Deserialize)]
pub enum Criterion {
    UserId(String),
    UserGroup(String),
    DeviceId(String),
    DeviceType(String),
    IPAddress(String),
    IPRange(String),
    TimeRange(TimeRange),
    Location(String),
    Application(String),
}

#[derive(Debug, Clone, Serialize, Deserialize)]
pub enum PolicyAction {
    Allow,
    Deny,
    AllowWithMonitoring,
    AllowWithRestrictions(Vec<Restriction>),
    RequireAdditionalAuth,
}

#[derive(Debug, Clone, Serialize, Deserialize)]
pub enum Restriction {
    BandwidthLimit(u64), // bytes per second
    SessionTimeLimit(u64), // seconds
    AllowedPorts(Vec<u16>),
    AllowedProtocols(Vec<String>),
}

#[derive(Debug, Clone, Serialize, Deserialize)]
pub enum Condition {
    RiskScore(f64),
    TrustLevel(TrustLevel),
    ComplianceStatus(bool),
    BehaviorAnomaly(f64),
    GeographicAnomaly(bool),
}

#[derive(Debug, Clone, Serialize, Deserialize)]
pub enum TrustLevel {
    Low,
    Medium,
    High,
    Verified,
}

pub struct PolicyEngine {
    pub policies: Arc<RwLock<HashMap<String, AccessPolicy>>>,
    pub policy_cache: Arc<RwLock<HashMap<String, PolicyDecision>>>,
    pub evaluation_metrics: Arc<RwLock<EvaluationMetrics>>,
}

#[derive(Debug, Clone)]
pub struct PolicyDecision {
    pub allowed: bool,
    pub action: PolicyAction,
    pub applied_policies: Vec<String>,
    pub restrictions: Vec<Restriction>,
    pub monitoring_required: bool,
    pub confidence: f64,
    pub reason: String,
}

#[derive(Debug, Clone)]
pub struct EvaluationMetrics {
    pub total_evaluations: u64,
    pub allow_decisions: u64,
    pub deny_decisions: u64,
    pub average_evaluation_time_ms: f64,
    pub cache_hit_rate: f64,
}

impl PolicyEngine {
    pub fn new() -> Self {
        Self {
            policies: Arc::new(RwLock::new(HashMap::new())),
            policy_cache: Arc::new(RwLock::new(HashMap::new())),
            evaluation_metrics: Arc::new(RwLock::new(EvaluationMetrics::default())),
        }
    }
    
    pub async fn evaluate_access_request(
        &self,
        request: &AccessRequest,
    ) -> Result<PolicyDecision, PolicyError> {
        let start_time = std::time::Instant::now();
        
        // Check cache first
        let cache_key = self.generate_cache_key(request);
        {
            let cache = self.policy_cache.read().await;
            if let Some(cached_decision) = cache.get(&cache_key) {
                self.update_cache_metrics(true).await;
                return Ok(cached_decision.clone());
            }
        }
        
        // Evaluate policies
        let policies = self.policies.read().await;
        let mut applicable_policies = Vec::new();
        
        // Find applicable policies
        for policy in policies.values() {
            if self.policy_matches_request(policy, request) {
                applicable_policies.push(policy.clone());
            }
        }
        
        // Sort by priority (higher priority first)
        applicable_policies.sort_by(|a, b| b.priority.cmp(&a.priority));
        
        // Evaluate policies in priority order
        let decision = self.make_policy_decision(&applicable_policies, request).await?;
        
        // Cache the decision
        {
            let mut cache = self.policy_cache.write().await;
            cache.insert(cache_key, decision.clone());
        }
        
        // Update metrics
        let evaluation_time = start_time.elapsed().as_millis() as f64;
        self.update_evaluation_metrics(&decision, evaluation_time).await;
        
        Ok(decision)
    }
    
    fn policy_matches_request(&self, policy: &AccessPolicy, request: &AccessRequest) -> bool {
        if !policy.enabled {
            return false;
        }
        
        // Check source criteria
        if !self.criteria_match(&policy.source_criteria, &request.source_context) {
            return false;
        }
        
        // Check destination criteria
        if !self.criteria_match(&policy.destination_criteria, &request.destination_context) {
            return false;
        }
        
        // Check conditions
        for condition in &policy.conditions {
            if !self.condition_satisfied(condition, request) {
                return false;
            }
        }
        
        true
    }
    
    fn criteria_match(&self, criteria: &[Criterion], context: &RequestContext) -> bool {
        if criteria.is_empty() {
            return true; // No criteria means match all
        }
        
        for criterion in criteria {
            match criterion {
                Criterion::UserId(user_id) => {
                    if context.user_id.as_ref() == Some(user_id) {
                        return true;
                    }
                }
                Criterion::UserGroup(group) => {
                    if context.user_groups.contains(group) {
                        return true;
                    }
                }
                Criterion::DeviceId(device_id) => {
                    if context.device_id.as_ref() == Some(device_id) {
                        return true;
                    }
                }
                Criterion::IPAddress(ip) => {
                    if context.ip_address.as_ref() == Some(ip) {
                        return true;
                    }
                }
                // Add more criterion matching logic
                _ => continue,
            }
        }
        
        false
    }
}

#[derive(Debug, Clone)]
pub struct AccessRequest {
    pub request_id: String,
    pub source_context: RequestContext,
    pub destination_context: RequestContext,
    pub requested_action: String,
    pub risk_score: Option<f64>,
    pub timestamp: chrono::DateTime<chrono::Utc>,
}

#[derive(Debug, Clone)]
pub struct RequestContext {
    pub user_id: Option<String>,
    pub user_groups: Vec<String>,
    pub device_id: Option<String>,
    pub device_type: Option<String>,
    pub ip_address: Option<String>,
    pub location: Option<String>,
    pub application: Option<String>,
}

#[derive(Debug)]
pub enum PolicyError {
    EvaluationFailed(String),
    InvalidPolicy(String),
    InternalError(String),
}
```

### Step 2: Integration with WireGuard Manager

```rust
// gateway/src/wireguard.rs - Enhanced with policy engine
use crate::policy_engine::{PolicyEngine, AccessRequest, RequestContext};

impl WireGuardManager {
    pub async fn register_peer_with_policy_check(
        &mut self,
        registration: PeerRegistration,
    ) -> Result<PeerResponse, WireGuardError> {
        // Create access request for policy evaluation
        let access_request = AccessRequest {
            request_id: uuid::Uuid::new_v4().to_string(),
            source_context: RequestContext {
                user_id: Some(registration.user_id.clone()),
                user_groups: registration.user_groups.clone().unwrap_or_default(),
                device_id: Some(registration.device_id.clone()),
                device_type: registration.device_type.clone(),
                ip_address: Some(registration.endpoint.ip().to_string()),
                location: registration.location.clone(),
                application: Some("wireguard_vpn".to_string()),
            },
            destination_context: RequestContext {
                user_id: None,
                user_groups: vec![],
                device_id: None,
                device_type: None,
                ip_address: Some(self.config.listen_addr.to_string()),
                location: Some("gateway".to_string()),
                application: Some("vpn_gateway".to_string()),
            },
            requested_action: "vpn_connection".to_string(),
            risk_score: None, // Could be provided by behavior analyzer
            timestamp: chrono::Utc::now(),
        };
        
        // Evaluate policy
        if let Some(policy_engine) = &self.policy_engine {
            let decision = policy_engine.evaluate_access_request(&access_request).await
                .map_err(|e| WireGuardError::PolicyViolation(format!("Policy evaluation failed: {:?}", e)))?;
            
            if !decision.allowed {
                return Err(WireGuardError::PolicyViolation(decision.reason));
            }
            
            // Apply restrictions if any
            if !decision.restrictions.is_empty() {
                self.apply_connection_restrictions(&registration.device_id, &decision.restrictions).await?;
            }
        }
        
        // Continue with normal peer registration
        self.register_peer(registration).await
    }
    
    async fn apply_connection_restrictions(
        &self,
        device_id: &str,
        restrictions: &[crate::policy_engine::Restriction],
    ) -> Result<(), WireGuardError> {
        for restriction in restrictions {
            match restriction {
                crate::policy_engine::Restriction::BandwidthLimit(limit) => {
                    self.set_bandwidth_limit(device_id, *limit).await?;
                }
                crate::policy_engine::Restriction::SessionTimeLimit(limit) => {
                    self.set_session_timeout(device_id, *limit).await?;
                }
                crate::policy_engine::Restriction::AllowedPorts(ports) => {
                    self.configure_port_restrictions(device_id, ports).await?;
                }
                crate::policy_engine::Restriction::AllowedProtocols(protocols) => {
                    self.configure_protocol_restrictions(device_id, protocols).await?;
                }
            }
        }
        Ok(())
    }
}
```

## 📋 Phase 3C: Web Console Integration (Week 4)

### Enhanced Web Console with Behavioral Insights

```typescript
// console/src/components/SecurityDashboard.tsx
import React, { useState, useEffect } from 'react';
import { Card, CardContent, CardHeader, CardTitle } from '@/components/ui/card';
import { Alert, AlertDescription } from '@/components/ui/alert';
import { Badge } from '@/components/ui/badge';

interface BehaviorMetrics {
  userId: string;
  deviceId: string;
  connectionCount: number;
  dataTransferred: number;
  connectionDuration: number;
  unusualHours: boolean;
  geographicAnomaly: boolean;
  riskScore: number;
}

interface AnomalyAlert {
  alertId: string;
  userId: string;
  deviceId: string;
  anomalyScore: number;
  detectedAnomalies: string[];
  severity: 'Low' | 'Medium' | 'High' | 'Critical';
  timestamp: string;
}

export const SecurityDashboard: React.FC = () => {
  const [behaviorMetrics, setBehaviorMetrics] = useState<BehaviorMetrics[]>([]);
  const [anomalyAlerts, setAnomalyAlerts] = useState<AnomalyAlert[]>([]);
  const [loading, setLoading] = useState(true);

  useEffect(() => {
    loadSecurityData();
    const interval = setInterval(loadSecurityData, 30000); // Refresh every 30 seconds
    return () => clearInterval(interval);
  }, []);

  const loadSecurityData = async () => {
    try {
      const [metricsResponse, alertsResponse] = await Promise.all([
        fetch('/api/behavior/metrics'),
        fetch('/api/behavior/alerts'),
      ]);

      const metrics = await metricsResponse.json();
      const alerts = await alertsResponse.json();

      setBehaviorMetrics(metrics);
      setAnomalyAlerts(alerts);
    } catch (error) {
      console.error('Failed to load security data:', error);
    } finally {
      setLoading(false);
    }
  };

  const getSeverityColor = (severity: string) => {
    switch (severity) {
      case 'Critical': return 'bg-red-500';
      case 'High': return 'bg-orange-500';
      case 'Medium': return 'bg-yellow-500';
      case 'Low': return 'bg-blue-500';
      default: return 'bg-gray-500';
    }
  };

  if (loading) {
    return <div>Loading security dashboard...</div>;
  }

  return (
    <div className="space-y-6">
      <h1 className="text-3xl font-bold">Security Dashboard</h1>
      
      {/* Anomaly Alerts */}
      <Card>
        <CardHeader>
          <CardTitle>Recent Anomaly Alerts</CardTitle>
        </CardHeader>
        <CardContent>
          {anomalyAlerts.length === 0 ? (
            <p>No anomalies detected</p>
          ) : (
            <div className="space-y-4">
              {anomalyAlerts.map((alert) => (
                <Alert key={alert.alertId}>
                  <AlertDescription>
                    <div className="flex items-center justify-between">
                      <div>
                        <strong>User:</strong> {alert.userId} | 
                        <strong> Device:</strong> {alert.deviceId} | 
                        <strong> Score:</strong> {alert.anomalyScore.toFixed(2)}
                      </div>
                      <Badge className={getSeverityColor(alert.severity)}>
                        {alert.severity}
                      </Badge>
                    </div>
                    <div className="mt-2">
                      <strong>Anomalies:</strong> {alert.detectedAnomalies.join(', ')}
                    </div>
                  </AlertDescription>
                </Alert>
              ))}
            </div>
          )}
        </CardContent>
      </Card>

      {/* Behavior Metrics */}
      <Card>
        <CardHeader>
          <CardTitle>User Behavior Overview</CardTitle>
        </CardHeader>
        <CardContent>
          <div className="overflow-x-auto">
            <table className="w-full border-collapse border border-gray-300">
              <thead>
                <tr className="bg-gray-100">
                  <th className="border border-gray-300 p-2">User ID</th>
                  <th className="border border-gray-300 p-2">Device ID</th>
                  <th className="border border-gray-300 p-2">Connections</th>
                  <th className="border border-gray-300 p-2">Data Transfer (MB)</th>
                  <th className="border border-gray-300 p-2">Risk Score</th>
                  <th className="border border-gray-300 p-2">Flags</th>
                </tr>
              </thead>
              <tbody>
                {behaviorMetrics.map((metric, index) => (
                  <tr key={index}>
                    <td className="border border-gray-300 p-2">{metric.userId}</td>
                    <td className="border border-gray-300 p-2">{metric.deviceId}</td>
                    <td className="border border-gray-300 p-2">{metric.connectionCount}</td>
                    <td className="border border-gray-300 p-2">
                      {(metric.dataTransferred / (1024 * 1024)).toFixed(2)}
                    </td>
                    <td className="border border-gray-300 p-2">
                      <Badge className={metric.riskScore > 0.7 ? 'bg-red-500' : 
                                       metric.riskScore > 0.4 ? 'bg-yellow-500' : 'bg-green-500'}>
                        {metric.riskScore.toFixed(2)}
                      </Badge>
                    </td>
                    <td className="border border-gray-300 p-2">
                      {metric.unusualHours && <Badge className="bg-orange-500 mr-1">Unusual Hours</Badge>}
                      {metric.geographicAnomaly && <Badge className="bg-red-500">Geo Anomaly</Badge>}
                    </td>
                  </tr>
                ))}
              </tbody>
            </table>
          </div>
        </CardContent>
      </Card>
    </div>
  );
};
```

### Policy Management Interface

```typescript
// console/src/components/PolicyManagement.tsx
import React, { useState, useEffect } from 'react';
import { Card, CardContent, CardHeader, CardTitle } from '@/components/ui/card';
import { Button } from '@/components/ui/button';
import { Input } from '@/components/ui/input';
import { Select, SelectContent, SelectItem, SelectTrigger, SelectValue } from '@/components/ui/select';

interface AccessPolicy {
  policyId: string;
  name: string;
  description: string;
  action: 'Allow' | 'Deny' | 'AllowWithMonitoring';
  priority: number;
  enabled: boolean;
  sourceCriteria: Criterion[];
  destinationCriteria: Criterion[];
}

interface Criterion {
  type: 'UserId' | 'UserGroup' | 'DeviceType' | 'IPAddress' | 'TimeRange';
  value: string;
}

export const PolicyManagement: React.FC = () => {
  const [policies, setPolicies] = useState<AccessPolicy[]>([]);
  const [showCreateForm, setShowCreateForm] = useState(false);
  const [newPolicy, setNewPolicy] = useState<Partial<AccessPolicy>>({
    name: '',
    description: '',
    action: 'Allow',
    priority: 100,
    enabled: true,
    sourceCriteria: [],
    destinationCriteria: [],
  });

  useEffect(() => {
    loadPolicies();
  }, []);

  const loadPolicies = async () => {
    try {
      const response = await fetch('/api/policies');
      const data = await response.json();
      setPolicies(data);
    } catch (error) {
      console.error('Failed to load policies:', error);
    }
  };

  const createPolicy = async () => {
    try {
      const response = await fetch('/api/policies', {
        method: 'POST',
        headers: { 'Content-Type': 'application/json' },
        body: JSON.stringify(newPolicy),
      });

      if (response.ok) {
        loadPolicies();
        setShowCreateForm(false);
        setNewPolicy({
          name: '',
          description: '',
          action: 'Allow',
          priority: 100,
          enabled: true,
          sourceCriteria: [],
          destinationCriteria: [],
        });
      }
    } catch (error) {
      console.error('Failed to create policy:', error);
    }
  };

  const togglePolicy = async (policyId: string, enabled: boolean) => {
    try {
      await fetch(`/api/policies/${policyId}`, {
        method: 'PATCH',
        headers: { 'Content-Type': 'application/json' },
        body: JSON.stringify({ enabled }),
      });
      loadPolicies();
    } catch (error) {
      console.error('Failed to toggle policy:', error);
    }
  };

  return (
    <div className="space-y-6">
      <div className="flex justify-between items-center">
        <h1 className="text-3xl font-bold">Policy Management</h1>
        <Button onClick={() => setShowCreateForm(true)}>Create Policy</Button>
      </div>

      {showCreateForm && (
        <Card>
          <CardHeader>
            <CardTitle>Create New Policy</CardTitle>
          </CardHeader>
          <CardContent className="space-y-4">
            <Input
              placeholder="Policy Name"
              value={newPolicy.name || ''}
              onChange={(e) => setNewPolicy({...newPolicy, name: e.target.value})}
            />
            <Input
              placeholder="Description"
              value={newPolicy.description || ''}
              onChange={(e) => setNewPolicy({...newPolicy, description: e.target.value})}
            />
            <Select
              value={newPolicy.action}
              onValueChange={(value) => setNewPolicy({...newPolicy, action: value as any})}
            >
              <SelectTrigger>
                <SelectValue placeholder="Select Action" />
              </SelectTrigger>
              <SelectContent>
                <SelectItem value="Allow">Allow</SelectItem>
                <SelectItem value="Deny">Deny</SelectItem>
                <SelectItem value="AllowWithMonitoring">Allow with Monitoring</SelectItem>
              </SelectContent>
            </Select>
            <Input
              type="number"
              placeholder="Priority"
              value={newPolicy.priority || 100}
              onChange={(e) => setNewPolicy({...newPolicy, priority: parseInt(e.target.value)})}
            />
            <div className="flex space-x-2">
              <Button onClick={createPolicy}>Create</Button>
              <Button variant="outline" onClick={() => setShowCreateForm(false)}>Cancel</Button>
            </div>
          </CardContent>
        </Card>
      )}

      <Card>
        <CardHeader>
          <CardTitle>Active Policies</CardTitle>
        </CardHeader>
        <CardContent>
          <div className="space-y-4">
            {policies.map((policy) => (
              <div key={policy.policyId} className="border rounded p-4">
                <div className="flex justify-between items-start">
                  <div>
                    <h3 className="font-semibold">{policy.name}</h3>
                    <p className="text-gray-600">{policy.description}</p>
                    <div className="flex space-x-2 mt-2">
                      <Badge className={policy.action === 'Allow' ? 'bg-green-500' : 
                                      policy.action === 'Deny' ? 'bg-red-500' : 'bg-yellow-500'}>
                        {policy.action}
                      </Badge>
                      <Badge>Priority: {policy.priority}</Badge>
                    </div>
                  </div>
                  <Button
                    onClick={() => togglePolicy(policy.policyId, !policy.enabled)}
                    variant={policy.enabled ? "destructive" : "default"}
                  >
                    {policy.enabled ? 'Disable' : 'Enable'}
                  </Button>
                </div>
              </div>
            ))}
          </div>
        </CardContent>
      </Card>
    </div>
  );
};
```

## 🚀 Quick Implementation Steps

To immediately start implementing these features:

1. **Add the behavior analyzer module** to the gateway
2. **Integrate behavioral tracking** into peer registration
3. **Create basic policy evaluation** for connection requests
4. **Add API endpoints** for security data
5. **Enhance the web console** with security dashboard
6. **Test with simple policies** and behavioral patterns

This foundation provides:
- ✅ Real-time behavioral analysis
- ✅ Policy-based access control
- ✅ Security monitoring dashboard
- ✅ Anomaly detection and alerting
- ✅ Web-based policy management

These features can be built incrementally while maintaining the existing WireGuard functionality, providing immediate value while laying the groundwork for more advanced zero-trust capabilities.
