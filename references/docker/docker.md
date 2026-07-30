# Docker & CI Reference

## Dockerfile.api

```dockerfile
# docker/Dockerfile.api
FROM node:20-alpine AS base
WORKDIR /app

# Install dependencies
COPY apps/api/package.json ./apps/api/
COPY packages/shared/package.json ./packages/shared/
COPY package.json turbo.json ./
RUN npm install --workspace=apps/api --workspace=packages/shared

# Copy source
COPY apps/api ./apps/api
COPY packages/shared ./packages/shared

EXPOSE 4000
CMD ["node", "apps/api/app.js"]
```

---

## Dockerfile.web

```dockerfile
# docker/Dockerfile.web
FROM node:20-alpine AS builder
WORKDIR /app

COPY apps/web/package.json ./apps/web/
COPY packages/shared/package.json ./packages/shared/
COPY package.json turbo.json ./
RUN npm install --workspace=apps/web --workspace=packages/shared

COPY apps/web ./apps/web
COPY packages/shared ./packages/shared

RUN npm run build --workspace=apps/web

# Serve with Nginx
FROM nginx:alpine
COPY --from=builder /app/apps/web/dist /usr/share/nginx/html
COPY docker/nginx.conf /etc/nginx/conf.d/default.conf
EXPOSE 80
```

---

## nginx.conf

```nginx
server {
  listen 80;

  location / {
    root /usr/share/nginx/html;
    index index.html;
    try_files $uri $uri/ /index.html;  # SPA fallback
  }

  location /api/ {
    proxy_pass http://api:4000;
    proxy_set_header Host $host;
    proxy_set_header X-Real-IP $remote_addr;
  }
}
```

This proxies `/api/` calls from the frontend to the Express container — no
CORS config needed, both are on the same Docker network.

---

## docker-compose.yml

```yaml
version: "3.9"

services:
  mongo:
    image: mongo:7
    restart: unless-stopped
    volumes:
      - mongo_data:/data/db
    environment:
      MONGO_INITDB_DATABASE: warehouse

  api:
    build:
      context: ..
      dockerfile: docker/Dockerfile.api
    restart: unless-stopped
    ports:
      - "4000:4000"
    environment:
      PORT: 4000
      MONGO_URI: mongodb://mongo:27017/warehouse
      JWT_SECRET: ${JWT_SECRET}
      JWT_REFRESH_SECRET: ${JWT_REFRESH_SECRET}
      NODE_ENV: production
    depends_on:
      - mongo

  web:
    build:
      context: ..
      dockerfile: docker/Dockerfile.web
    restart: unless-stopped
    ports:
      - "3000:80"
    depends_on:
      - api

volumes:
  mongo_data:
```

---

## CI Pipeline

Using GitHub Actions (or any CI runner). Create `.github/workflows/ci.yml`:

```yaml
name: CI

on:
  push:
    branches: [main, develop]
  pull_request:
    branches: [main]

jobs:
  ci:
    runs-on: ubuntu-latest

    steps:
      - uses: actions/checkout@v4

      - name: Setup Node
        uses: actions/setup-node@v4
        with:
          node-version: 20
          cache: npm

      - name: Install dependencies
        run: npm install

      - name: Lint
        run: npx turbo run lint

      - name: Test
        run: npx turbo run test
        # mongodb-memory-server handles the test DB — no external Mongo needed

      - name: Build
        run: npx turbo run build

      - name: Docker build check
        run: docker compose -f docker/docker-compose.yml build
```

---

## LAN Deployment — Quick Start

On the LAN server (Linux recommended):

```bash
# 1. Install Docker + Docker Compose
# 2. Clone the repo
git clone <repo> warehouse-app && cd warehouse-app

# 3. Set secrets
cp .env.example .env
# Edit .env: set JWT_SECRET, JWT_REFRESH_SECRET

# 4. Start
docker compose -f docker/docker-compose.yml up -d

# 5. Check logs
docker compose logs -f
```

App will be available at:
- Frontend: `http://<server-ip>:3000`
- API: `http://<server-ip>:4000`

Document the server's LAN IP in `DEPLOYMENT.md` so staff can bookmark it.

---

## Updating the app

```bash
git pull
docker compose -f docker/docker-compose.yml up -d --build
```

MongoDB data is preserved in the `mongo_data` volume across rebuilds.
