# System Architecture Documentation

## 📐 Architecture Overview

The Resource Monitoring Frontend follows a modern React architecture with clear separation of concerns and scalable patterns.

## 🏗️ Architectural Principles

### 1. Layered Architecture

```mermaid
graph TB
    subgraph "Presentation Layer"
        PL1[Pages] --> PL2[Components] 
        PL2 --> PL3[UI Elements]
    end
    
    subgraph "Business Logic Layer"
        BL1[Custom Hooks] --> BL2[State Management]
        BL2 --> BL3[Data Transformations]
    end
    
    subgraph "Data Access Layer"
        DAL1[React Query] --> DAL2[API Services]
        DAL2 --> DAL3[HTTP Client]
    end
    
    subgraph "External Services"
        ES1[LiveNX API] --> ES2[MongoDB]
        ES1 --> ES3[ClickHouse]
    end
    
    PL1 --> BL1
    BL1 --> DAL1
    DAL2 --> ES1
```

### 2. Component Architecture

```mermaid
graph TD
    subgraph "Smart Components (Pages)"
        SC1[NetworkDevices] --> SC2[DeviceDetails]
        SC2 --> SC3[DeviceMetrics]
    end
    
    subgraph "Container Components"
        CC1[TableWrapper] --> CC2[FilterSection]
        CC2 --> CC3[ContentSwitcher]
    end
    
    subgraph "Presentational Components"
        PC1[DonutChart] --> PC2[Search]
        PC2 --> PC3[Dropdown]
        PC3 --> PC4[Container]
    end
    
    SC1 --> CC1
    SC2 --> CC2
    SC3 --> CC3
    CC1 --> PC1
    CC2 --> PC2
    CC3 --> PC3
```

### 3. Data Flow Architecture

```mermaid
sequenceDiagram
    participant U as User
    participant C as Component
    participant H as Hook
    participant RQ as React Query
    participant API as API Layer
    participant S as Server
    
    U->>C: User Action
    C->>H: State Update
    H->>RQ: Query Trigger
    
    alt Cache Hit
        RQ-->>H: Cached Data
    else Cache Miss
        RQ->>API: API Call
        API->>S: HTTP Request
        S-->>API: Response
        API-->>RQ: Processed Data
        RQ-->>H: Fresh Data
    end
    
    H-->>C: Updated State
    C-->>U: UI Update
```

## 🧩 Component Design Patterns

### 1. Composition Pattern

Components are designed to be composable and reusable:

```typescript
// Container component that accepts any children
<Container isVertical spacing={{ top: 16 }}>
  <BreadcrumbNavigation />
  <FilterSection />
  <TableWrapper />
</Container>
```

### 2. Render Props Pattern

For complex components with shared logic:

```typescript
<ResizableContainer>
  {({ width }) => (
    <TypeWidgetGrid 
      widgets={widgets}
      visibleCount={calculateVisibleWidgets(width)}
    />
  )}
</ResizableContainer>
```

### 3. Custom Hook Pattern

Business logic separated from presentation:

```typescript
// Custom hook for pagination
function usePagination() {
  const [searchParams, setSearchParams] = useSearchParams();
  
  const page = useMemo(() => 
    parseInt(searchParams.get('page') || '1'), 
    [searchParams]
  );
  
  const setPage = useCallback((newPage: number) => {
    // Update URL parameters
  }, []);
  
  return { page, setPage };
}

// Usage in component
function DevicesTable() {
  const { page, setPage } = usePagination();
  // Component only handles rendering
}
```

## 📊 State Management Architecture

### 1. State Distribution

```mermaid
graph TD
    subgraph "Global State"
        GS1[Theme Context] --> GS2[Query Cache]
    end
    
    subgraph "URL State"
        US1[Filters] --> US2[Pagination]
        US2 --> US3[Sorting] 
        US3 --> US4[Navigation]
    end
    
    subgraph "Local State"
        LS1[Form Inputs] --> LS2[UI State]
        LS2 --> LS3[Temporary Data]
    end
    
    subgraph "Server State"
        SS1[Device Data] --> SS2[Metrics]
        SS2 --> SS3[Alerts]
    end
    
    GS2 --> SS1
    US1 --> SS1
    LS1 --> US1
```

### 2. URL State Management

Most application state lives in URL parameters for persistence and shareability:

```typescript
// URL structure examples
/resource-monitoring/devices?page=2&pageSize=50&sort=hostName&direction=asc&search=router&vendor.contains=cisco

/resource-monitoring/devices/123?tab=interfaces

/resource-monitoring/devices/123/metrics?view=historical&from=2023-01-01&to=2023-01-31
```

### 3. React Query State Management

```mermaid
graph LR
    subgraph "Query States"
        QS1[Loading] --> QS2[Success]
        QS2 --> QS3[Error]
        QS3 --> QS4[Refetching]
        QS4 --> QS2
    end
    
    subgraph "Cache Management"
        CM1[Background Updates] --> CM2[Stale While Revalidate]
        CM2 --> CM3[Optimistic Updates]
    end
    
    subgraph "Cache Layers"
        CL1[Memory Cache] --> CL2[Persistent Cache]
        CL2 --> CL3[Network Requests]
    end
```

## 🔄 Data Flow Patterns

### 1. Unidirectional Data Flow

```mermaid
flowchart TD
    A[User Interaction] --> B[Event Handler]
    B --> C[State Update]
    C --> D[Component Re-render]
    D --> E[Child Component Props]
    E --> F[UI Update]
    
    C --> G[Side Effects]
    G --> H[API Calls]
    H --> I[State Update]
    I --> D
```

### 2. Filter System Data Flow

```mermaid
sequenceDiagram
    participant U as User
    participant FD as FilterDropdown
    participant FS as FilterSection
    participant URL as URL State
    participant API as API Service
    participant T as Table
    
    U->>FD: Select filter value
    FD->>FS: Update filter state
    FS->>URL: Update search params
    URL->>API: Trigger new query
    API-->>T: Updated data
    T-->>U: Refreshed table
```

### 3. Pagination Data Flow

```mermaid
flowchart LR
    A[Page Change] --> B[Update URL Params]
    B --> C[Trigger API Query]
    C --> D[Fetch New Page Data]
    D --> E[Update Table]
    E --> F[Update Pagination UI]
    
    B --> G[Maintain Other State]
    G --> H[Preserve Filters]
    H --> I[Preserve Sort]
```

## 🎯 Performance Architecture

### 1. Caching Strategy

```mermaid
graph TB
    subgraph "Cache Hierarchy"
        L1[L1: Component State<br/>Immediate Access]
        L2[L2: React Query Cache<br/>5min - Static Data]
        L3[L3: Browser Cache<br/>HTTP Caching]
        L4[L4: Server Cache<br/>Database/API Layer]
    end
    
    L1 --> L2
    L2 --> L3
    L3 --> L4
    
    subgraph "Cache Policies"
        CP1[Static: 5+ minutes] --> CP2[Device Details, Lists]
        CP3[Dynamic: 1-5 minutes] --> CP4[Statistics, Alerts]
        CP5[Real-time: < 1 minute] --> CP6[Metrics, Performance]
    end
```

### 2. Bundle Architecture

```mermaid
graph LR
    subgraph "Code Splitting"
        CS1[Main Bundle] --> CS2[Page Chunks]
        CS2 --> CS3[Component Chunks]
        CS3 --> CS4[Vendor Chunks]
    end
    
    subgraph "Loading Strategy"
        LS1[Critical Path] --> LS2[Above Fold]
        LS2 --> LS3[Below Fold]
        LS3 --> LS4[On Demand]
    end
    
    CS1 --> LS1
    CS2 --> LS2
    CS3 --> LS3
    CS4 --> LS4
```

### 3. Rendering Optimization

```typescript
// Component memoization patterns
const ExpensiveComponent = React.memo(({ data, onAction }) => {
  const processedData = useMemo(() => 
    data.map(item => expensiveTransformation(item)), 
    [data]
  );
  
  const handleClick = useCallback((item) => {
    onAction(item);
  }, [onAction]);
  
  return (
    <div>
      {processedData.map(item => (
        <Item key={item.id} data={item} onClick={handleClick} />
      ))}
    </div>
  );
});

// Virtualization for large lists
const VirtualizedTable = ({ data }) => {
  const [visibleItems, setVisibleItems] = useState([]);
  
  useEffect(() => {
    // Calculate visible items based on scroll position
    const visible = calculateVisibleItems(data, scrollPosition);
    setVisibleItems(visible);
  }, [data, scrollPosition]);
  
  return (
    <div className="virtualized-container">
      {visibleItems.map(item => <TableRow key={item.id} data={item} />)}
    </div>
  );
};
```

## 🔌 Integration Architecture

### 1. API Integration Layers

```mermaid
graph TB
    subgraph "API Layer Architecture"
        AL1[API Client<br/>Axios + Interceptors] --> AL2[Service Layer<br/>Endpoint Definitions]
        AL2 --> AL3[Query Layer<br/>React Query Hooks]
        AL3 --> AL4[Component Layer<br/>UI Components]
    end
    
    subgraph "Error Handling"
        EH1[Global Error Handler] --> EH2[Component Error Boundaries]
        EH2 --> EH3[User-Friendly Messages]
    end
    
    subgraph "Authentication Flow"
        AF1[Request Interceptor] --> AF2[Token Validation]
        AF2 --> AF3[Auto-Redirect on 401]
    end
    
    AL1 --> EH1
    AL1 --> AF1
```

### 2. External Service Integration

```mermaid
sequenceDiagram
    participant FE as Frontend
    participant LNX as LiveNX Server
    participant MDB as MongoDB
    participant CH as ClickHouse
    
    FE->>LNX: API Request
    
    alt Device Data
        LNX->>MDB: Query Devices
        MDB-->>LNX: Device Records
    else Metrics Data
        LNX->>CH: Query Metrics
        CH-->>LNX: Time Series Data
    end
    
    LNX-->>FE: JSON Response
    FE->>FE: Update UI
```

### 3. Development Environment Integration

```mermaid
graph LR
    subgraph "Local Development"
        LD1[Frontend:5173] --> LD2[SSH Tunnels]
        LD2 --> LD3[Remote Services]
    end
    
    subgraph "Remote Environment"
        RE1[LiveNX:8093] --> RE2[MongoDB:27017]
        RE1 --> RE3[ClickHouse:4041]
    end
    
    LD2 -.-> RE1
    LD2 -.-> RE2
    LD2 -.-> RE3
```

## 🔧 Build & Deployment Architecture

### 1. Build Pipeline

```mermaid
flowchart TD
    A[Source Code] --> B[TypeScript Compilation]
    B --> C[Vite Bundling]
    C --> D[Asset Optimization]
    D --> E[Bundle Analysis]
    E --> F[Output Generation]
    
    F --> G[Development Build]
    F --> H[Production Build]
    
    G --> I[Dev Server]
    H --> J[Static Files]
    
    J --> K[LiveNX Integration]
    J --> L[Standalone Deployment]
```

### 2. Module Architecture

```mermaid
graph TB
    subgraph "Source Modules"
        SM1[Pages] --> SM2[Components]
        SM2 --> SM3[Services]
        SM3 --> SM4[Utilities]
        SM4 --> SM5[Types]
    end
    
    subgraph "Build Output"
        BO1[Main Bundle] --> BO2[Vendor Bundle]
        BO2 --> BO3[Async Chunks]
        BO3 --> BO4[CSS Bundle]
        BO4 --> BO5[Assets]
    end
    
    SM1 --> BO1
    SM2 --> BO2
    SM3 --> BO3
    SM4 --> BO4
    SM5 --> BO5
```

## 🧪 Testing Architecture

### 1. Testing Strategy

```mermaid
graph TB
    subgraph "Testing Pyramid"
        TP1[Unit Tests<br/>70%] --> TP2[Integration Tests<br/>20%]
        TP2 --> TP3[E2E Tests<br/>10%]
    end
    
    subgraph "Test Types"
        TT1[Component Tests] --> TT2[Hook Tests]
        TT2 --> TT3[Service Tests]
        TT3 --> TT4[Utility Tests]
    end
    
    subgraph "Testing Tools"
        TL1[Vitest] --> TL2[Testing Library]
        TL2 --> TL3[MSW Mocking]
        TL3 --> TL4[User Events]
    end
```

### 2. Test Environment Architecture

```typescript
// Test setup with providers
export const TestProviders = ({ children }: { children: React.ReactNode }) => {
  const queryClient = new QueryClient({
    defaultOptions: {
      queries: { retry: false, staleTime: 0 },
      mutations: { retry: false },
    },
  });

  return (
    <QueryClientProvider client={queryClient}>
      <BrowserRouter>
        <ThemeProvider>
          {children}
        </ThemeProvider>
      </BrowserRouter>
    </QueryClientProvider>
  );
};

// Mock service worker setup
export const server = setupServer(
  http.get('/api/devices', () => HttpResponse.json(mockDevices)),
  http.get('/api/metrics', () => HttpResponse.json(mockMetrics)),
);
```

## 📈 Scalability Considerations

### 1. Horizontal Scaling

```mermaid
graph LR
    subgraph "Frontend Scaling"
        FS1[CDN Distribution] --> FS2[Multiple Instances]
        FS2 --> FS3[Load Balancing]
    end
    
    subgraph "State Management"
        SM1[Stateless Components] --> SM2[URL-based State]
        SM2 --> SM3[Server State Cache]
    end
    
    subgraph "Performance"
        P1[Code Splitting] --> P2[Lazy Loading]
        P2 --> P3[Memoization]
    end
```

### 2. Feature Scaling

```mermaid
graph TD
    subgraph "Current Architecture"
        CA1[Monolithic Frontend] --> CA2[Shared Components]
        CA2 --> CA3[Common Services]
    end
    
    subgraph "Future Architecture"
        FA1[Micro-frontends] --> FA2[Independent Deployment]
        FA2 --> FA3[Shared Design System]
        FA3 --> FA4[Event-driven Communication]
    end
    
    CA1 -.-> FA1
    CA2 -.-> FA3
    CA3 -.-> FA4
```

## 🔒 Security Architecture

### 1. Authentication Flow

```mermaid
sequenceDiagram
    participant U as User
    participant FE as Frontend
    participant LNX as LiveNX Server
    participant AUTH as Auth Service
    
    U->>FE: Access Application
    FE->>LNX: API Request
    
    alt Authenticated
        LNX-->>FE: Data Response
    else Unauthenticated
        LNX-->>FE: 401 Unauthorized
        FE->>AUTH: Redirect to Login
        AUTH-->>U: Login Page
    end
```

### 2. Data Security

```mermaid
graph TB
    subgraph "Security Layers"
        SL1[HTTPS/TLS] --> SL2[Session Management]
        SL2 --> SL3[CSRF Protection]
        SL3 --> SL4[Input Validation]
    end
    
    subgraph "Data Protection"
        DP1[Client-side Validation] --> DP2[Server-side Validation]
        DP2 --> DP3[SQL Injection Prevention]
        DP3 --> DP4[XSS Protection]
    end
    
    SL4 --> DP1
```

This architecture documentation provides a comprehensive view of how the Resource Monitoring Frontend is structured, how data flows through the system, and how different components interact with each other. It serves as a foundation for understanding the system's design decisions and can guide future development efforts.