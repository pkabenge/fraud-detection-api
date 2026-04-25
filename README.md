# AI-Powered Fraud Detection API

Real-time ML inference API for transaction fraud detection, built with **FastAPI**, **scikit-learn**, and **GradientBoosting (GBoost)**. Containerized with **Docker**, orchestrated via **Kubernetes**, and monitored with **AWS CloudWatch**.

---

## Architecture

```
Transaction Request
      │
      ▼
 FastAPI /predict
      │
      ▼
GBoost Pipeline (scikit-learn)
  ├─ StandardScaler
  └─ GradientBoostingClassifier
      │
      ▼
Fraud Probability + Risk Level
      │
      ▼
 CloudWatch Logs / Metrics
```

**Deployment stack:**
```
GitHub Actions CI/CD
  └─ Build Docker image
  └─ Push to AWS ECR
  └─ kubectl rollout → EKS
        └─ 3-10 pods (HPA)
        └─ AWS ALB (HTTPS)
        └─ CloudWatch monitoring
```

---

## Project Structure

```
fraud-detection-api/
├── app/
│   └── main.py               # FastAPI application
├── model/
│   └── train_model.py        # Model training script
├── k8s/
│   ├── namespace.yaml        # Kubernetes namespace
│   ├── deployment.yaml       # Deployment (3 replicas, probes)
│   ├── service.yaml          # ClusterIP service + ALB Ingress
│   └── hpa.yaml              # Horizontal Pod Autoscaler
├── tests/
│   └── test_api.py           # Pytest test suite
├── .github/
│   └── workflows/
│       └── deploy.yml        # GitHub Actions CI/CD pipeline
├── Dockerfile                # Multi-stage Docker build
├── requirements.txt
└── README.md
```

---

## Quickstart (Local)

### 1. Train the model
```bash
pip install -r requirements.txt
python model/train_model.py
```

### 2. Run the API
```bash
uvicorn app.main:app --reload --port 8000
```

### 3. Test a prediction
```bash
curl -X POST http://localhost:8000/predict \
  -H "Content-Type: application/json" \
  -d '{
    "amount": 1200.00,
    "hour_of_day": 3,
    "distance_from_home": 250.0,
    "num_transactions_24h": 12,
    "is_foreign": true,
    "is_online": true
  }'
```

**Response:**
```json
{
  "transaction_id": "a3f2...",
  "is_fraud": true,
  "fraud_probability": 0.8743,
  "risk_level": "HIGH",
  "latency_ms": 2.1
}
```

Visit `http://localhost:8000/docs` for the full Swagger UI.

---

## Docker

```bash
# Build
docker build -t fraud-detection-api:latest .

# Run
docker run -p 8000:8000 fraud-detection-api:latest

# Push to ECR
aws ecr get-login-password --region us-east-1 | \
  docker login --username AWS --password-stdin <ACCOUNT>.dkr.ecr.us-east-1.amazonaws.com

docker tag fraud-detection-api:latest <ACCOUNT>.dkr.ecr.us-east-1.amazonaws.com/fraud-detection-api:latest
docker push <ACCOUNT>.dkr.ecr.us-east-1.amazonaws.com/fraud-detection-api:latest
```

---

## Kubernetes

```bash
# Apply all manifests
kubectl apply -f k8s/namespace.yaml
kubectl apply -f k8s/deployment.yaml
kubectl apply -f k8s/service.yaml
kubectl apply -f k8s/hpa.yaml

# Check pods
kubectl get pods -n ml-services

# Watch rollout
kubectl rollout status deployment/fraud-detection-api -n ml-services
```

> Update `k8s/deployment.yaml` with your ECR image URI before applying.
> Update `k8s/service.yaml` with your ACM certificate ARN for HTTPS.

---

## CI/CD (GitHub Actions)

The pipeline runs automatically on push to `main`:

| Step | Action |
|------|--------|
| **Test** | Trains model, runs pytest suite |
| **Build** | Docker multi-stage build |
| **Push** | Push image to AWS ECR (tagged with commit SHA) |
| **Deploy** | `kubectl set image` → rolling update on EKS |

**Required GitHub Secrets:**
- `AWS_ACCESS_KEY_ID`
- `AWS_SECRET_ACCESS_KEY`

---

## Monitoring (AWS CloudWatch)

All predictions are logged with structured JSON including:
- Transaction ID
- Fraud label + probability
- Risk level
- Inference latency (ms)

Configure a CloudWatch Log Group for `/ml-services/fraud-api` and set alarms on:
- High fraud rate (> 5% in 5min window)
- P99 latency > 100ms
- Pod crash loops (via EKS Container Insights)

---

## Author
**Polydor Kabengele** — DevSecOps Engineer | ML Engineer  
pkabengel1@gmail.com | pollykabengele@yahoo.fr  
github.com/polydor | linkedin.com/in/polydor-kabengele
