# 🚀 Development Setup Guide

## Overview

Complete guide for setting up a local Rustronaut development environment. This guide covers everything needed to develop both backend components and the React/TypeScript frontend.

**Target Audience**: Developers who want to contribute to or extend Rustronaut

## 📋 Prerequisites

### System Requirements
- **Operating System**: macOS, Linux, or Windows (with WSL2)
- **RAM**: Minimum 8GB, Recommended 16GB
- **Storage**: At least 5GB free space

### Required Software
- **Rust**: Latest stable (1.70+)
- **Node.js**: Version 18+ with npm
- **Git**: For repository management
- **Database**: PostgreSQL 14+ or SQLite (automatic)

### Optional Tools
- **Docker**: For containerized development
- **VS Code**: Recommended IDE with Rust and TypeScript extensions
- **Postman**: For API testing

## 🛠️ Installation Steps

### 1. Install Rust
```bash
# Install Rust using rustup
curl --proto '=https' --tlsv1.2 -sSf https://sh.rustup.rs | sh

# Restart shell or source the environment
source ~/.zshrc  # or ~/.bashrc

# Verify installation
rustc --version
cargo --version
```

### 2. Install Node.js and npm
```bash
# Using Homebrew on macOS
brew install node

# Or download from https://nodejs.org/
# Verify installation
node --version
npm --version
```

### 3. Clone Rustronaut Repository
```bash
# Clone the repository
git clone <repository-url>
cd rustronaut

# Verify project structure
ls -la
```

### 4. Database Setup

#### Option A: SQLite (Recommended for Development)
SQLite is automatically configured and requires no additional setup.

#### Option B: PostgreSQL (Production-like)
```bash
# Install PostgreSQL
brew install postgresql
brew services start postgresql

# Create database
createdb rustronaut_dev

# Update console/config/local.toml
[database]
url = "postgresql://username:password@localhost/rustronaut_dev"
```

## 🚀 Starting Development Environment

### 1. Build All Components
```bash
# Build entire workspace
cargo build --workspace

# Or build individual components
cd console && cargo build
cd gateway && cargo build  
cd tunnel && cargo build
```

### 2. Start Backend Services

#### Terminal 1: Console (Management Server)
```bash
cd console
cargo run
# Console starts on http://localhost:8081
```

#### Terminal 2: Gateway (Proxy)
```bash
cd gateway
cargo run
# Gateway starts on http://localhost:8080
```

#### Terminal 3: Tunnel (WireGuard Manager)
```bash
cd tunnel  
cargo run
# Tunnel service starts
```

### 3. Start Frontend Development (If Available)
```bash
cd console/web-console
npm install
npm run dev
# Frontend starts on http://localhost:3000
```

## 🧪 Verify Installation

### 1. Run Integration Tests
```bash
# From project root
./scripts/integration_test_enhanced.sh
```

### 2. Test API Endpoints
```bash
# Test console API
curl http://localhost:8081/api/v1/dashboard/stats

# Test gateway health
curl http://localhost:8080/health
```

### 3. Check Component Status
```bash
# Check if all components compiled successfully
cargo check --workspace
```

## 🔧 Development Configuration

### Console Configuration
Edit `console/config/local.toml`:
```toml
[server]
host = "127.0.0.1"
port = 8081

[database]
url = "sqlite:data/console.db"
max_connections = 5

[logging]
level = "debug"
```

### Gateway Configuration  
Edit `gateway/gateway.yml`:
```yaml
proxy:
  bind_addr: "127.0.0.1:8080"
  
security:
  policy_engine_enabled: true
  
logging:
  level: debug
```

## 🎯 Frontend Development Setup

### Prerequisites for Frontend Work
```bash
# Install frontend dependencies
cd console/web-console
npm install

# Verify TypeScript compilation
npm run type-check

# Start development server
npm run dev
```

### Available Frontend Commands
```bash
# Development server with hot reload
npm run dev

# Build for production
npm run build

# Static export for backend integration
npm run export

# Type checking
npm run type-check

# Linting
npm run lint
```

### Backend API Integration
The frontend is configured to proxy API requests to the backend console:

- **Development**: API calls go to `http://localhost:8081`
- **Production**: API calls use relative paths served by console
- **WebSocket**: Real-time connections at `/ws` endpoint

## 🚨 Troubleshooting

### Common Issues

#### "Cargo build failed"
```bash
# Update Rust toolchain
rustup update

# Clean build cache
cargo clean
cargo build
```

#### "Database connection failed"
```bash
# For SQLite: Check file permissions
ls -la console/data/

# For PostgreSQL: Check service status
brew services list | grep postgresql
```

#### "Frontend won't start"
```bash
# Clear npm cache
npm cache clean --force

# Reinstall dependencies
rm -rf node_modules
npm install
```

#### "Port already in use"
```bash
# Find process using port
lsof -i :8081

# Kill process
kill -9 <PID>
```

### Log Locations
- **Console logs**: Terminal output and `console/logs/`
- **Gateway logs**: Terminal output and `gateway/logs/`
- **Database logs**: Check `console/data/` for SQLite

## 🎯 Next Steps

After successful setup:

1. **Explore Codebase**: Review component architecture in each `src/` directory
2. **Run Tests**: Execute integration tests to understand functionality
3. **Check Documentation**: Review [Architecture Documentation](../architecture/README.md)
4. **Start Development**: Begin with [Frontend Development Guide](development/frontend-development.md)

## 🧭 Navigation

- **Parent**: [🛠️ Implementation](../README.md)
- **Next**: [Frontend Development Guide](../development/frontend-development.md)
- **Related**: [🏗️ Architecture](../../architecture/README.md)
