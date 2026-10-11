# Multi-Node Kubernetes Cluster with Multiple Applications and ReplicaSets

## 📌 Storyboard & Overview
This exercise simulates deploying an e-commerce platform with two microservices:
- **Product Catalog (`AppA`)**: Provides information about available products (2 replicas).
- **Shopping Cart (`AppB`)**: Manages user shopping carts (3 replicas).

To ensure high availability and fault tolerance, each replica of both services is distributed across multiple nodes using Kubernetes **Pod Anti-Affinity** (`podAntiAffinity`). Even if a worker node crashes or goes offline, the applications remain accessible.

---

## 📂 Project Structure

```text
Exs10/
├── Dockerfile.product              # Dockerfile for Product Catalog
├── Dockerfile.shopping             # Dockerfile for Shopping Cart
├── product_catalog.py             # Flask App: Product Catalog API
├── shopping_cart.py               # Flask App: Shopping Cart API
├── product_catalog_deployment.yaml # K8s Deployment with 2 replicas & podAntiAffinity
├── shopping_cart_deployment.yaml   # K8s Deployment with 3 replicas & podAntiAffinity
├── product_catalog_service.yaml    # NodePort Service for Product Catalog
├── shopping_cart_service.yaml      # NodePort Service for Shopping Cart
├── requirements.txt               # Python dependencies
└── README.md                       # Comprehensive guide and execution steps
```

---

## 🚀 Step-by-Step Execution Guide

### Step 0: Clean Up Any Existing Minikube Cluster
```bash
minikube stop
minikube delete
```

---

### Step 1: Set Up Minikube with Multiple Nodes

#### 1a. Start 3-node cluster
```bash
minikube start --nodes 3 -p devops-multinode --force
```

#### 1b. Enable the Registry Addon
In a multi-node cluster, images need to be shared across nodes. Enable the registry addon:
```bash
minikube -p devops-multinode addons enable registry
```

---

### Step 2 & 3: Application Code
- **Product Catalog** (`product_catalog.py`): Runs on port `80`, serves `GET /products`.
- **Shopping Cart** (`shopping_cart.py`): Runs on port `80`, serves `GET /cart` and `POST /cart`.

---

### Step 4: Build Docker Images Locally
```bash
docker build -t product-catalog:latest -f Dockerfile.product .
docker build -t shopping-cart:latest -f Dockerfile.shopping .
```

Verify images:
```bash
docker images
```

---

### Step 5: Load Images into Minikube Multi-Node Cluster
Load the built images into Minikube's multi-node environment:
```bash
minikube -p devops-multinode image load product-catalog:latest
minikube -p devops-multinode image load shopping-cart:latest
```

Verify inside minikube:
```bash
minikube -p devops-multinode ssh -- docker images
```

---

### Step 6: Deploy Applications to Kubernetes
Apply the deployment manifests:
```bash
kubectl apply -f product_catalog_deployment.yaml
kubectl apply -f shopping_cart_deployment.yaml
```

> **Note on Pod Anti-Affinity**: Both deployments specify `topologyKey: "kubernetes.io/hostname"` under `requiredDuringSchedulingIgnoredDuringExecution`. This guarantees that the scheduler will never schedule two pods of the same application onto the same node.

---

### Step 7: Expose Services via NodePort
```bash
kubectl apply -f product_catalog_service.yaml
kubectl apply -f shopping_cart_service.yaml
```

---

### Step 8: Verify Deployment and Node Distribution
Check pod status and node distribution:
```bash
kubectl get pods -o wide
```

Expected output showing pods distributed across `devops-multinode`, `devops-multinode-m02`, and `devops-multinode-m03`:
```text
NAME                               READY   STATUS    RESTARTS   AGE   IP           NODE                   NOMINATED NODE   READINESS GATES
product-catalog-5fdfc7f5fb-2jlnv   1/1     Running   0          29s   10.244.1.3   devops-multinode-m02   <none>           <none>
product-catalog-5fdfc7f5fb-7v556   1/1     Running   0          29s   10.244.2.4   devops-multinode-m03   <none>           <none>
shopping-cart-788c48675b-7l4lx     1/1     Running   0          25s   10.244.0.4   devops-multinode       <none>           <none>
shopping-cart-788c48675b-d48w2     1/1     Running   0          25s   10.244.1.4   devops-multinode-m02   <none>           <none>
shopping-cart-788c48675b-hml9r     1/1     Running   0          25s   10.244.2.5   devops-multinode-m03   <none>           <none>
```

---

### Step 9: Access Services via Minikube Tunnel / Service URL

#### 9a. Access Product Catalog Service:
```bash
minikube -p devops-multinode service product-catalog-service
```

#### 9b. Access Shopping Cart Service:
```bash
minikube -p devops-multinode service shopping-cart-service
```

Alternatively, get direct URLs without blocking:
```bash
minikube -p devops-multinode service product-catalog-service --url
minikube -p devops-multinode service shopping-cart-service --url
```

---

### Step 10: Test the Endpoints using cURL

Replace `<PRODUCT_CATALOG_URL>` and `<SHOPPING_CART_URL>` with your tunnel/NodePort URLs (e.g. `http://127.0.0.1:35855` and `http://127.0.0.1:38975`):

#### 1. Test Product Catalog Service:
```bash
curl http://127.0.0.1:35855/products
```
Expected output:
```json
[
  {"id": 1, "name": "Laptop", "price": 1200},
  {"id": 2, "name": "Phone", "price": 800},
  {"id": 3, "name": "Headphones", "price": 150}
]
```

#### 2. Test Shopping Cart Service (GET):
```bash
curl http://127.0.0.1:38975/cart
```
Expected output:
```json
[]
```

#### 3. Test Shopping Cart Service (POST):
```bash
curl -X POST http://127.0.0.1:38975/cart \
  -H "Content-Type: application/json" \
  -d '{"id": 1, "name": "Laptop", "quantity": 1}'
```
Expected output:
```json
[
  {"id": 1, "name": "Laptop", "quantity": 1}
]
```
