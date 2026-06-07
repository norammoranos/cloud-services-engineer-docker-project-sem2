FROM node:22-alpine3.22 AS deps

WORKDIR /app

COPY package.json package-lock.json ./
RUN npm ci --no-audit --no-fund \
    && npm cache clean --force

FROM deps AS dev

ENV NODE_ENV=development \
    NODE_OPTIONS=--openssl-legacy-provider \
    VUE_APP_MOMO_API_PATH=http://localhost:8082 \
    VUE_PUBLIC_PATH=/

COPY . .
RUN chown -R node:node /app

USER node

EXPOSE 3000

CMD ["npm", "run", "serve", "--", "--host", "0.0.0.0", "--port", "3000"]

FROM node:22-alpine3.22 AS build

WORKDIR /app

ARG MOMO_WEB_API_PATH=/api
ARG MOMO_WEB_PUBLIC_BASE=/

ENV NODE_ENV=production \
    NODE_OPTIONS=--openssl-legacy-provider \
    VUE_APP_MOMO_API_PATH=${MOMO_WEB_API_PATH} \
    VUE_PUBLIC_PATH=${MOMO_WEB_PUBLIC_BASE}

COPY --from=deps /app/node_modules ./node_modules
COPY . .
RUN npm run build

FROM nginxinc/nginx-unprivileged:1.29-alpine3.22

USER root
RUN apk upgrade --no-cache
USER 101

COPY nginx.conf /etc/nginx/conf.d/default.conf
COPY --from=build /app/dist /usr/share/nginx/html

EXPOSE 8080

HEALTHCHECK --interval=30s --timeout=3s --start-period=5s --retries=3 \
    CMD ["/bin/busybox", "wget", "-qO-", "http://127.0.0.1:8080/health"]
