# Deployment

## Dockerfile (멀티스테이지 빌드)

```dockerfile
# Stage 1: 빌드
FROM node:22-alpine AS builder
WORKDIR /app

COPY package*.json ./
RUN npm ci --include=dev

COPY . .
RUN npm run build

# Stage 2: 프로덕션
FROM node:22-alpine AS production
WORKDIR /app

ENV NODE_ENV=production

# 보안: 비루트 사용자
RUN addgroup -g 1001 -S nodejs && adduser -S nodejs -u 1001

COPY package*.json ./
RUN npm ci --only=production && npm cache clean --force

COPY --from=builder --chown=nodejs:nodejs /app/dist ./dist

USER nodejs
EXPOSE 3000

HEALTHCHECK --interval=30s --timeout=3s --start-period=10s \
  CMD wget -qO- http://localhost:3000/health || exit 1

CMD ["node", "dist/server.js"]
```

## docker-compose.yml (로컬 개발)

```yaml
version: '3.9'
services:
  api:
    build: .
    ports: ["3000:3000"]
    environment:
      NODE_ENV: development
      DATABASE_URL: postgresql://postgres:password@db:5432/appdb
      REDIS_URL: redis://redis:6379
    volumes:
      - ./src:/app/src  # 핫 리로드
    depends_on:
      db:    { condition: service_healthy }
      redis: { condition: service_healthy }

  db:
    image: postgres:16-alpine
    environment:
      POSTGRES_DB:       appdb
      POSTGRES_USER:     postgres
      POSTGRES_PASSWORD: password
    ports: ["5432:5432"]
    volumes: [pgdata:/var/lib/postgresql/data]
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U postgres"]
      interval: 5s
      timeout:  5s
      retries:  5

  redis:
    image: redis:7-alpine
    ports: ["6379:6379"]
    healthcheck:
      test:     ["CMD", "redis-cli", "ping"]
      interval: 5s

volumes:
  pgdata:
```

## Kubernetes 배포

```yaml
# deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: api-server
spec:
  replicas: 3
  selector:
    matchLabels: { app: api-server }
  template:
    metadata:
      labels: { app: api-server }
    spec:
      containers:
        - name: api-server
          image: registry.example.com/api-server:latest
          ports: [{ containerPort: 3000 }]
          env:
            - name:  DATABASE_URL
              valueFrom:
                secretKeyRef: { name: app-secrets, key: database-url }
          resources:
            requests: { cpu: 100m, memory: 128Mi }
            limits:   { cpu: 500m, memory: 512Mi }
          livenessProbe:
            httpGet: { path: /live, port: 3000 }
            initialDelaySeconds: 10
          readinessProbe:
            httpGet: { path: /ready, port: 3000 }
            initialDelaySeconds: 5
```

## 환경별 설정

```typescript
const configs = {
  development: { logLevel: 'debug', dbPoolMax: 5 },
  test:        { logLevel: 'warn',  dbPoolMax: 2 },
  production:  { logLevel: 'info',  dbPoolMax: 20 }
}

export const config = configs[process.env.NODE_ENV ?? 'development']
```

## 체크리스트
- [ ] 멀티스테이지 빌드로 이미지 크기 최소화
- [ ] 비루트 사용자로 실행
- [ ] Healthcheck 엔드포인트 구현
- [ ] Graceful shutdown (SIGTERM) 처리
- [ ] 시크릿은 환경변수 또는 Secrets Manager
- [ ] 리소스 limits/requests 설정
