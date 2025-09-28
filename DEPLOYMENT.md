# Deployment Documentation

## 🚀 Overview

This document covers the complete deployment strategy for the Resource Monitoring Frontend, including build processes, deployment environments, monitoring, and maintenance procedures.

## 🏗️ Build Process

### Build Configuration

The project uses Vite for building and bundling:

```typescript
// vite.config.ts
import { defineConfig } from 'vite';
import react from '@vitejs/plugin-react-swc';
import path from 'path';

export default defineConfig({
  plugins: [react()],
  
  // Build configuration
  build: {
    outDir: 'dist',
    sourcemap: process.env.NODE_ENV === 'development',
    minify: 'esbuild',
    target: 'esnext',
    
    // Bundle splitting for better caching
    rollupOptions: {
      output: {
        manualChunks: {
          // Vendor chunk for third-party libraries
          vendor: [
            'react',
            'react-dom',
            'react-router-dom',
            '@tanstack/react-query',
          ],
          // UI library chunk
          ui: ['@bluecat/pelagos'],
          // Chart library chunk
          charts: ['d3', 'chart.js'],
        },
      },
    },
    
    // Asset optimization
    assetsInlineLimit: 4096, // 4KB
    chunkSizeWarningLimit: 1000, // 1MB
  },
  
  // Development server
  server: {
    port: 5173,
    host: true,
    strictPort: true,
    cors: true,
  },
  
  // Preview server (for testing builds)
  preview: {
    port: 4173,
    host: true,
  },
  
  // Path resolution
  resolve: {
    alias: {
      '@': path.resolve(__dirname, './src'),
      '@components': path.resolve(__dirname, './src/components'),
      '@pages': path.resolve(__dirname, './src/pages'),
      '@services': path.resolve(__dirname, './src/services'),
      '@hooks': path.resolve(__dirname, './src/hooks'),
      '@helpers': path.resolve(__dirname, './src/helpers'),
      '@context': path.resolve(__dirname, './src/context'),
      '@styles': path.resolve(__dirname, './src/styles'),
    },
  },
  
  // Environment variable prefix
  envPrefix: ['VITE_', 'REACT_APP_'],
});
```

### Environment Configuration

```bash
# .env.example
# API Configuration
VITE_API_BASE_URL=http://localhost:8093
VITE_API_TIMEOUT=30000

# Feature Flags
VITE_ENABLE_DEBUG_MODE=false
VITE_ENABLE_PERFORMANCE_MONITORING=true
VITE_ENABLE_ERROR_REPORTING=true

# Build Configuration
VITE_BUILD_VERSION=1.0.0
VITE_BUILD_TIMESTAMP=
VITE_GIT_COMMIT_HASH=

# Monitoring
VITE_SENTRY_DSN=
VITE_ANALYTICS_ID=

# Development
VITE_MOCK_API=false
VITE_SHOW_DEV_TOOLS=false
```

### Build Scripts

```json
{
  "scripts": {
    // Development
    "dev": "vite --host",
    "dev:debug": "vite --host --debug",
    
    // Building
    "build": "npm run type-check && vite build",
    "build:analyze": "npm run build && npx vite-bundle-analyzer dist/stats.html",
    "build:dev": "vite build --mode development",
    "build:staging": "vite build --mode staging",
    "build:production": "vite build --mode production",
    
    // Type checking
    "type-check": "tsc --noEmit",
    "type-check:watch": "tsc --noEmit --watch",
    
    // Preview
    "preview": "vite preview",
    "preview:dist": "npm run build && npm run preview",
    
    // Quality checks
    "lint": "eslint . --ext ts,tsx --report-unused-disable-directives --max-warnings 0",
    "lint:fix": "npm run lint -- --fix",
    "test": "vitest",
    "test:coverage": "vitest run --coverage",
    "test:e2e": "playwright test",
    
    // Deployment utilities
    "clean": "rimraf dist coverage .temp",
    "pre-deploy": "npm run clean && npm run lint && npm run type-check && npm run test:coverage && npm run build",
    "deploy:staging": "npm run pre-deploy && npm run deploy:staging:run",
    "deploy:production": "npm run pre-deploy && npm run deploy:production:run"
  }
}
```

---

## 🏭 Deployment Environments

### Development Environment

```yaml
# docker-compose.dev.yml
version: '3.8'

services:
  frontend-dev:
    build:
      context: .
      dockerfile: Dockerfile.dev
    ports:
      - "5173:5173"
    volumes:
      - .:/app
      - /app/node_modules
    environment:
      - NODE_ENV=development
      - VITE_API_BASE_URL=http://localhost:8093
      - VITE_ENABLE_DEBUG_MODE=true
      - VITE_MOCK_API=true
    depends_on:
      - api-server
    networks:
      - dev-network

  api-server:
    image: resource-monitoring-api:dev
    ports:
      - "8093:8093"
    environment:
      - NODE_ENV=development
      - DB_HOST=mongodb
    depends_on:
      - mongodb
    networks:
      - dev-network

  mongodb:
    image: mongo:7
    ports:
      - "27017:27017"
    volumes:
      - mongo-data:/data/db
    networks:
      - dev-network

networks:
  dev-network:
    driver: bridge

volumes:
  mongo-data:
```

```dockerfile
# Dockerfile.dev
FROM node:23-alpine

WORKDIR /app

# Install dependencies
COPY package*.json ./
RUN npm ci

# Copy source code
COPY . .

# Expose port
EXPOSE 5173

# Start development server
CMD ["npm", "run", "dev"]
```

### Staging Environment

```yaml
# docker-compose.staging.yml
version: '3.8'

services:
  frontend-staging:
    build:
      context: .
      dockerfile: Dockerfile.staging
    ports:
      - "80:80"
      - "443:443"
    environment:
      - NODE_ENV=staging
      - VITE_API_BASE_URL=https://api-staging.livenx.com
      - VITE_ENABLE_DEBUG_MODE=false
      - VITE_ENABLE_PERFORMANCE_MONITORING=true
    volumes:
      - ./nginx.staging.conf:/etc/nginx/nginx.conf
      - ./ssl:/etc/nginx/ssl
    depends_on:
      - api-staging
    networks:
      - staging-network
    restart: unless-stopped

networks:
  staging-network:
    driver: bridge
```

```dockerfile
# Dockerfile.staging
# Build stage
FROM node:23-alpine as builder

WORKDIR /app

# Install dependencies
COPY package*.json ./
RUN npm ci --only=production

# Copy source and build
COPY . .
RUN npm run build:staging

# Production stage
FROM nginx:alpine

# Copy built files
COPY --from=builder /app/dist /usr/share/nginx/html

# Copy nginx configuration
COPY nginx.staging.conf /etc/nginx/nginx.conf

# Health check
HEALTHCHECK --interval=30s --timeout=3s --start-period=5s --retries=3 \
  CMD curl -f http://localhost/ || exit 1

EXPOSE 80 443

CMD ["nginx", "-g", "daemon off;"]
```

### Production Environment

```yaml
# docker-compose.prod.yml
version: '3.8'

services:
  frontend-prod:
    image: resource-monitoring-frontend:${VERSION}
    ports:
      - "80:80"
      - "443:443"
    environment:
      - NODE_ENV=production
      - VITE_API_BASE_URL=https://api.livenx.com
      - VITE_ENABLE_DEBUG_MODE=false
      - VITE_ENABLE_PERFORMANCE_MONITORING=true
      - VITE_ENABLE_ERROR_REPORTING=true
    volumes:
      - ./nginx.prod.conf:/etc/nginx/nginx.conf
      - ./ssl:/etc/nginx/ssl
      - logs:/var/log/nginx
    networks:
      - prod-network
    restart: always
    deploy:
      replicas: 2
      update_config:
        parallelism: 1
        delay: 10s
      restart_policy:
        condition: on-failure

networks:
  prod-network:
    external: true

volumes:
  logs:
    external: true
```

```dockerfile
# Dockerfile.prod
# Build stage
FROM node:23-alpine as builder

WORKDIR /app

# Install dependencies
COPY package*.json ./
RUN npm ci --only=production && npm cache clean --force

# Copy source and build
COPY . .
RUN npm run build:production

# Production stage
FROM nginx:alpine

# Install curl for health checks
RUN apk add --no-cache curl

# Copy built files
COPY --from=builder /app/dist /usr/share/nginx/html

# Copy nginx configuration
COPY nginx.prod.conf /etc/nginx/nginx.conf

# Create non-root user
RUN addgroup -g 1001 -S nginx && \
    adduser -S -D -H -u 1001 -h /var/cache/nginx -s /sbin/nologin -G nginx -g nginx nginx

# Set proper permissions
RUN chown -R nginx:nginx /usr/share/nginx/html && \
    chown -R nginx:nginx /var/cache/nginx && \
    chown -R nginx:nginx /var/log/nginx

# Health check
HEALTHCHECK --interval=30s --timeout=10s --start-period=5s --retries=3 \
  CMD curl -f http://localhost/health || exit 1

# Security settings
USER nginx

EXPOSE 80 443

CMD ["nginx", "-g", "daemon off;"]
```

---

## 🌐 Nginx Configuration

### Production Nginx Configuration

```nginx
# nginx.prod.conf
user nginx;
worker_processes auto;
error_log /var/log/nginx/error.log warn;
pid /var/run/nginx.pid;

events {
    worker_connections 1024;
    use epoll;
    multi_accept on;
}

http {
    include /etc/nginx/mime.types;
    default_type application/octet-stream;

    # Logging
    log_format main '$remote_addr - $remote_user [$time_local] "$request" '
                   '$status $body_bytes_sent "$http_referer" '
                   '"$http_user_agent" "$http_x_forwarded_for"';
    
    access_log /var/log/nginx/access.log main;

    # Performance settings
    sendfile on;
    tcp_nopush on;
    tcp_nodelay on;
    keepalive_timeout 65;
    types_hash_max_size 2048;
    server_tokens off;

    # Gzip compression
    gzip on;
    gzip_vary on;
    gzip_min_length 10240;
    gzip_proxied expired no-cache no-store private must-revalidate max-age=0;
    gzip_types
        application/javascript
        application/json
        application/xml
        text/css
        text/javascript
        text/plain
        text/xml
        image/svg+xml;

    # Security headers
    add_header X-Frame-Options DENY always;
    add_header X-Content-Type-Options nosniff always;
    add_header X-XSS-Protection "1; mode=block" always;
    add_header Referrer-Policy "strict-origin-when-cross-origin" always;
    add_header Content-Security-Policy "default-src 'self'; script-src 'self' 'unsafe-inline'; style-src 'self' 'unsafe-inline'; img-src 'self' data: https:; font-src 'self' data:; connect-src 'self' https://api.livenx.com wss:; frame-ancestors 'none';" always;
    add_header Strict-Transport-Security "max-age=31536000; includeSubDomains" always;

    # Rate limiting
    limit_req_zone $binary_remote_addr zone=api:10m rate=10r/s;
    limit_req_zone $binary_remote_addr zone=static:10m rate=100r/s;

    # Upstream for load balancing (if needed)
    upstream backend {
        server api.livenx.com:443;
        keepalive 32;
    }

    server {
        listen 80;
        server_name _;
        return 301 https://$host$request_uri;
    }

    server {
        listen 443 ssl http2;
        server_name livenx.com www.livenx.com;
        root /usr/share/nginx/html;
        index index.html;

        # SSL configuration
        ssl_certificate /etc/nginx/ssl/cert.pem;
        ssl_certificate_key /etc/nginx/ssl/key.pem;
        ssl_session_cache shared:SSL:50m;
        ssl_session_timeout 1d;
        ssl_session_tickets off;
        ssl_protocols TLSv1.2 TLSv1.3;
        ssl_ciphers ECDHE-ECDSA-AES128-GCM-SHA256:ECDHE-RSA-AES128-GCM-SHA256:ECDHE-ECDSA-AES256-GCM-SHA384:ECDHE-RSA-AES256-GCM-SHA384;
        ssl_prefer_server_ciphers off;

        # Static file serving with caching
        location ~* \.(js|css|png|jpg|jpeg|gif|ico|svg|woff|woff2|ttf|eot)$ {
            expires 1y;
            add_header Cache-Control "public, immutable";
            add_header X-Content-Type-Options nosniff;
            limit_req zone=static burst=50 nodelay;
        }

        # API proxy
        location /api/ {
            limit_req zone=api burst=20 nodelay;
            
            proxy_pass https://backend/;
            proxy_http_version 1.1;
            proxy_set_header Upgrade $http_upgrade;
            proxy_set_header Connection 'upgrade';
            proxy_set_header Host $host;
            proxy_set_header X-Real-IP $remote_addr;
            proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
            proxy_set_header X-Forwarded-Proto $scheme;
            proxy_cache_bypass $http_upgrade;
            proxy_connect_timeout 30s;
            proxy_send_timeout 30s;
            proxy_read_timeout 30s;
        }

        # Health check endpoint
        location /health {
            access_log off;
            return 200 "healthy\n";
            add_header Content-Type text/plain;
        }

        # React Router support - serve index.html for all routes
        location / {
            try_files $uri $uri/ /index.html;
            
            # Cache control for HTML files
            location = /index.html {
                expires 0;
                add_header Cache-Control "no-cache, no-store, must-revalidate";
                add_header Pragma "no-cache";
            }
        }

        # Security: Block access to hidden files
        location ~ /\. {
            deny all;
        }

        # Block access to source files
        location ~* \.(map|ts|tsx)$ {
            deny all;
        }

        error_page 404 /index.html;
        error_page 500 502 503 504 /50x.html;
        location = /50x.html {
            root /usr/share/nginx/html;
        }
    }
}
```

---

## 🚀 CI/CD Pipeline

### GitHub Actions Workflow

```yaml
# .github/workflows/deploy.yml
name: Deploy to Production

on:
  push:
    branches: [main]
    tags: ['v*']
  pull_request:
    branches: [main]

env:
  REGISTRY: ghcr.io
  IMAGE_NAME: ${{ github.repository }}
  NODE_VERSION: '23'

jobs:
  # Quality checks
  test:
    runs-on: ubuntu-latest
    
    steps:
      - name: Checkout
        uses: actions/checkout@v4

      - name: Setup Node.js
        uses: actions/setup-node@v4
        with:
          node-version: ${{ env.NODE_VERSION }}
          cache: 'npm'

      - name: Install dependencies
        run: npm ci

      - name: Lint
        run: npm run lint

      - name: Type check
        run: npm run type-check

      - name: Run tests
        run: npm run test:coverage

      - name: Upload coverage
        uses: codecov/codecov-action@v3
        with:
          file: ./coverage/lcov.info

  # Security scanning
  security:
    runs-on: ubuntu-latest
    needs: test
    
    steps:
      - name: Checkout
        uses: actions/checkout@v4

      - name: Run Snyk to check for vulnerabilities
        uses: snyk/actions/node@master
        env:
          SNYK_TOKEN: ${{ secrets.SNYK_TOKEN }}
        with:
          args: --severity-threshold=high

      - name: Run npm audit
        run: npm audit --audit-level=moderate

  # Build and push Docker image
  build:
    runs-on: ubuntu-latest
    needs: [test, security]
    if: github.event_name == 'push' || github.event_name == 'workflow_dispatch'
    
    outputs:
      image: ${{ steps.image.outputs.image }}
      digest: ${{ steps.build.outputs.digest }}
    
    steps:
      - name: Checkout
        uses: actions/checkout@v4

      - name: Setup Docker Buildx
        uses: docker/setup-buildx-action@v3

      - name: Log in to Container Registry
        uses: docker/login-action@v3
        with:
          registry: ${{ env.REGISTRY }}
          username: ${{ github.actor }}
          password: ${{ secrets.GITHUB_TOKEN }}

      - name: Extract metadata
        id: meta
        uses: docker/metadata-action@v5
        with:
          images: ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}
          tags: |
            type=ref,event=branch
            type=ref,event=pr
            type=semver,pattern={{version}}
            type=semver,pattern={{major}}.{{minor}}
            type=sha,prefix={{branch}}-

      - name: Build and push Docker image
        id: build
        uses: docker/build-push-action@v5
        with:
          context: .
          file: Dockerfile.prod
          platforms: linux/amd64,linux/arm64
          push: true
          tags: ${{ steps.meta.outputs.tags }}
          labels: ${{ steps.meta.outputs.labels }}
          cache-from: type=gha
          cache-to: type=gha,mode=max
          build-args: |
            VERSION=${{ github.sha }}
            BUILD_DATE=${{ steps.meta.outputs.date }}

      - name: Output image
        id: image
        run: echo "image=${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}:${{ github.sha }}" >> "$GITHUB_OUTPUT"

  # Deploy to staging
  deploy-staging:
    runs-on: ubuntu-latest
    needs: build
    if: github.ref == 'refs/heads/main'
    environment: staging
    
    steps:
      - name: Deploy to staging
        uses: appleboy/ssh-action@v1.0.3
        with:
          host: ${{ secrets.STAGING_HOST }}
          username: ${{ secrets.STAGING_USERNAME }}
          key: ${{ secrets.STAGING_SSH_KEY }}
          script: |
            cd /opt/resource-monitoring-frontend
            docker-compose -f docker-compose.staging.yml pull
            docker-compose -f docker-compose.staging.yml down
            docker-compose -f docker-compose.staging.yml up -d
            docker system prune -f

      - name: Run health check
        run: |
          sleep 30
          curl -f https://staging.livenx.com/health || exit 1

  # Deploy to production
  deploy-production:
    runs-on: ubuntu-latest
    needs: [build, deploy-staging]
    if: startsWith(github.ref, 'refs/tags/v')
    environment: production
    
    steps:
      - name: Deploy to production
        uses: appleboy/ssh-action@v1.0.3
        with:
          host: ${{ secrets.PRODUCTION_HOST }}
          username: ${{ secrets.PRODUCTION_USERNAME }}
          key: ${{ secrets.PRODUCTION_SSH_KEY }}
          script: |
            cd /opt/resource-monitoring-frontend
            
            # Create backup
            docker tag resource-monitoring-frontend:current resource-monitoring-frontend:backup-$(date +%Y%m%d-%H%M%S)
            
            # Deploy new version
            export VERSION=${{ github.ref_name }}
            docker-compose -f docker-compose.prod.yml pull
            docker-compose -f docker-compose.prod.yml up -d --no-deps frontend-prod
            
            # Wait for health check
            sleep 30
            if ! curl -f https://livenx.com/health; then
              echo "Health check failed, rolling back"
              docker-compose -f docker-compose.prod.yml rollback
              exit 1
            fi
            
            # Clean up old images
            docker system prune -f

      - name: Verify deployment
        run: |
          sleep 10
          curl -f https://livenx.com/health
          curl -f https://livenx.com/ | grep -q "Resource Monitoring"

      - name: Notify deployment success
        uses: 8398a7/action-slack@v3
        with:
          status: success
          channel: '#deployments'
          text: 'Resource Monitoring Frontend deployed successfully to production'
        env:
          SLACK_WEBHOOK_URL: ${{ secrets.SLACK_WEBHOOK }}

  # Rollback job (manual trigger)
  rollback:
    runs-on: ubuntu-latest
    if: github.event_name == 'workflow_dispatch'
    environment: production
    
    steps:
      - name: Rollback production deployment
        uses: appleboy/ssh-action@v1.0.3
        with:
          host: ${{ secrets.PRODUCTION_HOST }}
          username: ${{ secrets.PRODUCTION_USERNAME }}
          key: ${{ secrets.PRODUCTION_SSH_KEY }}
          script: |
            cd /opt/resource-monitoring-frontend
            docker-compose -f docker-compose.prod.yml down
            docker tag resource-monitoring-frontend:backup-latest resource-monitoring-frontend:current
            docker-compose -f docker-compose.prod.yml up -d
```

---

## 📊 Monitoring and Logging

### Application Monitoring

```typescript
// src/services/monitoring.ts
class MonitoringService {
  private static instance: MonitoringService;
  
  private constructor() {
    this.initSentry();
    this.initPerformanceMonitoring();
    this.initErrorBoundary();
  }
  
  static getInstance(): MonitoringService {
    if (!MonitoringService.instance) {
      MonitoringService.instance = new MonitoringService();
    }
    return MonitoringService.instance;
  }
  
  private initSentry() {
    if (import.meta.env.VITE_SENTRY_DSN) {
      Sentry.init({
        dsn: import.meta.env.VITE_SENTRY_DSN,
        environment: import.meta.env.NODE_ENV,
        release: import.meta.env.VITE_BUILD_VERSION,
        integrations: [
          new Sentry.BrowserTracing({
            routingInstrumentation: Sentry.reactRouterV6Instrumentation(
              React.useEffect,
              useLocation,
              useNavigationType,
              createRoutesFromChildren,
              matchRoutes
            ),
          }),
        ],
        tracesSampleRate: 0.1,
        beforeSend(event) {
          // Filter out development errors
          if (import.meta.env.NODE_ENV === 'development') {
            return null;
          }
          return event;
        },
      });
    }
  }
  
  private initPerformanceMonitoring() {
    // Web Vitals
    if ('requestIdleCallback' in window) {
      requestIdleCallback(() => {
        import('web-vitals').then(({ getCLS, getFID, getFCP, getLCP, getTTFB }) => {
          getCLS(this.sendMetric);
          getFID(this.sendMetric);
          getFCP(this.sendMetric);
          getLCP(this.sendMetric);
          getTTFB(this.sendMetric);
        });
      });
    }
    
    // Custom performance metrics
    this.trackNavigationTiming();
    this.trackResourceTiming();
  }
  
  private sendMetric = (metric: any) => {
    // Send to analytics service
    if (import.meta.env.VITE_ANALYTICS_ENDPOINT) {
      fetch(import.meta.env.VITE_ANALYTICS_ENDPOINT, {
        method: 'POST',
        headers: { 'Content-Type': 'application/json' },
        body: JSON.stringify({
          type: 'web-vital',
          name: metric.name,
          value: metric.value,
          rating: metric.rating,
          timestamp: Date.now(),
          url: window.location.href,
        }),
      }).catch(console.warn);
    }
  };
  
  private trackNavigationTiming() {
    if ('navigation' in performance) {
      const navigation = performance.getEntriesByType('navigation')[0] as PerformanceNavigationTiming;
      
      const metrics = {
        domContentLoaded: navigation.domContentLoadedEventEnd - navigation.domContentLoadedEventStart,
        loadComplete: navigation.loadEventEnd - navigation.loadEventStart,
        domInteractive: navigation.domInteractive - navigation.navigationStart,
        ttfb: navigation.responseStart - navigation.requestStart,
      };
      
      Object.entries(metrics).forEach(([name, value]) => {
        this.sendMetric({ name, value, rating: 'info' });
      });
    }
  }
  
  trackError(error: Error, context?: Record<string, any>) {
    console.error('Application Error:', error, context);
    
    if (import.meta.env.VITE_ENABLE_ERROR_REPORTING === 'true') {
      Sentry.captureException(error, {
        tags: {
          component: context?.component || 'unknown',
        },
        extra: context,
      });
    }
  }
  
  trackEvent(name: string, properties?: Record<string, any>) {
    // Custom event tracking
    if (import.meta.env.VITE_ANALYTICS_ENDPOINT) {
      fetch(import.meta.env.VITE_ANALYTICS_ENDPOINT, {
        method: 'POST',
        headers: { 'Content-Type': 'application/json' },
        body: JSON.stringify({
          type: 'custom-event',
          name,
          properties,
          timestamp: Date.now(),
          url: window.location.href,
        }),
      }).catch(console.warn);
    }
  }
}

export default MonitoringService.getInstance();
```

### Health Check Endpoint

```typescript
// src/services/healthCheck.ts
export interface HealthStatus {
  status: 'healthy' | 'degraded' | 'unhealthy';
  timestamp: number;
  version: string;
  services: {
    api: 'up' | 'down';
    database: 'up' | 'down';
  };
  metrics: {
    responseTime: number;
    errorRate: number;
    uptime: number;
  };
}

export async function performHealthCheck(): Promise<HealthStatus> {
  const startTime = Date.now();
  
  try {
    // Check API connectivity
    const apiResponse = await fetch('/api/health', {
      timeout: 5000,
    });
    
    const apiStatus = apiResponse.ok ? 'up' : 'down';
    const responseTime = Date.now() - startTime;
    
    return {
      status: apiStatus === 'up' ? 'healthy' : 'degraded',
      timestamp: Date.now(),
      version: import.meta.env.VITE_BUILD_VERSION || 'unknown',
      services: {
        api: apiStatus,
        database: 'up', // Assuming database is up if API is up
      },
      metrics: {
        responseTime,
        errorRate: 0, // Calculate based on recent error logs
        uptime: performance.now(),
      },
    };
  } catch (error) {
    return {
      status: 'unhealthy',
      timestamp: Date.now(),
      version: import.meta.env.VITE_BUILD_VERSION || 'unknown',
      services: {
        api: 'down',
        database: 'unknown',
      },
      metrics: {
        responseTime: Date.now() - startTime,
        errorRate: 100,
        uptime: performance.now(),
      },
    };
  }
}
```

---

## 🛡️ Security Configuration

### Content Security Policy

```typescript
// Security headers in index.html
const securityHeaders = {
  'Content-Security-Policy': [
    "default-src 'self'",
    "script-src 'self' 'unsafe-inline' 'unsafe-eval'", // React requires unsafe-inline
    "style-src 'self' 'unsafe-inline'",
    "img-src 'self' data: https:",
    "font-src 'self' data:",
    "connect-src 'self' https://api.livenx.com wss:",
    "frame-ancestors 'none'",
    "base-uri 'self'",
    "form-action 'self'",
  ].join('; '),
  'X-Frame-Options': 'DENY',
  'X-Content-Type-Options': 'nosniff',
  'X-XSS-Protection': '1; mode=block',
  'Referrer-Policy': 'strict-origin-when-cross-origin',
  'Permissions-Policy': [
    'camera=()',
    'microphone=()',
    'geolocation=()',
    'payment=()',
  ].join(', '),
};
```

### Environment Security

```bash
# Production environment variables
NODE_ENV=production
VITE_API_BASE_URL=https://api.livenx.com
VITE_ENABLE_DEBUG_MODE=false
VITE_ENABLE_PERFORMANCE_MONITORING=true
VITE_ENABLE_ERROR_REPORTING=true
VITE_BUILD_VERSION=${GITHUB_SHA}
VITE_SENTRY_DSN=${SENTRY_DSN}

# Security settings
DOCKER_CONTENT_TRUST=1
DOCKER_BUILDKIT=1
```

---

## 🚨 Disaster Recovery

### Backup Strategy

```bash
#!/bin/bash
# scripts/backup.sh

# Backup Docker images
docker save resource-monitoring-frontend:latest | gzip > backup-$(date +%Y%m%d).tar.gz

# Backup configuration
tar -czf config-backup-$(date +%Y%m%d).tar.gz \
  docker-compose.prod.yml \
  nginx.prod.conf \
  .env.production

# Upload to S3
aws s3 cp backup-$(date +%Y%m%d).tar.gz s3://backups/resource-monitoring/
aws s3 cp config-backup-$(date +%Y%m%d).tar.gz s3://backups/resource-monitoring/

# Cleanup old backups (keep last 30 days)
find . -name "backup-*.tar.gz" -mtime +30 -delete
find . -name "config-backup-*.tar.gz" -mtime +30 -delete
```

### Rollback Procedure

```bash
#!/bin/bash
# scripts/rollback.sh

BACKUP_VERSION=${1:-"latest"}

echo "Rolling back to version: $BACKUP_VERSION"

# Stop current containers
docker-compose -f docker-compose.prod.yml down

# Load backup image
if [ -f "backup-$BACKUP_VERSION.tar.gz" ]; then
  gunzip -c "backup-$BACKUP_VERSION.tar.gz" | docker load
else
  echo "Backup file not found, pulling from registry"
  docker pull "resource-monitoring-frontend:$BACKUP_VERSION"
fi

# Update docker-compose to use backup version
sed -i "s|image: resource-monitoring-frontend:.*|image: resource-monitoring-frontend:$BACKUP_VERSION|" docker-compose.prod.yml

# Start containers
docker-compose -f docker-compose.prod.yml up -d

# Wait for health check
sleep 30
if curl -f https://livenx.com/health; then
  echo "Rollback successful"
else
  echo "Rollback failed, manual intervention required"
  exit 1
fi
```

---

## 📈 Performance Optimization

### Build Optimization

```typescript
// vite.config.ts - Production optimizations
export default defineConfig({
  build: {
    rollupOptions: {
      output: {
        manualChunks: (id) => {
          // Vendor chunks
          if (id.includes('node_modules')) {
            if (id.includes('react')) return 'react';
            if (id.includes('@tanstack')) return 'query';
            if (id.includes('d3') || id.includes('chart')) return 'charts';
            return 'vendor';
          }
          
          // Feature-based chunks
          if (id.includes('/pages/')) {
            const page = id.split('/pages/')[1].split('/')[0];
            return `page-${page}`;
          }
          
          if (id.includes('/components/')) {
            if (id.includes('charts')) return 'chart-components';
            if (id.includes('Table')) return 'table-components';
            return 'components';
          }
        },
      },
    },
    
    // Minification settings
    minify: 'esbuild',
    target: 'esnext',
    
    // Source maps for production debugging
    sourcemap: 'hidden',
    
    // Asset optimization
    assetsInlineLimit: 4096,
    chunkSizeWarningLimit: 1000,
  },
  
  // Optimize dependencies
  optimizeDeps: {
    include: [
      'react',
      'react-dom',
      'react-router-dom',
      '@tanstack/react-query',
    ],
    exclude: ['@bluecat/pelagos'],
  },
});
```

### Runtime Performance

```typescript
// src/utils/performance.ts
export function measurePerformance<T>(
  fn: () => Promise<T> | T,
  label: string
): Promise<T> {
  return new Promise(async (resolve, reject) => {
    const start = performance.now();
    
    try {
      const result = await fn();
      const end = performance.now();
      const duration = end - start;
      
      if (duration > 100) { // Log slow operations
        console.warn(`Slow operation detected: ${label} took ${duration.toFixed(2)}ms`);
      }
      
      // Send to monitoring
      if (import.meta.env.VITE_ENABLE_PERFORMANCE_MONITORING === 'true') {
        monitoringService.trackEvent('performance', {
          operation: label,
          duration,
          timestamp: Date.now(),
        });
      }
      
      resolve(result);
    } catch (error) {
      reject(error);
    }
  });
}

// Lazy loading utility
export function createLazyComponent<T extends React.ComponentType<any>>(
  importFn: () => Promise<{ default: T }>,
  fallback?: React.ComponentType
) {
  return React.lazy(() =>
    measurePerformance(importFn, 'lazy-component-load')
  );
}
```

This comprehensive deployment documentation provides everything needed to deploy, monitor, and maintain the Resource Monitoring Frontend in production environments, ensuring reliable, secure, and performant operations.