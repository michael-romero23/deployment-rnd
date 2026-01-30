# Stage 1: Build
FROM node:20-alpine AS builder
WORKDIR /app

# Files are in the same directory, so we use ./
COPY package*.json ./
RUN npm install
COPY . .
RUN npm run build

# Stage 2: Production
FROM node:20-alpine AS runner
WORKDIR /app
ENV NODE_ENV=production

COPY --from=builder /app/package*.json ./
COPY --from=builder /app/dist ./dist
RUN npm install --only=production

EXPOSE 3000
CMD ["node", "dist/main"]