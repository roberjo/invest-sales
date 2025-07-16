# Investment Product Sales Tool - Testing Strategy

## Overview

This document outlines the comprehensive testing strategy for the Investment Product Sales Tool, covering unit, integration, end-to-end, performance, and security testing approaches.

## Testing Pyramid

```
        ┌─────────────────┐
        │   E2E Tests     │  ← Few, High Value
        │   (10%)         │
        └─────────────────┘
        ┌─────────────────┐
        │ Integration     │  ← Some, Medium Value
        │ Tests (20%)     │
        └─────────────────┘
        ┌─────────────────┐
        │   Unit Tests    │  ← Many, Fast, Reliable
        │   (70%)         │
        └─────────────────┘
```

## Unit Testing

### Backend Unit Tests (.NET)

#### Test Structure
```csharp
[TestFixture]
public class ProductServiceTests
{
    private Mock<IProductRepository> _mockRepository;
    private Mock<IAuditService> _mockAuditService;
    private ProductService _service;

    [SetUp]
    public void Setup()
    {
        _mockRepository = new Mock<IProductRepository>();
        _mockAuditService = new Mock<IAuditService>();
        _service = new ProductService(_mockRepository.Object, _mockAuditService.Object);
    }

    [Test]
    public async Task GetActiveProducts_ShouldReturnActiveProducts()
    {
        // Arrange
        var expectedProducts = new List<Product>
        {
            new Product { Id = Guid.NewGuid(), Name = "Test Product", IsActive = true }
        };
        _mockRepository.Setup(r => r.GetActiveProductsAsync()).ReturnsAsync(expectedProducts);

        // Act
        var result = await _service.GetActiveProductsAsync();

        // Assert
        Assert.That(result, Is.Not.Null);
        Assert.That(result.Count(), Is.EqualTo(1));
        Assert.That(result.First().Name, Is.EqualTo("Test Product"));
    }

    [Test]
    public async Task CreateProduct_WithValidData_ShouldCreateProduct()
    {
        // Arrange
        var request = new CreateProductRequest
        {
            Cusip = "123456789",
            Name = "Test Product",
            Type = "Annuity",
            BaseRate = 4.5m
        };
        var expectedProductId = Guid.NewGuid();
        _mockRepository.Setup(r => r.CreateProductAsync(It.IsAny<Product>()))
            .ReturnsAsync(expectedProductId);

        // Act
        var result = await _service.CreateProductAsync(request);

        // Assert
        Assert.That(result, Is.EqualTo(expectedProductId));
        _mockRepository.Verify(r => r.CreateProductAsync(It.IsAny<Product>()), Times.Once);
        _mockAuditService.Verify(a => a.LogActionAsync("CREATE_PRODUCT", It.IsAny<string>(), 
            It.IsAny<Guid?>(), null, It.IsAny<object>()), Times.Once);
    }

    [Test]
    public void CreateProduct_WithInvalidCusip_ShouldThrowValidationException()
    {
        // Arrange
        var request = new CreateProductRequest
        {
            Cusip = "123", // Invalid CUSIP
            Name = "Test Product",
            Type = "Annuity",
            BaseRate = 4.5m
        };

        // Act & Assert
        var exception = Assert.ThrowsAsync<ValidationException>(
            async () => await _service.CreateProductAsync(request));
        Assert.That(exception.Message, Does.Contain("CUSIP"));
    }
}
```

#### Test Categories
```csharp
[TestFixture]
public class ProductValidationTests
{
    [Test]
    [TestCase("123456789", true)]
    [TestCase("12345678", false)]
    [TestCase("1234567890", false)]
    [TestCase("12345678A", false)]
    public void ValidateCusip_ShouldValidateCorrectly(string cusip, bool expected)
    {
        var validator = new CreateProductRequestValidator();
        var request = new CreateProductRequest { Cusip = cusip };
        
        var result = validator.Validate(request);
        
        Assert.That(result.IsValid, Is.EqualTo(expected));
    }
}
```

### Frontend Unit Tests (React)

#### Component Testing
```typescript
import { render, screen, fireEvent } from '@testing-library/react';
import { ProductCard } from '../components/ProductCard';

describe('ProductCard', () => {
  const mockProduct = {
    id: '1',
    cusip: '123456789',
    name: 'Test Annuity',
    type: 'Annuity',
    baseRate: 4.5,
    bonusRate: 0.5,
    minimumInvestment: 10000,
    isActive: true
  };

  it('renders product information correctly', () => {
    render(<ProductCard product={mockProduct} />);
    
    expect(screen.getByText('Test Annuity')).toBeInTheDocument();
    expect(screen.getByText('4.5%')).toBeInTheDocument();
    expect(screen.getByText('$10,000')).toBeInTheDocument();
  });

  it('calls onSelect when select button is clicked', () => {
    const mockOnSelect = jest.fn();
    render(<ProductCard product={mockProduct} onSelect={mockOnSelect} />);
    
    fireEvent.click(screen.getByText('Select'));
    expect(mockOnSelect).toHaveBeenCalledWith(mockProduct);
  });

  it('displays inactive state correctly', () => {
    const inactiveProduct = { ...mockProduct, isActive: false };
    render(<ProductCard product={inactiveProduct} />);
    
    expect(screen.getByText('Inactive')).toBeInTheDocument();
    expect(screen.getByText('Select')).toBeDisabled();
  });
});
```

#### Hook Testing
```typescript
import { renderHook, act } from '@testing-library/react';
import { useProducts } from '../hooks/useProducts';

describe('useProducts', () => {
  it('fetches products successfully', async () => {
    const mockProducts = [
      { id: '1', name: 'Product 1' },
      { id: '2', name: 'Product 2' }
    ];
    
    global.fetch = jest.fn().mockResolvedValue({
      ok: true,
      json: async () => ({ products: mockProducts })
    });

    const { result } = renderHook(() => useProducts());

    await act(async () => {
      await result.current.refetch();
    });

    expect(result.current.data).toEqual(mockProducts);
    expect(result.current.isLoading).toBe(false);
    expect(result.current.error).toBeNull();
  });

  it('handles error state correctly', async () => {
    global.fetch = jest.fn().mockRejectedValue(new Error('Network error'));

    const { result } = renderHook(() => useProducts());

    await act(async () => {
      await result.current.refetch();
    });

    expect(result.current.error).toBeTruthy();
    expect(result.current.isLoading).toBe(false);
  });
});
```

## Integration Testing

### API Integration Tests

#### Test Setup
```csharp
[TestFixture]
public class ProductApiIntegrationTests
{
    private TestServer _server;
    private HttpClient _client;

    [SetUp]
    public void Setup()
    {
        var builder = new WebHostBuilder()
            .UseStartup<TestStartup>()
            .ConfigureAppConfiguration((context, config) =>
            {
                config.AddJsonFile("appsettings.Testing.json");
            });

        _server = new TestServer(builder);
        _client = _server.CreateClient();
    }

    [Test]
    public async Task GetProducts_ShouldReturnProducts()
    {
        // Arrange
        var token = await GetValidToken();

        // Act
        _client.DefaultRequestHeaders.Authorization = 
            new AuthenticationHeaderValue("Bearer", token);
        var response = await _client.GetAsync("/api/products");

        // Assert
        Assert.That(response.IsSuccessStatusCode, Is.True);
        var content = await response.Content.ReadAsStringAsync();
        var products = JsonSerializer.Deserialize<ProductsResponse>(content);
        Assert.That(products.Products, Is.Not.Empty);
    }

    [Test]
    public async Task CreateProduct_WithValidData_ShouldCreateProduct()
    {
        // Arrange
        var token = await GetValidToken();
        var request = new CreateProductRequest
        {
            Cusip = "123456789",
            Name = "Integration Test Product",
            Type = "Annuity",
            BaseRate = 4.5m
        };

        // Act
        _client.DefaultRequestHeaders.Authorization = 
            new AuthenticationHeaderValue("Bearer", token);
        var json = JsonSerializer.Serialize(request);
        var content = new StringContent(json, Encoding.UTF8, "application/json");
        var response = await _client.PostAsync("/api/products", content);

        // Assert
        Assert.That(response.IsSuccessStatusCode, Is.True);
        var responseContent = await response.Content.ReadAsStringAsync();
        var createdProduct = JsonSerializer.Deserialize<ProductResponse>(responseContent);
        Assert.That(createdProduct.Product.Cusip, Is.EqualTo("123456789"));
    }
}
```

### Database Integration Tests

#### Repository Tests
```csharp
[TestFixture]
public class ProductRepositoryTests
{
    private DbContext _context;
    private ProductRepository _repository;

    [SetUp]
    public void Setup()
    {
        var options = new DbContextOptionsBuilder<ApplicationDbContext>()
            .UseInMemoryDatabase(databaseName: Guid.NewGuid().ToString())
            .Options;
        _context = new ApplicationDbContext(options);
        _repository = new ProductRepository(_context);
    }

    [Test]
    public async Task GetActiveProducts_ShouldReturnOnlyActiveProducts()
    {
        // Arrange
        var activeProduct = new Product { Name = "Active", IsActive = true };
        var inactiveProduct = new Product { Name = "Inactive", IsActive = false };
        _context.Products.AddRange(activeProduct, inactiveProduct);
        await _context.SaveChangesAsync();

        // Act
        var result = await _repository.GetActiveProductsAsync();

        // Assert
        Assert.That(result.Count(), Is.EqualTo(1));
        Assert.That(result.First().Name, Is.EqualTo("Active"));
    }

    [Test]
    public async Task CreateProduct_ShouldCreateProductWithAudit()
    {
        // Arrange
        var product = new Product
        {
            Cusip = "123456789",
            Name = "Test Product",
            Type = "Annuity",
            BaseRate = 4.5m
        };

        // Act
        var result = await _repository.CreateProductAsync(product);

        // Assert
        Assert.That(result, Is.Not.EqualTo(Guid.Empty));
        var createdProduct = await _context.Products.FindAsync(result);
        Assert.That(createdProduct, Is.Not.Null);
        Assert.That(createdProduct.Cusip, Is.EqualTo("123456789"));
    }
}
```

## End-to-End Testing

### Playwright E2E Tests

#### Test Configuration
```typescript
// playwright.config.ts
import { PlaywrightTestConfig } from '@playwright/test';

const config: PlaywrightTestConfig = {
  testDir: './tests/e2e',
  timeout: 30000,
  expect: {
    timeout: 5000
  },
  use: {
    baseURL: process.env.BASE_URL || 'http://localhost:3000',
    trace: 'on-first-retry',
    screenshot: 'only-on-failure'
  },
  projects: [
    {
      name: 'chromium',
      use: { browserName: 'chromium' }
    },
    {
      name: 'firefox',
      use: { browserName: 'firefox' }
    },
    {
      name: 'webkit',
      use: { browserName: 'webkit' }
    }
  ]
};

export default config;
```

#### E2E Test Examples
```typescript
// tests/e2e/product-comparison.spec.ts
import { test, expect } from '@playwright/test';

test.describe('Product Comparison', () => {
  test.beforeEach(async ({ page }) => {
    await page.goto('/');
    await page.fill('[data-testid="email"]', 'test@example.com');
    await page.fill('[data-testid="password"]', 'password123');
    await page.click('[data-testid="login-button"]');
    await page.waitForURL('/dashboard');
  });

  test('should allow user to compare products', async ({ page }) => {
    // Navigate to product catalog
    await page.click('[data-testid="product-catalog"]');
    
    // Select first product
    await page.click('[data-testid="product-1-select"]');
    
    // Select second product
    await page.click('[data-testid="product-2-select"]');
    
    // Open comparison view
    await page.click('[data-testid="compare-button"]');
    
    // Verify comparison view
    await expect(page.locator('[data-testid="comparison-table"]')).toBeVisible();
    await expect(page.locator('[data-testid="growth-chart"]')).toBeVisible();
    
    // Verify products are displayed
    await expect(page.locator('text=Product 1')).toBeVisible();
    await expect(page.locator('text=Product 2')).toBeVisible();
  });

  test('should calculate growth projections', async ({ page }) => {
    // Navigate to comparison
    await page.click('[data-testid="product-catalog"]');
    await page.click('[data-testid="product-1-select"]');
    await page.click('[data-testid="compare-button"]');
    
    // Set investment amount
    await page.fill('[data-testid="investment-amount"]', '10000');
    await page.click('[data-testid="calculate-button"]');
    
    // Verify projections
    await expect(page.locator('[data-testid="projection-year-1"]')).toContainText('$10,450');
    await expect(page.locator('[data-testid="projection-year-5"]')).toContainText('$12,460');
  });

  test('should export comparison report', async ({ page }) => {
    // Setup comparison
    await page.click('[data-testid="product-catalog"]');
    await page.click('[data-testid="product-1-select"]');
    await page.click('[data-testid="compare-button"]');
    
    // Export report
    const downloadPromise = page.waitForEvent('download');
    await page.click('[data-testid="export-pdf"]');
    const download = await downloadPromise;
    
    // Verify download
    expect(download.suggestedFilename()).toContain('comparison-report');
  });
});
```

#### Admin E2E Tests
```typescript
// tests/e2e/admin-product-management.spec.ts
import { test, expect } from '@playwright/test';

test.describe('Admin Product Management', () => {
  test.beforeEach(async ({ page }) => {
    await page.goto('/');
    await page.fill('[data-testid="email"]', 'admin@example.com');
    await page.fill('[data-testid="password"]', 'admin123');
    await page.click('[data-testid="login-button"]');
    await page.waitForURL('/admin');
  });

  test('should create new product', async ({ page }) => {
    await page.click('[data-testid="add-product"]');
    
    // Fill product form
    await page.fill('[data-testid="cusip"]', '987654321');
    await page.fill('[data-testid="name"]', 'New Test Product');
    await page.selectOption('[data-testid="type"]', 'Annuity');
    await page.fill('[data-testid="base-rate"]', '4.75');
    await page.fill('[data-testid="minimum-investment"]', '5000');
    
    await page.click('[data-testid="save-product"]');
    
    // Verify product created
    await expect(page.locator('text=New Test Product')).toBeVisible();
    await expect(page.locator('text=4.75%')).toBeVisible();
  });

  test('should update product rates', async ({ page }) => {
    await page.click('[data-testid="edit-product-1"]');
    
    // Update rate
    await page.fill('[data-testid="base-rate"]', '5.0');
    await page.click('[data-testid="save-changes"]');
    
    // Verify rate updated
    await expect(page.locator('text=5.0%')).toBeVisible();
  });
});
```

## Performance Testing

### Load Testing with k6

#### API Load Tests
```javascript
// tests/performance/api-load-test.js
import http from 'k6/http';
import { check, sleep } from 'k6';
import { Rate } from 'k6/metrics';

const errorRate = new Rate('errors');

export const options = {
  stages: [
    { duration: '2m', target: 10 }, // Ramp up
    { duration: '5m', target: 10 }, // Stay at 10 users
    { duration: '2m', target: 50 }, // Ramp up to 50
    { duration: '5m', target: 50 }, // Stay at 50 users
    { duration: '2m', target: 0 },  // Ramp down
  ],
  thresholds: {
    http_req_duration: ['p(95)<2000'], // 95% of requests under 2s
    http_req_failed: ['rate<0.1'],     // Error rate under 10%
    errors: ['rate<0.1'],
  },
};

const BASE_URL = __ENV.BASE_URL || 'https://api.investment-tool.com';

export default function() {
  const token = getAuthToken();
  
  const params = {
    headers: {
      'Authorization': `Bearer ${token}`,
      'Content-Type': 'application/json',
    },
  };

  // Test product listing
  const productsResponse = http.get(`${BASE_URL}/api/products`, params);
  check(productsResponse, {
    'products endpoint status is 200': (r) => r.status === 200,
    'products response time < 1000ms': (r) => r.timings.duration < 1000,
  });

  // Test product comparison
  const comparisonPayload = JSON.stringify({
    productIds: ['product-1', 'product-2'],
    investmentAmount: 10000,
    timeHorizon: 5
  });
  
  const comparisonResponse = http.post(
    `${BASE_URL}/api/comparisons`, 
    comparisonPayload, 
    params
  );
  
  check(comparisonResponse, {
    'comparison endpoint status is 201': (r) => r.status === 201,
    'comparison response time < 2000ms': (r) => r.timings.duration < 2000,
  });

  sleep(1);
}

function getAuthToken() {
  // Implementation to get auth token
  return 'mock-token';
}
```

#### Database Performance Tests
```sql
-- tests/performance/database-performance.sql
-- Test product queries with different filters

-- Test 1: Basic product listing
EXPLAIN ANALYZE
SELECT * FROM products 
WHERE is_active = true 
ORDER BY created_at DESC 
LIMIT 20;

-- Test 2: Filtered product search
EXPLAIN ANALYZE
SELECT * FROM products 
WHERE is_active = true 
  AND product_type = 'Annuity'
  AND base_rate BETWEEN 4.0 AND 5.0
ORDER BY base_rate DESC;

-- Test 3: Complex comparison query
EXPLAIN ANALYZE
SELECT 
    p.product_id,
    p.cusip,
    p.product_name,
    p.base_rate,
    p.bonus_rate,
    pa.start_date,
    pa.end_date
FROM products p
JOIN product_availability pa ON p.product_id = pa.product_id
WHERE p.is_active = true 
  AND pa.is_active = true 
  AND CURRENT_DATE BETWEEN pa.start_date AND pa.end_date
  AND p.product_type = 'Annuity'
ORDER BY p.base_rate DESC;

-- Test 4: Audit log query performance
EXPLAIN ANALYZE
SELECT * FROM audit_log 
WHERE timestamp > CURRENT_DATE - INTERVAL '30 days'
  AND action_type = 'UPDATE_PRODUCT'
ORDER BY timestamp DESC;
```

## Security Testing

### OWASP ZAP Security Tests

#### ZAP Configuration
```yaml
# .zap/zap.conf
# ZAP Configuration for Security Testing
api:
  key: "your-zap-api-key"
  port: 8080

# Scan policies
scan:
  policy: "OWASP Top 10"
  maxDuration: 3600
  maxAlerts: 100

# Authentication
auth:
  method: "form"
  loginUrl: "https://investment-tool.com/login"
  loginData:
    username: "test@example.com"
    password: "testpassword"
  loggedInRegex: "dashboard"

# Target configuration
target:
  url: "https://investment-tool.com"
  scope:
    - "https://investment-tool.com/*"
    - "https://api.investment-tool.com/*"
```

#### Security Test Scripts
```javascript
// tests/security/security-scan.js
const ZapClient = require('zaproxy');

const zap = new ZapClient({
  apiKey: process.env.ZAP_API_KEY,
  proxy: 'http://localhost:8080'
});

async function runSecurityScan() {
  try {
    // Start ZAP
    await zap.core.newSession();
    
    // Spider the application
    const spiderResult = await zap.spider.scan({
      url: 'https://investment-tool.com',
      maxChildren: 10
    });
    
    // Wait for spider to complete
    while (true) {
      const progress = await zap.spider.status(spiderResult.scan);
      if (progress >= 100) break;
      await new Promise(resolve => setTimeout(resolve, 1000));
    }
    
    // Run active scan
    const activeScanResult = await zap.ascan.scan({
      url: 'https://investment-tool.com',
      recurse: true
    });
    
    // Wait for active scan to complete
    while (true) {
      const progress = await zap.ascan.status(activeScanResult.scan);
      if (progress >= 100) break;
      await new Promise(resolve => setTimeout(resolve, 1000));
    }
    
    // Get alerts
    const alerts = await zap.core.alerts();
    
    // Filter high and medium risk alerts
    const highRiskAlerts = alerts.filter(alert => 
      alert.risk === 'High' || alert.risk === 'Medium'
    );
    
    if (highRiskAlerts.length > 0) {
      console.error('Security vulnerabilities found:', highRiskAlerts);
      process.exit(1);
    }
    
    console.log('Security scan completed successfully');
  } catch (error) {
    console.error('Security scan failed:', error);
    process.exit(1);
  }
}

runSecurityScan();
```

### Penetration Testing

#### Manual Security Tests
```bash
#!/bin/bash
# tests/security/manual-security-tests.sh

echo "Running manual security tests..."

# Test 1: SQL Injection
echo "Testing SQL Injection..."
curl -X POST "https://api.investment-tool.com/api/products" \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"cusip": "123456789\"; DROP TABLE products; --"}' \
  -w "Status: %{http_code}\n"

# Test 2: XSS
echo "Testing XSS..."
curl -X POST "https://api.investment-tool.com/api/products" \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"name": "<script>alert(\"XSS\")</script>"}' \
  -w "Status: %{http_code}\n"

# Test 3: Authentication Bypass
echo "Testing Authentication Bypass..."
curl -X GET "https://api.investment-tool.com/api/products" \
  -w "Status: %{http_code}\n"

# Test 4: Authorization Test
echo "Testing Authorization..."
curl -X POST "https://api.investment-tool.com/api/users" \
  -H "Authorization: Bearer $USER_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"email": "test@example.com", "role": "SystemAdministrator"}' \
  -w "Status: %{http_code}\n"

echo "Manual security tests completed"
```

## Accessibility Testing

### Automated Accessibility Tests

#### Jest-Axe Integration
```typescript
// tests/accessibility/accessibility.test.tsx
import { render } from '@testing-library/react';
import { axe, toHaveNoViolations } from 'jest-axe';
import { ProductCatalog } from '../../components/ProductCatalog';

expect.extend(toHaveNoViolations);

describe('Accessibility', () => {
  it('should not have any accessibility violations', async () => {
    const { container } = render(<ProductCatalog />);
    const results = await axe(container);
    expect(results).toHaveNoViolations();
  });

  it('should have proper focus management', async () => {
    const { container } = render(<ProductCatalog />);
    const results = await axe(container, {
      rules: {
        'focus-order-semantics': { enabled: true },
        'focus-visible': { enabled: true }
      }
    });
    expect(results).toHaveNoViolations();
  });

  it('should have proper color contrast', async () => {
    const { container } = render(<ProductCatalog />);
    const results = await axe(container, {
      rules: {
        'color-contrast': { enabled: true }
      }
    });
    expect(results).toHaveNoViolations();
  });
});
```

#### Manual Accessibility Checklist
```markdown
# Accessibility Testing Checklist

## Navigation
- [ ] Skip links work correctly
- [ ] Focus indicators are visible
- [ ] Tab order is logical
- [ ] Keyboard navigation works

## Forms
- [ ] All inputs have labels
- [ ] Error messages are associated with fields
- [ ] Required fields are indicated
- [ ] Form validation is announced

## Content
- [ ] Images have alt text
- [ ] Color is not the only way to convey information
- [ ] Text has sufficient contrast
- [ ] Text can be resized up to 200%

## Interactive Elements
- [ ] Buttons have descriptive text
- [ ] Links have meaningful text
- [ ] Touch targets are at least 44px
- [ ] Disabled states are communicated
```

## Test Data Management

### Test Data Setup
```csharp
// TestData/TestDataSeeder.cs
public class TestDataSeeder
{
    private readonly ApplicationDbContext _context;

    public TestDataSeeder(ApplicationDbContext context)
    {
        _context = context;
    }

    public async Task SeedTestDataAsync()
    {
        // Seed products
        var products = new List<Product>
        {
            new Product
            {
                Cusip = "123456789",
                Name = "Premium Annuity Plus",
                Type = "Annuity",
                BaseRate = 4.5m,
                BonusRate = 0.5m,
                MinimumInvestment = 10000,
                IsActive = true
            },
            new Product
            {
                Cusip = "987654321",
                Name = "High-Yield CD",
                Type = "CD",
                BaseRate = 4.2m,
                BonusRate = 0m,
                MinimumInvestment = 5000,
                IsActive = true
            }
        };

        _context.Products.AddRange(products);

        // Seed users
        var users = new List<User>
        {
            new User
            {
                Email = "test@example.com",
                FirstName = "Test",
                LastName = "User",
                Roles = new List<string> { "SalesRepresentative" }
            },
            new User
            {
                Email = "admin@example.com",
                FirstName = "Admin",
                LastName = "User",
                Roles = new List<string> { "SystemAdministrator" }
            }
        };

        _context.Users.AddRange(users);

        await _context.SaveChangesAsync();
    }
}
```

## Test Reporting

### Test Results Dashboard
```typescript
// tests/reporting/test-reporter.ts
import { writeFileSync } from 'fs';
import { join } from 'path';

interface TestResult {
  name: string;
  status: 'passed' | 'failed' | 'skipped';
  duration: number;
  error?: string;
}

interface TestSuite {
  name: string;
  total: number;
  passed: number;
  failed: number;
  skipped: number;
  duration: number;
  results: TestResult[];
}

class TestReporter {
  private suites: TestSuite[] = [];

  addSuite(suite: TestSuite) {
    this.suites.push(suite);
  }

  generateReport() {
    const totalTests = this.suites.reduce((sum, suite) => sum + suite.total, 0);
    const totalPassed = this.suites.reduce((sum, suite) => sum + suite.passed, 0);
    const totalFailed = this.suites.reduce((sum, suite) => sum + suite.failed, 0);
    const totalSkipped = this.suites.reduce((sum, suite) => sum + suite.skipped, 0);

    const report = {
      summary: {
        total: totalTests,
        passed: totalPassed,
        failed: totalFailed,
        skipped: totalSkipped,
        coverage: (totalPassed / totalTests) * 100
      },
      suites: this.suites,
      timestamp: new Date().toISOString()
    };

    const reportPath = join(process.cwd(), 'test-results', 'report.json');
    writeFileSync(reportPath, JSON.stringify(report, null, 2));

    console.log(`Test Report Generated: ${reportPath}`);
    console.log(`Coverage: ${report.summary.coverage.toFixed(2)}%`);
  }
}

export default TestReporter;
```

## Continuous Testing

### GitHub Actions Test Workflow
```yaml
# .github/workflows/test.yml
name: Test Suite

on:
  push:
    branches: [ main, develop ]
  pull_request:
    branches: [ main ]

jobs:
  unit-tests:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      
      - name: Setup .NET
        uses: actions/setup-dotnet@v3
        with:
          dotnet-version: 8.0.x
      
      - name: Setup Node.js
        uses: actions/setup-node@v3
        with:
          node-version: '18'
          cache: 'npm'
      
      - name: Run backend unit tests
        run: |
          cd backend
          dotnet test --collect:"XPlat Code Coverage" --results-directory ./coverage
      
      - name: Run frontend unit tests
        run: |
          cd frontend
          npm test -- --coverage
      
      - name: Upload coverage reports
        uses: codecov/codecov-action@v3

  integration-tests:
    runs-on: ubuntu-latest
    needs: unit-tests
    steps:
      - uses: actions/checkout@v3
      
      - name: Setup test database
        run: |
          docker run --name test-db -e POSTGRES_DB=test_db -e POSTGRES_USER=test -e POSTGRES_PASSWORD=test -d postgres:14
          sleep 10
      
      - name: Run integration tests
        run: |
          cd backend
          dotnet test --filter Category=Integration

  e2e-tests:
    runs-on: ubuntu-latest
    needs: unit-tests
    steps:
      - uses: actions/checkout@v3
      
      - name: Setup Playwright
        run: |
          cd frontend
          npm install
          npx playwright install
      
      - name: Run E2E tests
        run: |
          cd frontend
          npm run test:e2e

  performance-tests:
    runs-on: ubuntu-latest
    needs: unit-tests
    steps:
      - uses: actions/checkout@v3
      
      - name: Setup k6
        run: |
          wget https://github.com/grafana/k6/releases/download/v0.45.0/k6-v0.45.0-linux-amd64.tar.gz
          tar xzf k6-v0.45.0-linux-amd64.tar.gz
          sudo cp k6-v0.45.0-linux-amd64/k6 /usr/local/bin
      
      - name: Run performance tests
        run: |
          k6 run tests/performance/api-load-test.js

  security-tests:
    runs-on: ubuntu-latest
    needs: unit-tests
    steps:
      - uses: actions/checkout@v3
      
      - name: Run OWASP ZAP scan
        uses: zaproxy/action-full-scan@v0.8.0
        with:
          target: 'https://staging.investment-tool.com'
          rules_file_name: '.zap/rules.tsv'
          cmd_options: '-a'
```

---

**Last Updated**: 2024-01-15
**Version**: 1.0.0
**Next Review**: 2024-04-15 