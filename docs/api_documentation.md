# Investment Product Sales Tool - API Documentation

## Overview

This document provides comprehensive API documentation for the Investment Product Sales Tool. The API follows RESTful principles and uses JWT tokens for authentication via Okta integration.

## Base URL

- **Development**: `http://localhost:5000/api`
- **Staging**: `https://api-staging.investment-tool.com/api`
- **Production**: `https://api.investment-tool.com/api`

## Authentication

### Okta OAuth 2.0 Flow

The API uses Okta OAuth 2.0 with JWT tokens for authentication.

#### 1. Authorization Code Flow

```http
GET /oauth2/v1/authorize?
  client_id={client_id}&
  response_type=code&
  scope=openid profile email&
  redirect_uri={redirect_uri}&
  state={state}
```

#### 2. Token Exchange

```http
POST /oauth2/v1/token
Content-Type: application/x-www-form-urlencoded

grant_type=authorization_code&
code={authorization_code}&
redirect_uri={redirect_uri}&
client_id={client_id}&
client_secret={client_secret}
```

#### 3. Using JWT Tokens

Include the JWT token in the Authorization header:

```http
Authorization: Bearer {jwt_token}
```

### Token Validation

The API validates JWT tokens using Okta's public keys. Tokens must:
- Be signed by Okta
- Not be expired
- Contain required claims (sub, aud, iss)
- Have valid user permissions

## Error Handling

### Standard Error Response Format

```json
{
  "error": {
    "code": "VALIDATION_ERROR",
    "message": "Invalid input parameters",
    "details": [
      {
        "field": "cusip",
        "message": "CUSIP must be exactly 9 characters"
      }
    ],
    "timestamp": "2024-01-15T10:30:00Z",
    "requestId": "req-12345678-1234-1234-1234-123456789012"
  }
}
```

### HTTP Status Codes

| Code | Description |
|------|-------------|
| 200 | Success |
| 201 | Created |
| 400 | Bad Request |
| 401 | Unauthorized |
| 403 | Forbidden |
| 404 | Not Found |
| 422 | Validation Error |
| 429 | Rate Limited |
| 500 | Internal Server Error |

## Rate Limiting

- **Standard Users**: 100 requests per minute
- **Administrators**: 500 requests per minute
- **System Administrators**: 1000 requests per minute

Rate limit headers:
```http
X-RateLimit-Limit: 100
X-RateLimit-Remaining: 95
X-RateLimit-Reset: 1642248600
```

## API Endpoints

### Authentication

#### GET /auth/me
Get current user information.

**Headers:**
```http
Authorization: Bearer {jwt_token}
```

**Response:**
```json
{
  "user": {
    "id": "550e8400-e29b-41d4-a716-446655440000",
    "email": "user@example.com",
    "firstName": "John",
    "lastName": "Doe",
    "roles": ["SalesRepresentative"],
    "lastLogin": "2024-01-15T10:30:00Z"
  }
}
```

### Products

#### GET /products
Get list of available products with filtering.

**Query Parameters:**
- `type` (string, optional): Filter by product type (Annuity, CD, Other)
- `minRate` (number, optional): Minimum base rate
- `maxRate` (number, optional): Maximum base rate
- `minInvestment` (number, optional): Minimum investment amount
- `active` (boolean, optional): Filter by active status
- `page` (number, optional): Page number (default: 1)
- `limit` (number, optional): Items per page (default: 20, max: 100)

**Headers:**
```http
Authorization: Bearer {jwt_token}
```

**Response:**
```json
{
  "products": [
    {
      "id": "550e8400-e29b-41d4-a716-446655440000",
      "cusip": "123456789",
      "name": "Premium Annuity Plus",
      "type": "Annuity",
      "description": "High-yield annuity with bonus rates",
      "baseRate": 4.5,
      "bonusRate": 0.5,
      "minimumInvestment": 10000,
      "maximumInvestment": 1000000,
      "returnOfPremium": true,
      "returnOfPremiumPercentage": 100,
      "terms": "5-year term with annual compounding",
      "availability": {
        "startDate": "2024-01-01",
        "endDate": "2024-12-31"
      },
      "isActive": true,
      "createdAt": "2024-01-01T00:00:00Z",
      "updatedAt": "2024-01-15T10:30:00Z"
    }
  ],
  "pagination": {
    "page": 1,
    "limit": 20,
    "total": 150,
    "totalPages": 8
  }
}
```

#### GET /products/{id}
Get detailed information about a specific product.

**Headers:**
```http
Authorization: Bearer {jwt_token}
```

**Response:**
```json
{
  "product": {
    "id": "550e8400-e29b-41d4-a716-446655440000",
    "cusip": "123456789",
    "name": "Premium Annuity Plus",
    "type": "Annuity",
    "description": "High-yield annuity with bonus rates",
    "baseRate": 4.5,
    "bonusRate": 0.5,
    "minimumInvestment": 10000,
    "maximumInvestment": 1000000,
    "returnOfPremium": true,
    "returnOfPremiumPercentage": 100,
    "terms": "5-year term with annual compounding",
    "availability": {
      "startDate": "2024-01-01",
      "endDate": "2024-12-31"
    },
    "rateHistory": [
      {
        "effectiveDate": "2024-01-01",
        "baseRate": 4.5,
        "bonusRate": 0.5,
        "createdBy": "admin@example.com"
      }
    ],
    "isActive": true,
    "createdAt": "2024-01-01T00:00:00Z",
    "updatedAt": "2024-01-15T10:30:00Z"
  }
}
```

#### POST /products
Create a new product (Admin only).

**Headers:**
```http
Authorization: Bearer {jwt_token}
Content-Type: application/json
```

**Request Body:**
```json
{
  "cusip": "123456789",
  "name": "Premium Annuity Plus",
  "type": "Annuity",
  "description": "High-yield annuity with bonus rates",
  "baseRate": 4.5,
  "bonusRate": 0.5,
  "minimumInvestment": 10000,
  "maximumInvestment": 1000000,
  "returnOfPremium": true,
  "returnOfPremiumPercentage": 100,
  "terms": "5-year term with annual compounding",
  "availability": {
    "startDate": "2024-01-01",
    "endDate": "2024-12-31"
  }
}
```

**Response:**
```json
{
  "product": {
    "id": "550e8400-e29b-41d4-a716-446655440000",
    "cusip": "123456789",
    "name": "Premium Annuity Plus",
    "type": "Annuity",
    "description": "High-yield annuity with bonus rates",
    "baseRate": 4.5,
    "bonusRate": 0.5,
    "minimumInvestment": 10000,
    "maximumInvestment": 1000000,
    "returnOfPremium": true,
    "returnOfPremiumPercentage": 100,
    "terms": "5-year term with annual compounding",
    "availability": {
      "startDate": "2024-01-01",
      "endDate": "2024-12-31"
    },
    "isActive": true,
    "createdAt": "2024-01-01T00:00:00Z",
    "updatedAt": "2024-01-01T00:00:00Z"
  }
}
```

#### PUT /products/{id}
Update an existing product (Admin only).

**Headers:**
```http
Authorization: Bearer {jwt_token}
Content-Type: application/json
```

**Request Body:**
```json
{
  "name": "Premium Annuity Plus Updated",
  "baseRate": 4.75,
  "bonusRate": 0.75,
  "availability": {
    "startDate": "2024-01-01",
    "endDate": "2024-12-31"
  }
}
```

#### DELETE /products/{id}
Deactivate a product (Admin only).

**Headers:**
```http
Authorization: Bearer {jwt_token}
```

**Response:**
```json
{
  "message": "Product deactivated successfully"
}
```

### Product Comparisons

#### POST /comparisons
Create a product comparison.

**Headers:**
```http
Authorization: Bearer {jwt_token}
Content-Type: application/json
```

**Request Body:**
```json
{
  "productIds": [
    "550e8400-e29b-41d4-a716-446655440000",
    "550e8400-e29b-41d4-a716-446655440001"
  ],
  "investmentAmount": 10000,
  "timeHorizon": 5
}
```

**Response:**
```json
{
  "comparison": {
    "id": "550e8400-e29b-41d4-a716-446655440000",
    "products": [
      {
        "id": "550e8400-e29b-41d4-a716-446655440000",
        "cusip": "123456789",
        "name": "Premium Annuity Plus",
        "type": "Annuity",
        "baseRate": 4.5,
        "bonusRate": 0.5,
        "projections": {
          "year1": 10450,
          "year2": 10920,
          "year3": 11411,
          "year4": 11924,
          "year5": 12460
        }
      }
    ],
    "investmentAmount": 10000,
    "timeHorizon": 5,
    "createdAt": "2024-01-15T10:30:00Z"
  }
}
```

#### GET /comparisons/{id}
Get a specific comparison.

**Headers:**
```http
Authorization: Bearer {jwt_token}
```

#### GET /comparisons
Get user's comparison history.

**Query Parameters:**
- `page` (number, optional): Page number
- `limit` (number, optional): Items per page

### Investment Calculator

#### POST /calculator/projections
Calculate growth projections for an investment.

**Headers:**
```http
Authorization: Bearer {jwt_token}
Content-Type: application/json
```

**Request Body:**
```json
{
  "productId": "550e8400-e29b-41d4-a716-446655440000",
  "investmentAmount": 10000,
  "timeHorizon": 5,
  "compoundingFrequency": "annual"
}
```

**Response:**
```json
{
  "projections": {
    "initialInvestment": 10000,
    "finalValue": 12460,
    "totalReturn": 2460,
    "annualizedReturn": 4.5,
    "yearlyBreakdown": [
      {
        "year": 1,
        "value": 10450,
        "interest": 450
      },
      {
        "year": 2,
        "value": 10920,
        "interest": 470
      }
    ]
  }
}
```

### User Management

#### GET /users
Get list of users (Admin only).

**Headers:**
```http
Authorization: Bearer {jwt_token}
```

**Query Parameters:**
- `role` (string, optional): Filter by role
- `active` (boolean, optional): Filter by active status
- `page` (number, optional): Page number
- `limit` (number, optional): Items per page

#### GET /users/{id}
Get user details (Admin only).

#### PUT /users/{id}/roles
Update user roles (Admin only).

**Headers:**
```http
Authorization: Bearer {jwt_token}
Content-Type: application/json
```

**Request Body:**
```json
{
  "roles": ["SalesRepresentative", "SalesManager"]
}
```

### Audit Trail

#### GET /audit
Get audit log entries (Admin only).

**Headers:**
```http
Authorization: Bearer {jwt_token}
```

**Query Parameters:**
- `userId` (string, optional): Filter by user ID
- `actionType` (string, optional): Filter by action type
- `tableName` (string, optional): Filter by table name
- `startDate` (string, optional): Filter by start date
- `endDate` (string, optional): Filter by end date
- `page` (number, optional): Page number
- `limit` (number, optional): Items per page

**Response:**
```json
{
  "auditEntries": [
    {
      "id": "550e8400-e29b-41d4-a716-446655440000",
      "userId": "550e8400-e29b-41d4-a716-446655440000",
      "userEmail": "user@example.com",
      "actionType": "UPDATE_PRODUCT",
      "tableName": "products",
      "recordId": "550e8400-e29b-41d4-a716-446655440000",
      "oldValues": {
        "baseRate": 4.5
      },
      "newValues": {
        "baseRate": 4.75
      },
      "ipAddress": "192.168.1.1",
      "userAgent": "Mozilla/5.0...",
      "timestamp": "2024-01-15T10:30:00Z"
    }
  ],
  "pagination": {
    "page": 1,
    "limit": 20,
    "total": 150,
    "totalPages": 8
  }
}
```

### System Health

#### GET /health
Get system health status.

**Response:**
```json
{
  "status": "healthy",
  "timestamp": "2024-01-15T10:30:00Z",
  "services": {
    "database": "healthy",
    "authentication": "healthy",
    "rateService": "healthy"
  },
  "version": "1.0.0"
}
```

## Data Models

### Product Model
```typescript
interface Product {
  id: string;
  cusip: string;
  name: string;
  type: 'Annuity' | 'CD' | 'Other';
  description: string;
  baseRate: number;
  bonusRate: number;
  minimumInvestment: number;
  maximumInvestment?: number;
  returnOfPremium: boolean;
  returnOfPremiumPercentage?: number;
  terms: string;
  availability: {
    startDate: string;
    endDate: string;
  };
  isActive: boolean;
  createdAt: string;
  updatedAt: string;
}
```

### User Model
```typescript
interface User {
  id: string;
  email: string;
  firstName: string;
  lastName: string;
  roles: string[];
  isActive: boolean;
  lastLogin?: string;
  createdAt: string;
  updatedAt: string;
}
```

### Comparison Model
```typescript
interface Comparison {
  id: string;
  productIds: string[];
  investmentAmount: number;
  timeHorizon: number;
  products: ProductWithProjections[];
  createdAt: string;
}
```

## SDK Examples

### JavaScript/TypeScript

```typescript
import { InvestmentToolAPI } from '@investment-tool/sdk';

const api = new InvestmentToolAPI({
  baseURL: 'https://api.investment-tool.com/api',
  token: 'your-jwt-token'
});

// Get products
const products = await api.products.list({
  type: 'Annuity',
  minRate: 4.0,
  maxRate: 5.0
});

// Create comparison
const comparison = await api.comparisons.create({
  productIds: ['product-1', 'product-2'],
  investmentAmount: 10000,
  timeHorizon: 5
});

// Calculate projections
const projections = await api.calculator.projections({
  productId: 'product-1',
  investmentAmount: 10000,
  timeHorizon: 5
});
```

### cURL Examples

```bash
# Get products
curl -H "Authorization: Bearer YOUR_TOKEN" \
  "https://api.investment-tool.com/api/products?type=Annuity&minRate=4.0"

# Create comparison
curl -X POST \
  -H "Authorization: Bearer YOUR_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"productIds":["id1","id2"],"investmentAmount":10000,"timeHorizon":5}' \
  "https://api.investment-tool.com/api/comparisons"

# Calculate projections
curl -X POST \
  -H "Authorization: Bearer YOUR_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"productId":"id1","investmentAmount":10000,"timeHorizon":5}' \
  "https://api.investment-tool.com/api/calculator/projections"
```

## Webhook Integration

### Webhook Events

The API supports webhooks for real-time notifications:

- `product.created` - New product created
- `product.updated` - Product updated
- `product.deactivated` - Product deactivated
- `rate.updated` - Product rates updated
- `comparison.created` - New comparison created

### Webhook Configuration

```json
{
  "url": "https://your-app.com/webhooks",
  "events": ["product.created", "rate.updated"],
  "secret": "your-webhook-secret"
}
```

### Webhook Payload Example

```json
{
  "event": "product.updated",
  "timestamp": "2024-01-15T10:30:00Z",
  "data": {
    "productId": "550e8400-e29b-41d4-a716-446655440000",
    "cusip": "123456789",
    "changes": {
      "baseRate": {
        "old": 4.5,
        "new": 4.75
      }
    }
  }
}
```

## Versioning

API versioning is handled through the URL path:

- Current version: `/api/v1/`
- Legacy version: `/api/v0/` (deprecated)

Version deprecation schedule:
- v0: Deprecated as of 2024-01-01, removal on 2024-07-01
- v1: Current stable version
- v2: Planned for Q2 2024

## Support

For API support:
- **Documentation**: This document
- **SDK**: Available on npm as `@investment-tool/sdk`
- **Postman Collection**: Available in `/docs/postman/`
- **Support**: api-support@investment-tool.com
- **Status Page**: https://status.investment-tool.com 