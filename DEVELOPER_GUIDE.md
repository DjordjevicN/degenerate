# Developer Guide

## 👩‍💻 Overview

This guide provides practical instructions and best practices for developing features in the Resource Monitoring Frontend. It covers common tasks, workflows, and patterns that developers will encounter.

## 🛠️ Development Setup

### Environment Prerequisites

```bash
# Required Node.js versions
nvm install 22    # For livenx-web dependency
nvm install 23    # For resource monitoring frontend

# Check installations
node --version    # Should be 23.x for development
npm --version     # Should be 10.x+
```

### Development Workflow

#### 1. Project Setup
```bash
# Clone the repository
git clone <repository-url>
cd resource-monitoring-frontend

# Install dependencies
nvm use 23
npm install

# Set up SSH tunnels for API access
ssh -f -N -L 27017:127.0.0.1:27017 admin@10.244.17.62  # MongoDB
ssh -f -N -L 8093:127.0.0.1:8093 admin@10.244.17.62    # Java Server
ssh -f -N -L 4041:127.0.0.1:4041 admin@10.244.17.62    # ClickHouse

# Start livenx-web (required dependency)
cd ../livenx-web
nvm use 22
npm install
npm run sandbox

# Return to resource monitoring and start development
cd ../resource-monitoring-frontend
nvm use 23
npm run dev
```

#### 2. Daily Development Workflow
```bash
# Start development environment
./scripts/dev-setup.sh  # Custom script for tunnel setup

# Run tests
npm test

# Check code quality
npm run lint
npm run format

# Build for testing
npm run build

# Preview production build
npm run preview
```

---

## 🏗️ Adding New Features

### 1. Adding a New Page

#### Step-by-Step Guide

**1. Create the page component:**
```typescript
// src/pages/NewFeature/NewFeature.tsx
import React from 'react';
import { useSearchParams } from 'react-router-dom';

import BreadcrumbNavigation from '@components/BreadcrumbNavigation/BreadcrumbNavigation';
import Container from '@components/Container/Container';
import ErrorState from '@components/ErrorState/ErrorState';

const NewFeature: React.FC = () => {
  const [searchParams] = useSearchParams();
  
  // Custom logic here
  
  return (
    <Container isVertical>
      <Container isVertical align="start" className="layer01">
        <BreadcrumbNavigation />
        <Container
          isVertical
          align="start"
          gap={16}
          spacing={{ top: 16, right: 32, left: 32, bottom: 16 }}
        >
          <h1 className="pageTitle">New Feature</h1>
          {/* Feature content */}
        </Container>
      </Container>
      
      {/* Main content area */}
      <Container isVertical spacing={16} gap={16} align="start">
        {/* Feature implementation */}
      </Container>
    </Container>
  );
};

export default NewFeature;
```

**2. Add page styles:**
```less
// src/pages/NewFeature/styles.less
.new-feature {
  &__header {
    display: flex;
    justify-content: space-between;
    align-items: center;
    margin-bottom: var(--spacing-lg);
  }
  
  &__content {
    display: grid;
    grid-template-columns: 1fr 300px;
    gap: var(--spacing-lg);
    
    @media (max-width: 768px) {
      grid-template-columns: 1fr;
    }
  }
  
  &__main {
    min-height: 500px;
  }
  
  &__sidebar {
    background: var(--layer-01);
    border-radius: var(--border-radius-md);
    padding: var(--spacing-md);
  }
}
```

**3. Add to routing:**
```typescript
// src/App.tsx
import NewFeature from './pages/NewFeature/NewFeature';

// Inside Routes component
<Route path="new-feature" element={<NewFeature />} />
```

**4. Create tests:**
```typescript
// src/pages/NewFeature/__tests__/NewFeature.test.tsx
import { screen, waitFor } from '@testing-library/react';
import { renderWithProviders } from '@tests/renderWithProviders';

import NewFeature from '../NewFeature';

describe('NewFeature', () => {
  it('renders page title', () => {
    renderWithProviders(<NewFeature />);
    expect(screen.getByText('New Feature')).toBeInTheDocument();
  });
  
  it('shows breadcrumb navigation', () => {
    renderWithProviders(<NewFeature />);
    expect(screen.getByRole('navigation')).toBeInTheDocument();
  });
});
```

### 2. Adding a New Component

#### Component Template

```typescript
// src/components/NewComponent/NewComponent.tsx
import React, { useState, useCallback, useMemo } from 'react';
import { useQuery } from '@tanstack/react-query';

import Container from '@components/Container/Container';
import ErrorState from '@components/ErrorState/ErrorState';

import './styles.less';

interface NewComponentProps {
  // Define props with clear documentation
  data?: DataType[];
  onAction?: (item: DataType) => void;
  loading?: boolean;
  error?: Error | null;
  className?: string;
  // Optional configuration
  config?: {
    showHeader?: boolean;
    enableActions?: boolean;
  };
}

const NewComponent: React.FC<NewComponentProps> = ({
  data = [],
  onAction,
  loading = false,
  error = null,
  className = '',
  config = {},
}) => {
  // Local state
  const [selectedItem, setSelectedItem] = useState<DataType | null>(null);
  
  // Memoized computations
  const processedData = useMemo(() => {
    return data.map(item => ({
      ...item,
      processed: true,
      // Add computed properties
    }));
  }, [data]);
  
  // Event handlers
  const handleItemClick = useCallback((item: DataType) => {
    setSelectedItem(item);
    onAction?.(item);
  }, [onAction]);
  
  // Early returns for error states
  if (error) {
    return (
      <ErrorState
        title="Failed to load data"
        text={error.message}
      />
    );
  }
  
  if (loading) {
    return <div className="loading-skeleton">Loading...</div>;
  }
  
  // Main render
  return (
    <Container className={`new-component ${className}`}>
      {config.showHeader && (
        <div className="new-component__header">
          <h2>Component Title</h2>
        </div>
      )}
      
      <div className="new-component__content">
        {processedData.map(item => (
          <div 
            key={item.id}
            className="new-component__item"
            onClick={() => handleItemClick(item)}
          >
            {/* Item content */}
          </div>
        ))}
      </div>
    </Container>
  );
};

export default NewComponent;
```

#### Component Styles Template

```less
// src/components/NewComponent/styles.less
.new-component {
  display: flex;
  flex-direction: column;
  background: var(--layer-01);
  border-radius: var(--border-radius-md);
  overflow: hidden;
  
  &__header {
    padding: var(--spacing-md) var(--spacing-lg);
    border-bottom: 1px solid var(--border-color);
    background: var(--layer-02);
    
    h2 {
      margin: 0;
      font-size: var(--font-size-lg);
      font-weight: var(--font-weight-semibold);
      color: var(--text-primary);
    }
  }
  
  &__content {
    flex: 1;
    padding: var(--spacing-md);
    
    // Handle empty state
    &:empty::after {
      content: 'No data available';
      display: block;
      text-align: center;
      color: var(--text-secondary);
      padding: var(--spacing-xl);
    }
  }
  
  &__item {
    padding: var(--spacing-sm) var(--spacing-md);
    border-radius: var(--border-radius-sm);
    cursor: pointer;
    transition: background-color 0.2s ease;
    
    &:hover {
      background: var(--layer-hover);
    }
    
    &:focus {
      outline: 2px solid var(--focus-color);
      outline-offset: 2px;
    }
    
    &--selected {
      background: var(--layer-selected);
      border-left: 3px solid var(--primary-color);
    }
  }
  
  // Loading state
  &.loading {
    .new-component__content {
      opacity: 0.6;
      pointer-events: none;
    }
  }
  
  // Error state
  &.error {
    border: 1px solid var(--error-color);
  }
  
  // Responsive design
  @media (max-width: 768px) {
    &__header {
      padding: var(--spacing-sm) var(--spacing-md);
      
      h2 {
        font-size: var(--font-size-md);
      }
    }
    
    &__content {
      padding: var(--spacing-sm);
    }
  }
}

// Dark theme adjustments
body.dark-theme .new-component {
  &__header {
    border-bottom-color: var(--border-color-dark);
  }
  
  &__item {
    &:hover {
      background: var(--layer-hover-dark);
    }
  }
}
```

#### Component Tests Template

```typescript
// src/components/NewComponent/__tests__/NewComponent.test.tsx
import { screen, fireEvent, waitFor } from '@testing-library/react';
import { renderWithProviders } from '@tests/renderWithProviders';
import userEvent from '@testing-library/user-event';

import NewComponent from '../NewComponent';

const mockData = [
  { id: '1', name: 'Item 1' },
  { id: '2', name: 'Item 2' },
];

describe('NewComponent', () => {
  it('renders component with data', () => {
    renderWithProviders(<NewComponent data={mockData} />);
    
    expect(screen.getByText('Item 1')).toBeInTheDocument();
    expect(screen.getByText('Item 2')).toBeInTheDocument();
  });
  
  it('handles item click events', async () => {
    const onAction = jest.fn();
    const user = userEvent.setup();
    
    renderWithProviders(
      <NewComponent data={mockData} onAction={onAction} />
    );
    
    await user.click(screen.getByText('Item 1'));
    
    expect(onAction).toHaveBeenCalledWith(mockData[0]);
  });
  
  it('shows loading state', () => {
    renderWithProviders(<NewComponent loading={true} />);
    
    expect(screen.getByText('Loading...')).toBeInTheDocument();
  });
  
  it('handles error state', () => {
    const error = new Error('Test error');
    renderWithProviders(<NewComponent error={error} />);
    
    expect(screen.getByText('Failed to load data')).toBeInTheDocument();
    expect(screen.getByText('Test error')).toBeInTheDocument();
  });
  
  it('renders with optional config', () => {
    renderWithProviders(
      <NewComponent 
        data={mockData} 
        config={{ showHeader: true }} 
      />
    );
    
    expect(screen.getByText('Component Title')).toBeInTheDocument();
  });
});
```

### 3. Adding New API Endpoints

#### API Service Pattern

```typescript
// src/services/api/newService/api.ts
import apiClient from '../../apiClient';
import type { NewDataType, NewDataParams, NewDataResponse } from './types';

export const fetchNewData = async (
  params: NewDataParams
): Promise<NewDataResponse> => {
  try {
    const response = await apiClient.get('/api/new-endpoint', {
      params,
      timeout: 10000, // Custom timeout for this endpoint
    });
    
    // Validate response structure
    if (!response.data || !Array.isArray(response.data.items)) {
      throw new Error('Invalid response structure');
    }
    
    return {
      items: response.data.items,
      totalCount: response.data.totalCount || 0,
      metadata: response.data.metadata || {},
    };
  } catch (error) {
    console.error('Failed to fetch new data:', error);
    throw error;
  }
};

export const createNewData = async (
  data: Omit<NewDataType, 'id' | 'createdAt' | 'updatedAt'>
): Promise<NewDataType> => {
  try {
    const response = await apiClient.post('/api/new-endpoint', data);
    return response.data;
  } catch (error) {
    console.error('Failed to create new data:', error);
    throw error;
  }
};

export const updateNewData = async (
  id: string,
  updates: Partial<NewDataType>
): Promise<NewDataType> => {
  try {
    const response = await apiClient.patch(`/api/new-endpoint/${id}`, updates);
    return response.data;
  } catch (error) {
    console.error(`Failed to update data ${id}:`, error);
    throw error;
  }
};

export const deleteNewData = async (id: string): Promise<void> => {
  try {
    await apiClient.delete(`/api/new-endpoint/${id}`);
  } catch (error) {
    console.error(`Failed to delete data ${id}:`, error);
    throw error;
  }
};
```

#### React Query Hooks

```typescript
// src/services/api/newService/query.ts
import { useQuery, useMutation, useQueryClient } from '@tanstack/react-query';
import type { UseQueryResult, UseMutationResult } from '@tanstack/react-query';

import { fetchNewData, createNewData, updateNewData, deleteNewData } from './api';
import type { NewDataType, NewDataParams, NewDataResponse } from './types';

// Query hook for fetching data
export const useNewData = (
  params: NewDataParams
): UseQueryResult<NewDataResponse, Error> => {
  return useQuery({
    queryKey: ['newData', params],
    queryFn: () => fetchNewData(params),
    staleTime: 5 * 60 * 1000, // 5 minutes
    retry: 2,
    retryDelay: (attemptIndex) => Math.min(1000 * 2 ** attemptIndex, 30000),
  });
};

// Mutation hook for creating data
export const useCreateNewData = (): UseMutationResult<
  NewDataType,
  Error,
  Omit<NewDataType, 'id' | 'createdAt' | 'updatedAt'>
> => {
  const queryClient = useQueryClient();
  
  return useMutation({
    mutationFn: createNewData,
    onSuccess: (newItem) => {
      // Invalidate queries to refetch data
      queryClient.invalidateQueries({ queryKey: ['newData'] });
      
      // Optionally add to cache optimistically
      queryClient.setQueryData(
        ['newData'], 
        (oldData: NewDataResponse | undefined) => {
          if (!oldData) return oldData;
          return {
            ...oldData,
            items: [newItem, ...oldData.items],
            totalCount: oldData.totalCount + 1,
          };
        }
      );
    },
    onError: (error) => {
      console.error('Create mutation error:', error);
      // Handle error (show notification, etc.)
    },
  });
};

// Mutation hook for updating data
export const useUpdateNewData = (): UseMutationResult<
  NewDataType,
  Error,
  { id: string; updates: Partial<NewDataType> }
> => {
  const queryClient = useQueryClient();
  
  return useMutation({
    mutationFn: ({ id, updates }) => updateNewData(id, updates),
    onMutate: async ({ id, updates }) => {
      // Cancel outgoing refetches
      await queryClient.cancelQueries({ queryKey: ['newData'] });
      
      // Snapshot previous state
      const previousData = queryClient.getQueryData(['newData']);
      
      // Optimistically update
      queryClient.setQueryData(['newData'], (old: NewDataResponse | undefined) => {
        if (!old) return old;
        return {
          ...old,
          items: old.items.map(item =>
            item.id === id ? { ...item, ...updates } : item
          ),
        };
      });
      
      return { previousData };
    },
    onError: (error, variables, context) => {
      // Rollback optimistic update
      if (context?.previousData) {
        queryClient.setQueryData(['newData'], context.previousData);
      }
    },
    onSettled: () => {
      // Refetch after mutation
      queryClient.invalidateQueries({ queryKey: ['newData'] });
    },
  });
};

// Mutation hook for deleting data
export const useDeleteNewData = (): UseMutationResult<void, Error, string> => {
  const queryClient = useQueryClient();
  
  return useMutation({
    mutationFn: deleteNewData,
    onSuccess: (_, deletedId) => {
      // Remove from cache
      queryClient.setQueryData(['newData'], (oldData: NewDataResponse | undefined) => {
        if (!oldData) return oldData;
        return {
          ...oldData,
          items: oldData.items.filter(item => item.id !== deletedId),
          totalCount: oldData.totalCount - 1,
        };
      });
    },
  });
};
```

### 4. Adding Custom Hooks

#### Custom Hook Template

```typescript
// src/hooks/useCustomHook.ts
import { useState, useEffect, useCallback, useMemo } from 'react';
import { useSearchParams } from 'react-router-dom';

interface UseCustomHookOptions {
  defaultValue?: any;
  autoUpdate?: boolean;
  debounceMs?: number;
}

interface UseCustomHookReturn {
  value: any;
  setValue: (value: any) => void;
  isLoading: boolean;
  error: Error | null;
  reset: () => void;
}

export const useCustomHook = (
  options: UseCustomHookOptions = {}
): UseCustomHookReturn => {
  const {
    defaultValue = null,
    autoUpdate = true,
    debounceMs = 300,
  } = options;
  
  // State management
  const [value, setValue] = useState(defaultValue);
  const [isLoading, setIsLoading] = useState(false);
  const [error, setError] = useState<Error | null>(null);
  
  // URL params integration (if needed)
  const [searchParams, setSearchParams] = useSearchParams();
  
  // Debounced value
  const debouncedValue = useMemo(() => {
    const timer = setTimeout(() => value, debounceMs);
    return () => clearTimeout(timer);
  }, [value, debounceMs]);
  
  // Reset function
  const reset = useCallback(() => {
    setValue(defaultValue);
    setError(null);
    setIsLoading(false);
  }, [defaultValue]);
  
  // Side effects
  useEffect(() => {
    if (!autoUpdate) return;
    
    const performUpdate = async () => {
      setIsLoading(true);
      setError(null);
      
      try {
        // Perform async operations here
        await new Promise(resolve => setTimeout(resolve, 1000));
        
        // Update URL params if needed
        const newParams = new URLSearchParams(searchParams);
        newParams.set('customValue', String(value));
        setSearchParams(newParams);
        
      } catch (err) {
        setError(err as Error);
      } finally {
        setIsLoading(false);
      }
    };
    
    performUpdate();
  }, [value, autoUpdate, searchParams, setSearchParams]);
  
  // Cleanup
  useEffect(() => {
    return () => {
      // Cleanup logic
    };
  }, []);
  
  return {
    value,
    setValue,
    isLoading,
    error,
    reset,
  };
};

// Hook tests
// src/hooks/__tests__/useCustomHook.test.ts
import { renderHook, act, waitFor } from '@testing-library/react';
import { BrowserRouter } from 'react-router-dom';

import { useCustomHook } from '../useCustomHook';

const wrapper = ({ children }: { children: React.ReactNode }) => (
  <BrowserRouter>{children}</BrowserRouter>
);

describe('useCustomHook', () => {
  it('initializes with default value', () => {
    const { result } = renderHook(
      () => useCustomHook({ defaultValue: 'test' }),
      { wrapper }
    );
    
    expect(result.current.value).toBe('test');
    expect(result.current.isLoading).toBe(false);
    expect(result.current.error).toBe(null);
  });
  
  it('updates value correctly', async () => {
    const { result } = renderHook(() => useCustomHook(), { wrapper });
    
    act(() => {
      result.current.setValue('new value');
    });
    
    expect(result.current.value).toBe('new value');
  });
  
  it('handles reset correctly', () => {
    const { result } = renderHook(
      () => useCustomHook({ defaultValue: 'default' }),
      { wrapper }
    );
    
    act(() => {
      result.current.setValue('changed');
    });
    
    act(() => {
      result.current.reset();
    });
    
    expect(result.current.value).toBe('default');
  });
});
```

---

## 🎨 Styling Guidelines

### CSS Architecture

```scss
// Style organization
src/styles/
├── index.less          // Main entry point
├── base/
│   ├── reset.less      // CSS reset
│   ├── typography.less // Font definitions
│   └── layout.less     // Basic layout classes
├── themes/
│   ├── light.less      // Light theme variables
│   ├── dark.less       // Dark theme variables
│   └── tokens.less     // Design tokens
├── utils/
│   ├── mixins.less     // Reusable mixins
│   ├── functions.less  // Less functions
│   └── helpers.less    // Utility classes
└── vendors/
    └── pelagos.less    // Third-party overrides
```

### Design Tokens

```less
// src/styles/themes/tokens.less
:root {
  // Color Tokens
  --color-primary-50: #e3f2fd;
  --color-primary-100: #bbdefb;
  --color-primary-500: #2196f3;
  --color-primary-900: #0d47a1;
  
  --color-gray-50: #fafafa;
  --color-gray-100: #f5f5f5;
  --color-gray-500: #9e9e9e;
  --color-gray-900: #212121;
  
  // Semantic Colors
  --color-success: #4caf50;
  --color-warning: #ff9800;
  --color-error: #f44336;
  --color-info: #2196f3;
  
  // Typography Tokens
  --font-family-primary: 'Inter', -apple-system, sans-serif;
  --font-family-mono: 'JetBrains Mono', 'Consolas', monospace;
  
  --font-size-xs: 0.75rem;   // 12px
  --font-size-sm: 0.875rem;  // 14px
  --font-size-md: 1rem;      // 16px
  --font-size-lg: 1.125rem;  // 18px
  --font-size-xl: 1.25rem;   // 20px
  
  --font-weight-normal: 400;
  --font-weight-medium: 500;
  --font-weight-semibold: 600;
  --font-weight-bold: 700;
  
  --line-height-tight: 1.25;
  --line-height-normal: 1.5;
  --line-height-relaxed: 1.75;
  
  // Spacing Tokens
  --space-1: 0.25rem;  // 4px
  --space-2: 0.5rem;   // 8px
  --space-3: 0.75rem;  // 12px
  --space-4: 1rem;     // 16px
  --space-6: 1.5rem;   // 24px
  --space-8: 2rem;     // 32px
  --space-12: 3rem;    // 48px
  
  // Border Radius
  --radius-sm: 0.125rem; // 2px
  --radius-md: 0.25rem;  // 4px
  --radius-lg: 0.5rem;   // 8px
  --radius-xl: 1rem;     // 16px
  
  // Shadows
  --shadow-sm: 0 1px 2px 0 rgba(0, 0, 0, 0.05);
  --shadow-md: 0 4px 6px -1px rgba(0, 0, 0, 0.1);
  --shadow-lg: 0 10px 15px -3px rgba(0, 0, 0, 0.1);
  
  // Z-Index Scale
  --z-dropdown: 1000;
  --z-sticky: 1020;
  --z-fixed: 1030;
  --z-modal: 1040;
  --z-popover: 1050;
  --z-tooltip: 1060;
}
```

### Component Styling Pattern

```less
// Component-specific styling template
.component-name {
  // Layout
  display: flex;
  flex-direction: column;
  
  // Spacing
  padding: var(--space-4);
  margin: var(--space-2) 0;
  gap: var(--space-3);
  
  // Appearance
  background: var(--layer-01);
  border: 1px solid var(--border-color);
  border-radius: var(--radius-md);
  box-shadow: var(--shadow-sm);
  
  // Typography
  font-family: var(--font-family-primary);
  font-size: var(--font-size-md);
  line-height: var(--line-height-normal);
  
  // States
  &:hover {
    background: var(--layer-hover);
    box-shadow: var(--shadow-md);
  }
  
  &:focus-within {
    outline: 2px solid var(--color-primary-500);
    outline-offset: 2px;
  }
  
  &[data-state="loading"] {
    opacity: 0.6;
    pointer-events: none;
  }
  
  &[data-state="error"] {
    border-color: var(--color-error);
    background: color-mix(in srgb, var(--color-error) 5%, transparent);
  }
  
  // Nested elements (BEM methodology)
  &__header {
    display: flex;
    justify-content: space-between;
    align-items: center;
    padding-bottom: var(--space-3);
    border-bottom: 1px solid var(--border-color);
    
    h2 {
      margin: 0;
      font-size: var(--font-size-lg);
      font-weight: var(--font-weight-semibold);
      color: var(--text-primary);
    }
  }
  
  &__content {
    flex: 1;
    min-height: 0; // Prevents flex item overflow
  }
  
  &__footer {
    display: flex;
    justify-content: flex-end;
    gap: var(--space-3);
    padding-top: var(--space-3);
    border-top: 1px solid var(--border-color);
  }
  
  // Modifiers
  &--compact {
    padding: var(--space-2);
    gap: var(--space-2);
  }
  
  &--bordered {
    border-width: 2px;
  }
  
  &--elevated {
    box-shadow: var(--shadow-lg);
  }
  
  // Responsive design
  @media (max-width: 768px) {
    padding: var(--space-3);
    gap: var(--space-2);
    
    &__header {
      flex-direction: column;
      align-items: flex-start;
      gap: var(--space-2);
      
      h2 {
        font-size: var(--font-size-md);
      }
    }
    
    &__footer {
      flex-direction: column;
      
      .button {
        width: 100%;
      }
    }
  }
  
  @media (max-width: 480px) {
    margin: var(--space-1) 0;
    border-radius: var(--radius-sm);
  }
}

// Dark theme support
[data-theme="dark"] .component-name {
  background: var(--layer-01-dark);
  border-color: var(--border-color-dark);
  color: var(--text-primary-dark);
  
  &:hover {
    background: var(--layer-hover-dark);
  }
  
  &__header {
    border-bottom-color: var(--border-color-dark);
    
    h2 {
      color: var(--text-primary-dark);
    }
  }
  
  &__footer {
    border-top-color: var(--border-color-dark);
  }
}

// High contrast theme support
@media (prefers-contrast: high) {
  .component-name {
    border-width: 2px;
    border-color: var(--text-primary);
    
    &:focus-within {
      outline-width: 3px;
    }
  }
}

// Reduced motion support
@media (prefers-reduced-motion: reduce) {
  .component-name {
    transition: none;
    
    * {
      transition: none !important;
      animation: none !important;
    }
  }
}

// Print styles
@media print {
  .component-name {
    border: 1px solid #000;
    box-shadow: none;
    break-inside: avoid;
    
    &__header {
      border-bottom: 1px solid #000;
    }
    
    &__footer {
      border-top: 1px solid #000;
    }
  }
}
```

---

## 🧪 Testing Patterns

### Component Testing Strategy

```typescript
// Complete component test example
import { screen, fireEvent, waitFor, within } from '@testing-library/react';
import userEvent from '@testing-library/user-event';
import { renderWithProviders } from '@tests/renderWithProviders';
import { server } from '@tests/mocks/server';
import { http, HttpResponse } from 'msw';

import ComponentName from '../ComponentName';

// Mock data
const mockData = [
  { id: '1', name: 'Item 1', status: 'active' },
  { id: '2', name: 'Item 2', status: 'inactive' },
];

describe('ComponentName', () => {
  // Basic rendering test
  it('renders with default props', () => {
    renderWithProviders(<ComponentName />);
    expect(screen.getByRole('main')).toBeInTheDocument();
  });
  
  // Props testing
  it('renders with custom data', () => {
    renderWithProviders(<ComponentName data={mockData} />);
    
    expect(screen.getByText('Item 1')).toBeInTheDocument();
    expect(screen.getByText('Item 2')).toBeInTheDocument();
  });
  
  // User interaction testing
  it('handles click events', async () => {
    const onItemClick = jest.fn();
    const user = userEvent.setup();
    
    renderWithProviders(
      <ComponentName data={mockData} onItemClick={onItemClick} />
    );
    
    const firstItem = screen.getByText('Item 1');
    await user.click(firstItem);
    
    expect(onItemClick).toHaveBeenCalledWith(mockData[0]);
    expect(onItemClick).toHaveBeenCalledTimes(1);
  });
  
  // Keyboard interaction testing
  it('supports keyboard navigation', async () => {
    const user = userEvent.setup();
    
    renderWithProviders(<ComponentName data={mockData} />);
    
    const firstItem = screen.getByText('Item 1');
    firstItem.focus();
    
    await user.keyboard('{ArrowDown}');
    expect(screen.getByText('Item 2')).toHaveFocus();
    
    await user.keyboard('{Enter}');
    // Assert expected behavior
  });
  
  // Loading state testing
  it('shows loading state', () => {
    renderWithProviders(<ComponentName loading={true} />);
    expect(screen.getByText('Loading...')).toBeInTheDocument();
  });
  
  // Error state testing
  it('handles error state', () => {
    const error = new Error('Test error');
    renderWithProviders(<ComponentName error={error} />);
    
    expect(screen.getByText('Test error')).toBeInTheDocument();
    expect(screen.getByRole('button', { name: /retry/i })).toBeInTheDocument();
  });
  
  // Empty state testing
  it('shows empty state when no data', () => {
    renderWithProviders(<ComponentName data={[]} />);
    expect(screen.getByText(/no items found/i)).toBeInTheDocument();
  });
  
  // API integration testing
  it('fetches data on mount', async () => {
    server.use(
      http.get('/api/items', () => {
        return HttpResponse.json({ items: mockData });
      })
    );
    
    renderWithProviders(<ComponentName />);
    
    await waitFor(() => {
      expect(screen.getByText('Item 1')).toBeInTheDocument();
    });
  });
  
  // Form testing
  it('submits form with correct data', async () => {
    const onSubmit = jest.fn();
    const user = userEvent.setup();
    
    renderWithProviders(<ComponentName onSubmit={onSubmit} />);
    
    await user.type(screen.getByLabelText(/name/i), 'New Item');
    await user.click(screen.getByRole('button', { name: /submit/i }));
    
    expect(onSubmit).toHaveBeenCalledWith({ name: 'New Item' });
  });
  
  // Accessibility testing
  it('meets accessibility standards', () => {
    renderWithProviders(<ComponentName data={mockData} />);
    
    // Check ARIA attributes
    const listbox = screen.getByRole('listbox');
    expect(listbox).toHaveAttribute('aria-label');
    
    // Check keyboard accessibility
    const items = screen.getAllByRole('option');
    items.forEach(item => {
      expect(item).toHaveAttribute('tabindex');
    });
  });
  
  // Responsive behavior testing
  it('adapts to mobile viewport', () => {
    // Mock window dimensions
    Object.defineProperty(window, 'innerWidth', {
      writable: true,
      configurable: true,
      value: 375,
    });
    
    renderWithProviders(<ComponentName data={mockData} />);
    
    // Assert mobile-specific behavior
    expect(screen.getByTestId('mobile-layout')).toBeInTheDocument();
  });
});
```

### Integration Testing

```typescript
// Integration test example
describe('Device Management Flow', () => {
  it('completes full device management workflow', async () => {
    const user = userEvent.setup();
    
    // Mock API responses
    server.use(
      http.get('/api/devices', () => HttpResponse.json({ items: mockDevices })),
      http.post('/api/devices', () => HttpResponse.json(newDevice)),
      http.patch('/api/devices/1', () => HttpResponse.json(updatedDevice)),
    );
    
    // Render the main application
    renderWithProviders(<App />);
    
    // Navigate to devices page
    await user.click(screen.getByText('Devices'));
    
    // Verify devices load
    await waitFor(() => {
      expect(screen.getByText('router-01')).toBeInTheDocument();
    });
    
    // Add new device
    await user.click(screen.getByText('Add Device'));
    await user.type(screen.getByLabelText('Hostname'), 'new-router');
    await user.type(screen.getByLabelText('IP Address'), '192.168.1.100');
    await user.click(screen.getByText('Create Device'));
    
    // Verify device was added
    await waitFor(() => {
      expect(screen.getByText('new-router')).toBeInTheDocument();
    });
    
    // Edit device
    await user.click(screen.getByTestId('device-1-menu'));
    await user.click(screen.getByText('Edit'));
    await user.clear(screen.getByLabelText('Hostname'));
    await user.type(screen.getByLabelText('Hostname'), 'updated-router');
    await user.click(screen.getByText('Save Changes'));
    
    // Verify device was updated
    await waitFor(() => {
      expect(screen.getByText('updated-router')).toBeInTheDocument();
    });
  });
});
```

---

## 📋 Code Review Checklist

### Before Submitting

- [ ] **Code Quality**
  - [ ] No console.log statements in production code
  - [ ] All TypeScript errors resolved
  - [ ] ESLint warnings addressed
  - [ ] Code formatted with Prettier
  
- [ ] **Functionality**
  - [ ] Feature works as expected
  - [ ] Error states handled gracefully
  - [ ] Loading states implemented
  - [ ] Edge cases considered
  
- [ ] **Performance**
  - [ ] Components memoized where appropriate
  - [ ] Event handlers use useCallback
  - [ ] Expensive calculations use useMemo
  - [ ] No unnecessary re-renders
  
- [ ] **Testing**
  - [ ] Unit tests written and passing
  - [ ] Integration tests for complex workflows
  - [ ] Accessibility tested
  - [ ] Mobile responsiveness verified
  
- [ ] **Documentation**
  - [ ] Code commented where necessary
  - [ ] Props interfaces documented
  - [ ] README updated if needed
  - [ ] API changes documented

### Review Guidelines

```typescript
// Good patterns to look for

// 1. Proper error handling
const { data, error, isLoading } = useQuery(/* ... */);

if (error) return <ErrorState error={error} />;
if (isLoading) return <LoadingSpinner />;

// 2. Accessible components
<button
  aria-label="Delete item"
  aria-describedby="delete-description"
  onClick={handleDelete}
>
  <TrashIcon />
</button>

// 3. Proper TypeScript usage
interface ComponentProps {
  data: DataType[];
  onAction: (item: DataType) => void;
  variant?: 'primary' | 'secondary';
}

// 4. Memoization for performance
const expensiveValue = useMemo(() => 
  data.map(item => processItem(item)), 
  [data]
);

// 5. Clean component structure
const Component: React.FC<Props> = ({ data, onAction }) => {
  // Hooks first
  const [state, setState] = useState();
  const query = useQuery(/* ... */);
  
  // Event handlers
  const handleClick = useCallback(() => {
    onAction(data);
  }, [onAction, data]);
  
  // Early returns
  if (query.error) return <ErrorState />;
  
  // Main render
  return <div>{/* content */}</div>;
};
```

This developer guide provides comprehensive patterns and examples for common development tasks in the Resource Monitoring Frontend. It should help developers quickly understand the codebase conventions and implement features consistently.