# Dockerfile for Bun application
FROM oven/bun:1.0.30 AS builder

WORKDIR /app

# Copy package files
COPY package*.json ./
COPY bun.lock ./

# Install dependencies
RUN bun install

# Copy source code
COPY . .

# Generate Prisma client
RUN bunx prisma generate

# Build the application
RUN bun run build

# Production image
FROM oven/bun:1.0.30

WORKDIR /app

# Copy built assets from builder
COPY --from=builder /app/dist ./dist
COPY --from=builder /app/node_modules ./node_modules
COPY --from=builder /app/package.json ./package.json
COPY --from=builder /app/prisma ./prisma

# Expose port
EXPOSE 3000

# Start the application
ENTRYPOINT ["bun", "run", "start"]