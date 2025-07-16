# Investment Product Sales Tool - Product Requirements Document

## Executive Summary

The Investment Product Sales Tool is a browser-based sales application designed to help bankers present investment products (Annuities, CDs, and other investment products) to customers. The tool provides real-time product comparisons, rate calculations, and growth projections to facilitate informed decision-making during customer interactions.

## Product Overview

### Purpose
Enable bankers to efficiently present and compare investment products with customers, showing projected returns and growth scenarios for standardized investment amounts.

### Target Users
- **Primary Users**: Bank sales representatives and financial advisors
- **Secondary Users**: Administrative staff for product management
- **Tertiary Users**: System administrators for user management

### Key Value Propositions
- Streamlined product comparison workflow
- Real-time rate calculations and projections
- Centralized product management
- Comprehensive audit trail for compliance
- Secure, role-based access control

## Functional Requirements

### Core Features

#### 1. Product Display and Comparison
- **Product Catalog View**: Display all available investment products categorized by type (Annuities, CDs, Other)
- **Product Filtering**: Filter by product type, rate ranges, terms, and availability dates
- **Real-time Rates**: Show current base rates, bonus rates, and any promotional rates
- **Investment Calculator**: Calculate growth projections for $10,000 investment across different time periods
- **Comparative Analysis**: Side-by-side comparison of selected products with key metrics
- **Return of Premium**: Display return of premium options where applicable

#### 2. Product Information Management
- **CUSIP-based Storage**: All products identified and stored by CUSIP values
- **Product Details**: Comprehensive product information including terms, conditions, and rate structures
- **Time-based Validity**: Products valid only during specified date ranges
- **Rate History**: Track historical rate changes for trending analysis

#### 3. Administrative Interface
- **Product CRUD Operations**: Create, Read, Update, Delete investment products
- **Product Staging**: Schedule products to become available during specific periods
- **Rate Management**: Update base rates, bonus rates, and promotional rates
- **Availability Management**: Set start and end dates for product availability
- **Bulk Operations**: Import/export product data and rates

#### 4. Security and Compliance
- **Authentication**: Okta OAuth 2.0 JWT token-based authentication
- **Authorization**: Role-based access control using Okta claims groups
- **Audit Trail**: Comprehensive logging of all user actions and system changes
- **Data Protection**: Encryption at rest and in transit
- **Session Management**: Secure session handling with appropriate timeouts

### User Roles and Permissions

#### Sales Representative
- View available products and rates
- Access investment calculator
- Generate product comparisons
- View own activity history

#### Sales Manager
- All Sales Representative permissions
- Access to team performance metrics
- View team activity reports

#### Product Administrator
- All Sales permissions
- CRUD operations on products
- Manage product availability periods
- Update rates and terms
- Access to full audit trail

#### System Administrator
- All permissions
- User management
- System configuration
- Security settings management

## Technical Requirements

### Architecture
- **Frontend**: React with Vite build tool and Tailwind CSS
- **Backend**: ASP.NET Core 8 Lambda functions
- **Database**: Aurora V2 (PostgreSQL-compatible)
- **Authentication**: Okta OAuth 2.0
- **Cloud Platform**: AWS

### Performance Requirements
- **Response Time**: < 2 seconds for product queries
- **Concurrent Users**: Support 100+ concurrent users
- **Availability**: 99.9% uptime during business hours
- **Data Refresh**: Real-time rate updates within 5 minutes

### Security Requirements
- **Authentication**: Multi-factor authentication via Okta
- **Authorization**: Role-based access control
- **Data Encryption**: AES-256 encryption at rest, TLS 1.3 in transit
- **Audit Logging**: All user actions logged with timestamps and user identification
- **Compliance**: SOC 2 Type II compliance ready

## User Experience Requirements

### Sales Representative Workflow
1. Login via Okta SSO
2. View dashboard with available products
3. Filter/search products based on customer needs
4. Select products for comparison
5. Input investment amount (default $10,000)
6. Review growth projections and terms
7. Generate comparison report for customer

### Administrator Workflow
1. Login via Okta SSO
2. Access admin dashboard
3. Manage product catalog
4. Set availability periods
5. Update rates and terms
6. Review audit trail
7. Generate compliance reports

## Data Requirements

### Product Data Structure
- CUSIP identifier
- Product name and description
- Product type (Annuity, CD, Other)
- Base rate and bonus rate structures
- Terms and conditions
- Availability period (start/end dates)
- Return of premium options
- Minimum/maximum investment amounts

### User Data
- User identification (from Okta)
- Role assignments
- Activity history
- Preferences and settings

### Audit Data
- Action timestamps
- User identification
- Action type and details
- Data changes (before/after)
- System events

## Integration Requirements

### External Systems
- **Okta**: Authentication and authorization
- **Rate Feed Systems**: Real-time rate updates
- **Compliance Systems**: Audit trail integration
- **Reporting Systems**: Data export capabilities

### API Requirements
- RESTful API design
- JWT token validation
- Rate limiting and throttling
- Error handling and logging

## Compliance and Regulatory Requirements

### Financial Services Compliance
- Audit trail retention for 7 years
- User action logging for regulatory review
- Data privacy compliance (GDPR, CCPA)
- Financial product disclosure requirements

### Security Compliance
- Regular security assessments
- Penetration testing
- Vulnerability management
- Incident response procedures

## Success Metrics

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

## Timeline and Milestones

### Phase 1: Foundation (Weeks 1-4)
- Infrastructure setup
- Authentication integration
- Basic product display

### Phase 2: Core Features (Weeks 5-8)
- Product comparison functionality
- Investment calculator
- Administrative interface

### Phase 3: Advanced Features (Weeks 9-12)
- Audit trail implementation
- Advanced filtering and search
- Reporting capabilities

### Phase 4: Testing and Deployment (Weeks 13-16)
- Security testing
- Performance testing
- User acceptance testing
- Production deployment

## Risk Assessment

### Technical Risks
- Integration complexity with Okta
- Performance with large product catalogs
- Real-time rate update reliability

### Business Risks
- User adoption challenges
- Compliance requirement changes
- Market rate volatility impact

### Mitigation Strategies
- Comprehensive testing protocols
- Phased rollout approach
- Regular stakeholder communication
- Continuous monitoring and optimization

## Appendices

### Glossary
- **CUSIP**: Committee on Uniform Securities Identification Procedures identifier
- **JWT**: JSON Web Token
- **RBAC**: Role-Based Access Control
- **Aurora V2**: Amazon Aurora PostgreSQL-compatible database

### References
- Okta Developer Documentation
- AWS Lambda Best Practices
- Financial Services Security Guidelines
- React Development Standards