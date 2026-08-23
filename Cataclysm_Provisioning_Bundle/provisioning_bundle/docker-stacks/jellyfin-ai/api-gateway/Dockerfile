
FROM node:20-alpine

WORKDIR /app

# Install dependencies
COPY package*.json ./
RUN npm ci --only=production

# Copy application code
COPY . .

# Create necessary directories
RUN mkdir -p /app/logs

EXPOSE 3000


# Install curl for healthcheck
RUN apk add --no-cache curl

# Health check
HEALTHCHECK --interval=30s --timeout=5s --start-period=15s --retries=3 \
    CMD curl -f http://localhost:3000/health || exit 1
# Non-root. Alpine ships an unused uid 1000 `node`; PMOVES pins 65532 to match
# the rest of the fleet so compose `user:` overrides are uniform.
RUN addgroup -g 65532 -S pmoves     && adduser -u 65532 -S -G pmoves -h /app -s /sbin/nologin pmoves     && chown -R 65532:65532 /app
USER pmoves:pmoves

CMD ["npm", "start"]
