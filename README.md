# TechCorp - Kurumsal Web Uygulaması

> Flask tabanlı kurumsal web sitesi — Docker, Kubernetes (GKE) ve Cloud Build CI/CD ile deploy edilmiştir.

---

## 📁 Proje Yapısı

```
corporate-site/
├── app/
│   ├── app.py                  # Flask uygulaması
│   ├── requirements.txt        # Python bağımlılıkları
│   ├── templates/              # HTML şablonları
│   │   ├── base.html
│   │   ├── index.html
│   │   ├── hakkimizda.html
│   │   ├── hizmetler.html
│   │   └── iletisim.html
│   └── static/
│       └── css/
│           └── style.css
├── k8s/
│   ├── deployment.yaml         # Kubernetes Deployment
│   ├── service.yaml            # Kubernetes Service (LoadBalancer)
│   ├── pv.yaml                 # Persistent Volume
│   ├── pvc.yaml                # Persistent Volume Claim
│   ├── networkpolicy.yaml      # Network Policy
│   └── hpa.yaml                # Horizontal Pod Autoscaler
├── Dockerfile                  # Docker image tanımı
├── cloudbuild.yaml             # CI/CD pipeline (Cloud Build)
└── README.md
```

---

## 🏗️ Uygulama Mimarisi

```
Kullanıcı → LoadBalancer Service (port 80)
              ↓
         Kubernetes Pod (Flask app, port 5000)
              ↓
         Persistent Volume (uygulama verisi)
```

- **Frontend:** HTML/CSS (Jinja2 template engine)
- **Backend:** Python Flask + Gunicorn (2 worker)
- **Container:** Docker (python:3.11-slim base image)
- **Servis:** 4 sayfa — Ana Sayfa, Hakkımızda, Hizmetler, İletişim

---

## ☸️ Kubernetes Mimarisi

```
GKE Cluster
├── Deployment (techcorp-deployment)
│   ├── Pod 1 (techcorp container)
│   ├── Pod 2 (techcorp container)
│   └── Pod 3 (techcorp container)
├── Service (techcorp-service) → LoadBalancer
├── HorizontalPodAutoscaler (min:2, max:5)
├── PersistentVolume (1Gi)
├── PersistentVolumeClaim (1Gi)
└── NetworkPolicy (ingress/egress kuralları)
```

---

## 🔄 CI/CD Pipeline Akışı

```
GitHub Push
    ↓
Cloud Build Tetiklenir
    ↓
1. Docker image build (gcr.io/$PROJECT_ID/techcorp:$COMMIT_SHA)
    ↓
2. Image → Google Container Registry'e push
    ↓
3. GKE cluster'a bağlan
    ↓
4. kubectl set image → Rolling Update başlar
    ↓
Yeni Pod'lar ayağa kalkar → Eski Pod'lar kaldırılır
```

---

## 🚀 Kurulum ve Çalıştırma

### Gereksinimler
- Google Cloud hesabı
- `gcloud` CLI kurulu
- `kubectl` kurulu
- Docker kurulu

### 1. GKE Cluster Oluşturma

```bash
# Google Cloud'a giriş
gcloud auth login

# Proje seçimi
gcloud config set project YOUR_PROJECT_ID

# GKE cluster oluştur
gcloud container clusters create techcorp-cluster \
  --num-nodes=3 \
  --zone=europe-west1-b \
  --machine-type=e2-medium

# Cluster'a bağlan
gcloud container clusters get-credentials techcorp-cluster \
  --zone=europe-west1-b
```

### 2. Docker Image Build & Push

```bash
# Image oluştur
docker build -t gcr.io/YOUR_PROJECT_ID/techcorp:v1 .

# GCR'a push et
docker push gcr.io/YOUR_PROJECT_ID/techcorp:v1
```

### 3. Kubernetes Manifest'lerini Uygula

```bash
# deployment.yaml içindeki YOUR_PROJECT_ID'yi güncelleyin!
sed -i 's/YOUR_PROJECT_ID/your-actual-project-id/g' k8s/deployment.yaml

# PV ve PVC oluştur
kubectl apply -f k8s/pv.yaml
kubectl apply -f k8s/pvc.yaml

# Deployment ve Service
kubectl apply -f k8s/deployment.yaml
kubectl apply -f k8s/service.yaml

# NetworkPolicy
kubectl apply -f k8s/networkpolicy.yaml

# HPA (Autoscaling)
kubectl apply -f k8s/hpa.yaml
```

### 4. Uygulamaya Erişim

```bash
# External IP'yi al (1-2 dakika bekleyin)
kubectl get service techcorp-service

# Pod durumlarını kontrol et
kubectl get pods
```

---

## 🔁 Rolling Update

GitHub'a yeni commit push edildiğinde Cloud Build otomatik olarak çalışır ve rolling update yapar. Manuel olarak da yapılabilir:

```bash
# Yeni image ile update
kubectl set image deployment/techcorp-deployment \
  techcorp=gcr.io/YOUR_PROJECT_ID/techcorp:v2

# Update durumunu izle
kubectl rollout status deployment/techcorp-deployment
```

---

## ⏪ Rollback

```bash
# Bir önceki versiyona geri dön
kubectl rollout undo deployment/techcorp-deployment

# Belirli bir revizyona geri dön
kubectl rollout undo deployment/techcorp-deployment --to-revision=2

# Revizyon geçmişini gör
kubectl rollout history deployment/techcorp-deployment
```

---

## 📈 Scaling

```bash
# Manuel scaling — replica sayısını artır
kubectl scale deployment techcorp-deployment --replicas=5

# HPA durumunu izle (otomatik scaling)
kubectl get hpa techcorp-hpa

# HPA detayları
kubectl describe hpa techcorp-hpa
```

HPA, CPU kullanımı %70'i veya bellek kullanımı %80'i geçtiğinde otomatik olarak pod sayısını 2'den 5'e kadar artırır.

---

## 🛡️ NetworkPolicy

`networkpolicy.yaml` ile:
- **Ingress:** Sadece 5000 numaralı porttan gelen trafiğe izin verilir
- **Egress:** Sadece HTTP (80), HTTPS (443) ve DNS (53/UDP) çıkışına izin verilir
- Diğer tüm trafik engellenir

---

## 💾 Persistent Volume / PVC

- `pv.yaml`: 1Gi kapasiteli, `ReadWriteOnce` erişim modlu PersistentVolume tanımlar
- `pvc.yaml`: PV'ye bağlanan PersistentVolumeClaim tanımlar
- `deployment.yaml`: `/app/data` dizinine PVC mount edilir

---

## ⚙️ CI/CD — Cloud Build Kurulumu

1. GitHub reponuzu Google Cloud Source Repositories ile bağlayın
2. Cloud Build > Triggers > Yeni Trigger oluşturun
3. Tetikleyici: `main` branch'e push
4. Config dosyası: `cloudbuild.yaml`

```bash
# Cloud Build trigger oluştur (gcloud ile)
gcloud builds triggers create github \
  --repo-name=corporate-site \
  --repo-owner=YOUR_GITHUB_USERNAME \
  --branch-pattern=^main$ \
  --build-config=cloudbuild.yaml
```

---

## 🛠️ Kullanılan Teknolojiler

| Teknoloji | Versiyon | Kullanım Amacı |
|---|---|---|
| Python | 3.11 | Uygulama dili |
| Flask | 3.0.3 | Web framework |
| Gunicorn | 22.0.0 | WSGI sunucu |
| Docker | latest | Containerization |
| Kubernetes | 1.28+ | Orchestration |
| GKE | - | Yönetilen K8s |
| Cloud Build | - | CI/CD pipeline |
| GCR | - | Container registry |

---

## 👥 Grup Bilgileri

- **Ders:** Bulut Bilişim
- **Sunum Tarihi:** 3 Haziran 2025
- **Grup Üyeleri:** [İsimlerinizi ekleyin]
