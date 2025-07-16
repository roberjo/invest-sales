# Investment Product Sales Tool - Data Modeling Patterns

## Overview

This document outlines the data modeling patterns, design principles, and best practices used in the Investment Product Sales Tool database design.

## Design Principles

### 1. Domain-Driven Design (DDD)

```
┌─────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────┐
│                                                                 Domain Model                                                           │
├─────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  ┌─────────────────┐    ┌─────────────────┐    ┌─────────────────┐        │
│  │   User Domain   │    │ Product Domain  │    │  Audit Domain   │        │
│  │                 │    │                 │    │                 │        │
│  │ • User          │    │ • Product       │    │ • AuditLog      │        │
│  │ • UserRole      │    │ • RateHistory   │    │ • UserSession   │        │
│  │ • UserSession   │    │ • Availability  │    │ • Comparison    │        │
│  │                 │    │                 │    │                 │        │
│  └─────────────────┘    └─────────────────┘    └─────────────────┘        │
│           │                       │                       │                │
│           │                       │                       │                │
│           ▼                       ▼                       ▼                │
│  ┌─────────────────────────────────────────────────────────────────────┐    │
│  │                    Aggregate Boundaries                            │    │
│  │                                                                     │    │
│  │  ┌─────────────────┐  ┌─────────────────┐  ┌─────────────────┐    │    │
│  │  │   User          │  │   Product       │  │   Audit         │    │    │
│  │  │   Aggregate     │  │   Aggregate     │  │   Aggregate     │    │    │
│  │  │                 │  │                 │  │                 │    │    │
│  │  │ • User (Root)   │  │ • Product       │  │ • AuditLog      │    │    │
│  │  │ • UserRole      │  │   (Root)        │  │   (Root)        │    │    │
│  │  │ • UserSession   │  │ • RateHistory   │  │ • UserSession   │    │    │
│  │  │                 │  │ • Availability  │  │ • Comparison    │    │    │
│  │  └─────────────────┘  └─────────────────┘  └─────────────────┘    │    │
│  └─────────────────────────────────────────────────────────────────────┘    │
└─────────────────────────────────────────────────────────────────────────────┘
```

### 2. Event Sourcing Pattern

```csharp
// Domain Events
public abstract class DomainEvent
{
    public Guid EventId { get; set; } = Guid.NewGuid();
    public DateTime Timestamp { get; set; } = DateTime.UtcNow;
    public string EventType { get; set; }
    public Guid AggregateId { get; set; }
    public long Version { get; set; }
}

public class ProductCreatedEvent : DomainEvent
{
    public string Cusip { get; set; }
    public string ProductName { get; set; }
    public string ProductType { get; set; }
    public decimal BaseRate { get; set; }
    public decimal MinimumInvestment { get; set; }
    public string CreatedBy { get; set; }
}

public class RateChangedEvent : DomainEvent
{
    public decimal OldBaseRate { get; set; }
    public decimal NewBaseRate { get; set; }
    public decimal OldBonusRate { get; set; }
    public decimal NewBonusRate { get; set; }
    public DateTime EffectiveDate { get; set; }
    public string ChangedBy { get; set; }
}

// Event Store
public class EventStore
{
    private readonly ApplicationDbContext _context;

    public async Task SaveEventsAsync(Guid aggregateId, IEnumerable<DomainEvent> events, long expectedVersion)
    {
        var eventList = events.ToList();
        var version = expectedVersion;

        foreach (var @event in eventList)
        {
            version++;
            @event.Version = version;

            var eventData = new EventData
            {
                EventId = @event.EventId,
                AggregateId = aggregateId,
                EventType = @event.GetType().Name,
                EventData = JsonSerializer.Serialize(@event),
                Version = version,
                Timestamp = @event.Timestamp
            };

            _context.EventStore.Add(eventData);
        }

        await _context.SaveChangesAsync();
    }

    public async Task<IEnumerable<DomainEvent>> GetEventsAsync(Guid aggregateId, long fromVersion = 0)
    {
        var events = await _context.EventStore
            .Where(e => e.AggregateId == aggregateId && e.Version > fromVersion)
            .OrderBy(e => e.Version)
            .ToListAsync();

        return events.Select(e => JsonSerializer.Deserialize<DomainEvent>(e.EventData));
    }
}
```

### 3. CQRS Pattern (Command Query Responsibility Segregation)

```csharp
// Commands
public abstract class Command
{
    public Guid CommandId { get; set; } = Guid.NewGuid();
    public DateTime Timestamp { get; set; } = DateTime.UtcNow;
    public string UserId { get; set; }
}

public class CreateProductCommand : Command
{
    public string Cusip { get; set; }
    public string ProductName { get; set; }
    public string ProductType { get; set; }
    public decimal BaseRate { get; set; }
    public decimal MinimumInvestment { get; set; }
    public string Description { get; set; }
}

public class UpdateProductRateCommand : Command
{
    public Guid ProductId { get; set; }
    public decimal NewBaseRate { get; set; }
    public decimal NewBonusRate { get; set; }
    public DateTime EffectiveDate { get; set; }
}

// Queries
public abstract class Query<TResult>
{
    public Guid QueryId { get; set; } = Guid.NewGuid();
    public DateTime Timestamp { get; set; } = DateTime.UtcNow;
}

public class GetProductsQuery : Query<List<ProductDto>>
{
    public string ProductType { get; set; }
    public bool IsActive { get; set; } = true;
    public decimal? MinRate { get; set; }
    public decimal? MaxRate { get; set; }
}

public class GetProductComparisonQuery : Query<ProductComparisonDto>
{
    public List<Guid> ProductIds { get; set; }
    public decimal InvestmentAmount { get; set; }
    public int Years { get; set; } = 5;
}

// Command Handlers
public class CreateProductCommandHandler : ICommandHandler<CreateProductCommand>
{
    private readonly ApplicationDbContext _context;
    private readonly IEventStore _eventStore;

    public async Task HandleAsync(CreateProductCommand command)
    {
        // Validate command
        await ValidateCommandAsync(command);

        // Create product
        var product = new Product
        {
            ProductId = Guid.NewGuid(),
            Cusip = command.Cusip,
            ProductName = command.ProductName,
            ProductType = command.ProductType,
            BaseRate = command.BaseRate,
            MinimumInvestment = command.MinimumInvestment,
            Description = command.Description,
            IsActive = true,
            CreatedAt = DateTime.UtcNow,
            CreatedBy = command.UserId
        };

        _context.Products.Add(product);

        // Create event
        var @event = new ProductCreatedEvent
        {
            AggregateId = product.ProductId,
            Cusip = product.Cusip,
            ProductName = product.ProductName,
            ProductType = product.ProductType,
            BaseRate = product.BaseRate,
            MinimumInvestment = product.MinimumInvestment,
            CreatedBy = command.UserId
        };

        await _eventStore.SaveEventsAsync(product.ProductId, new[] { @event }, 0);
        await _context.SaveChangesAsync();
    }
}

// Query Handlers
public class GetProductsQueryHandler : IQueryHandler<GetProductsQuery, List<ProductDto>>
{
    private readonly ApplicationDbContext _context;

    public async Task<List<ProductDto>> HandleAsync(GetProductsQuery query)
    {
        var productsQuery = _context.Products.AsQueryable();

        if (!string.IsNullOrEmpty(query.ProductType))
            productsQuery = productsQuery.Where(p => p.ProductType == query.ProductType);

        if (query.IsActive)
            productsQuery = productsQuery.Where(p => p.IsActive);

        if (query.MinRate.HasValue)
            productsQuery = productsQuery.Where(p => p.BaseRate >= query.MinRate.Value);

        if (query.MaxRate.HasValue)
            productsQuery = productsQuery.Where(p => p.BaseRate <= query.MaxRate.Value);

        var products = await productsQuery
            .Select(p => new ProductDto
            {
                ProductId = p.ProductId,
                Cusip = p.Cusip,
                ProductName = p.ProductName,
                ProductType = p.ProductType,
                BaseRate = p.BaseRate,
                BonusRate = p.BonusRate,
                MinimumInvestment = p.MinimumInvestment,
                Description = p.Description
            })
            .ToListAsync();

        return products;
    }
}
```

## Data Access Patterns

### 1. Repository Pattern

```csharp
// Repository Interface
public interface IRepository<TEntity, TKey> where TEntity : class
{
    Task<TEntity> GetByIdAsync(TKey id);
    Task<IEnumerable<TEntity>> GetAllAsync();
    Task<IEnumerable<TEntity>> FindAsync(Expression<Func<TEntity, bool>> predicate);
    Task<TEntity> AddAsync(TEntity entity);
    Task UpdateAsync(TEntity entity);
    Task DeleteAsync(TKey id);
    Task<bool> ExistsAsync(TKey id);
}

// Generic Repository Implementation
public class Repository<TEntity, TKey> : IRepository<TEntity, TKey> where TEntity : class
{
    private readonly ApplicationDbContext _context;
    private readonly DbSet<TEntity> _dbSet;

    public Repository(ApplicationDbContext context)
    {
        _context = context;
        _dbSet = context.Set<TEntity>();
    }

    public async Task<TEntity> GetByIdAsync(TKey id)
    {
        return await _dbSet.FindAsync(id);
    }

    public async Task<IEnumerable<TEntity>> GetAllAsync()
    {
        return await _dbSet.ToListAsync();
    }

    public async Task<IEnumerable<TEntity>> FindAsync(Expression<Func<TEntity, bool>> predicate)
    {
        return await _dbSet.Where(predicate).ToListAsync();
    }

    public async Task<TEntity> AddAsync(TEntity entity)
    {
        var result = await _dbSet.AddAsync(entity);
        await _context.SaveChangesAsync();
        return result.Entity;
    }

    public async Task UpdateAsync(TEntity entity)
    {
        _dbSet.Update(entity);
        await _context.SaveChangesAsync();
    }

    public async Task DeleteAsync(TKey id)
    {
        var entity = await GetByIdAsync(id);
        if (entity != null)
        {
            _dbSet.Remove(entity);
            await _context.SaveChangesAsync();
        }
    }

    public async Task<bool> ExistsAsync(TKey id)
    {
        return await _dbSet.FindAsync(id) != null;
    }
}

// Specific Repository
public interface IProductRepository : IRepository<Product, Guid>
{
    Task<IEnumerable<Product>> GetActiveProductsAsync();
    Task<IEnumerable<Product>> GetProductsByTypeAsync(string productType);
    Task<Product> GetByCusipAsync(string cusip);
    Task<IEnumerable<Product>> GetProductsWithRatesAsync();
}

public class ProductRepository : Repository<Product, Guid>, IProductRepository
{
    public ProductRepository(ApplicationDbContext context) : base(context)
    {
    }

    public async Task<IEnumerable<Product>> GetActiveProductsAsync()
    {
        return await _dbSet
            .Where(p => p.IsActive)
            .Include(p => p.RateHistory)
            .Include(p => p.Availability)
            .ToListAsync();
    }

    public async Task<IEnumerable<Product>> GetProductsByTypeAsync(string productType)
    {
        return await _dbSet
            .Where(p => p.ProductType == productType && p.IsActive)
            .ToListAsync();
    }

    public async Task<Product> GetByCusipAsync(string cusip)
    {
        return await _dbSet
            .FirstOrDefaultAsync(p => p.Cusip == cusip);
    }

    public async Task<IEnumerable<Product>> GetProductsWithRatesAsync()
    {
        return await _dbSet
            .Include(p => p.RateHistory.OrderByDescending(r => r.EffectiveDate))
            .Where(p => p.IsActive)
            .ToListAsync();
    }
}
```

### 2. Unit of Work Pattern

```csharp
public interface IUnitOfWork : IDisposable
{
    IProductRepository Products { get; }
    IUserRepository Users { get; }
    IAuditLogRepository AuditLogs { get; }
    Task<int> SaveChangesAsync();
    Task BeginTransactionAsync();
    Task CommitTransactionAsync();
    Task RollbackTransactionAsync();
}

public class UnitOfWork : IUnitOfWork
{
    private readonly ApplicationDbContext _context;
    private readonly IProductRepository _productRepository;
    private readonly IUserRepository _userRepository;
    private readonly IAuditLogRepository _auditLogRepository;
    private IDbContextTransaction _transaction;

    public UnitOfWork(ApplicationDbContext context)
    {
        _context = context;
        _productRepository = new ProductRepository(context);
        _userRepository = new UserRepository(context);
        _auditLogRepository = new AuditLogRepository(context);
    }

    public IProductRepository Products => _productRepository;
    public IUserRepository Users => _userRepository;
    public IAuditLogRepository AuditLogs => _auditLogRepository;

    public async Task<int> SaveChangesAsync()
    {
        return await _context.SaveChangesAsync();
    }

    public async Task BeginTransactionAsync()
    {
        _transaction = await _context.Database.BeginTransactionAsync();
    }

    public async Task CommitTransactionAsync()
    {
        await _transaction?.CommitAsync();
    }

    public async Task RollbackTransactionAsync()
    {
        await _transaction?.RollbackAsync();
    }

    public void Dispose()
    {
        _transaction?.Dispose();
        _context?.Dispose();
    }
}
```

### 3. Specification Pattern

```csharp
public abstract class Specification<T>
{
    public abstract Expression<Func<T, bool>> ToExpression();
    public bool IsSatisfiedBy(T entity)
    {
        return ToExpression().Compile()(entity);
    }

    public Specification<T> And(Specification<T> specification)
    {
        return new AndSpecification<T>(this, specification);
    }

    public Specification<T> Or(Specification<T> specification)
    {
        return new OrSpecification<T>(this, specification);
    }

    public Specification<T> Not()
    {
        return new NotSpecification<T>(this);
    }
}

public class ActiveProductsSpecification : Specification<Product>
{
    public override Expression<Func<Product, bool>> ToExpression()
    {
        return product => product.IsActive;
    }
}

public class ProductTypeSpecification : Specification<Product>
{
    private readonly string _productType;

    public ProductTypeSpecification(string productType)
    {
        _productType = productType;
    }

    public override Expression<Func<Product, bool>> ToExpression()
    {
        return product => product.ProductType == _productType;
    }
}

public class MinimumRateSpecification : Specification<Product>
{
    private readonly decimal _minimumRate;

    public MinimumRateSpecification(decimal minimumRate)
    {
        _minimumRate = minimumRate;
    }

    public override Expression<Func<Product, bool>> ToExpression()
    {
        return product => product.BaseRate >= _minimumRate;
    }
}

// Composite Specifications
public class AndSpecification<T> : Specification<T>
{
    private readonly Specification<T> _left;
    private readonly Specification<T> _right;

    public AndSpecification(Specification<T> left, Specification<T> right)
    {
        _left = left;
        _right = right;
    }

    public override Expression<Func<T, bool>> ToExpression()
    {
        var leftExpression = _left.ToExpression();
        var rightExpression = _right.ToExpression();
        var invokedExpression = Expression.Invoke(rightExpression, leftExpression.Parameters);

        return Expression.Lambda<Func<T, bool>>(
            Expression.AndAlso(leftExpression.Body, invokedExpression),
            leftExpression.Parameters);
    }
}

// Usage in Repository
public async Task<IEnumerable<Product>> GetProductsAsync(Specification<Product> specification)
{
    return await _dbSet
        .Where(specification.ToExpression())
        .ToListAsync();
}

// Example usage
var activeAnnuities = new ActiveProductsSpecification()
    .And(new ProductTypeSpecification("Annuity"))
    .And(new MinimumRateSpecification(4.0m));

var products = await productRepository.GetProductsAsync(activeAnnuities);
```

## Data Validation Patterns

### 1. Fluent Validation

```csharp
public class ProductValidator : AbstractValidator<Product>
{
    public ProductValidator()
    {
        RuleFor(x => x.Cusip)
            .NotEmpty()
            .Length(9)
            .Matches(@"^[A-Z0-9]{9}$")
            .WithMessage("CUSIP must be exactly 9 alphanumeric characters");

        RuleFor(x => x.ProductName)
            .NotEmpty()
            .MaximumLength(255)
            .WithMessage("Product name is required and cannot exceed 255 characters");

        RuleFor(x => x.ProductType)
            .NotEmpty()
            .IsInEnum()
            .WithMessage("Product type must be a valid enum value");

        RuleFor(x => x.BaseRate)
            .GreaterThan(0)
            .LessThanOrEqualTo(20)
            .WithMessage("Base rate must be between 0 and 20%");

        RuleFor(x => x.BonusRate)
            .GreaterThanOrEqualTo(0)
            .LessThanOrEqualTo(5)
            .WithMessage("Bonus rate must be between 0 and 5%");

        RuleFor(x => x.MinimumInvestment)
            .GreaterThan(0)
            .WithMessage("Minimum investment must be greater than 0");

        RuleFor(x => x.MaximumInvestment)
            .GreaterThan(x => x.MinimumInvestment)
            .When(x => x.MaximumInvestment.HasValue)
            .WithMessage("Maximum investment must be greater than minimum investment");

        RuleFor(x => x.ReturnOfPremiumPercentage)
            .InclusiveBetween(0, 100)
            .When(x => x.ReturnOfPremium)
            .WithMessage("Return of premium percentage must be between 0 and 100%");
    }
}
```

### 2. Domain Validation

```csharp
public class Product : IAggregateRoot
{
    private readonly List<DomainEvent> _domainEvents = new();

    public Guid ProductId { get; private set; }
    public string Cusip { get; private set; }
    public string ProductName { get; private set; }
    public ProductType ProductType { get; private set; }
    public decimal BaseRate { get; private set; }
    public decimal BonusRate { get; private set; }
    public decimal MinimumInvestment { get; private set; }
    public decimal? MaximumInvestment { get; private set; }
    public bool ReturnOfPremium { get; private set; }
    public decimal? ReturnOfPremiumPercentage { get; private set; }
    public bool IsActive { get; private set; }

    public IReadOnlyCollection<DomainEvent> DomainEvents => _domainEvents.AsReadOnly();

    private Product() { }

    public static Product Create(
        string cusip,
        string productName,
        ProductType productType,
        decimal baseRate,
        decimal minimumInvestment,
        string createdBy)
    {
        ValidateProductCreation(cusip, productName, baseRate, minimumInvestment);

        var product = new Product
        {
            ProductId = Guid.NewGuid(),
            Cusip = cusip,
            ProductName = productName,
            ProductType = productType,
            BaseRate = baseRate,
            BonusRate = 0,
            MinimumInvestment = minimumInvestment,
            ReturnOfPremium = false,
            IsActive = true,
            CreatedAt = DateTime.UtcNow,
            CreatedBy = createdBy
        };

        product.AddDomainEvent(new ProductCreatedEvent
        {
            AggregateId = product.ProductId,
            Cusip = product.Cusip,
            ProductName = product.ProductName,
            ProductType = product.ProductType.ToString(),
            BaseRate = product.BaseRate,
            MinimumInvestment = product.MinimumInvestment,
            CreatedBy = product.CreatedBy
        });

        return product;
    }

    public void UpdateRates(decimal newBaseRate, decimal newBonusRate, DateTime effectiveDate, string updatedBy)
    {
        ValidateRateUpdate(newBaseRate, newBonusRate);

        var oldBaseRate = BaseRate;
        var oldBonusRate = BonusRate;

        BaseRate = newBaseRate;
        BonusRate = newBonusRate;
        UpdatedAt = DateTime.UtcNow;
        UpdatedBy = updatedBy;

        AddDomainEvent(new RateChangedEvent
        {
            AggregateId = ProductId,
            OldBaseRate = oldBaseRate,
            NewBaseRate = newBaseRate,
            OldBonusRate = oldBonusRate,
            NewBonusRate = newBonusRate,
            EffectiveDate = effectiveDate,
            ChangedBy = updatedBy
        });
    }

    public void Deactivate(string deactivatedBy)
    {
        if (!IsActive)
            throw new InvalidOperationException("Product is already inactive");

        IsActive = false;
        UpdatedAt = DateTime.UtcNow;
        UpdatedBy = deactivatedBy;

        AddDomainEvent(new ProductDeactivatedEvent
        {
            AggregateId = ProductId,
            DeactivatedBy = deactivatedBy
        });
    }

    private static void ValidateProductCreation(string cusip, string productName, decimal baseRate, decimal minimumInvestment)
    {
        if (string.IsNullOrWhiteSpace(cusip))
            throw new ArgumentException("CUSIP is required", nameof(cusip));

        if (cusip.Length != 9)
            throw new ArgumentException("CUSIP must be exactly 9 characters", nameof(cusip));

        if (string.IsNullOrWhiteSpace(productName))
            throw new ArgumentException("Product name is required", nameof(productName));

        if (baseRate <= 0 || baseRate > 20)
            throw new ArgumentException("Base rate must be between 0 and 20%", nameof(baseRate));

        if (minimumInvestment <= 0)
            throw new ArgumentException("Minimum investment must be greater than 0", nameof(minimumInvestment));
    }

    private void ValidateRateUpdate(decimal newBaseRate, decimal newBonusRate)
    {
        if (newBaseRate <= 0 || newBaseRate > 20)
            throw new ArgumentException("Base rate must be between 0 and 20%", nameof(newBaseRate));

        if (newBonusRate < 0 || newBonusRate > 5)
            throw new ArgumentException("Bonus rate must be between 0 and 5%", nameof(newBonusRate));

        if (newBaseRate + newBonusRate > 25)
            throw new ArgumentException("Total rate cannot exceed 25%");
    }

    private void AddDomainEvent(DomainEvent domainEvent)
    {
        _domainEvents.Add(domainEvent);
    }

    public void ClearDomainEvents()
    {
        _domainEvents.Clear();
    }
}
```

## Performance Patterns

### 1. Lazy Loading

```csharp
public class Product
{
    private readonly ILazyLoader _lazyLoader;

    public Product(ILazyLoader lazyLoader)
    {
        _lazyLoader = lazyLoader;
    }

    private ICollection<RateHistory> _rateHistory;
    public virtual ICollection<RateHistory> RateHistory
    {
        get => _lazyLoader.Load(this, ref _rateHistory);
        set => _rateHistory = value;
    }
}
```

### 2. Eager Loading

```csharp
public async Task<Product> GetProductWithDetailsAsync(Guid productId)
{
    return await _context.Products
        .Include(p => p.RateHistory.OrderByDescending(r => r.EffectiveDate))
        .Include(p => p.Availability.Where(a => a.IsActive))
        .FirstOrDefaultAsync(p => p.ProductId == productId);
}
```

### 3. Query Optimization

```csharp
public class ProductQueryService
{
    private readonly ApplicationDbContext _context;

    public async Task<List<ProductDto>> GetProductsOptimizedAsync(ProductQueryParameters parameters)
    {
        var query = _context.Products.AsNoTracking();

        // Apply filters
        if (parameters.IsActive.HasValue)
            query = query.Where(p => p.IsActive == parameters.IsActive.Value);

        if (!string.IsNullOrEmpty(parameters.ProductType))
            query = query.Where(p => p.ProductType == parameters.ProductType);

        if (parameters.MinRate.HasValue)
            query = query.Where(p => p.BaseRate >= parameters.MinRate.Value);

        if (parameters.MaxRate.HasValue)
            query = query.Where(p => p.BaseRate <= parameters.MaxRate.Value);

        // Select only needed fields
        var result = await query
            .Select(p => new ProductDto
            {
                ProductId = p.ProductId,
                Cusip = p.Cusip,
                ProductName = p.ProductName,
                ProductType = p.ProductType,
                BaseRate = p.BaseRate,
                BonusRate = p.BonusRate,
                MinimumInvestment = p.MinimumInvestment,
                MaximumInvestment = p.MaximumInvestment,
                ReturnOfPremium = p.ReturnOfPremium,
                ReturnOfPremiumPercentage = p.ReturnOfPremiumPercentage
            })
            .ToListAsync();

        return result;
    }
}
```

---

**Last Updated**: 2024-01-15
**Version**: 1.0.0
**Next Review**: 2024-04-15 