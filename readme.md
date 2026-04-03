Rode: docker.io/gitea/act_runner:latest

```bash
docker-compose up -d
```

Crie o projeto projeto-teste no Gitea

Acessar o gitea ir no repositorio

- configurações
- Ações
- Runners
- Criar novo Runner
- copiar o token
- ir no docker-compose
- colocar o token no GITEA_RUNNER_REGISTRATION_TOKEN
- salvar o docker-compose
- docker-compose restart

no terminal:

```bash
npx create-next-app@latest projeto-teste --yes
cd projeto-teste
code .
git remote add origin http://localhost:3000/gittea/projeto-gitea.git
git add .
git commit -m "first commit"
git push -u origin main
```

crie o arquivo .github/workflows/ci.yml

```yaml
name: CI

on:
  push:
    branches:
      - main
      - master
  pull_request:

jobs:
  build-and-docker:
    runs-on: ubuntu-latest

    steps:
      - name: Checkout
        uses: https://gitea.com/actions/checkout@v4
        continue-on-error: true

      - name: Setup Node.js
        uses: https://gitea.com/actions/setup-node@v4
        with:
          node-version: 22
          cache: npm

      - name: Install dependencies
        run: |
          if [ -f package-lock.json ]; then npm ci; else npm install; fi

      - name: Lint
        run: npm run lint

      - name: Build Next.js
        run: npm run build

      - name: Read app version
        run: |
          APP_VERSION=$(node -p "require('./package.json').version")
          SHA="${GITEA_SHA:-${GITHUB_SHA}}"
          SHORT_SHA=$(echo "${SHA}" | cut -c1-7)
          RUN_NUMBER="${GITEA_RUN_NUMBER:-${GITHUB_RUN_NUMBER:-0}}"
          BUILD_TAG="build-${RUN_NUMBER}-${SHORT_SHA}"
          echo "APP_VERSION=${APP_VERSION}" >> "$GITHUB_ENV"
          echo "BUILD_TAG=${BUILD_TAG}" >> "$GITHUB_ENV"

      - name: Build Docker image (latest + version + build)
        run: |
          docker build \
            --build-arg APP_VERSION="${APP_VERSION}" \
            -t projeto-teste:latest \
            -t projeto-teste:${APP_VERSION} \
            -t projeto-teste:${BUILD_TAG} \
            .

      - name: Prepare release tag
        run: |
          EVENT_NAME="${GITEA_EVENT_NAME:-${GITHUB_EVENT_NAME:-${GITHUB_EVENT_NAME}}}"
          if [ "${EVENT_NAME}" != "push" ]; then
            echo "Not a push event, skipping release prep."
            exit 0
          fi
          RELEASE_TAG="v${APP_VERSION}-${BUILD_TAG}"
          echo "RELEASE_TAG=${RELEASE_TAG}" >> "$GITHUB_ENV"

      - name: Create and push git tag
        run: |
          EVENT_NAME="${GITEA_EVENT_NAME:-${GITHUB_EVENT_NAME:-${GITHUB_EVENT_NAME}}}"
          if [ "${EVENT_NAME}" != "push" ]; then
            echo "Not a push event, skipping tag creation."
            exit 0
          fi

          git config user.name "gitea-actions"
          git config user.email "gitea-actions@local"
          git fetch --tags
          if git rev-parse "${RELEASE_TAG}" >/dev/null 2>&1; then
            echo "Tag ${RELEASE_TAG} already exists, skipping tag creation."
            exit 0
          fi
          git tag -a "${RELEASE_TAG}" -m "Release ${RELEASE_TAG}"
          git push origin "${RELEASE_TAG}"

      - name: Create Gitea release
        env:
          GITEA_TOKEN: ${{ secrets.GITEA_TOKEN }}
          TOKEN_SECRET: ${{ secrets.TOKEN }}
          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
        run: |
          EVENT_NAME="${GITEA_EVENT_NAME:-${GITHUB_EVENT_NAME:-${GITHUB_EVENT_NAME}}}"
          if [ "${EVENT_NAME}" != "push" ]; then
            echo "Not a push event, skipping release creation."
            exit 0
          fi

          TOKEN="${GITEA_TOKEN:-${TOKEN_SECRET:-${GITHUB_TOKEN}}}"
          if [ -z "${TOKEN}" ]; then
            echo "Missing token. Set GITEA_TOKEN (recommended), TOKEN or GITHUB_TOKEN in repository secrets."
            exit 1
          fi

          SERVER_URL="${GITEA_SERVER_URL:-${GITHUB_SERVER_URL}}"
          REPOSITORY="${GITEA_REPOSITORY:-${GITHUB_REPOSITORY}}"
          SHA="${GITEA_SHA:-${GITHUB_SHA}}"
          RELEASE_URL="${SERVER_URL}/api/v1/repos/${REPOSITORY}/releases"
          TAG_URL="${RELEASE_URL}/tags/${RELEASE_TAG}"

          STATUS=$(curl -s -o /dev/null -w "%{http_code}" \
            -H "Authorization: token ${TOKEN}" \
            -H "Accept: application/json" \
            "${TAG_URL}")

          if [ "${STATUS}" = "200" ]; then
            echo "Release ${RELEASE_TAG} already exists, skipping creation."
            exit 0
          fi

          curl -sS -X POST "${RELEASE_URL}" \
            -H "Authorization: token ${TOKEN}" \
            -H "Accept: application/json" \
            -H "Content-Type: application/json" \
            -d "{\"tag_name\":\"${RELEASE_TAG}\",\"target_commitish\":\"${SHA}\",\"name\":\"${RELEASE_TAG}\",\"body\":\"Release automatica gerada pelo CI.\",\"draft\":false,\"prerelease\":false}"
```

Altere o arquivo next.config.js para:

```javascript
import type { NextConfig } from "next";

const nextConfig: NextConfig = {
  output: "standalone",
};

export default nextConfig;
```

Crie o arquivo Dockerfile:

```Dockerfile
# syntax=docker/dockerfile:1

FROM node:22-alpine AS base
WORKDIR /app
ENV NEXT_TELEMETRY_DISABLED=1

FROM base AS deps
COPY package.json package-lock.json* ./
RUN npm ci

FROM base AS builder
COPY --from=deps /app/node_modules ./node_modules
COPY . .
RUN npm run build

FROM node:22-alpine AS runner
WORKDIR /app

ARG APP_VERSION=dev
LABEL org.opencontainers.image.title="projeto-teste"
LABEL org.opencontainers.image.version="$APP_VERSION"

ENV NODE_ENV=production
ENV NEXT_TELEMETRY_DISABLED=1
ENV HOSTNAME=0.0.0.0
ENV PORT=3000

RUN addgroup -S nodejs && adduser -S nextjs -G nodejs

COPY --from=builder /app/public ./public
COPY --from=builder /app/.next/standalone ./
COPY --from=builder /app/.next/static ./.next/static

USER nextjs
EXPOSE 3000

CMD ["node", "server.js"]
```

Crie o arquivo .dockerignore:

```
.git
.gitea
.next
node_modules
npm-debug.log*
README.md
```
