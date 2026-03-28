FROM node:22-alpine AS builder

# Install pnpm
RUN corepack enable && corepack prepare pnpm@10.32.1 --activate

WORKDIR /app

# Copy configuration files first for better layer caching
COPY package.json pnpm-lock.yaml pnpm-workspace.yaml ./
COPY tsconfig.base.json tsconfig.json ./
COPY vitest.config.ts ./



# Copy all workspace packages (needed for workspace dependencies resolution)
COPY packages/core packages/core
COPY packages/protobufs packages/protobufs
COPY packages/transport-http packages/transport-http
COPY packages/transport-web-bluetooth packages/transport-web-bluetooth
COPY packages/transport-web-serial packages/transport-web-serial
COPY packages/ui packages/ui
COPY packages/web/package.json packages/web/
COPY packages/web/tsconfig.json packages/web/

# Install root dependencies
RUN pnpm install --frozen-lockfile

# Copy remaining source code
COPY . .

# Build the web application
WORKDIR /app/packages/web
RUN pnpm run build

# Production stage - minimal nginx image
FROM nginx:1.29-alpine

# Create non-root user for security
RUN addgroup -g 1000 -S appgroup && \
    adduser -u 1000 -S appuser -G appgroup

# Copy custom nginx configuration
COPY --chown=appuser:appgroup default.conf /etc/nginx/conf.d/default.conf

# Copy built artifacts from builder stage
COPY --from=builder --chown=appuser:appgroup /app/packages/web/dist /usr/share/nginx/html

# Set proper permissions
RUN chown -R appuser:appgroup /usr/share/nginx/html /var/cache/nginx /var/log/nginx /etc/nginx/conf.d /run

# Expose HTTP port
EXPOSE 8080

# Health check for容器 orchestration
HEALTHCHECK --interval=30s --timeout=10s --start-period=5s --retries=3 \
    CMD wget --no-verbose --tries=1 --spider http://localhost:8080/health || exit 1

# Switch to non-root user
USER appuser

# Start nginx in foreground
CMD ["nginx", "-g", "daemon off;"]
