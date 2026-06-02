# TechCorp - Kurumsal Web Uygulaması ve Bulut Mimarisi Raporu

> **Bartın Üniversitesi - Bilgisayar Mühendisliği**
> **Bulut Bilişim Dersi Dönem Sonu Projesi**
> **Sunum Tarihi:** 3 Haziran 2026

## 👥 Proje Ekibi
* **Kaan Kuzucanlı** (23010310051)
* **Azmi Cankurtaran** (23640310034)
* **Ahmet Nihat Karkaç** (23010310045)

---

## 📄 Proje Özeti ve Amacı

Bu proje, kurumsal bir web uygulamasının modern bulut bilişim prensiplerine uygun olarak Google Kubernetes Engine (GKE) üzerinde dağıtılmasını amaçlamaktadır. Projenin temel motivasyonu; yüksek erişilebilirlik (High Availability), yüke göre otomatik ölçeklendirme (Auto-scaling) ve sürekli entegrasyon/dağıtım (CI/CD) gibi endüstri standartlarındaki mimari konseptleri gerçek bir senaryo üzerinde uygulamaktır. 

Geliştirilen Flask tabanlı web uygulaması, Docker ile konteynerize edilmiş ve Cloud Build kullanılarak sıfır kesinti (zero-downtime) prensibiyle tam otomatik bir dağıtım hattına entegre edilmiştir. Bu sayede hem operasyonel maliyetlerin optimize edilmesi hem de insan hatasından arındırılmış, güvenli bir bulut altyapısı tasarlanması hedeflenmiştir.

---

## 📁 1. Proje Dizin Yapısı

Projemiz, kodun okunabilirliğini ve yönetilebilirliğini artırmak için mikroservis mimarisi mantığına uygun şekilde modüllere ayrılmıştır:

```text
corporate-site/
├── app/
│   ├── app.py                  # Backend: Flask uygulama mantığı
│   ├── requirements.txt        # Python bağımlılıkları (Gunicorn, Flask vb.)
│   ├── templates/              # Frontend: Jinja2 HTML şablonları
│   └── static/
│       └── css/
│           └── style.css       # Tasarım dosyaları
├── k8s/
│   ├── deployment.yaml         # K8s: Pod yönetimi ve replikalar
│   ├── service.yaml            # K8s: LoadBalancer yapılandırması
│   ├── pv.yaml                 # K8s: Kalıcı veri depolama birimi
│   ├── pvc.yaml                # K8s: Depolama alanı talebi
│   ├── networkpolicy.yaml      # K8s: Güvenlik duvarı kuralları
│   └── hpa.yaml                # K8s: Otomatik ölçeklendirme
├── Dockerfile                  # Konteyner imajı oluşturma talimatları
├── cloudbuild.yaml             # Google Cloud CI/CD pipeline yapılandırması
└── README.md                   # Proje dokümantasyonu
```

---

## 🏗️ 2. Proje ve Sistem Mimarisi

* **Frontend & Backend:** Kullanıcı arayüzü Jinja2 şablon motoru ve HTML/CSS ile hazırlanmış, backend tarafında Python Flask kullanılmıştır. Gunicorn, WSGI sunucusu olarak kullanılarak (2 worker) eşzamanlı HTTP istekleri optimize edilmiştir.
* **Konteynerizasyon:** Uygulama, imaj boyutunu küçük tutmak ve güvenlik açıklarını minimize etmek amacıyla `python:3.11-slim` baz imajı kullanılarak Dockerize edilmiştir.
* **Orkestrasyon:** Konteynerlerin yönetimi, yük dağıtımı (Load Balancing) ve sağlık kontrolleri için Google Kubernetes Engine (GKE) tercih edilmiştir.

### Trafik ve Veri Akışı (Traffic Flow)
Dışarıdan gelen bir web isteğinin sistem içindeki yolculuğu aşağıdaki gibidir:

```text
[ İnternet / Kullanıcı ] 
           │
           ▼
[ LoadBalancer Service ] (Gelen trafiği K8s cluster'ına alır ve dengeler - Port 80)
           │
           ├─────────► [ Pod 1 ] (Flask App - Port 5000)
           ├─────────► [ Pod 2 ] (Flask App - Port 5000)
           └─────────► [ Pod 3 ] (Flask App - Port 5000)
                         │
                         ▼
                 [ Persistent Volume ] (Kalıcı uygulama verisi - /app/data)
```

---

## ☸️ 3. Kubernetes (K8s) Kaynakları ve Mimari Kararlar

Projemizde sistemin sürekliliğini ve güvenliğini garanti altına almak için aşağıdaki yapılandırmalar tercih edilmiştir:

### 3.1. Yüksek Erişilebilirlik (Deployment)
**Dosya:** `deployment.yaml`
Kapsayıcılar doğası gereği geçicidir. Deployment objesi sayesinde sistemin "İstenen Durumu (Desired State)" tanımlanmıştır. Herhangi bir Pod çökerse, Kubernetes kontrol döngüsü anında yeni bir Pod ayağa kaldırır (Self-healing). 

### 3.2. Dış Trafik Yönetimi (Service & LoadBalancer)
**Dosya:** `service.yaml`
Cluster dışından erişim için `LoadBalancer` servisi kullanılmıştır. Dışarıdan gelen HTTP (Port 80) trafiği, sistem tarafından otomatik olarak Pod'ların dinlediği Port 5000'e yönlendirilir ve trafik aktif Pod'lar arasında dengeli paylaştırılır.

### 3.3. Dinamik Kaynak Yönetimi (HPA)
**Dosya:** `hpa.yaml`
Performans optimizasyonu için HPA yapılandırılmıştır. CPU kullanımı %70'i veya Bellek kullanımı %80'i aştığında tetiklenir. Sistem boştayken minimum 2 Pod çalışır, yük altındayken maksimum 5 Pod'a kadar otomatik ölçeklenir.

### 3.4. Ağ Güvenliği (NetworkPolicy)
**Dosya:** `networkpolicy.yaml`
"Sıfır Güven (Zero-trust)" yaklaşımı benimsenmiştir. Ingress tarafında yalnızca `5000` portuna, Egress tarafında sadece HTTP (80), HTTPS (443) ve DNS (53) trafiğine izin verilerek saldırı yüzeyi daraltılmıştır.

### 3.5. Kalıcı Veri Yönetimi (PV & PVC)
**Dosya:** `pv.yaml`, `pvc.yaml`
Uygulama verilerinin kaybolmasını önlemek için 1Gi kapasiteli Persistent Volume yapılandırılmış ve `ReadWriteOnce` erişim moduyla `/app/data` dizinine bağlanmıştır.

---

## 🔄 4. CI/CD Süreci ve Otomasyon

CI/CD hattı, Google Cloud Build kullanılarak tam otomatik hale getirilmiştir. 

1. **Tetikleme:** GitHub `main` dalına kod push'landığında süreç otomatik başlar.
2. **Derleme:** `cloudbuild.yaml` devreye girer. Dockerfile ile imaj derlenir.
3. **Kayıt:** Derlenen imaj Google Container Registry'e (GCR) aktarılır.
4. **Dağıtım:** Cloud Build, GKE cluster'ına bağlanır ve "Rolling Update" başlatarak kesintisiz geçiş sağlar.

---

## ⚙️ 5. Operasyonel Yönetim, Ölçekleme ve Kurtarma

Aşağıdaki komut setleri, sistemin manuel olarak yönetilmesi ve olası felaket senaryolarında kurtarılması için belgelenmiştir.

### 5.1. Alt Yapı ve Obje Kurulumu
```bash
# Cluster Oluşturma ve Bağlanma
gcloud container clusters create techcorp-cluster --num-nodes=3 --zone=europe-west1-b --machine-type=e2-medium
gcloud container clusters get-credentials techcorp-cluster --zone=europe-west1-b

# K8s Objelerini Uygulama
kubectl apply -f k8s/pv.yaml
kubectl apply -f k8s/pvc.yaml
kubectl apply -f k8s/deployment.yaml
kubectl apply -f k8s/service.yaml
kubectl apply -f k8s/networkpolicy.yaml
kubectl apply -f k8s/hpa.yaml
```

### 5.2. Versiyon Geri Alma (Rollback) Stratejisi
Hatalı bir güncelleme sonrası sistemi stabil versiyona döndürme:

```bash
# Dağıtım geçmişini ve revizyonları görüntüleme
kubectl rollout history deployment/techcorp-deployment

# Bir önceki stabil versiyona anında geri dönme
kubectl rollout undo deployment/techcorp-deployment
```

---

## 🛠️ 6. Kullanılan Teknoloji Yığını (Tech Stack)

| Kategori | Teknoloji / Araç | Sürüm | Tercih Nedeni |
| :--- | :--- | :--- | :--- |
| **Uygulama Dili** | Python | 3.11 | Hızlı geliştirme süreci ve geniş kütüphane desteği |
| **Web Framework** | Flask & Gunicorn | 3.0.3 / 22.0.0 | Mikroservis mimarisine uygun hafif yapı |
| **Konteynerizasyon** | Docker | Latest | İşletim sisteminden bağımsız, taşınabilir ortam |
| **Orkestrasyon** | Kubernetes (GKE)| 1.28+ | Yük dengeleme ve kendini onarma yetenekleri |
| **CI/CD Hattı** | Cloud Build | - | Dağıtım süreçlerini otomatikleştirip hatayı minimize etmek |

---

## 🎯 7. Sonuç ve Kazanımlar

Bu proje kapsamında, tek başına çalışan yerel bir uygulamanın (monolitik Flask) modern bulut teknolojileri kullanılarak nasıl ölçeklenebilir ve dayanıklı bir sisteme dönüştürüleceği uygulamalı olarak test edilmiştir. 

Kubernetes'in sunduğu HPA ve NetworkPolicy gibi yetenekler sayesinde kaynak kullanımı optimize edilmiş ve ağ güvenliği artırılmıştır. Ayrıca Cloud Build entegrasyonu ile koda yapılan her müdahalenin anında, manuel bir müdahale gerektirmeden canlı sisteme yansıması başarıyla sağlanmıştır. Projenin ilerleyen fazlarında; sistem sağlığını canlı izlemek adına Prometheus ve Grafana araçlarının entegre edilmesi ve veritabanı katmanının mikroservis olarak dışarı çıkarılması hedeflenmektedir.
