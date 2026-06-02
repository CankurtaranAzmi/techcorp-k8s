# TechCorp - Kurumsal Web Uygulaması ve Bulut Mimarisi Raporu

> **Bartın Üniversitesi - Bilgisayar Mühendisliği**
> **Bulut Bilişim Dersi Dönem Sonu Projesi**
> **Sunum Tarihi:** 3 Haziran 2026

## 👥 Proje Ekibi
* **Kaan Kuzucanlı** (23010310051)
* **Azmi Cankurtaran** (Öğrenci Numarasını Giriniz)

---

## 🏗️ 1. Proje ve Sistem Mimarisi

TechCorp, modern bulut mimarisi standartlarına uygun, yüke göre kendi kendini ölçekleyebilen (Auto-scaling) ve sürekli entegrasyon/sürekli dağıtım (CI/CD) süreçlerine sahip bir yapı olarak tasarlanmıştır.

* **Frontend & Backend (Monolitik Mikroservis):** Kullanıcı arayüzü Jinja2 şablon motoru ve HTML/CSS ile hazırlanmış, backend tarafında ise Python Flask kullanılmıştır. Gunicorn, WSGI sunucusu olarak kullanılarak (2 worker) eşzamanlı HTTP istekleri optimize edilmiştir.
* **Konteynerizasyon:** Uygulama, imaj boyutunu küçük tutmak ve güvenlik açıklarını minimize etmek amacıyla `python:3.11-slim` baz imajı kullanılarak Dockerize edilmiştir.
* **Orkestrasyon:** Konteynerlerin yönetimi, yük dağıtımı (Load Balancing) ve sağlık kontrolleri (Health Checks) için Google Kubernetes Engine (GKE) tercih edilmiştir.

### Trafik ve Veri Akışı (Traffic Flow)
Dışarıdan gelen bir web isteğinin sistem içindeki yolculuğu aşağıdaki gibidir:

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

---

## ☸️ 2. Kubernetes (K8s) Kaynakları ve Mimari Kararlar

Projemizde sistemin sürekliliğini, güvenliğini ve performansını garanti altına almak için çeşitli Kubernetes objeleri kullanılmıştır. Her bir bileşen, belirli bir bulut mimarisi problemini çözmek üzere özel olarak yapılandırılmıştır. `k8s/` dizini altında bulunan manifest dosyalarının işlevleri ve tercih nedenleri aşağıda detaylandırılmıştır:

### 2.1. Yüksek Erişilebilirlik ve Durum Yönetimi (Deployment)
**Dosya:** `deployment.yaml`
**Kapsam:** Uygulamanın GKE üzerinde nasıl çalıştırılacağını tanımlar. 
**Mimari Karar:** Kapsayıcılar (Containers) doğası gereği geçicidir. Deployment objesi sayesinde sistemin "İstenen Durumu (Desired State)" tanımlanmıştır. Herhangi bir Pod donanım arızası veya yazılımsal bir hata sebebiyle çökerse, Kubernetes'in kontrol döngüsü bunu fark eder ve anında yeni bir Pod ayağa kaldırır (Self-healing). Ayrıca güncellemeler sırasında "Rolling Update" stratejisi kullanılarak kesintisiz geçiş (Zero-downtime) sağlanır.

### 2.2. Dış Trafik Yönetimi (Service & LoadBalancer)
**Dosya:** `service.yaml`
**Kapsam:** İnternetten gelen kullanıcı isteklerini karşılar ve arkadaki Pod'lara dağıtır.
**Mimari Karar:** Cluster dışından uygulamaya erişim sağlamak için `LoadBalancer` tipinde bir servis kullanılmıştır. Dışarıdan gelen HTTP (Port 80) trafiği, sistem tarafından otomatik olarak Pod'ların dinlediği Port 5000'e yönlendirilir. Bu sayede trafik tek bir noktaya yığılmaz, aktif Pod'lar arasında dengeli bir şekilde paylaştırılır.

### 2.3. Dinamik Kaynak Yönetimi ve Ölçeklendirme (HPA)
**Dosya:** `hpa.yaml` (Horizontal Pod Autoscaler)
**Kapsam:** Sisteme gelen yüke göre Pod sayısını otomatik olarak artırır veya azaltır.
**Mimari Karar:** "Kullandığın kadar öde" prensibini ve performans optimizasyonunu sağlamak için HPA yapılandırılmıştır. Ortalama CPU kullanımı %70'i veya Bellek (RAM) kullanımı %80'i aştığında tetiklenir. Sistem boştayken minimum 2 Pod çalışır, yük altındayken maksimum 5 Pod'a kadar otomatik ölçeklenir.

### 2.4. Ağ Güvenliği ve İzolasyon (NetworkPolicy)
**Dosya:** `networkpolicy.yaml`
**Kapsam:** Pod'ların ağ üzerindeki iletişim kurallarını belirler.
**Mimari Karar:** "Sıfır Güven (Zero-trust)" yaklaşımı benimsenmiştir. Ingress tarafında yalnızca `5000` portuna izin verilirken, Egress tarafında sadece HTTP (80), HTTPS (443) ve DNS (53) trafiğine izin verilerek potansiyel saldırı yüzeyi daraltılmıştır.

### 2.5. Kalıcı Veri Yönetimi (PV & PVC)
**Dosya:** `pv.yaml`, `pvc.yaml`
**Kapsam:** Konteynerlerin yaşam döngüsünden bağımsız, kalıcı depolama alanı sağlar.
**Mimari Karar:** Uygulama verilerinin Pod'lar yeniden başlatıldığında kaybolmasını önlemek için 1Gi kapasiteli Persistent Volume yapılandırılmış ve `ReadWriteOnce` erişim moduyla `/app/data` dizinine bağlanmıştır.

---

## 🔄 3. CI/CD Süreci ve Otomasyon (Sürekli Entegrasyon ve Dağıtım)

CI/CD hattı, Google Cloud Build kullanılarak tam otomatik hale getirilmiştir. 

**Pipeline Akış Adımları:**
1. **Tetikleme (Trigger):** GitHub `main` dalına yeni bir kod push'landığında süreç otomatik başlar.
2. **Derleme (Build):** `cloudbuild.yaml` devreye girer. Dockerfile ile yeni uygulama derlenir ve anlık commit hash'i ile etiketlenir.
3. **Kayıt (Push):** Derlenen Docker imajı Google Container Registry'e (GCR) aktarılır.
4. **Dağıtım (Deploy):** Cloud Build, GKE cluster'ına bağlanır. Yeni imaj sisteme uygulanarak "Rolling Update" başlatılır ve kesintisiz geçiş sağlanır.

---

## ⚙️ 4. Operasyonel Yönetim ve Felaket Kurtarma (Disaster Recovery)

Hata durumlarında sistemi geriye döndürme (Rollback) senaryolarını da kapsayan operasyonel adımlar:

### 4.1. GKE Cluster ve Alt Yapı Kurulumu
    gcloud auth login
    gcloud config set project YOUR_PROJECT_ID

    gcloud container clusters create techcorp-cluster \
      --num-nodes=3 \
      --zone=europe-west1-b \
      --machine-type=e2-medium

    gcloud container clusters get-credentials techcorp-cluster --zone=europe-west1-b

### 4.2. K8s Obje Dağıtımı (Deployment)
    sed -i 's/YOUR_PROJECT_ID/your-actual-project-id/g' k8s/deployment.yaml

    kubectl apply -f k8s/pv.yaml
    kubectl apply -f k8s/pvc.yaml
    kubectl apply -f k8s/deployment.yaml
    kubectl apply -f k8s/service.yaml
    kubectl apply -f k8s/networkpolicy.yaml
    kubectl apply -f k8s/hpa.yaml

### 4.3. Versiyon Geri Alma (Rollback) Stratejisi
    # Dağıtım geçmişini görüntüleme
    kubectl rollout history deployment/techcorp-deployment

    # Bir önceki stabil versiyona geri dönme
    kubectl rollout undo deployment/techcorp-deployment

---

## 🛠️ 5. Kullanılan Teknoloji Yığını (Tech Stack)

| Kategori | Teknoloji / Araç | Sürüm | Tercih Nedeni |
| :--- | :--- | :--- | :--- |
| **Uygulama Dili** | Python | 3.11 | Hızlı geliştirme süreci ve geniş kütüphane desteği |
| **Web Framework** | Flask & Gunicorn | 3.0.3 / 22.0.0 | Mikroservis mimarisine uygun hafif yapı ve asenkron WSGI desteği |
| **Konteynerizasyon** | Docker | Latest | İşletim sisteminden bağımsız, taşınabilir (portable) ortam |
| **Orkestrasyon** | Kubernetes (GKE)| 1.28+ | High Availability, Load Balancing ve Self-healing gereksinimleri |
| **CI/CD Hattı** | Cloud Build | - | Geliştirme süreçlerini otomatikleştirerek insan hatasını minimize etmek |
| **İmaj Yönetimi** | GCR | - | Docker imajlarını güvenli ve versiyonlanmış olarak saklamak |
