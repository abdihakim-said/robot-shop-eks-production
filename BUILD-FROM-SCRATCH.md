# 🏗️ Building Robot Shop From Scratch

This guide shows how to build all Docker images from source code and deploy using your own container registry.

## 🎯 **Why Build From Scratch?**

- **Complete Control**: Modify source code and configurations
- **Security**: Know exactly what's in your images
- **Customization**: Add your own features or branding
- **Learning**: Understand the complete build process
- **Portfolio Value**: Shows end-to-end DevOps skills

## 📋 **Prerequisites**

- Docker installed and running
- Docker Hub account (or AWS ECR, Google GCR, etc.)
- kubectl and eksctl configured
- Helm 3.x installed

## 🔧 **Step 1: Build All Docker Images**

### **Build Script for All Services**

```bash
#!/bin/bash
# build-all-images.sh

# Your Docker registry (change this to your Docker Hub username or ECR URL)
REGISTRY="your-dockerhub-username"
VERSION="v1.0.0"

# Services to build
SERVICES=("web" "cart" "catalogue" "user" "payment" "shipping" "ratings" "dispatch" "mongo" "mysql")

echo "🏗️ Building all Robot Shop images..."

for service in "${SERVICES[@]}"; do
    echo "Building $service..."
    cd $service
    docker build -t $REGISTRY/rs-$service:$VERSION .
    docker build -t $REGISTRY/rs-$service:latest .
    cd ..
done

echo "✅ All images built successfully!"
```

### **Individual Service Builds**

```bash
# Web Frontend (AngularJS + Nginx)
cd web
docker build -t your-registry/rs-web:v1.0.0 .

# Cart Service (Node.js)
cd cart
docker build -t your-registry/rs-cart:v1.0.0 .

# Catalogue Service (Node.js)
cd catalogue
docker build -t your-registry/rs-catalogue:v1.0.0 .

# User Service (Node.js)
cd user
docker build -t your-registry/rs-user:v1.0.0 .

# Payment Service (Python)
cd payment
docker build -t your-registry/rs-payment:v1.0.0 .

# Shipping Service (Java)
cd shipping
docker build -t your-registry/rs-shipping:v1.0.0 .

# Ratings Service (PHP)
cd ratings
docker build -t your-registry/rs-ratings:v1.0.0 .

# Dispatch Service (Go)
cd dispatch
docker build -t your-registry/rs-dispatch:v1.0.0 .

# MongoDB with Data
cd mongo
docker build -t your-registry/rs-mongodb:v1.0.0 .

# MySQL with Schema
cd mysql
docker build -t your-registry/rs-mysql-db:v1.0.0 .
```

## 🚀 **Step 2: Push Images to Registry**

### **Push to Docker Hub**

```bash
#!/bin/bash
# push-all-images.sh

REGISTRY="your-dockerhub-username"
VERSION="v1.0.0"
SERVICES=("web" "cart" "catalogue" "user" "payment" "shipping" "ratings" "dispatch" "mongodb" "mysql-db")

# Login to Docker Hub
docker login

echo "🚀 Pushing all images to Docker Hub..."

for service in "${SERVICES[@]}"; do
    echo "Pushing rs-$service..."
    docker push $REGISTRY/rs-$service:$VERSION
    docker push $REGISTRY/rs-$service:latest
done

echo "✅ All images pushed successfully!"
```

### **Push to AWS ECR**

```bash
# Create ECR repositories
aws ecr create-repository --repository-name robot-shop/web --region us-east-1
aws ecr create-repository --repository-name robot-shop/cart --region us-east-1
# ... repeat for all services

# Get ECR login token
aws ecr get-login-password --region us-east-1 | docker login --username AWS --password-stdin <account-id>.dkr.ecr.us-east-1.amazonaws.com

# Tag and push
docker tag your-registry/rs-web:v1.0.0 <account-id>.dkr.ecr.us-east-1.amazonaws.com/robot-shop/web:v1.0.0
docker push <account-id>.dkr.ecr.us-east-1.amazonaws.com/robot-shop/web:v1.0.0
```

## ⚙️ **Step 3: Update Helm Configuration**

### **Update values.yaml**

```yaml
# EKS/helm/values.yaml
image:
  repo: your-dockerhub-username  # Change from 'robotshop'
  version: v1.0.0               # Change from 'latest'
  pullPolicy: IfNotPresent

# For ECR
# image:
#   repo: <account-id>.dkr.ecr.us-east-1.amazonaws.com/robot-shop
#   version: v1.0.0
#   pullPolicy: IfNotPresent
```

### **Update Individual Service Images (if needed)**

```yaml
# In deployment templates, you can override specific images:
# EKS/helm/templates/web-deployment.yaml
spec:
  template:
    spec:
      containers:
      - name: web
        image: your-registry/rs-web:v1.0.0  # Custom image
```

## 🔄 **Step 4: Deploy with Custom Images**

```bash
# Create EKS cluster (same as before)
eksctl create cluster \
  --name robot-shop-custom \
  --region us-east-1 \
  --nodegroup-name workers \
  --node-type t3.small \
  --nodes 3 \
  --managed

# Configure OIDC, EBS CSI, ALB Controller (same as before)
# ... (all the same setup steps)

# Deploy with your custom images
kubectl create namespace robot-shop
cd EKS/helm
helm install robot-shop --namespace robot-shop .
```

## 🛠️ **Step 5: Customization Examples**

### **Add Custom Branding to Web Service**

```dockerfile
# web/Dockerfile
FROM nginx:1.21.6

EXPOSE 8080

# Add custom environment variables
ENV CATALOGUE_HOST=catalogue \
    USER_HOST=user \
    CART_HOST=cart \
    SHIPPING_HOST=shipping \
    PAYMENT_HOST=payment \
    RATINGS_HOST=ratings \
    COMPANY_NAME="Your Company" \
    CUSTOM_THEME="blue"

COPY entrypoint.sh /root/
ENTRYPOINT ["/root/entrypoint.sh"]

COPY default.conf.template /etc/nginx/conf.d/default.conf.template
COPY static /usr/share/nginx/html

# Add custom CSS or branding files
COPY custom-theme.css /usr/share/nginx/html/css/
```

### **Add Custom Monitoring to Services**

```dockerfile
# cart/Dockerfile
FROM node:16-alpine

# Add monitoring dependencies
RUN npm install -g prom-client

WORKDIR /opt/app

COPY package.json .
RUN npm install

# Add custom monitoring endpoint
COPY monitoring.js .
COPY server.js .

EXPOSE 8080 9090

CMD ["node", "server.js"]
```

## 📊 **Step 6: Advanced Customizations**

### **Multi-Stage Builds for Optimization**

```dockerfile
# shipping/Dockerfile (Java service optimization)
# Build stage
FROM maven:3.8-openjdk-11 AS builder
WORKDIR /app
COPY pom.xml .
COPY src ./src
RUN mvn clean package -DskipTests

# Runtime stage
FROM openjdk:11-jre-slim
WORKDIR /app
COPY --from=builder /app/target/shipping-1.0.jar app.jar
EXPOSE 8080
CMD ["java", "-jar", "app.jar"]
```

### **Security Hardening**

```dockerfile
# user/Dockerfile (Security hardened)
FROM node:16-alpine

# Create non-root user
RUN addgroup -g 1001 -S nodejs && \
    adduser -S nodejs -u 1001

# Install security updates
RUN apk update && apk upgrade

WORKDIR /opt/app
COPY package*.json ./
RUN npm ci --only=production && npm cache clean --force

# Copy source code
COPY --chown=nodejs:nodejs . .

# Switch to non-root user
USER nodejs

EXPOSE 8080
CMD ["node", "server.js"]
```

## 🔍 **Step 7: Verification**

```bash
# Check your custom images are being used
kubectl describe pod -n robot-shop | grep Image:

# Should show your registry instead of robotshop/*
# Image: your-dockerhub-username/rs-web:v1.0.0
# Image: your-dockerhub-username/rs-cart:v1.0.0
```

## 💡 **Benefits of Building From Scratch**

### **Technical Benefits:**
- **Complete control** over image contents
- **Security scanning** of your own images
- **Custom optimizations** for your use case
- **Version control** of your image builds
- **Reduced external dependencies**

### **Portfolio Benefits:**
- **Shows end-to-end skills** - from source to production
- **Demonstrates Docker expertise** - multi-stage builds, optimization
- **Security awareness** - hardened images, non-root users
- **DevOps maturity** - complete CI/CD pipeline capability

### **Learning Benefits:**
- **Understand each service** - what it does and how it works
- **Docker best practices** - optimization, security, multi-stage builds
- **Registry management** - pushing, tagging, versioning
- **Troubleshooting skills** - debugging build issues

## 🚀 **Next Level: CI/CD Pipeline**

Once you have custom images, you can create a complete CI/CD pipeline:

```yaml
# .github/workflows/build-and-deploy.yml
name: Build and Deploy Robot Shop
on:
  push:
    branches: [main]

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v2
      - name: Build and push images
        run: |
          ./build-all-images.sh
          ./push-all-images.sh
      - name: Deploy to EKS
        run: |
          helm upgrade robot-shop ./EKS/helm
```

## 📈 **Cost Considerations**

### **Image Storage Costs:**
- **Docker Hub**: Free for public repos, $5/month for private
- **AWS ECR**: $0.10/GB/month storage + data transfer
- **Google GCR**: $0.026/GB/month storage

### **Build Time:**
- **Java services**: 5-10 minutes (Maven dependencies)
- **Node.js services**: 2-5 minutes (npm install)
- **Go services**: 1-2 minutes (fast compilation)
- **Total build time**: ~30-45 minutes for all services

## 🎯 **Recommendation**

For **learning and portfolio purposes**, I recommend:

1. **Start with existing images** (current approach) - Get the system working
2. **Build 2-3 services from scratch** - Show Docker skills
3. **Customize one service** - Add your own features
4. **Document the process** - Show your understanding

This approach demonstrates both **practical deployment skills** and **deep technical understanding** without overwhelming complexity.
