# Investment Product Sales Tool

## 🎯 Project Overview

The Investment Product Sales Tool is a modern, browser-based application designed to streamline the presentation and comparison of investment products (Annuities, CDs, and other investment products) by bankers and financial advisors to their customers. The tool provides real-time product comparisons, rate calculations, and growth projections to facilitate informed decision-making during customer interactions.

## 🏗️ Architecture Overview

```
┌─────────────────┐    ┌─────────────────┐    ┌─────────────────┐
│   React Frontend│    │ ASP.NET Core    │    │ Aurora V2       │
│   (Vite + TS)   │◄──►│ Lambda Functions│◄──►│ (PostgreSQL)    │
└─────────────────┘    └─────────────────┘    └─────────────────┘
         │                       │                       │
         │                       │                       │
         ▼                       ▼                       ▼
┌─────────────────┐    ┌─────────────────┐    ┌─────────────────┐
│   Okta Auth     │    │   AWS Lambda    │    │   CloudWatch    │
│   (OAuth 2.0)   │    │   (Serverless)  │    │   (Monitoring)  │
└─────────────────┘    └─────────────────┘    └─────────────────┘
```

## 🚀 Quick Start

### Prerequisites
- Node.js 18+ 
- .NET 8 SDK
- AWS CLI configured
- Terraform CLI (v1.5+)
- Terraform Cloud account
- Okta developer account

### Development Setup
```bash
# Clone the repository
git clone <repository-url>
cd invest-sales

# Install frontend dependencies
cd frontend
npm install

# Start development server
npm run dev

# In another terminal, start backend
cd ../backend
dotnet run

# Initialize Terraform (first time only)
cd ../infrastructure
terraform init
terraform login
```

### Environment Configuration
```bash
# Frontend (.env)
VITE_API_BASE_URL=http://localhost:5000
VITE_OKTA_DOMAIN=your-okta-domain
VITE_OKTA_CLIENT_ID=your-client-id

# Backend (appsettings.Development.json)
{
  "ConnectionStrings": {
    "DefaultConnection": "your-aurora-connection-string"
  },
  "Okta": {
    "Domain": "your-okta-domain",
    "Audience": "your-api-audience"
  }
}
```

## 📚 Documentation Structure

### Core Documentation
- **[Product Requirements Document](prd_investment_tool.md)** - Complete functional and technical requirements
- **[Database Design](database_design.md)** - Schema design and data architecture
- **[UI Development Plan](ui_development_plan.md)** - Frontend architecture and component design
- **[UI Implementation Plan](ui_implementation_plan.md)** - Detailed implementation specifications

### Additional Documentation
- **[API Documentation](api_documentation.md)** - Complete API specifications
- **[Security Guide](security_guide.md)** - Security implementation and compliance
- **[Deployment Guide](deployment_guide.md)** - Production deployment procedures
- **[Testing Strategy](testing_strategy.md)** - Comprehensive testing approach

## 🎯 Key Features

### For Sales Representatives
- **Product Catalog**: Browse and filter investment products
- **Real-time Comparisons**: Side-by-side product comparison
- **Investment Calculator**: Growth projections for $10,000 investments
- **Export Capabilities**: Generate reports for customer presentations

### For Administrators
- **Product Management**: CRUD operations for investment products
- **Rate Management**: Update rates and availability periods
- **User Management**: Role-based access control
- **Audit Trail**: Comprehensive activity logging

### For System Administrators
- **System Monitoring**: Performance and health monitoring
- **Security Management**: User access and authentication
- **Compliance Reporting**: Regulatory compliance tools
- **Backup & Recovery**: Data protection and disaster recovery

## 🔐 Security & Compliance

- **Authentication**: Okta OAuth 2.0 with JWT tokens
- **Authorization**: Role-based access control (RBAC)
- **Data Protection**: AES-256 encryption at rest, TLS 1.3 in transit
- **Compliance**: SOC 2 Type II ready, 7-year audit trail retention
- **Privacy**: GDPR and CCPA compliant

## 📊 Performance Targets

- **Response Time**: < 2 seconds for product queries
- **Concurrent Users**: Support 100+ concurrent users
- **Availability**: 99.9% uptime during business hours
- **Data Refresh**: Real-time rate updates within 5 minutes

## 🛠️ Technology Stack

### Frontend
- **React 18.2+** with TypeScript
- **Vite 5.x** for build tooling
- **Tailwind CSS 3.x** for styling
- **React Query/TanStack Query** for data fetching
- **React Router 6.x** for routing

### Backend
- **ASP.NET Core 8** Lambda functions
- **Entity Framework Core** for data access
- **JWT authentication** with Okta integration
- **AWS Lambda** for serverless execution

### Infrastructure
- **Amazon Aurora V2** (PostgreSQL-compatible)
- **AWS Lambda** for serverless compute
- **Amazon CloudWatch** for monitoring
- **AWS IAM** for security and access control
- **Terraform Cloud** for Infrastructure as Code (IaC)

## 📈 Success Metrics

### Business Metrics
- User adoption rate
- Product comparison frequency
- Sales conversion tracking
- User satisfaction scores

### Technical Metrics
- System uptime and performance
- API response times
- Error rates
- Security incident frequency

## 🚦 Development Phases

### Phase 1: Foundation (Weeks 1-4)
- [x] Infrastructure setup
- [x] Authentication integration
- [ ] Basic product display

### Phase 2: Core Features (Weeks 5-8)
- [ ] Product comparison functionality
- [ ] Investment calculator
- [ ] Administrative interface

### Phase 3: Advanced Features (Weeks 9-12)
- [ ] Audit trail implementation
- [ ] Advanced filtering and search
- [ ] Reporting capabilities

### Phase 4: Testing and Deployment (Weeks 13-16)
- [ ] Security testing
- [ ] Performance testing
- [ ] User acceptance testing
- [ ] Production deployment

## 🤝 Contributing

### Development Workflow
1. Create feature branch from `main`
2. Implement changes with tests
3. Run linting and type checking
4. Submit pull request with description
5. Code review and approval
6. Merge to main branch

### Code Standards
- TypeScript strict mode enabled
- ESLint and Prettier configured
- Unit test coverage > 80%
- Accessibility compliance (WCAG 2.1 AA)

## 📞 Support

### Getting Help
- **Documentation**: Check the docs directory
- **Issues**: Create GitHub issues for bugs
- **Questions**: Use GitHub Discussions
- **Security**: Report security issues privately

### Contact
- **Project Lead**: [Contact Information]
- **Technical Lead**: [Contact Information]
- **Product Owner**: [Contact Information]

## 📄 License

[License Information]

---

**Last Updated**: [Date]
**Version**: 1.0.0
**Status**: In Development 