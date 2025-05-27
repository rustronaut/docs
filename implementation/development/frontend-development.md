# 🎨 Frontend Development Guide

## Overview

Comprehensive guide for developing the React/TypeScript web console for Rustronaut. This guide covers the complete implementation of the management interface that will connect to the production-ready backend APIs.

**Status**: 🚧 **High Priority Development**  
**Backend Status**: ✅ **Complete and Ready**  
**Frontend Status**: ❌ **Needs Implementation**

## 🎯 Development Context

### What's Already Built
- ✅ **Complete Backend APIs**: All management endpoints implemented and tested
- ✅ **Real-time WebSocket**: Live updates for policies and security events  
- ✅ **Database Integration**: PostgreSQL/SQLite working with migrations
- ✅ **Static Asset Serving**: Frontend build integration ready
- ✅ **CORS Configuration**: Development and production-ready

### What Needs Implementation
- ❌ **React Components**: Policy management, security dashboard, gateway monitoring
- ❌ **TypeScript Interfaces**: API response types and data models
- ❌ **Real-time Integration**: WebSocket client implementation
- ❌ **State Management**: Application state and API integration
- ❌ **UI/UX Design**: Modern dashboard with Tailwind CSS

## 🏗️ Frontend Architecture

### Technology Stack
- **Framework**: Next.js 14 with App Router
- **Language**: TypeScript
- **Styling**: Tailwind CSS
- **State Management**: React hooks with Context API
- **Real-time**: WebSocket client
- **Charts**: Chart.js or Recharts for analytics
- **HTTP Client**: Fetch API with error handling

### Project Structure
```
console/web-console/
├── app/                     # Next.js App Router
│   ├── layout.tsx          # Root layout
│   ├── page.tsx            # Dashboard home
│   ├── dashboard/          # Dashboard routes
│   ├── policies/           # Policy management
│   ├── gateways/           # Gateway monitoring
│   └── analytics/          # Security analytics
├── components/             # Reusable components
│   ├── ui/                 # Base UI components
│   ├── dashboard/          # Dashboard-specific
│   ├── policies/           # Policy management
│   └── gateways/           # Gateway monitoring
├── lib/                    # Utilities and API client
│   ├── api.ts              # API client
│   ├── websocket.ts        # WebSocket client
│   └── types.ts            # TypeScript types
└── public/                 # Static assets
```

## 🚀 Getting Started

### 1. Setup Development Environment
```bash
# Navigate to frontend directory
cd console/web-console

# Install dependencies
npm install

# Start development server
npm run dev

# Frontend available at http://localhost:3000
```

### 2. Verify Backend Connection
Ensure the console backend is running:
```bash
# In separate terminal
cd console
cargo run

# Verify API endpoint
curl http://localhost:8081/api/v1/dashboard/stats
```

## 🔧 API Integration

### Available Backend Endpoints

#### Dashboard & Status
```typescript
GET /api/v1/dashboard/stats     // System statistics
GET /api/v1/dashboard/status    // System health status
GET /api/v1/gateways           // Gateway list and status
```

#### Policy Management
```typescript
GET    /api/v1/policies        // List all policies  
POST   /api/v1/policies        // Create new policy
PUT    /api/v1/policies/:id    // Update policy
DELETE /api/v1/policies/:id    // Delete policy
```

#### Analytics & Security
```typescript
GET /api/v1/analytics/security     // Security metrics
GET /api/v1/analytics/behavior     // Behavioral analysis data
GET /api/v1/analytics/performance  // Performance metrics
```

#### WebSocket Endpoints
```typescript
/ws/policies    // Real-time policy updates
/ws/security    // Security event stream  
/ws/gateways    // Gateway status updates
```

### TypeScript API Client
Create `lib/api.ts`:
```typescript
// Base API configuration
const API_BASE = process.env.NODE_ENV === 'development' 
  ? 'http://localhost:8081' 
  : '';

class ApiClient {
  private async request<T>(endpoint: string, options?: RequestInit): Promise<T> {
    const response = await fetch(`${API_BASE}${endpoint}`, {
      headers: {
        'Content-Type': 'application/json',
        ...options?.headers,
      },
      ...options,
    });
    
    if (!response.ok) {
      throw new Error(`API Error: ${response.status}`);
    }
    
    return response.json();
  }
  
  // Dashboard endpoints
  async getDashboardStats() {
    return this.request('/api/v1/dashboard/stats');
  }
  
  async getDashboardStatus() {
    return this.request('/api/v1/dashboard/status');  
  }
  
  // Gateway management
  async getGateways() {
    return this.request('/api/v1/gateways');
  }
  
  // Policy management
  async getPolicies() {
    return this.request('/api/v1/policies');
  }
  
  async createPolicy(policy: PolicyCreateRequest) {
    return this.request('/api/v1/policies', {
      method: 'POST',
      body: JSON.stringify(policy),
    });
  }
}

export const apiClient = new ApiClient();
```

### WebSocket Integration
Create `lib/websocket.ts`:
```typescript
type WebSocketEventType = 'policies' | 'security' | 'gateways';

class WebSocketClient {
  private connections: Map<string, WebSocket> = new Map();
  
  connect(eventType: WebSocketEventType, onMessage: (data: any) => void) {
    const wsUrl = `ws://localhost:8081/ws/${eventType}`;
    const ws = new WebSocket(wsUrl);
    
    ws.onmessage = (event) => {
      const data = JSON.parse(event.data);
      onMessage(data);
    };
    
    ws.onerror = (error) => {
      console.error(`WebSocket error for ${eventType}:`, error);
    };
    
    this.connections.set(eventType, ws);
    return ws;
  }
  
  disconnect(eventType: WebSocketEventType) {
    const ws = this.connections.get(eventType);
    if (ws) {
      ws.close();
      this.connections.delete(eventType);
    }
  }
}

export const wsClient = new WebSocketClient();
```

## 🧩 Component Implementation

### 1. Dashboard Overview
Create `components/dashboard/DashboardOverview.tsx`:
```typescript
'use client';
import { useEffect, useState } from 'react';
import { apiClient } from '@/lib/api';

interface DashboardStats {
  activeGateways: number;
  totalPolicies: number;
  securityEvents: number;
  systemHealth: 'healthy' | 'warning' | 'error';
}

export default function DashboardOverview() {
  const [stats, setStats] = useState<DashboardStats | null>(null);
  const [loading, setLoading] = useState(true);
  
  useEffect(() => {
    const fetchStats = async () => {
      try {
        const data = await apiClient.getDashboardStats();
        setStats(data);
      } catch (error) {
        console.error('Failed to fetch dashboard stats:', error);
      } finally {
        setLoading(false);
      }
    };
    
    fetchStats();
    const interval = setInterval(fetchStats, 30000); // Update every 30s
    return () => clearInterval(interval);
  }, []);
  
  if (loading) return <div>Loading dashboard...</div>;
  
  return (
    <div className="grid grid-cols-1 md:grid-cols-4 gap-6">
      <StatCard 
        title="Active Gateways" 
        value={stats?.activeGateways || 0}
        status={stats?.systemHealth}
      />
      <StatCard 
        title="Total Policies" 
        value={stats?.totalPolicies || 0}
      />
      <StatCard 
        title="Security Events" 
        value={stats?.securityEvents || 0}
      />
      <StatCard 
        title="System Health" 
        value={stats?.systemHealth || 'unknown'}
      />
    </div>
  );
}
```

### 2. Gateway List Component
Create `components/gateways/GatewayList.tsx`:
```typescript
'use client';
import { useEffect, useState } from 'react';
import { apiClient } from '@/lib/api';
import { wsClient } from '@/lib/websocket';

interface Gateway {
  id: string;
  name: string;
  status: 'healthy' | 'warning' | 'error';
  region: string;
  activeConnections: number;
  lastSeen: string;
}

export default function GatewayList() {
  const [gateways, setGateways] = useState<Gateway[]>([]);
  
  useEffect(() => {
    // Initial load
    const fetchGateways = async () => {
      const data = await apiClient.getGateways();
      setGateways(data);
    };
    fetchGateways();
    
    // Real-time updates
    wsClient.connect('gateways', (update) => {
      setGateways(prev => 
        prev.map(gw => 
          gw.id === update.id ? { ...gw, ...update } : gw
        )
      );
    });
    
    return () => wsClient.disconnect('gateways');
  }, []);
  
  return (
    <div className="space-y-4">
      {gateways.map(gateway => (
        <GatewayCard key={gateway.id} gateway={gateway} />
      ))}
    </div>
  );
}
```

### 3. Policy Management Interface
Create `components/policies/PolicyManagement.tsx`:
```typescript
'use client';
import { useState, useEffect } from 'react';
import { apiClient } from '@/lib/api';

interface Policy {
  id: string;
  name: string;
  description: string;
  rules: PolicyRule[];
  enabled: boolean;
  createdAt: string;
}

export default function PolicyManagement() {
  const [policies, setPolicies] = useState<Policy[]>([]);
  const [showCreateForm, setShowCreateForm] = useState(false);
  
  useEffect(() => {
    const fetchPolicies = async () => {
      const data = await apiClient.getPolicies();
      setPolicies(data);
    };
    fetchPolicies();
  }, []);
  
  const handleCreatePolicy = async (policyData: PolicyCreateRequest) => {
    try {
      const newPolicy = await apiClient.createPolicy(policyData);
      setPolicies(prev => [...prev, newPolicy]);
      setShowCreateForm(false);
    } catch (error) {
      console.error('Failed to create policy:', error);
    }
  };
  
  return (
    <div className="space-y-6">
      <div className="flex justify-between items-center">
        <h2 className="text-2xl font-bold">Policy Management</h2>
        <button 
          onClick={() => setShowCreateForm(true)}
          className="bg-blue-600 text-white px-4 py-2 rounded"
        >
          Create Policy
        </button>
      </div>
      
      {showCreateForm && (
        <PolicyCreateForm 
          onSubmit={handleCreatePolicy}
          onCancel={() => setShowCreateForm(false)}
        />
      )}
      
      <PolicyList policies={policies} />
    </div>
  );
}
```

## 🎨 UI/UX Guidelines

### Design System
- **Color Scheme**: Dark theme with blue accents
- **Typography**: Inter font family
- **Spacing**: Tailwind's spacing scale (4, 6, 8, 12)
- **Components**: Consistent button styles, form inputs, cards

### Responsive Design
- **Mobile First**: Start with mobile layout
- **Breakpoints**: sm (640px), md (768px), lg (1024px), xl (1280px)
- **Grid System**: CSS Grid and Flexbox for layouts

### Accessibility
- **Keyboard Navigation**: All interactive elements accessible
- **Screen Reader**: Proper ARIA labels and semantic HTML
- **Color Contrast**: WCAG AA compliance

## 🧪 Development Workflow

### 1. Component Development
```bash
# Create component
mkdir -p components/feature-name
touch components/feature-name/ComponentName.tsx

# Add to index
echo "export { default } from './ComponentName';" > components/feature-name/index.ts
```

### 2. API Integration Testing
```bash
# Test API endpoints
curl http://localhost:8081/api/v1/dashboard/stats

# Test WebSocket connection
wscat -c ws://localhost:8081/ws/policies
```

### 3. Build and Deploy
```bash
# Development build
npm run dev

# Production build  
npm run build

# Export static files for backend integration
npm run export
```

## 📊 Implementation Priority

### Phase 1: Core Dashboard (Week 1)
1. **Dashboard Overview** - System stats and health
2. **Gateway List** - Gateway monitoring  
3. **Basic Navigation** - App shell and routing

### Phase 2: Policy Management (Week 2)  
1. **Policy List** - View existing policies
2. **Policy Editor** - Create/edit policies
3. **Real-time Updates** - WebSocket integration

### Phase 3: Analytics (Week 3)
1. **Security Dashboard** - Security metrics
2. **Performance Charts** - System performance
3. **Behavioral Analysis** - User behavior monitoring

## 🔗 Resources

- **Backend API**: Console running on `http://localhost:8081`
- **WebSocket**: Real-time connections at `/ws/*`  
- **Database**: All data persisted and ready for frontend
- **Documentation**: [Web Console Feature Docs](../../features/web-console/README.md)

## 🧭 Navigation

- **Parent**: [🛠️ Implementation](../README.md)
- **Previous**: [Development Setup](../setup-guides/development-setup.md)
- **Related**: [🎯 Web Console Features](../../features/web-console/README.md)
