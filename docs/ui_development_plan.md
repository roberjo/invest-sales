# Investment Product Sales Tool - UI Development Plan

## Overview

This document outlines the UI/UX development plan for the Investment Product Sales Tool using React, Vite, and Tailwind CSS. The plan focuses on creating a modern, responsive, and user-friendly interface for both sales representatives and administrators.

## Technology Stack

### Frontend Framework
- **React 18.2+**: Core framework with hooks and functional components
- **Vite 5.x**: Build tool and development server
- **TypeScript**: Type safety and better developer experience
- **Tailwind CSS 3.x**: Utility-first CSS framework

### Additional Libraries
- **React Router 6.x**: Client-side routing
- **React Hook Form**: Form management and validation
- **React Query/TanStack Query**: Data fetching and caching
- **React Table/TanStack Table**: Data table components
- **Chart.js/React-Chartjs-2**: Data visualization
- **Headless UI**: Accessible UI components
- **Heroicons**: Icon library
- **React Hot Toast**: Toast notifications
- **Framer Motion**: Animations and transitions

## Design System

### Color Palette
```css
/* Primary Colors */
--primary-50: #eff6ff;
--primary-500: #3b82f6;
--primary-600: #2563eb;
--primary-700: #1d4ed8;

/* Secondary Colors */
--secondary-50: #f8fafc;
--secondary-500: #64748b;
--secondary-600: #475569;
--secondary-700: #334155;

/* Success/Error Colors */
--success-500: #10b981;
--error-500: #ef4444;
--warning-500: #f59e0b;

/* Neutral Colors */
--gray-50: #f9fafb;
--gray-100: #f3f4f6;
--gray-200: #e5e7eb;
--gray-300: #d1d5db;
--gray-400: #9ca3af;
--gray-500: #6b7280;
--gray-600: #4b5563;
--gray-700: #374151;
--gray-800: #1f2937;
--gray-900: #111827;
```

### Typography
```css
/* Font Families */
--font-sans: 'Inter', system-ui, -apple-system, sans-serif;
--font-mono: 'JetBrains Mono', 'Fira Code', monospace;

/* Font Scales */
--text-xs: 0.75rem;
--text-sm: 0.875rem;
--text-base: 1rem;
--text-lg: 1.125rem;
--text-xl: 1.25rem;
--text-2xl: 1.5rem;
--text-3xl: 1.875rem;
--text-4xl: 2.25rem;
```

### Spacing and Layout
```css
/* Spacing Scale */
--spacing-1: 0.25rem;
--spacing-2: 0.5rem;
--spacing-4: 1rem;
--spacing-6: 1.5rem;
--spacing-8: 2rem;
--spacing-12: 3rem;
--spacing-16: 4rem;

/* Container Sizes */
--container-sm: 640px;
--container-md: 768px;
--container-lg: 1024px;
--container-xl: 1280px;
--container-2xl: 1536px;
```

## Component Architecture

### Core Components Structure
```
src/
├── components/
│   ├── ui/                    # Reusable UI components
│   │   ├── Button/
│   │   ├── Input/
│   │   ├── Modal/
│   │   ├── Table/
│   │   ├── Card/
│   │   └── ...
│   ├── layout/                # Layout components
│   │   ├── Header/
│   │   ├── Sidebar/
│   │   ├── Footer/
│   │   └── MainLayout/
│   ├── forms/                 # Form components
│   │   ├── ProductForm/
│   │   ├── LoginForm/
│   │   └── ...
│   ├── charts/                # Chart components
│   │   ├── GrowthChart/
│   │   ├── ComparisonChart/
│   │   └── ...
│   └── features/              # Feature-specific components
│       ├── ProductCatalog/
│       ├── ProductComparison/
│       ├── AdminDashboard/
│       └── ...
├── hooks/                     # Custom React hooks
├── utils/                     # Utility functions
├── types/                     # TypeScript type definitions
├── services/                  # API service layer
├── contexts/                  # React contexts
└── pages/                     # Page components
```

## User Interface Specifications

### 1. Authentication Pages

#### Login Page
```typescript
// Features:
- Okta OAuth2 integration
- Clean, professional design
- Loading states
- Error handling
- Responsive layout

// Components:
- LoginForm
- OktaButton
- LoadingSpinner
- ErrorMessage
```

#### Authentication Guard
```typescript
// Features:
- Route protection
- Token validation
- Automatic redirects
- Role-based access control
```

### 2. Sales Representative Interface

#### Dashboard
```typescript
// Layout: 
- Header with user info and logout
- Main content area
- Quick action buttons
- Recent activity sidebar

// Components:
- DashboardHeader
- ProductSummaryCards
- QuickActionButtons
- RecentActivityPanel
- StatisticsOverview
```

#### Product Catalog
```typescript
// Features:
- Filterable product grid
- Search functionality
- Product type tabs
- Sort options
- Pagination

// Components:
- ProductGrid
- ProductCard
- FilterPanel
- SearchBar
- Pagination
- ProductTypeToggle
```

#### Product Comparison
```typescript
// Features:
- Side-by-side comparison
- Investment calculator
- Growth projections
- Export functionality
- Print-friendly view

// Components:
- ComparisonTable
- InvestmentCalculator
- GrowthChart
- ExportButton
- PrintButton
```

### 3. Administrator Interface

#### Admin Dashboard
```typescript
// Features:
- System overview
- User activity metrics
- Product statistics
- Audit trail summary

// Components:
- AdminDashboardHeader
- MetricsCards
- ActivityChart
- SystemHealth
- QuickActions
```

#### Product Management
```typescript
// Features:
- Product CRUD operations
- Bulk operations
- Rate management
- Availability scheduling
- Import/export

// Components:
- ProductTable
- ProductForm
- RateHistoryTable
- AvailabilityScheduler
- BulkActionsPanel
```

#### User Management
```typescript
// Features:
- User list with roles
- Role assignment
- Activity monitoring
- Session management

// Components:
- UserTable
- RoleAssignmentModal
- ActivityLog
- SessionMonitor
```

#### Audit Trail
```typescript
// Features:
- Comprehensive audit log
- Advanced filtering
- Export capabilities
- Real-time updates

// Components:
- AuditTable
- AuditFilters
- ExportOptions
- RealTimeUpdates
```

## Responsive Design Strategy

### Breakpoints
```css
/* Mobile First Approach */
--mobile: 320px;
--tablet: 768px;
--desktop: 1024px;
--large-desktop: 1280px;
--extra-large: 1536px;
```

### Grid System
```typescript
// 12-column grid system using Tailwind CSS
// Responsive utilities for different screen sizes
// Flexbox for complex layouts
// CSS Grid for dashboard layouts
```

### Mobile Optimizations
- Touch-friendly buttons (44px minimum)
- Collapsible navigation
- Swipe gestures for tables
- Optimized form inputs
- Reduced animation on mobile

## State Management

### Global State Architecture
```typescript
// React Context for:
- User authentication state
- User preferences
- Application settings
- Theme selection

// React Query for:
- Server state management
- Data fetching and caching
- Optimistic updates
- Background refetching

// Local State for:
- Form state (React Hook Form)
- UI state (modals, dropdowns)
- Component-specific state
```

### Data Flow
```typescript
// Authentication Flow
Okta OAuth → JWT Token → Context → Protected Routes

// Product Data Flow
API → React Query → Components → UI Updates

// Form Data Flow
User Input → React Hook Form → Validation → API → Success/Error
```

## Performance Optimization

### Code Splitting
```typescript
// Route-based code splitting
const Dashboard = lazy(() => import('./pages/Dashboard'));
const ProductCatalog = lazy(() => import('./pages/ProductCatalog'));
const AdminPanel = lazy(() => import('./pages/AdminPanel'));

// Component-based code splitting
const HeavyChart = lazy(() => import('./components/HeavyChart'));
```

### Optimization Techniques
- React.memo for expensive components
- useMemo and useCallback for optimization
- Virtual scrolling for large lists
- Image optimization and lazy loading
- Bundle size optimization with Vite

### Caching Strategy
- React Query for API response caching
- Service worker for offline capability
- Local storage for user preferences
- Session storage for temporary data

## Development Workflow

### Project Setup
```bash
# Create Vite project with TypeScript
npm create vite@latest investment-tool -- --template react-ts

# Install