# 🚗 Uber Clone — Full-Stack Using Microservices

<p align="center">
  <a href="https://skillicons.dev">
    <img src="https://skillicons.dev/icons?i=react,ts,nodejs,npm,express,mongodb,redis,kubernetes,docker,gcp,terraform,githubactions,kafka,firebase" />
  </a>
  
  <br>
  <img src="https://img.shields.io/badge/Socket.io-010101?style=for-the-badge&logo=socket.io&logoColor=white" />
  <img src="https://img.shields.io/badge/Helm-0F1628?style=for-the-badge&logo=helm&logoColor=white" />
  <img src="https://img.shields.io/badge/fastlane-00F2FF?style=for-the-badge&logo=fastlane&logoColor=black" />
  <img src="https://img.shields.io/badge/Skaffold-239120?style=for-the-badge&logo=skaffold&logoColor=white" />
</p>

> A production-grade, cloud-native ride-sharing platform built with a microservices architecture. Covers the full engineering stack: a **React Native** dual-flavor mobile app, **7 Node.js/TypeScript** backend microservices, a **shared private NPM package**, and a complete **GitOps CI/CD pipeline** running on **Google Kubernetes Engine**.

---

## 📁 Repository Structure

This project is split across **three GitHub repositories**, each with a distinct responsibility:


| Repository | Description |
|---|---|
| `uber-clone` | All 7 backend microservices + Kubernetes manifests + Helm chart + Skaffold config |
| `rn_app_uber` | React Native mobile app (User & Driver dual-flavor) |
| `uber-clone-infra` | Terraform IaC for provisioning the GKE cluster on Google Cloud |

---

## 📦 The `@uber-clone/common` Shared NPM Package

One of the core architectural decisions is a **private shared NPM package** that lives inside `uber-clone/common/`. Every microservice installs it as a dependency, ensuring a **single source of truth** for shared logic.

### What's Inside

| Module | Purpose |
|---|---|
| `errors/` | Typed custom error classes (`BadRequestError`, `NotFoundError`, `NotAuthorizedError`, `DatabaseConnectionError`, `RequestValidationError`) |
| `middlewares/` | Express middleware: `currentUser` (JWT decode), `requireAuth` (guard), `errorHandler` (centralized), `validateRequest` |
| `events/subjects.ts` | The canonical `Subjects` enum — every Kafka topic string in the entire system defined once |
| `events/kafka-client.ts` | Singleton `KafkaClient` wrapper (built on `kafkajs`) shared by all services |
| `events/kafka-publisher.ts` | Abstract base class for typed Kafka producers |
| `events/kafka-listener.ts` | Abstract base class for typed Kafka consumers |
| `events/types.ts` | Shared TypeScript interfaces for all event payloads |
| `redis/redis-client.ts` | Shared Redis client singleton |
| `metrics/app.metrics.ts` | Prometheus metrics setup via `prom-client` |
| `utils/logger.ts` | Structured `winston` logger |

### Kafka Topics Defined in `Subjects` Enum

```typescript
// A single enum governs every event name in the system
export enum Subjects {
  // User & Auth
  UserCreated, UserUpdated, UserSignedIn, UserSignedOut,
  // Ride Lifecycle
  RideRequested, RideAccepted, RideStarted, RideCompleted, RideCancelled, RideRated,
  // Driver
  DriverOnline, DriverOffline, DriverLocationUpdated, DriverRideAccepted,
  // Payments
  PaymentInitiated, PaymentCompleted, PaymentFailed, PaymentRefunded,
  // Notifications
  NotificationEmail, NotificationSMS, NotificationPush, NotificationInApp,
}
```

### Publishing a New Version

The package is published to the **npm registry** under the scoped name `@uber-clone/common`. The workflow is a single command:

```bash
# In uber-clone/common/
npm run pub
# This runs: git add . && git commit && npm version patch && npm run build && npm publish
```

This bumps the patch version (currently **v1.0.24**), compiles TypeScript to `build/`, and publishes. Each microservice then updates its `package.json` to pull the latest version.

---

## 🖥️ Local Development with Skaffold & Kubernetes

The entire backend stack — all 7 microservices, Kafka, Redis, MongoDB instances, Ingress — runs **locally inside Kubernetes** (Docker Desktop) using a single command. No `docker-compose`. No manual `kubectl apply`. Just Skaffold.

### Prerequisites

- [Docker Desktop](https://www.docker.com/products/docker-desktop/) with **Kubernetes enabled**
- [Skaffold](https://skaffold.dev/docs/install/) (`v4beta11+`)
- [kubectl](https://kubernetes.io/docs/tasks/tools/)
- [Node.js](https://nodejs.org/) `>= 22.11.0`
- Git Bash (used by Skaffold hooks on Windows)

### How It Works — Step by Step

Running `skaffold dev` from the root of `uber-clone/` triggers the following automated sequence:

#### Phase 1: Pre-flight Hooks (`manifests.hooks.before`)

Skaffold executes these shell commands **before** deploying any manifests:

1. **Creates the namespace**: `kubectl create namespace uber-clone --dry-run=client -o yaml | kubectl apply -f -`
2. **Installs Strimzi Kafka Operator**: Applies the Strimzi CRD bundle so Kafka can run inside Kubernetes natively
3. **Waits for CRDs**: Sleeps 10s then waits for `kafkas.kafka.strimzi.io` to be established
4. **Injects the JWT Secret**: Creates `jwt-secret` Kubernetes Secret in the `uber-clone` namespace
5. **Injects Firebase credentials**: Mounts `firebase-service-account-key.json` as a Kubernetes Secret
6. **Cleans stuck Kafka finalizers**: Patches any leftover `KafkaTopic` resources that may have finalizers from a previous session
7. **Installs NGINX Ingress Controller**: Applies the official NGINX Ingress manifest for local cloud-style routing

#### Phase 2: Build All Service Images

Skaffold builds **7 Docker images in parallel** (using BuildKit, `concurrency: 0`) using each service's `Dockerfile.dev`. Images are built locally and **not pushed** to any registry:

```yaml
artifacts:
  - image: hitendraparmar/auth       # → auth/Dockerfile.dev
  - image: hitendraparmar/location   # → location/Dockerfile.dev
  - image: hitendraparmar/ride       # → ride/Dockerfile.dev
  - image: hitendraparmar/driver     # → driver-service/Dockerfile.dev
  - image: hitendraparmar/passenger  # → passenger-service/Dockerfile.dev
  - image: hitendraparmar/notification # → notification-service/Dockerfile.dev
  - image: hitendraparmar/payment    # → payment/Dockerfile.dev
```

#### Phase 3: Deploy Kubernetes Manifests

All files in `infra/k8s/local-dev/` are applied, which include:

- **Kafka cluster** (`02-kafka.yaml`) — Strimzi `Kafka` CR, single-node for local dev
- **Kafka Topics** (`04-kafka-topics.yaml`) — All 28 topics pre-created via `KafkaTopic` CRs
- **Redis Cluster** (`05-redis-cluster.yaml` + `05-redis-configmap.yaml`)
- **Per-service MongoDB deployments** — Each service gets its own `mongo` pod (`auth-mongo-depl.yaml`, `ride-mongo-depl.yaml`, etc.)
- **All 7 microservice Deployments + Services**
- **NGINX Ingress** (`ingress-srv.yaml`) — Routes `/api/users/*`, `/api/ride/*`, `/api/location/*`, etc. to the correct service
- **Kubernetes Secrets** — JWT, Firebase, Razorpay, Twilio, SMTP, Google Maps

#### Phase 4: File Sync (Hot-Reload Without Rebuilding)

Each artifact is configured with **manual sync** for TypeScript source files:

```yaml
sync:
  manual:
    - src: "src/**/*.ts"
      dest: /app
```

When you save a `.ts` file, Skaffold **copies it directly into the running container** without rebuilding the Docker image. The `ts-node-dev` or `nodemon` process inside the container picks up the change and restarts the service in seconds.

#### Phase 5: Port Forwarding

Skaffold automatically forwards all service ports to `localhost`:

| Service | Port |
|---|---|
| auth-srv | `localhost:3000` |
| location-srv | `localhost:3001` |
| ride-srv | `localhost:3002` |
| driver-srv | `localhost:3003` |
| passenger-srv | `localhost:3004` |
| notification-srv | `localhost:3005` |
| payment-srv | `localhost:3006` |

### Start Local Development

```bash
# From the root of uber-clone/
skaffold dev
```

Skaffold watches all source files. **Ctrl+C** tears everything down cleanly.

---

## 🔧 Backend Microservices

All services are built with **Node.js + Express + TypeScript** and share the `@uber-clone/common` package.

| Service | Port | Database | Key Responsibility |
|---|---|---|---|
| **auth** | 3000 | MongoDB | Registration, login, JWT issuance, session management |
| **location** | 3001 | Redis only | Real-time driver location via WebSocket + Redis `GEOADD`/`GEORADIUS` |
| **ride** | 3002 | MongoDB | Ride lifecycle orchestration (Requested → Accepted → Started → Completed) |
| **driver-service** | 3003 | MongoDB | Driver profile CRUD, ratings, vehicle details |
| **passenger-service** | 3004 | MongoDB | Passenger profile CRUD, ride history |
| **notification-service** | 3005 | MongoDB | Firebase FCM push notifications, Twilio SMS, SMTP email |
| **payment** | 3006 | MongoDB | Razorpay payment order creation and verification |

### Event-Driven Flow (Example: Ride Request)

```
Passenger requests ride
      │
      ▼
Ride Service → publishes "ride.requested" to Kafka
      │
      ├──► Notification Service → sends FCM push to nearby drivers
      │
      ├──► Driver Service → updates driver state
      │
Driver accepts → publishes "ride.accepted"
      │
      ▼
Ride Service → finalizes match, updates DB
      │
      ▼
Notification Service → notifies passenger "Driver is on the way"
```

---

## 📱 Mobile App (`rn_app_uber`)

A single **React Native** codebase that compiles into **two completely separate apps** using Android Build Flavors.

### Dual-Flavor Architecture

| Flavor | App ID Suffix | Metro Port | Env File | Target |
|---|---|---|---|---|
| **User** | `.user` | `8082` | `.env.user` | Passengers booking rides |
| **Driver** | `.driver` | `8083` | `.env.driver` | Drivers accepting rides |

Both flavors run simultaneously from the same codebase with isolated configs:

```bash
# Run User app (in two terminals)
npm run start:user          # Metro on port 8082
npm run rn:user             # Android build with userDebug flavor

# Run Driver app (in two other terminals)
npm run start:driver        # Metro on port 8083
npm run rn:driver           # Android build with driverDebug flavor
```

### Key Libraries

| Library | Purpose |
|---|---|
| `react-native-maps` | Google Maps with live driver markers and route polylines |
| `react-native-maps-directions` | Route drawing between pickup and dropoff |
| `react-native-google-places-autocomplete` | Location search with Places API |
| `socket.io-client` | WebSocket connection for real-time tracking |
| `@react-native-firebase/messaging` | Firebase Cloud Messaging for push notifications |
| `react-native-razorpay` | In-app Razorpay payment sheet |
| `react-native-geolocation-service` | GPS location for driver broadcasts |
| `@reduxjs/toolkit` + `redux-persist` | Global state with persistence |
| `@gorhom/bottom-sheet` | Ride booking and confirmation sheets |
| `react-native-reanimated` | Smooth animations (radar, pulse, map transitions) |
| `react-native-config` | Environment variable injection per flavor |

### Fastlane — Automated App Distribution

**Fastlane** automates building and distributing release APKs to **Firebase App Distribution** for testing. It is configured in `android/fastlane/Fastfile`:

```ruby
# Build & distribute User app to Firebase testers
lane :distribute_user do
  increment_build_version    # Auto-increments versionCode in build.gradle
  ENV["ENVFILE"] = ".env.user"
  gradle(task: "assemble", flavor: "user", build_type: "Release")
  firebase_app_distribution(app: ENV["FIREBASE_APP_ID_USER"], groups: "testers")
end

# Build & distribute Driver app
lane :distribute_driver do
  increment_build_version
  ENV["ENVFILE"] = ".env.driver"
  gradle(task: "assemble", flavor: "driver", build_type: "Release")
  firebase_app_distribution(app: ENV["FIREBASE_APP_ID_DRIVER"], groups: "testers")
end
```

Run locally:
```bash
npm run distribute:user    # → cd android && bundle exec fastlane distribute_user
npm run distribute:driver  # → cd android && bundle exec fastlane distribute_driver
npm run distribute:all     # Both at once
```

### Mobile CI/CD — GitHub Actions

On every push to `main`, the **Android Firebase Distribution** workflow (`.github/workflows/android-distribute.yml`) runs automatically:

1. Sets up Node.js 22, Java 17, Ruby 3.2, Android SDK
2. Runs `npm ci` and caches Gradle dependencies
3. Injects `.env`, `.env.user`, `.env.driver` from GitHub Secrets
4. Writes Firebase service account credentials from `FIREBASE_CREDENTIALS` secret
5. Calls `bundle exec fastlane distribute_user` to build & ship the APK

---

## ☁️ Production Infrastructure

### Overview

Production runs on **Google Kubernetes Engine (GKE)** on Google Cloud Platform. The infrastructure is fully codified — from the cluster itself to the Kubernetes Secrets.

### Layer 1: GKE Cluster — Terraform (`uber-clone-infra`)

The entire GCP infrastructure is defined as code in Terraform and stored remotely in a **GCS bucket backend** (`uber-clone-terraform-state`).

**What Terraform provisions:**

| Resource | Details |
|---|---|
| **GKE Standard Cluster** | VPC-native, `asia-south1` region, deletion protection disabled (demo) |
| **Node Pool** | `e2-standard-2` **Spot instances**, autoscales 1–3 nodes |
| **VPC + Subnets** | Custom VPC with dedicated pod and service IP ranges |
| **SSD StorageClass** | `pd-ssd` backed PersistentVolumes for database pods |
| **Kubernetes Namespace** | `uber-clone` namespace |
| **All Kubernetes Secrets** | JWT, Firebase, Razorpay, Twilio, Google Maps, SMTP, MongoDB — sourced from **GCP Secret Manager** |
| **Strimzi Kafka Operator** | Deployed via Helm release |
| **ArgoCD** | Deployed via Helm into `argocd` namespace |
| **NGINX Ingress Controller** | Deployed via Helm into `ingress-nginx` namespace |

**Secrets are pulled from GCP Secret Manager** (not hardcoded anywhere):
```hcl
data "google_secret_manager_secret_version" "razorpay_id" { secret = "razorpay-key-id" }
data "google_secret_manager_secret_version" "jwt_key"     { secret = "jwt-secret-key" }
# ...and so on for all 13 secrets
```

**Deploy:**
```bash
cd uber-clone-infra/terraform
terraform init
terraform plan
terraform apply   # ~10-15 min for GKE to spin up

# Connect kubectl to the new cluster
gcloud container clusters get-credentials uber-clone-gke --region asia-south1 --project <PROJECT_ID>
```

**Destroy (avoid unexpected charges):**
```bash
terraform destroy -auto-approve
# Also manually delete: Artifact Registry images, Compute disks, static IPs, the GCS state bucket
```

### Layer 2: Application Deployment — Helm + ArgoCD (GitOps)

The application is deployed to GKE via a **Helm chart** (`infra/k8s/prod/uber-clone-chart/`) managed by **ArgoCD** following a strict **GitOps** model.

**How GitOps works:**

```
Developer pushes to `gcp-production` branch
           │
           ▼
  GitHub Actions CI/CD Pipeline
  ├─ Builds 7 Docker images (tagged with git SHA)
  ├─ Pushes images to Docker Hub (hitendraparmar/<service>:<sha>)
  └─ Updates infra/k8s/prod/uber-clone-chart/values.yaml
     with new image tags via sed + git commit
           │
           ▼
  ArgoCD detects change in Git repo
  └─ Auto-syncs Helm chart to GKE cluster
     ├─ selfHeal: true  (reverts manual changes)
     └─ prune: true     (removes deleted resources)
```

**ArgoCD Application manifest** (`infra/k8s/prod/argocd-app.yaml`):
```yaml
spec:
  source:
    repoURL: https://github.com/hitendrasinh-Parmar/uber-clone.git
    path: infra/k8s/prod/uber-clone-chart
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
```

### Layer 3: Helm Chart — `uber-clone-chart`

The Helm chart in `infra/k8s/prod/uber-clone-chart/` is the single configuration file (`values.yaml`) that drives the entire production deployment:

**Microservice configuration example:**
```yaml
microservices:
  auth:
    image: "hitendraparmar/auth:<git-sha>"  # Updated by GitHub Actions on every deploy
    port: 3000
    requiresMongo: true
    mongoDbName: "auth"
    kafkaClientId: "auth-producer-client"
    kafkaConsumerGroup: "auth-service-consumer-group"
    ingressPath: "/(api/users/?.*)"

  notification:
    requiresTwilio: true      # Mounts Twilio secret
    requiresFirebase: true    # Mounts Firebase secret
    requiresSmtp: true        # Mounts SMTP secret

  payment:
    requiresRazorpay: true    # Mounts Razorpay secret
```

**Infrastructure in `values.yaml`:**
```yaml
global:
  kafkaBrokers: "uber-kafka-kafka-bootstrap:9092"
  redisUrl: "redis://redis-cluster:6379"

kafka:
  clusterName: "uber-kafka"
  topics:           # 28 topics, each with 3-6 partitions
    - name: "ride.requested"
      partitions: 3
    - name: "payment.completed"
      partitions: 4
    # ...
```

### Layer 4: GitHub Actions CI/CD (Backend)

**Workflow:** `.github/workflows/deploy.yaml`
**Trigger:** Push to `gcp-production` branch (ignores `infra/` path changes to avoid loops)

```yaml
jobs:
  build-and-deploy:
    steps:
      # 1. Build all 7 service images with the git SHA as the tag
      - docker build -t hitendraparmar/$IMAGE:${{ github.sha }} ./$SERVICE
      - docker push hitendraparmar/$IMAGE:${{ github.sha }}

      # 2. Update Helm values.yaml with new image tags (GitOps trigger)
      - sed -i "s|image: \"hitendraparmar/$IMAGE.*\"|image: \"hitendraparmar/$IMAGE:$TAG\"|g" values.yaml
      - git commit -m "chore: update image tags to $TAG [skip ci]"
      - git push
```

This single push triggers ArgoCD to deploy the new version within seconds.

---

## 🔑 Secrets Management

| Environment | How Secrets Are Managed |
|---|---|
| **Local Dev** | Kubernetes Secrets created by Skaffold pre-flight hooks directly from local files |
| **Production** | Secrets stored in **GCP Secret Manager**, pulled by Terraform and injected as Kubernetes Secrets at cluster creation time |
| **Mobile CI** | Stored as **GitHub Actions Secrets** (`ENV_FILE_USER`, `ENV_FILE_DRIVER`, `FIREBASE_CREDENTIALS`, `FIREBASE_APP_ID_USER`, etc.) |

---

## 📊 Observability Dashboards

| Tool | Purpose |
|---|---|
| **ArgoCD Dashboard** | Live view of deployment sync status, resource health, and rollback history |
| **Kafka UI** | Message topic inspection, consumer group lag monitoring |
| **Redis Insight** | Real-time geospatial driver location visualization |
| **GCP Console** | GKE node utilization, pod logs, load balancer metrics |
| **Firebase Console** | Push notification delivery rates and FCM analytics |
| **MongoDB Compass** | Per-service database inspection and query analysis |

---

## 📋 Full Tech Stack Summary

| Category | Technology |
|---|---|
| **Mobile App** | React Native 0.84, TypeScript, React Navigation, Redux Toolkit |
| **Maps & Location** | Google Maps API, react-native-maps, Google Places Autocomplete |
| **Push Notifications** | Firebase Cloud Messaging (FCM), `@react-native-firebase` |
| **Payments (Mobile)** | Razorpay SDK (`react-native-razorpay`) |
| **Mobile Distribution** | Fastlane + Firebase App Distribution |
| **Mobile CI** | GitHub Actions (`android-distribute.yml`) |
| **Backend Runtime** | Node.js 22 + Express + TypeScript |
| **Shared Package** | `@uber-clone/common` (private npm, v1.0.24) |
| **Real-Time** | Socket.io (WebSockets) |
| **Message Broker** | Apache Kafka via Strimzi Operator |
| **Primary Database** | MongoDB (one instance per service) |
| **Cache / Geo** | Redis (Geospatial indexing + concurrency locks via Redlock) |
| **Notifications (Backend)** | Firebase Admin SDK, Twilio SMS, Nodemailer SMTP |
| **Payments (Backend)** | Razorpay API |
| **Containerization** | Docker (multi-stage, `Dockerfile.dev` + `Dockerfile`) |
| **Orchestration** | Kubernetes (GKE Standard) |
| **Local Dev** | Skaffold (`skaffold dev`) |
| **Package Management (K8s)** | Helm |
| **GitOps / CD** | ArgoCD (automated sync + self-heal) |
| **Backend CI** | GitHub Actions (`deploy.yaml`) |
| **Infrastructure as Code** | Terraform (GKE, VPC, IAM, Secrets) |
| **Cloud Provider** | Google Cloud Platform — `asia-south1` |
| **Secret Management** | GCP Secret Manager |
| **Ingress / API Gateway** | NGINX Ingress Controller |

---

## 🚀 Quick Start

### Backend (Local)
```bash
git clone https://github.com/hitendrasinh-Parmar/uber-clone.git
cd uber-clone
skaffold dev
```

### Mobile App
```bash
git clone https://github.com/hitendrasinh-Parmar/rn-uber-clone.git
cd rn_app_uber
npm install

# Terminal 1 — User app Metro
npm run start:user

# Terminal 2 — User app Android
npm run rn:user

# Terminal 3 — Driver app Metro
npm run start:driver

# Terminal 4 — Driver app Android
npm run rn:driver

# Deployment & Distribution

This project uses **Fastlane** for automated builds and distribution via Firebase App Distribution.

## Setup
1. Install Fastlane:
   ```sh
   bundle install
   ```
2. Install Fastlane plugins:
   - For Android: `cd android && bundle exec fastlane install_plugins`

## Distribution Scripts

### Android
- **User App**: `npm run distribute:user`
- **Driver App**: `npm run distribute:driver`
- **All**: `npm run distribute:all`

```

### Production Infrastructure
```bash
git clone https://github.com/hitendrasinh-Parmar/uber-clone-infra.git
cd uber-clone-infra/terraform
# Create terraform.tfvars with your GCP project_id and secrets
terraform init && terraform apply
```

---
## 📸 Screenshots & Demo

🏗️ Infrastructure & Orchestration

https://github.com/user-attachments/assets/bec403e9-20b9-49a6-8769-30e60cd1db23

## Backend

<img width="1122" height="693" alt="Screenshot 2026-04-27 094323" src="https://github.com/user-attachments/assets/47069f02-c694-4f44-be87-1d699f71dc4b" />

📱 Mobile Experience

## Driver App
https://github.com/user-attachments/assets/6a58fe20-9654-4c05-86a4-db35d611433c

## User App
https://github.com/user-attachments/assets/13d29aab-a87a-4037-b90c-08998b776e32


