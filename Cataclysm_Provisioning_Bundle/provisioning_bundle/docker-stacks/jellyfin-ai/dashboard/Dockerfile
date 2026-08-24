# Stage 1: Build the React application
FROM node:18-alpine AS build

WORKDIR /app

# Copy package.json and package-lock.json
COPY package*.json ./

# Install dependencies
RUN npm install

# Copy the rest of the application files
COPY . .

# Build the application
RUN npm run build

# Stage 2: Serve the static files with Nginx
# nginx-unprivileged rather than stock nginx + a bare USER directive.
# Stock nginx binds :80 (privileged) and writes to root-owned /var/cache/nginx
# and /var/run, so simply adding `USER nginx` produces an image that builds
# clean and then fails to start. The unprivileged variant is the same nginx
# built to run as uid 101 on :8080 with writable paths already relocated.
# NOTE: the port contract changes 80 -> 8080; any compose mapping must follow.
FROM nginxinc/nginx-unprivileged:1.25-alpine

# Copy the built application from the 'build' stage
COPY --from=build /app/build /usr/share/nginx/html

# Expose port 80
EXPOSE 8080

# Start Nginx
USER 101

CMD ["nginx", "-g", "daemon off;"]
