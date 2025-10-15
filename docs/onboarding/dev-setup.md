# Developer Setup Guide

## Welcome! 👋

This guide will help you set up your local development environment for the Blockchain DApp Platform. Follow these steps to get up and running quickly.

## Table of Contents

- [Prerequisites](#prerequisites)
- [Repository Setup](#repository-setup)
- [Environment Configuration](#environment-configuration)
- [Component Setup](#component-setup)
  - [Backend Setup (Go)](#backend-setup-go)
  - [Frontend Setup (React)](#frontend-setup-react)
  - [Mobile Setup (React Native)](#mobile-setup-react-native)
  - [Infrastructure Setup (Terraform)](#infrastructure-setup-terraform)
- [Running the Application](#running-the-application)
- [Development Workflow](#development-workflow)
- [Testing](#testing)
- [Troubleshooting](#troubleshooting)
- [Next Steps](#next-steps)

## Prerequisites

Before you begin, ensure you have the following tools installed on your development machine:

### Required Tools

| Tool | Minimum Version | Installation Guide |
|------|----------------|--------------------|
| **Git** | 2.30+ | [git-scm.com](https://git-scm.com/downloads) |
| **Docker** | 20.10+ | [docs.docker.com](https://docs.docker.com/get-docker/) |
| **Docker Compose** | 2.0+ | Usually included with Docker Desktop |
| **Node.js** | 18.x LTS | [nodejs.org](https://nodejs.org/) |
| **npm** | 8.0+ | Included with Node.js |
| **Go** | 1.19+ | [go.dev](https://go.dev/doc/install) |
| **Make** | 4.0+ | Pre-installed on macOS/Linux, [Windows](http://gnuwin32.sourceforge.net/packages/make.htm) |

### Optional Tools (for specific tasks)

| Tool | Purpose | Installation |
|------|---------|-------------|
| **Terraform** | Infrastructure provisioning | [terraform.io](https://www.terraform.io/downloads) (v1.5+) |
| **kubectl** | Kubernetes management | [kubernetes.io](https://kubernetes.io/docs/tasks/tools/) (v1.24+) |
| **AWS CLI** | AWS resource management | [aws.amazon.com](https://aws.amazon.com/cli/) (v2.0+) |
| **React Native CLI** | Mobile development | [reactnative.dev](https://reactnative.dev/docs/environment-setup) |
| **Xcode** | iOS development | Mac App Store (macOS only) |
| **Android Studio** | Android development | [developer.android.com](https://developer.android.com/studio) |

### Verify Installations

Run these commands to verify your installations:

```bash
# Check versions
git --version
docker --version
docker-compose --version
node --version
npm --version
go version
make --version

# Optional tools
terraform --version
kubectl version --client
aws --version
```

## Repository Setup

### 1. Clone the Repository

```bash
# Clone via HTTPS
git clone https://github.com/your-org/Blockchain-DApp.git

# OR clone via SSH (recommended if you have SSH keys set up)
git clone git@github.com:your-org/Blockchain-DApp.git

# Navigate to the project directory
cd Blockchain-DApp
```

### 2. Configure Git

```bash
# Set your name and email (if not already configured)
git config user.name "Your Name"
git config user.email "your.email@example.com"

# Verify configuration
git config --list
```

### 3. Create a Feature Branch

```bash
# Always work on a feature branch, never directly on main
git checkout -b feature/your-feature-name
```

## Environment Configuration

### 1. Create Environment Files

Each component needs its own environment configuration:

**Backend (.env)**
```bash
cd backend
cp .env.example .env  # If .env.example exists

# OR create .env manually
cat > .env << EOF
# Database Configuration
DB_HOST=localhost
DB_PORT=5432
DB_NAME=blockchain_dapp_dev
DB_USER=postgres
DB_PASSWORD=postgres
DB_SSL_MODE=disable

# Redis Configuration
REDIS_HOST=localhost
REDIS_PORT=6379
REDIS_PASSWORD=

# Server Configuration
PORT=8080
ENV=development

# JWT Secret (generate a secure random string)
JWT_SECRET=your-secret-key-here-change-in-production

# Blockchain Configuration (example)
BLOCKCHAIN_RPC_URL=http://localhost:8545
BLOCKCHAIN_NETWORK=testnet
EOF
```

**Frontend (.env)**
```bash
cd ../app
cat > .env.local << EOF
# API Configuration
REACT_APP_API_URL=http://localhost:8080

# Environment
REACT_APP_ENV=development

# Feature Flags
REACT_APP_ENABLE_ANALYTICS=false
EOF
```

**Mobile (.env)**
```bash
cd ../mobile
cat > .env << EOF
# API Configuration
API_URL=http://localhost:8080

# Environment
ENV=development
EOF
```

### 2. Start Local Services (Docker)

Create a `docker-compose.yml` in the project root for local development services:

```bash
cd ..
cat > docker-compose.dev.yml << EOF
version: '3.8'

services:
  postgres:
    image: postgres:14-alpine
    container_name: blockchain-postgres
    environment:
      POSTGRES_DB: blockchain_dapp_dev
      POSTGRES_USER: postgres
      POSTGRES_PASSWORD: postgres
    ports:
      - "5432:5432"
    volumes:
      - postgres_data:/var/lib/postgresql/data
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U postgres"]
      interval: 10s
      timeout: 5s
      retries: 5

  redis:
    image: redis:7-alpine
    container_name: blockchain-redis
    ports:
      - "6379:6379"
    healthcheck:
      test: ["CMD", "redis-cli", "ping"]
      interval: 10s
      timeout: 5s
      retries: 5

  # Optional: Local blockchain node (Ganache)
  ganache:
    image: trufflesuite/ganache:latest
    container_name: blockchain-ganache
    ports:
      - "8545:8545"
    command:
      - --deterministic
      - --accounts=10
      - --networkId=1337

volumes:
  postgres_data:
EOF

# Start services
docker-compose -f docker-compose.dev.yml up -d

# Verify services are running
docker-compose -f docker-compose.dev.yml ps
```

## Component Setup

### Backend Setup (Go)

```bash
cd backend

# Download dependencies
go mod download

# Verify dependencies
go mod verify

# Run database migrations (if migration files exist)
# go run cmd/migrate/main.go up

# Build the application
go build -o bin/server cmd/server/main.go

# Run tests
go test ./...

# Run with hot reload (install air first: go install github.com/cosmtrek/air@latest)
air  # OR go run cmd/server/main.go
```

**Expected Output**:
```
Server starting on :8080
Connected to PostgreSQL
Connected to Redis
```

**Useful Commands**:
```bash
# Format code
go fmt ./...

# Run linter (install golangci-lint first)
golangci-lint run

# Check for vulnerabilities
go list -json -m all | nancy sleuth

# Generate test coverage
go test -coverprofile=coverage.out ./...
go tool cover -html=coverage.out
```

### Frontend Setup (React)

```bash
cd app

# Install dependencies
npm ci  # Use 'ci' for clean install based on package-lock.json

# Start development server
npm start

# The app will open in your browser at http://localhost:3000
```

**Expected Output**:
```
Webpack compiled successfully!
App running at:
- Local:   http://localhost:3000
- Network: http://192.168.1.x:3000
```

**Useful Commands**:
```bash
# Run tests
npm test

# Run tests with coverage
npm test -- --coverage

# Build for production
npm run build

# Lint code
npm run lint

# Format code (if Prettier is configured)
npm run format

# Analyze bundle size
npm run build
npx source-map-explorer 'build/static/js/*.js'
```

### Mobile Setup (React Native)

#### Prerequisites

**For iOS (macOS only)**:
- Xcode 14+
- CocoaPods: `sudo gem install cocoapods`

**For Android**:
- Android Studio with Android SDK
- Java Development Kit (JDK) 11
- Set `ANDROID_HOME` environment variable

#### Setup Steps

```bash
cd mobile

# Install JavaScript dependencies
npm ci

# iOS: Install CocoaPods dependencies
cd ios
pod install
cd ..

# Android: No additional steps needed
```

#### Run on iOS Simulator

```bash
# Start Metro bundler
npm start

# In another terminal, run iOS
npm run ios

# OR specify a device
npm run ios -- --simulator="iPhone 14 Pro"
```

#### Run on Android Emulator

```bash
# Start an Android emulator first (via Android Studio)
# OR list available devices
adb devices

# Start Metro bundler
npm start

# In another terminal, run Android
npm run android
```

#### Run on Physical Device

**iOS**:
1. Open `mobile/ios/BlockchainDApp.xcworkspace` in Xcode
2. Connect your iPhone via USB
3. Select your device in Xcode
4. Update `API_URL` in `.env` to your computer's local IP
5. Click Run

**Android**:
1. Enable Developer Options and USB Debugging on your device
2. Connect via USB
3. Update `API_URL` in `.env` to your computer's local IP
4. Run: `npm run android`

### Infrastructure Setup (Terraform)

**⚠️ Warning**: Only needed if you're working on infrastructure. This will create real AWS resources and may incur costs.

#### Prerequisites

1. AWS Account with appropriate permissions
2. AWS CLI configured with credentials
3. Terraform installed (v1.5+)

#### Setup Steps

```bash
# Configure AWS credentials
aws configure
# Enter your AWS Access Key ID, Secret Access Key, Region (us-east-1), and output format (json)

# Verify AWS access
aws sts get-caller-identity

# Navigate to dev environment
cd infra/env/dev

# Initialize Terraform
terraform init

# Review the plan
terraform plan

# Apply changes (only if you're sure!)
# terraform apply
```

**Note**: Always test infrastructure changes in the `dev` environment first.

## Running the Application

### Full Stack Local Development

**Option 1: Using Makefile** (recommended)

```bash
# From project root
make dev-setup     # One-time setup
make dev           # Start all services
```

**Option 2: Manual Start**

**Terminal 1 - Backend**:
```bash
cd backend
go run cmd/server/main.go
```

**Terminal 2 - Frontend**:
```bash
cd app
npm start
```

**Terminal 3 - Mobile** (optional):
```bash
cd mobile
npm start
```

### Access the Application

- **Web App**: http://localhost:3000
- **Backend API**: http://localhost:8080
- **API Health Check**: http://localhost:8080/health
- **PostgreSQL**: localhost:5432
- **Redis**: localhost:6379

## Development Workflow

### 1. Working on a Feature

```bash
# Create a feature branch
git checkout -b feature/add-user-authentication

# Make your changes
# ... edit code ...

# Stage changes
git add .

# Commit with descriptive message
git commit -m "feat: add JWT authentication to user login"

# Push to remote
git push origin feature/add-user-authentication
```

### 2. Commit Message Convention

We follow [Conventional Commits](https://www.conventionalcommits.org/):

```
feat: add new feature
fix: bug fix
docs: documentation changes
style: code style changes (formatting, etc.)
refactor: code refactoring
test: add or update tests
chore: maintenance tasks
```

### 3. Creating a Pull Request

1. Push your branch to GitHub
2. Go to the repository on GitHub
3. Click "New Pull Request"
4. Select your feature branch
5. Fill in the PR template:
   - Description of changes
   - Related issue number
   - Testing performed
   - Screenshots (if UI changes)

### 4. Code Review Process

- All PRs require at least one approval
- CI checks must pass (tests, linting, security scans)
- Address review comments
- Squash commits before merging (if requested)

## Testing

### Backend Tests

```bash
cd backend

# Run all tests
go test ./...

# Run tests with verbose output
go test -v ./...

# Run tests with coverage
go test -cover ./...

# Run specific test
go test -run TestUserLogin ./internal/auth

# Run tests in parallel
go test -parallel 4 ./...
```

### Frontend Tests

```bash
cd app

# Run tests in watch mode
npm test

# Run all tests once
npm test -- --watchAll=false

# Run with coverage
npm test -- --coverage --watchAll=false

# Update snapshots
npm test -- -u
```

### E2E Tests (Future)

```bash
# Install Playwright or Cypress
npm install -D @playwright/test

# Run E2E tests
npm run test:e2e
```

## Troubleshooting

### Common Issues and Solutions

#### Issue: Backend can't connect to PostgreSQL

**Solution**:
```bash
# Check if PostgreSQL container is running
docker-compose -f docker-compose.dev.yml ps

# Check container logs
docker logs blockchain-postgres

# Restart PostgreSQL
docker-compose -f docker-compose.dev.yml restart postgres

# Verify connection
psql -h localhost -U postgres -d blockchain_dapp_dev
```

#### Issue: Port already in use

**Solution**:
```bash
# Find process using port 8080
lsof -i :8080  # macOS/Linux
netstat -ano | findstr :8080  # Windows

# Kill the process
kill -9 <PID>  # macOS/Linux
taskkill /PID <PID> /F  # Windows

# OR change port in .env file
```

#### Issue: Frontend can't connect to backend API

**Solution**:
1. Verify backend is running: `curl http://localhost:8080/health`
2. Check `REACT_APP_API_URL` in `app/.env.local`
3. Check browser console for CORS errors
4. Ensure backend has CORS enabled for `http://localhost:3000`

#### Issue: Go module errors

**Solution**:
```bash
cd backend

# Clean module cache
go clean -modcache

# Re-download dependencies
go mod download

# Tidy up go.mod and go.sum
go mod tidy
```

#### Issue: npm install failures

**Solution**:
```bash
# Clear npm cache
npm cache clean --force

# Delete node_modules and package-lock.json
rm -rf node_modules package-lock.json

# Reinstall
npm install
```

#### Issue: Docker services won't start

**Solution**:
```bash
# Stop all containers
docker-compose -f docker-compose.dev.yml down

# Remove volumes (WARNING: deletes data)
docker-compose -f docker-compose.dev.yml down -v

# Rebuild and start
docker-compose -f docker-compose.dev.yml up -d --build
```

#### Issue: React Native build failures

**iOS**:
```bash
cd mobile/ios
pod deintegrate
pod install
cd ..
npm run ios
```

**Android**:
```bash
cd mobile/android
./gradlew clean
cd ..
npm run android
```

### Getting Help

1. **Check Documentation**: Review [docs/](../../docs/)
2. **Search Issues**: Check GitHub Issues for similar problems
3. **Ask the Team**: Post in team Slack channel
4. **Create an Issue**: If it's a bug, create a GitHub Issue with:
   - Steps to reproduce
   - Expected vs actual behavior
   - Environment details (OS, versions)
   - Error messages and logs

## Next Steps

Now that your environment is set up:

1. ✅ **Explore the Codebase**: Familiarize yourself with the project structure
2. ✅ **Read Architecture Docs**: Understand system design ([docs/architecture/](../architecture/))
3. ✅ **Pick a Starter Issue**: Look for "good first issue" labels on GitHub
4. ✅ **Join Team Meetings**: Participate in standups and planning sessions
5. ✅ **Review Runbooks**: Learn operational procedures ([docs/runbooks/](../runbooks/))

## Quick Reference

### Essential Commands

```bash
# Start everything
make dev

# Run tests
make test

# Stop services
make stop

# Clean up
make clean

# See all available commands
make help
```

### Helpful Resources

- [System Overview](../architecture/system-overview.md)
- [Data Flow](../architecture/data-flow.md)
- [Runbooks](../runbooks/)
- [GitHub Repository](https://github.com/your-org/Blockchain-DApp)
- [Team Wiki](https://wiki.example.com)

---

**Welcome to the team! Happy coding! 🚀**