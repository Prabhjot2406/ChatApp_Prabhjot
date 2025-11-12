# Stage 1: Node.js build environment
FROM node:18-alpine AS first-env
WORKDIR /app
COPY package*.json ./
RUN npm ci --only=production

# Stage 2: Build React application
FROM node:18-alpine AS build-stage
WORKDIR /app
COPY package*.json ./
RUN npm ci
COPY . .
RUN npm run build

# Stage 3: Nginx runtime server
FROM nginx:alpine AS runtime
COPY --from=build-stage /app/build /usr/share/nginx/html
COPY nginx.conf /etc/nginx/nginx.conf
EXPOSE 80
CMD ["nginx", "-g", "daemon off;"]


