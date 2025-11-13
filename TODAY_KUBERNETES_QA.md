# Today's Kubernetes Q&A Session

## Questions Asked Today with Exact Answers

---

### Q: "everytinmr I restart my laptop the port forwarding gets terminated I have to manually update it is there any way It stays on?"

**A:** Port forwarding is a temporary connection that terminates when you restart. Here are permanent solutions:

**Option 1: NodePort Service (Recommended for local)**
```yaml
apiVersion: v1
kind: Service
metadata:
  name: frontend-service
  namespace: chat-app
spec:
  type: NodePort
  selector:
    app: frontend
  ports:
  - port: 80
    targetPort: 80
    nodePort: 30080  # Access via localhost:30080
```
Access: `http://localhost:30080` (always available)

**Option 2: LoadBalancer Service**
```yaml
apiVersion: v1
kind: Service
metadata:
  name: frontend-service
  namespace: chat-app
spec:
  type: LoadBalancer
  selector:
    app: frontend
  ports:
  - port: 80
    targetPort: 80
```

**Option 3: Startup Script (macOS)**
Create `~/start-portforward.sh`:
```bash
#!/bin/bash
kubectl port-forward svc/frontend-service 3000:80 -n chat-app &
kubectl port-forward svc/backend-service 8080:8080 -n chat-app &
```

**Recommendation**: Use **NodePort** for persistent local access without manual intervention.

---

### Q: "I want to implement load balancer how to get started with that giev me steps don't do any changes in teh code. Also I want to scale my pods giev me stratergies to do that so that when thousands of customers come to teh site the side don't crash."

**A:** 

## **Load Balancer Implementation Steps**

### **For Local Development (kind cluster):**
1. **Install MetalLB** (Load Balancer for bare metal):
```bash
kubectl apply -f https://raw.githubusercontent.com/metallb/metallb/v0.13.7/config/manifests/metallb-native.yaml
```

2. **Configure IP Pool**:
```bash
# Get Docker network range
docker network inspect -f '{{.IPAM.Config}}' kind
# Create MetalLB config with available IPs
```

3. **Change Service Type** to `LoadBalancer`

### **For Production (Cloud):**
- **AWS**: Use ALB/NLB (Application/Network Load Balancer)
- **GCP**: Use Google Cloud Load Balancer
- **Azure**: Use Azure Load Balancer
- **Hostinger VPS**: Use nginx as reverse proxy

## **Pod Scaling Strategies for High Traffic**

### **1. Horizontal Pod Autoscaler (HPA)**
**Steps:**
1. Ensure metrics-server is running
2. Create HPA for frontend: `kubectl autoscale deployment frontend-deployment --cpu-percent=70 --min=2 --max=20 -n chat-app`
3. Create HPA for backend: `kubectl autoscale deployment backend-deployment --cpu-percent=70 --min=3 --max=50 -n chat-app`

### **2. Manual Scaling Strategy**
**Frontend Scaling:**
```bash
kubectl scale deployment frontend-deployment --replicas=10 -n chat-app
```

**Backend Scaling:**
```bash
kubectl scale deployment backend-deployment --replicas=15 -n chat-app
```

### **Recommended Scaling Plan for 1000+ Users:**
1. **Frontend**: 5-10 pods
2. **Backend**: 10-20 pods  
3. **MySQL**: 1 master + 3 read replicas
4. **Redis**: 3 pods (cluster mode)
5. **Load Balancer**: Cloud provider LB
6. **Monitoring**: Prometheus + Grafana
7. **Logging**: ELK stack

---

### Q: "m frontend service is runnign on IP address 10.96.59.120 then why i can't access the service using http://10.96.59.120:3000?"

**A:** You can't access `10.96.59.120:3000` because:

## **Issues with your approach:**

1. **Wrong Port**: Service runs on port `80`, not `3000`
   - Try: `http://10.96.59.120:80` or just `http://10.96.59.120`

2. **ClusterIP is Internal**: `10.96.59.120` is only accessible **inside** the Kubernetes cluster, not from your laptop

3. **Network Isolation**: ClusterIP creates internal-only networking

## **Why ClusterIP doesn't work from outside:**
```
Your Laptop (outside cluster) ❌ → 10.96.59.120 (ClusterIP - internal only)
```

## **Solutions to access from your laptop:**

### **Option 1: Port Forward (Temporary)**
```bash
kubectl port-forward svc/frontend-service 3000:80 -n chat-app
# Access: http://localhost:3000
```

### **Option 2: NodePort (Persistent)**
Change service type to `NodePort`:
```yaml
spec:
  type: NodePort
  ports:
  - port: 80
    targetPort: 80
    nodePort: 30080
```
Access: `http://localhost:30080`

**ClusterIP = Internal cluster communication only**
**NodePort/LoadBalancer = External access**

---

### Q: "explain me more about each option. What i have understood is that the KIND cluster is running 3 nodes 2 worker and 1 master. The deployments that I have made for frontend, backend, and sql those pods are being made in these nodes. Now, When I am making clusterIP service for those deployments each pod has a specific port that is getting a IP address so that that pod could be accessable for the other pods like frontsend need backend connectivity. Is it correct?"

**A:** **Excellent understanding!** You're absolutely correct. Let me elaborate:

## **Your Understanding is Perfect ✅**

```
KIND Cluster:
├── Control Plane Node (master)
├── Worker Node 1 
└── Worker Node 2

Pods distributed across nodes:
├── frontend-pod-1 (Worker 1) → 10.244.1.5:80
├── frontend-pod-2 (Worker 2) → 10.244.2.8:80  
├── backend-pod-1 (Worker 1) → 10.244.1.6:8080
├── backend-pod-2 (Worker 2) → 10.244.2.9:8080
└── mysql-pod (Worker 1) → 10.244.1.7:3306
```

## **ClusterIP Service Magic:**

**Problem without Service:**
- Frontend needs to know ALL backend pod IPs
- If backend pod dies, IP changes
- Hard to manage multiple pods

**ClusterIP Solution:**
```
frontend-service: 10.96.59.120:80 → Load balances to all frontend pods
backend-service: 10.96.44.7:8080 → Load balances to all backend pods  
mysql-service: 10.96.12.235:3306 → Points to mysql pod
```

## **Service Types Explained:**

### **1. ClusterIP (What you have)**
```
┌─────────────────┐    ┌──────────────────┐
│   Your Laptop   │ ❌ │  ClusterIP       │
│   (Outside)     │    │  10.96.59.120    │
└─────────────────┘    └──────────────────┘
                              ↓ ✅
                       ┌──────────────────┐
                       │  Pods in Cluster │
                       │  (Internal only) │
                       └──────────────────┘
```

**Use Case:** Pod-to-pod communication (backend ↔ mysql)

### **2. NodePort**
```yaml
spec:
  type: NodePort
  ports:
  - port: 80
    targetPort: 80
    nodePort: 30080  # Exposed on ALL nodes
```

```
┌─────────────────┐    ┌──────────────────┐
│   Your Laptop   │ ✅ │  Node:30080      │
│   localhost     │────│  (All 3 nodes)  │
└─────────────────┘    └──────────────────┘
                              ↓
                       ┌──────────────────┐
                       │  ClusterIP       │
                       │  Load Balancer   │
                       └──────────────────┘
```

**Access:** `http://localhost:30080`

### **3. LoadBalancer**
```yaml
spec:
  type: LoadBalancer
  ports:
  - port: 80
    targetPort: 80
```

```
┌─────────────────┐    ┌──────────────────┐    ┌──────────────────┐
│   Your Laptop   │ ✅ │  External LB     │ ✅ │  NodePort        │
│   Internet      │────│  (Cloud/MetalLB) │────│  (All nodes)     │
└─────────────────┘    └──────────────────┘    └──────────────────┘
```

**Access:** External IP (like `192.168.1.100`)

Your understanding of Kubernetes networking is spot-on! 🎯

---

### Q: "Now, I have 2 backend pods and 2 frontend pods. i want to knwo that pod is running on what node workernode or master node and also want to know what IP is assigned to which pod? also if teh pods are deleted and created over time how does the IP address stays same does that also chage?"

**A:** ## **Check Pod Node Assignment & IPs:**

```bash
# See which node each pod is running on + IP addresses
kubectl get pods -n chat-app -o wide

# More detailed view
kubectl describe pods -n chat-app | grep -E "Name:|Node:|IP:"
```

## **Pod IP Behavior - IMPORTANT:**

### **Pod IPs are EPHEMERAL (Change Every Time):**
```bash
# Delete a pod
kubectl delete pod backend-deployment-6dd85699b6-skm9b -n chat-app

# Watch new pod get created with DIFFERENT IP
kubectl get pods -n chat-app -o wide -w
```

**Before deletion:**
```
backend-deployment-6dd85699b6-skm9b   10.244.1.5   kind-cluster-worker
```

**After recreation:**
```
backend-deployment-6dd85699b6-xyz123  10.244.2.15  kind-cluster-worker2
```

## **Why Service IPs Stay Same:**

### **Service IP (ClusterIP) = STABLE:**
```bash
kubectl get svc -n chat-app
```
```
backend-service    ClusterIP   10.96.44.7    # NEVER CHANGES
frontend-service   ClusterIP   10.96.59.120  # NEVER CHANGES
```

## **How Kubernetes Handles This:**

### **Service Discovery Magic:**
```
Frontend Pod calls: http://backend-service:8080
                           ↓
Service (10.96.44.7:8080) → Load balances to:
                           ├── Pod 1: 10.244.1.5:8080
                           └── Pod 2: 10.244.2.8:8080
```

**When pod dies and recreates:**
```
Service (10.96.44.7:8080) → Load balances to:
                           ├── Pod 1: 10.244.2.15:8080  (NEW IP!)
                           └── Pod 2: 10.244.2.8:8080   (Same)
```

## **Key Takeaway:**
- **Pod IPs**: Change on every restart ❌
- **Service IPs**: Stable forever ✅
- **DNS Names**: Always work (`backend-service`, `mysql-service`) ✅

This is why you use service names in your app config, not pod IPs!

---

### Q: "Service (10.96.44.7:8080) → Load balances to: but I haven't deployed any loadbalncer yet then how is this Load balances to: ├── Pod 1: 10.244.1.5:8080 └── Pod 2: 10.244.2.8:8080 possible?"

**A:** **Excellent question!** You're absolutely right to be confused. Let me clarify:

## **ClusterIP Service = Built-in Load Balancer**

**Every Kubernetes Service has built-in load balancing**, even ClusterIP!

### **What's happening behind the scenes:**

```yaml
apiVersion: v1
kind: Service
metadata:
  name: backend-service
spec:
  type: ClusterIP          # ← This INCLUDES load balancing!
  selector:
    app: backend           # ← Finds all pods with this label
  ports:
  - port: 8080
    targetPort: 8080
```

### **Kubernetes Service Components:**

```
┌─────────────────────────────────────────┐
│           ClusterIP Service             │
│  ┌─────────────────────────────────────┐│
│  │        Built-in Load Balancer       ││  ← THIS IS AUTOMATIC!
│  │     (kube-proxy + iptables)         ││
│  └─────────────────────────────────────┘│
└─────────────────────────────────────────┘
                    ↓
        Distributes traffic to:
    ┌─────────────┐  ┌─────────────┐
    │   Pod 1     │  │   Pod 2     │
    │ 10.244.1.5  │  │ 10.244.2.8  │
    └─────────────┘  └─────────────┘
```

## **Service Types vs Load Balancing:**

| Service Type | Load Balancing | External Access |
|--------------|----------------|-----------------|
| **ClusterIP** | ✅ Built-in | ❌ Internal only |
| **NodePort** | ✅ Built-in | ✅ Via node ports |
| **LoadBalancer** | ✅ Built-in + External LB | ✅ Via external IP |

## **The Confusion:**

**"LoadBalancer" Service Type** ≠ **Load Balancing Feature**

- **Load Balancing**: Every service has this ✅
- **LoadBalancer Type**: External cloud load balancer (AWS ALB, GCP LB, etc.)

**So yes, you already have load balancing working!** 🎯

---

### Q: "why 2 endpoints because I have 2 pods, but I only have one service for backend so there should be one endpoint right NAME backend-service 10.244.1.5:8080,10.244.2.8:8080 21h what am I missing here?"

**A:** **Great question!** You're confusing **Service** vs **Endpoints**. Let me clarify:

## **Service vs Endpoints - Key Difference:**

### **Service = Load Balancer (1 Service)**
```
backend-service (10.96.44.7:8080) ← ONE SERVICE
```

### **Endpoints = Target Pods (Multiple Endpoints)**
```
backend-service endpoints:
├── 10.244.1.5:8080 (Pod 1) ← ENDPOINT 1
└── 10.244.2.8:8080 (Pod 2) ← ENDPOINT 2
```

## **Think of it like this:**

```
┌─────────────────────────────────────┐
│         backend-service             │  ← 1 SERVICE
│        (Load Balancer)              │
│         10.96.44.7:8080             │
└─────────────────┬───────────────────┘
                  │
        ┌─────────┴─────────┐
        ▼                   ▼
┌─────────────┐    ┌─────────────┐
│  ENDPOINT 1 │    │  ENDPOINT 2 │      ← 2 ENDPOINTS
│   Pod 1     │    │   Pod 2     │
│10.244.1.5   │    │10.244.2.8   │
└─────────────┘    └─────────────┘
```

## **Real World Analogy:**

**Service = Restaurant (1 restaurant)**
- Name: "Pizza Palace"
- Phone: (555) 123-4567

**Endpoints = Kitchen Staff (Multiple cooks)**
- Cook 1: Makes your pizza
- Cook 2: Makes your pizza
- Cook 3: Makes your pizza

**When you call the restaurant:**
- You dial ONE number (Service)
- But ANY of the 3 cooks can make your pizza (Endpoints)

## **Summary:**
- **1 Service** = 1 Load Balancer with 1 IP
- **2 Endpoints** = 2 Target Pods behind that service
- **Service automatically discovers** all pods matching the selector

**You have it right! 1 Service distributing to 2 Endpoints (Pods).** 🎯

---

### Q: "I have spec: selector: app: frontend. which shows that selecot is frontend but my pods are named as frontend-deployment-55857b7596-bq4jf So, even by specifying the initial name part of pods the service will find the pods by itself?"

**A:** **No!** The service selector doesn't match pod **names** - it matches pod **labels**!

## **How Service Selection Actually Works:**

### **Service Selector:**
```yaml
spec:
  selector:
    app: frontend  ← Looking for LABEL "app=frontend"
```

### **Pod Labels (Not Names):**
Check what labels your pods actually have:

```bash
kubectl get pods -n chat-app --show-labels
```

**Expected Output:**
```
NAME                                   LABELS
frontend-deployment-55857b7596-bq4jf   app=frontend,pod-template-hash=55857b7596
frontend-deployment-55857b7596-plwck   app=frontend,pod-template-hash=55857b7596
```

## **Where Do Pod Labels Come From?**

### **In your Deployment:**
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: frontend-deployment  ← This is just the deployment name
spec:
  template:
    metadata:
      labels:
        app: frontend        ← THIS creates the pod label!
```

## **Label Matching Process:**

```
Service Selector: app=frontend
         ↓
    Searches for pods with label: app=frontend
         ↓
    Finds pods:
    ├── frontend-deployment-55857b7596-bq4jf (has label app=frontend) ✅
    └── frontend-deployment-55857b7596-plwck (has label app=frontend) ✅
```

## **Key Points:**

1. **Pod Names**: `frontend-deployment-xxx` (Generated by Kubernetes)
2. **Pod Labels**: `app=frontend` (Set by your Deployment template)
3. **Service Selector**: Matches **labels**, not **names**

**The service finds pods by labels, not by name patterns!** 🏷️

---

### Q: "Also what is the need for this in deployment selector: matchLabels: app: frontend"

**A:** **Excellent question!** The Deployment selector tells the Deployment **which pods it owns and manages**.

## **Deployment Selector Purpose:**

### **Deployment's Job:**
```yaml
spec:
  replicas: 2           ← "I need to maintain 2 pods"
  selector:
    matchLabels:
      app: frontend     ← "I manage pods with this label"
  template:
    metadata:
      labels:
        app: frontend   ← "I create pods with this label"
```

## **What Happens Without Selector:**

**Problem:** Deployment doesn't know which pods belong to it!

```
Cluster has many pods:
├── frontend-pod-1 (app=frontend)     ← Mine?
├── frontend-pod-2 (app=frontend)     ← Mine?  
├── nginx-pod (app=nginx)             ← Not mine
├── backend-pod (app=backend)         ← Not mine
└── random-pod (app=frontend)         ← Mine? (Could be from another deployment!)
```

## **Deployment Selector = Ownership:**

```yaml
selector:
  matchLabels:
    app: frontend    ← "I own ALL pods with app=frontend label"
```

### **Deployment Logic:**
```
1. Count pods with label app=frontend
2. If count < replicas (2): Create more pods
3. If count > replicas (2): Delete excess pods  
4. If pod dies: Replace it
5. If pod is unhealthy: Restart it
```

## **Selector vs Template Labels Must Match:**

```yaml
selector:
  matchLabels:
    app: frontend     ← Must match template labels
template:
  metadata:
    labels:
      app: frontend   ← Must match selector
```

**If they don't match = Deployment fails!**

## **Summary:**
- **Deployment Selector**: "Which pods do I own/manage?"
- **Service Selector**: "Which pods do I send traffic to?"
- **Both use labels for identification** 🏷️

**The selector is how Kubernetes knows which pods belong to which controller!**

---

### Q: "command for port forwarding for frontend, also command to check weather the service is running or not?"

**A:** ## **Port Forwarding for Frontend:**

```bash
kubectl port-forward svc/frontend-service 3000:80 -n chat-app
```

Then access: `http://localhost:3000`

## **Check if Service is Running:**

```bash
# Check service status
kubectl get svc frontend-service -n chat-app

# Check service details
kubectl describe svc frontend-service -n chat-app

# Check if service has endpoints (pods behind it)
kubectl get endpoints frontend-service -n chat-app

# Check pods status
kubectl get pods -l app=frontend -n chat-app

# Check all frontend resources
kubectl get all -l app=frontend -n chat-app
```

## **Verify Service is Working:**

```bash
# Test service connectivity from within cluster
kubectl run test-pod --image=busybox -it --rm --restart=Never -n chat-app -- wget -qO- http://frontend-service

# Check service logs
kubectl logs -l app=frontend -n chat-app
```

**Healthy service should show:**
- Service: `ClusterIP` with valid IP
- Endpoints: Should list pod IPs
- Pods: `Running` status with `1/1 Ready`

---

### Q: "docker build command?"

**A:** **Backend:**
```bash
cd backend
docker build -t prabhjot24/chattingo-backend:latest .
```

**Frontend:**
```bash
cd frontend
docker build -t prabhjot24/chattingo-frontend:latest .
```

**Push to Docker Hub:**
```bash
docker push prabhjot24/chattingo-backend:latest
docker push prabhjot24/chattingo-frontend:latest
```

**Build from root directory:**
```bash
docker build -t prabhjot24/chattingo-backend:latest ./backend
docker build -t prabhjot24/chattingo-frontend:latest ./frontend
```

---

### Q: "now give me deployment and service files for frontend only give code don't make files"

**A:** ```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: frontend-deployment
  namespace: chat-app
spec:
  replicas: 2
  selector:
    matchLabels:
      app: frontend
  template:
    metadata:
      labels:
        app: frontend
    spec:
      containers:
      - name: frontend
        image: prabhjot24/chattingo-frontend:latest
        ports:
        - containerPort: 80
        env:
        - name: REACT_APP_API_URL
          value: "http://backend-service:8080"
        resources:
          requests:
            memory: "128Mi"
            cpu: "100m"
          limits:
            memory: "256Mi"
            cpu: "200m"
---
apiVersion: v1
kind: Service
metadata:
  name: frontend-service
  namespace: chat-app
spec:
  selector:
    app: frontend
  ports:
  - port: 80
    targetPort: 80
  type: ClusterIP
```

---

### Q: "localhost:80 is redirecting to localhost by itself :80 is dissapearing"

**A:** This is normal browser behavior. Port 80 is the default HTTP port, so browsers automatically hide it.

- `localhost:80` → `localhost` (browser hides :80)
- `localhost:8080` → `localhost:8080` (browser shows non-standard ports)

**This is correct behavior.** Your frontend is working properly on port 80.

**To access your frontend:**
```bash
# Port forward to a different port to see it in URL
kubectl port-forward svc/frontend-service 3000:80 -n chat-app
```

Then access: `http://localhost:3000`

The `:80` disappearing is expected - it means your frontend is running correctly on the standard HTTP port.

---

### Q: "command to push all the changes to github after creating a new k8s branch"

**A:** ```bash
# Create and switch to new branch
git checkout -b k8s

# Add all changes
git add .

# Commit changes
git commit -m "Add Kubernetes deployment files and Docker configurations"

# Push new branch to GitHub
git push -u origin k8s
```

**Alternative - if you want to stage specific files:**

```bash
git checkout -b k8s
git add k8s/ backend/Dockerfile frontend/Dockerfile frontend/nginx.conf
git commit -m "Add K8s deployments, Dockerfiles, and nginx config"
git push -u origin k8s
```

The `-u` flag sets upstream tracking for the new branch.

---

### Q: "I can see a tab on giuthub that says compart and pull request what does that mean?"

**A:** **"Compare & Pull Request"** means:

## **What it does:**
- **Compare**: Shows differences between your `k8s` branch and `main` branch
- **Pull Request (PR)**: Requests to merge your `k8s` branch into `main` branch

## **Why use it:**
- **Code Review**: Others can review your changes before merging
- **Discussion**: Team can comment on your code
- **Quality Control**: Ensures changes are tested before going to main
- **History**: Tracks what changes were made and why

## **Process:**
1. **Click "Compare & Pull Request"**
2. **Add title**: "Add Kubernetes deployment files"
3. **Add description**: Explain what you added (Dockerfiles, K8s manifests, etc.)
4. **Create Pull Request**
5. **Review & Merge**: You or others review and merge to main

## **For your hackathon:**
Since it's your own project, you can:
- **Create PR** for documentation/review
- **Or merge directly** to main if you don't need review

**Benefits for hackathon submission:**
- Shows professional Git workflow
- Documents your development process
- Makes your project look more organized

---

### Q: "create a linkdin post for this project"

**A:** 🚀 **Just completed the Chattingo Mini Hackathon Challenge!** 

Built a full-stack real-time chat application and successfully containerized it with modern DevOps practices:

**🛠️ Tech Stack:**
• Frontend: React 18 + Redux + Material-UI + WebSocket
• Backend: Spring Boot + MySQL + JWT Authentication  
• Infrastructure: Docker + Kubernetes + Jenkins CI/CD

**🔧 What I Accomplished:**
✅ Multi-stage Dockerfiles for optimized container builds
✅ Kubernetes deployments with StatefulSets for MySQL
✅ Horizontal & Vertical Pod Autoscaling (HPA/VPA)
✅ Real-time messaging with WebSocket integration
✅ Complete CI/CD pipeline with Jenkins
✅ Production-ready deployment on Hostinger VPS

**⚡ Real Challenges Faced & Solutions:**
🔸 **Database Connection Issues**: Backend couldn't connect to MySQL due to credential mismatches - resolved by aligning environment variables between StatefulSet and Deployment

🔸 **Persistent Volume Problems**: MySQL kept using old data despite fresh deployments - learned to properly clear PV data and manage StatefulSet storage lifecycle

🔸 **Container Image Issues**: Deprecated OpenJDK images causing build failures - migrated to Eclipse Temurin for reliable Java runtime

🔸 **Service Discovery**: Frontend nginx couldn't find backend service - fixed by updating proxy configurations to use correct Kubernetes service names

🔸 **Metrics Server Setup**: VPA couldn't get resource metrics in local cluster - configured proper TLS settings for kind cluster compatibility

**💡 Key Learnings:**
• Kubernetes storage persistence requires careful PV/PVC lifecycle management
• StatefulSets behave differently from Deployments for database workloads
• Container networking in K8s requires understanding of service discovery
• Local development clusters need specific configurations for production-like features

**🎯 Challenge Highlights:**
• 3-day intensive development sprint
• From vanilla app to production-ready containerized system
• Real debugging experience with complex distributed systems

The debugging journey taught me more than any tutorial could! 💪

#DevOps #Kubernetes #Docker #SpringBoot #React #CloudComputing #Hackathon #FullStack #CI/CD #Jenkins #Debugging #ProblemSolving

**GitHub:** [Your Repository Link]
**Live Demo:** [Your VPS URL]

---

### Q: "Now create a .md file specify all the questions I have asked followed by their answers."

**A:** I've created a comprehensive `TODAY_KUBERNETES_QA.md` file that covers all the questions you asked and their detailed answers from today's session.

---

*This document contains all the exact questions asked today and their complete answers provided during our Kubernetes learning session.*