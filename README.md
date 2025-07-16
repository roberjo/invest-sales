# Investment Product Sales Tool

[![Build Status](https://github.com/your-org/invest-sales/workflows/CI/badge.svg)](https://github.com/your-org/invest-sales/actions)
[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](https://opensource.org/licenses/MIT)
[![.NET](https://img.shields.io/badge/.NET-8.0-blue.svg)](https://dotnet.microsoft.com/)
[![React](https://img.shields.io/badge/React-18.0-blue.svg)](https://reactjs.org/)
[![PostgreSQL](https://img.shields.io/badge/PostgreSQL-15.0-blue.svg)](https://www.postgresql.org/)

A modern, browser-based sales tool designed for bankers to present and compare investment products with real-time rates, growth projections, and comprehensive administrative management.

## 📋 Table of Contents

- [Features](#-features)
- [Architecture](#-architecture)
- [Technology Stack](#-technology-stack)
- [Quick Start](#-quick-start)
- [Installation](#-installation)
- [Configuration](#-configuration)
- [Usage](#-usage)
- [API Documentation](#-api-documentation)
- [Development](#-development)
- [Testing](#-testing)
- [Deployment](#-deployment)
- [Security](#-security)
- [Contributing](#-contributing)
- [License](#-license)
- [Support](#-support)

## ✨ Features

### 🏦 Sales Representative Features
- **Product Comparison**: Side-by-side comparison of investment products
- **Real-time Rates**: Live rate updates and calculations
- **Growth Projections**: Interactive charts showing investment growth over time
- **Client Presentations**: Professional presentation mode for client meetings
- **Rate History**: Historical rate tracking and analysis
- **Mobile Responsive**: Works seamlessly on tablets and mobile devices

### 👨‍💼 Administrative Features
- **Product Management**: Add, edit, and manage investment products
- **Rate Management**: Update rates with effective dates and audit trails
- **User Management**: Role-based access control and user administration
- **Audit Logging**: Comprehensive audit trails for compliance
- **Reporting**: Sales analytics and performance reports
- **Data Export**: Export comparison data and reports

### 🔒 Security & Compliance
- **OAuth 2.0 Authentication**: Secure authentication via Okta
- **Role-based Access Control**: Granular permissions system
- **Audit Trails**: Complete audit logging for regulatory compliance
- **Data Encryption**: End-to-end encryption for sensitive data
- **GDPR Compliance**: Data retention and privacy controls

## 🏗️ Architecture

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                           Investment Product Sales Tool                    │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  ┌─────────────────┐    ┌─────────────────┐    ┌─────────────────┐        │
│  │   Frontend      │    │   API Gateway   │    │   Backend       │        │
│  │   (React)       │◄──►│   (API Gateway) │◄──►│   (ASP.NET)     │        │
│  │                 │    │                 │    │                 │        │
│  │ • React 18      │    │ • Rate Limiting │    │ • Lambda        │        │
│  │ • TypeScript    │    │ • Authentication│    │   Functions     │        │
│  │ • Tailwind CSS  │    │ • CORS          │    │ • Entity        │        │
│  │ • Vite          │    │ • Logging       │    │   Framework     │        │
│  └─────────────────┘    └─────────────────┘    └─────────────────┘        │
│           │                       │                       │                │
│           │                       │                       │                │
│           ▼                       ▼                       ▼                │
│  ┌─────────────────────────────────────────────────────────────────────┐    │
│  │                    Aurora PostgreSQL Cluster                       │    │
│  │                                                                     │    │
│  │  ┌─────────────────┐  ┌─────────────────┐  ┌─────────────────┐    │    │
│  │  │   Primary       │  │   Read Replica  │  │   Read Replica  │    │    │
│  │  │   Instance      │  │   Instance      │  │   Instance      │    │    │
│  │  │   (Writer)      │  │   (Reader)      │  │   (Reader)      │    │    │
│  │  └─────────────────┘  └─────────────────┘  └─────────────────┘    │    │
│  └─────────────────────────────────────────────────────────────────────┘    │
│                                                                             │
│  ┌─────────────────┐    ┌─────────────────┐    ┌─────────────────┐        │
│  │   Terraform     │    │   Monitoring    │    │   Backup        │        │
│  │   Cloud         │    │   (CloudWatch)  │    │   (S3)          │        │
│  └─────────────────┘    └─────────────────┘    └─────────────────┘        │
└─────────────────────────────────────────────────────────────────────────────┘
```

## 🛠️ Technology Stack

### Frontend
- **React 18** - Modern UI framework with hooks and functional components
- **TypeScript** - Type-safe JavaScript development
- **Tailwind CSS** - Utility-first CSS framework for rapid UI development
- **Vite** - Fast build tool and development server
- **React Router** - Client-side routing
- **React Query** - Server state management and caching
- **Recharts** - Interactive charts and data visualization

### Backend
- **ASP.NET Core 8** - High-performance web framework
- **Entity Framework Core** - Object-relational mapping
- **AWS Lambda** - Serverless compute for scalability
- **API Gateway** - RESTful API management
- **C# 12** - Modern programming language with latest features

### Database
- **Amazon Aurora PostgreSQL** - Fully managed relational database
- **Row-Level Security** - Fine-grained access control
- **Audit Logging** - Comprehensive audit trails
- **Data Encryption** - At-rest and in-transit encryption

### Infrastructure
- **Terraform Cloud** - Infrastructure as Code
- **AWS** - Cloud infrastructure and services
- **Docker** - Containerization
- **GitHub Actions** - CI/CD pipeline

### Security & Authentication
- **Okta** - OAuth 2.0 authentication and authorization
- **JWT Tokens** - Secure token-based authentication
- **HTTPS/TLS** - Encrypted communication
- **CORS** - Cross-origin resource sharing

## 🚀 Quick Start

### Prerequisites
- [.NET 8 SDK](https://dotnet.microsoft.com/download/dotnet/8.0)
- [Node.js 18+](https://nodejs.org/)
- [PostgreSQL 15+](https://www.postgresql.org/)
- [Docker](https://www.docker.com/) (optional)

### Quick Installation

1. **Clone the repository**
   ```bash
   git clone https://github.com/your-org/invest-sales.git
   cd invest-sales
   ```

2. **Set up the database**
   ```bash
   # Using Docker
   docker run --name postgres-invest -e POSTGRES_PASSWORD=password -e POSTGRES_DB=investment_tool -p 5432:5432 -d postgres:15
   
   # Or install PostgreSQL locally
   ```

3. **Configure the application**
   ```bash
   # Copy configuration files
   cp src/Backend/appsettings.Development.json.example src/Backend/appsettings.Development.json
   cp src/Frontend/.env.example src/Frontend/.env
   ```

4. **Run the application**
   ```bash
   # Start the backend
   cd src/Backend
   dotnet restore
   dotnet ef database update
   dotnet run
   
   # Start the frontend (in a new terminal)
   cd src/Frontend
   npm install
   npm run dev
   ```

5. **Access the application**
   - Frontend: http://localhost:5173
   - Backend API: http://localhost:5000
   - API Documentation: http://localhost:5000/swagger

## 📦 Installation

### Detailed Installation Steps

#### 1. Backend Setup

```bash
# Navigate to backend directory
cd src/Backend

# Install dependencies
dotnet restore

# Configure database connection
# Edit appsettings.Development.json with your database connection string

# Run database migrations
dotnet ef database update

# Start the application
dotnet run
```

#### 2. Frontend Setup

```bash
# Navigate to frontend directory
cd src/Frontend

# Install dependencies
npm install

# Configure environment variables
cp .env.example .env
# Edit .env with your configuration

# Start development server
npm run dev
```

#### 3. Database Setup

```sql
-- Create database
CREATE DATABASE investment_tool;

-- Create user (optional)
CREATE USER investment_user WITH PASSWORD 'secure_password';
GRANT ALL PRIVILEGES ON DATABASE investment_tool TO investment_user;
```

## ⚙️ Configuration

### Environment Variables

#### Backend (.env)
```bash
# Database
DATABASE_CONNECTION_STRING=Host=localhost;Database=investment_tool;Username=postgres;Password=password

# Authentication
OKTA_DOMAIN=your-domain.okta.com
OKTA_CLIENT_ID=your-client-id
OKTA_CLIENT_SECRET=your-client-secret

# AWS Configuration
AWS_REGION=us-east-1
AWS_ACCESS_KEY_ID=your-access-key
AWS_SECRET_ACCESS_KEY=your-secret-key

# Application Settings
ASPNETCORE_ENVIRONMENT=Development
ASPNETCORE_URLS=http://localhost:5000
```

#### Frontend (.env)
```bash
# API Configuration
VITE_API_BASE_URL=http://localhost:5000
VITE_API_TIMEOUT=30000

# Authentication
VITE_OKTA_DOMAIN=your-domain.okta.com
VITE_OKTA_CLIENT_ID=your-client-id
VITE_OKTA_REDIRECT_URI=http://localhost:5173/callback

# Feature Flags
VITE_ENABLE_ANALYTICS=true
VITE_ENABLE_DEBUG_MODE=false
```

### Database Configuration

```json
{
  "ConnectionStrings": {
    "DefaultConnection": "Host=localhost;Database=investment_tool;Username=postgres;Password=password;Pooling=true;Minimum Pool Size=5;Maximum Pool Size=100;"
  },
  "Database": {
    "EnableSensitiveDataLogging": false,
    "EnableDetailedErrors": true,
    "CommandTimeout": 30
  }
}
```

## 📖 Usage

### For Sales Representatives

1. **Login to the application**
   - Navigate to the login page
   - Use your Okta credentials to authenticate

2. **Browse Products**
   - View available investment products
   - Filter by product type, rate range, or investment amount

3. **Compare Products**
   - Select multiple products for comparison
   - Enter investment amount and time horizon
   - View side-by-side comparison with growth projections

4. **Create Presentations**
   - Generate professional presentations for clients
   - Export comparison data to PDF or Excel
   - Save comparison sessions for future reference

### For Administrators

1. **Manage Products**
   - Add new investment products
   - Update product information and rates
   - Set product availability dates

2. **User Management**
   - Manage user accounts and roles
   - Monitor user activity and sessions
   - Generate user access reports

3. **System Monitoring**
   - View audit logs and system events
   - Monitor performance metrics
   - Generate compliance reports

## 📚 API Documentation

### RESTful API Endpoints

#### Authentication
```http
POST /api/auth/login
POST /api/auth/logout
GET  /api/auth/user
```

#### Products
```http
GET    /api/products
GET    /api/products/{id}
POST   /api/products
PUT    /api/products/{id}
DELETE /api/products/{id}
GET    /api/products/{id}/rates
POST   /api/products/{id}/rates
```

#### Comparisons
```http
POST /api/comparisons
GET   /api/comparisons/{id}
GET   /api/comparisons/user/{userId}
```

#### Users
```http
GET    /api/users
GET    /api/users/{id}
PUT    /api/users/{id}
DELETE /api/users/{id}
GET    /api/users/{id}/roles
POST   /api/users/{id}/roles
```

### API Response Format

```json
{
  "success": true,
  "data": {
    "productId": "550e8400-e29b-41d4-a716-446655440001",
    "cusip": "123456789",
    "productName": "Premium Annuity Plus",
    "baseRate": 4.5,
    "bonusRate": 0.5,
    "minimumInvestment": 10000
  },
  "message": "Product retrieved successfully",
  "timestamp": "2024-01-15T10:30:00Z"
}
```

## 🛠️ Development

### Development Environment Setup

1. **Install Development Tools**
   ```bash
   # Install .NET 8 SDK
   # Install Node.js 18+
   # Install PostgreSQL 15+
   # Install Docker Desktop
   ```

2. **Clone and Setup**
   ```bash
   git clone https://github.com/your-org/invest-sales.git
   cd invest-sales
   
   # Setup pre-commit hooks
   pre-commit install
   ```

3. **Database Setup**
   ```bash
   # Using Docker
   docker-compose up -d postgres
   
   # Run migrations
   dotnet ef database update --project src/Backend
   ```

4. **Start Development Servers**
   ```bash
   # Backend
   cd src/Backend
   dotnet watch run
   
   # Frontend (new terminal)
   cd src/Frontend
   npm run dev
   ```

### Code Style and Standards

#### C# (.NET)
- Follow Microsoft C# coding conventions
- Use `dotnet format` for code formatting
- Enable nullable reference types
- Use async/await patterns consistently

#### TypeScript/JavaScript
- Use ESLint and Prettier for code formatting
- Follow Airbnb JavaScript style guide
- Use TypeScript strict mode
- Prefer functional components with hooks

#### Database
- Use Entity Framework migrations
- Follow naming conventions (snake_case for PostgreSQL)
- Include proper indexes and constraints
- Document all stored procedures

### Testing

#### Backend Testing
```bash
# Run unit tests
dotnet test src/Backend.Tests

# Run integration tests
dotnet test src/Backend.IntegrationTests

# Run with coverage
dotnet test --collect:"XPlat Code Coverage"
```

#### Frontend Testing
```bash
# Run unit tests
npm test

# Run with coverage
npm run test:coverage

# Run E2E tests
npm run test:e2e
```

## 🧪 Testing

### Test Categories

#### Unit Tests
- **Backend**: Service layer, business logic, utilities
- **Frontend**: Component rendering, hooks, utilities
- **Database**: Repository methods, data access patterns

#### Integration Tests
- **API Endpoints**: Full request/response cycles
- **Database Integration**: Entity Framework operations
- **Authentication**: OAuth flow and token validation

#### End-to-End Tests
- **User Workflows**: Complete user journeys
- **Cross-browser Testing**: Chrome, Firefox, Safari, Edge
- **Mobile Testing**: Responsive design validation

### Test Data

```csharp
// Sample test data
public static class TestData
{
    public static Product CreateSampleProduct()
    {
        return new Product
        {
            ProductId = Guid.NewGuid(),
            Cusip = "123456789",
            ProductName = "Test Annuity",
            ProductType = "Annuity",
            BaseRate = 4.5m,
            MinimumInvestment = 10000m,
            IsActive = true
        };
    }
}
```

## 🚀 Deployment

### Production Deployment

#### Using Terraform Cloud

1. **Configure Terraform Cloud**
   ```bash
   # Set up Terraform Cloud workspace
   terraform login
   terraform workspace new production
   ```

2. **Deploy Infrastructure**
   ```bash
   # Initialize Terraform
   terraform init
   
   # Plan deployment
   terraform plan
   
   # Apply changes
   terraform apply
   ```

3. **Deploy Application**
   ```bash
   # Build application
   dotnet publish -c Release
   
   # Deploy to AWS Lambda
   aws lambda update-function-code --function-name invest-sales-api --zip-file fileb://deploy.zip
   ```

#### Manual Deployment

1. **Build Application**
   ```bash
   # Backend
   dotnet publish -c Release -o ./publish
   
   # Frontend
   npm run build
   ```

2. **Deploy to AWS**
   ```bash
   # Upload to S3
   aws s3 sync ./publish s3://your-bucket-name/
   
   # Update CloudFront
   aws cloudfront create-invalidation --distribution-id YOUR_DISTRIBUTION_ID --paths "/*"
   ```

### Environment Configuration

#### Production Environment Variables
```bash
# Database
DATABASE_CONNECTION_STRING=Host=your-aurora-cluster.amazonaws.com;Database=investment_tool;Username=admin;Password=secure_password

# Authentication
OKTA_DOMAIN=your-domain.okta.com
OKTA_CLIENT_ID=your-production-client-id
OKTA_CLIENT_SECRET=your-production-client-secret

# AWS Configuration
AWS_REGION=us-east-1
AWS_ACCESS_KEY_ID=your-production-access-key
AWS_SECRET_ACCESS_KEY=your-production-secret-key

# Application Settings
ASPNETCORE_ENVIRONMENT=Production
ASPNETCORE_URLS=https://api.yourdomain.com
```

## 🔒 Security

### Security Features

- **Authentication**: OAuth 2.0 with Okta
- **Authorization**: Role-based access control (RBAC)
- **Data Encryption**: AES-256 encryption at rest and in transit
- **Audit Logging**: Comprehensive audit trails for compliance
- **Input Validation**: Server-side validation and sanitization
- **SQL Injection Prevention**: Parameterized queries and ORM usage
- **XSS Protection**: Content Security Policy (CSP) headers
- **CSRF Protection**: Anti-forgery tokens

### Security Best Practices

1. **Regular Security Updates**
   - Keep dependencies updated
   - Monitor security advisories
   - Apply patches promptly

2. **Access Control**
   - Principle of least privilege
   - Regular access reviews
   - Multi-factor authentication

3. **Data Protection**
   - Encrypt sensitive data
   - Implement data retention policies
   - Regular backup testing

4. **Monitoring and Alerting**
   - Security event monitoring
   - Intrusion detection
   - Automated alerting

## 🤝 Contributing

We welcome contributions! Please read our contributing guidelines.

### How to Contribute

1. **Fork the repository**
   ```bash
   git clone https://github.com/your-username/invest-sales.git
   cd invest-sales
   ```

2. **Create a feature branch**
   ```bash
   git checkout -b feature/your-feature-name
   ```

3. **Make your changes**
   - Follow the coding standards
   - Add tests for new functionality
   - Update documentation

4. **Commit your changes**
   ```bash
   git add .
   git commit -m "feat: add new feature description"
   ```

5. **Push and create a pull request**
   ```bash
   git push origin feature/your-feature-name
   # Create PR on GitHub
   ```

### Development Guidelines

- **Code Review**: All changes require review
- **Testing**: Maintain >80% code coverage
- **Documentation**: Update docs for new features
- **Performance**: Consider performance impact
- **Security**: Follow security best practices

### Issue Reporting

When reporting issues, please include:
- **Environment**: OS, browser, version
- **Steps to reproduce**: Detailed steps
- **Expected behavior**: What should happen
- **Actual behavior**: What actually happens
- **Screenshots**: If applicable
- **Logs**: Error messages and stack traces

## 📄 License

This project is licensed under the MIT License - see the [LICENSE](LICENSE) file for details.

```
MIT License

Copyright (c) 2024 Investment Product Sales Tool

Permission is hereby granted, free of charge, to any person obtaining a copy
of this software and associated documentation files (the "Software"), to deal
in the Software without restriction, including without limitation the rights
to use, copy, modify, merge, publish, distribute, sublicense, and/or sell
copies of the Software, and to permit persons to whom the Software is
furnished to do so, subject to the following conditions:

The above copyright notice and this permission notice shall be included in all
copies or substantial portions of the Software.

THE SOFTWARE IS PROVIDED "AS IS", WITHOUT WARRANTY OF ANY KIND, EXPRESS OR
IMPLIED, INCLUDING BUT NOT LIMITED TO THE WARRANTIES OF MERCHANTABILITY,
FITNESS FOR A PARTICULAR PURPOSE AND NONINFRINGEMENT. IN NO EVENT SHALL THE
AUTHORS OR COPYRIGHT HOLDERS BE LIABLE FOR ANY CLAIM, DAMAGES OR OTHER
LIABILITY, WHETHER IN AN ACTION OF CONTRACT, TORT OR OTHERWISE, ARISING FROM,
OUT OF OR IN CONNECTION WITH THE SOFTWARE OR THE USE OR OTHER DEALINGS IN THE
SOFTWARE.
```

## 📞 Support

### Getting Help

- **Documentation**: Check our [docs](docs/) directory
- **Issues**: [GitHub Issues](https://github.com/your-org/invest-sales/issues)
- **Discussions**: [GitHub Discussions](https://github.com/your-org/invest-sales/discussions)
- **Email**: support@yourdomain.com

### Community

- **Slack**: Join our [Slack workspace](https://your-slack-workspace.slack.com)
- **Discord**: Join our [Discord server](https://discord.gg/your-invite)
- **Twitter**: Follow us [@yourhandle](https://twitter.com/yourhandle)

### Professional Support

For enterprise customers and professional support:
- **Email**: enterprise@yourdomain.com
- **Phone**: +1 (555) 123-4567
- **Hours**: Monday-Friday, 9 AM - 6 PM EST

## 🙏 Acknowledgments

- **Okta** for authentication services
- **AWS** for cloud infrastructure
- **Terraform** for infrastructure as code
- **React** and **ASP.NET Core** communities
- **Contributors** who have helped improve this project

---

**Made with ❤️ by the Investment Product Sales Tool Team**

[Back to Top](#investment-product-sales-tool)
