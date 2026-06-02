# TechCorp - Kurumsal Web Uygulaması ve Bulut Mimarisi Raporu

> **Bartın Üniversitesi - Bilgisayar Mühendisliği**
> **Bulut Bilişim Dersi Dönem Sonu Projesi**
> **Sunum Tarihi:** 3 Haziran 2026

## 👥 Proje Ekibi
* **Kaan Kuzucanlı** (23010310051)
* **Azmi Cankurtaran** (23640310034)
* **Ahmet Nihat Karkaç** (23010310045)

## 🏗️ 1. Proje ve Sistem Mimarisi

TechCorp, modern bulut mimarisi standartlarına uygun, yüke göre kendi kendini ölçekleyebilen (Auto-scaling) ve sürekli entegrasyon/sürekli dağıtım (CI/CD) süreçlerine sahip bir yapı olarak tasarlanmıştır.

* **Frontend & Backend (Monolitik Mikroservis):** Kullanıcı arayüzü Jinja2 şablon motoru ve HTML/CSS ile hazırlanmış, backend tarafında ise Python Flask kullanılmıştır. Gunicorn, WSGI sunucusu olarak kullanılarak (2 worker) eşzamanlı HTTP istekleri optimize edilmiştir.
* **Konteynerizasyon:** Uygulama, imaj boyutunu küçük tutmak ve güvenlik açıklarını minimize etmek amacıyla `python:3.11-slim` baz imajı kullanılarak Dockerize edilmiştir.
* **Orkestrasyon:** Konteynerlerin yönetimi, yük dağıtımı (Load Balancing) ve sağlık kontrolleri (Health Checks) için Google Kubernetes Engine (GKE) tercih edilmiştir.

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

## ☸️ 2. Kubernetes (K8s) Kaynakları ve Mimari Kararlar

Projemizde sistemin sürekliliğini, güvenliğini ve performansını garanti altına almak için çeşitli Kubernetes objeleri kullanılmıştır. Her bir bileşen, belirli bir bulut mimarisi problemini çözmek üzere özel olarak yapılandırılmıştır. `k8s/` dizini altında bulunan manifest dosyalarının işlevleri ve tercih nedenleri aşağıda detaylandırılmıştır:

### 2.1. Yüksek Erişilebilirlik ve Durum Yönetimi (Deployment)
* **Dosya:** `deployment.yaml`
* **Kapsam:** Uygulamanın GKE üzerinde nasıl çalıştırılacağını tanımlar. 
* **Mimari Karar:** Kapsayıcılar (Containers) doğası gereği geçicidir. Deployment objesi sayesinde sistemin "İstenen Durumu (Desired State)" tanımlanmıştır. Herhangi bir Pod donanım arızası veya yazılımsal bir hata sebebiyle çökerse, Kubernetes'in kontrol döngüsü (Control Loop) bunu fark eder ve anında yeni bir Pod ayağa kaldırır (Self-healing). Ayrıca güncellemeler sırasında "Rolling Update" stratejisi kullanılarak kesintisiz geçiş (Zero-downtime) sağlanır.

### 2.2. Dış Trafik Yönetimi (Service & LoadBalancer)
* **Dosya:** `service.yaml`
* **Kapsam:** İnternetten gelen kullanıcı isteklerini karşılar ve arkadaki Pod'lara dağıtır.
* **Mimari Karar:** Cluster dışından uygulamaya erişim sağlamak için `LoadBalancer` tipinde bir servis kullanılmıştır. Dışarıdan gelen HTTP (Port 80) trafiği, sistem tarafından otomatik olarak Pod'ların dinlediği Port 5000'e (TargetPort) yönlendirilir. Bu sayede trafik tek bir noktaya yığılmaz, aktif Pod'lar arasında dengeli bir şekilde paylaştırılır.

### 2.3. Dinamik Kaynak Yönetimi ve Ölçeklendirme (HPA)
* **Dosya:** `hpa.yaml` (Horizontal Pod Autoscaler)
* **Kapsam:** Sisteme gelen yüke göre Pod sayısını otomatik olarak artırır veya azaltır.
* **Mimari Karar:** Bulut bilişimin en büyük avantajlarından olan "Kullandığın kadar öde" prensibini ve performans optimizasyonunu sağlamak için HPA yapılandırılmıştır.
  * **Metrikler:** Ortalama CPU kullanımı %70'i veya Bellek (RAM) kullanımı %80'i aştığında tetiklenir.
  * **Sınırlar:** Sistem boştayken minimum 2 Pod çalışır (Maliyet tasarrufu). Yük altındayken ise sistem kendini maksimum 5 Pod'a kadar otomatik olarak ölçekleyebilir (Performans garantisi).

### 2.4. Ağ Güvenliği ve İzolasyon (NetworkPolicy)
* **Dosya:** `networkpolicy.yaml`
* **Kapsam:** Pod'ların ağ üzerindeki iletişim kurallarını (Ingress/Egress) belirler.
* **Mimari Karar:** "En Az Ayrıcalık Prensibi (Principle of Least Privilege)" ve "Sıfır Güven (Zero-trust)" yaklaşımları benimsenmiştir.
  * **Ingress (Gelen Trafik):** Pod'lara yalnızca belirtilen 5000 portu üzerinden gelecek bağlantılara izin verilir.
  * **Egress (Giden Trafik):** Uygulamanın dış dünyayla iletişimi yalnızca HTTP (80), HTTPS (443) ve DNS çözümlemesi (53/UDP) ile sınırlandırılmıştır. Bu sayede olası bir güvenlik ihlalinde, saldırganın sistem içinden başka portlara veya dışarıdaki zararlı sunuculara erişimi engellenmiştir (Lateral movement koruması).

### 2.5. Kalıcı Veri Yönetimi (PV & PVC)
* **Dosya:** `pv.yaml`, `pvc.yaml`
* **Kapsam:** Konteynerlerin yaşam döngüsünden bağımsız, kalıcı depolama alanı sağlar.
* **Mimari Karar:** Uygulama verilerinin ve logların Pod'lar yeniden başlatıldığında kaybolmasını önlemek için 1Gi (

## 🔄 3. CI/CD Süreci ve Otomasyon (Sürekli Entegrasyon ve Dağıtım)

Modern yazılım geliştirme döngüsünün (SDLC) en kritik parçalarından biri olan CI/CD hattı, bu projede Google Cloud Build kullanılarak tam otomatik hale getirilmiştir. Manuel müdahaleleri ortadan kaldıran bu mimari, kodun güvenli ve kesintisiz bir şekilde canlı ortama alınmasını (Zero-downtime deployment) sağlar.

**Pipeline Akış Adımları:**
1. **Tetikleme (Trigger):** Geliştirici, GitHub deposundaki `main` dalına (branch) yeni bir kod push'ladığında veya Pull Request onaylandığında Cloud Build otomatik olarak tetiklenir.
2. **Derleme (Build):** `cloudbuild.yaml` konfigürasyonu devreye girer. Dockerfile kullanılarak yeni uygulama sürümü derlenir. İmaj, o anki GitHub commit hash'i (`$COMMIT_SHA`) ile etiketlenerek versiyon kontrolü ve izlenebilirlik (traceability) sağlanır.
3. **Kayıt (Push):** Derlenen Docker imajı, güvenli depolama için Google Container Registry'e (GCR) aktarılır (`gcr.io/$PROJECT_ID/techcorp:$COMMIT_SHA`).
4. **Dağıtım (Deploy):** Cloud Build, GKE cluster'ına bağlanarak `kubectl set image` komutunu çalıştırır. Kubernetes, yeni imajı fark ettiğinde "Rolling Update" stratejisini başlatır. Eski Pod'lar kademeli olarak kapatılırken, eşzamanlı olarak yeni Pod'lar trafiğe açılır. Sistem hiçbir an kapalı kalmaz.

---

## ⚙️ 4. Operasyonel Yönetim ve Felaket Kurtarma (Disaster Recovery)

Projenin operasyonel süreçleri salt kurulumdan ibaret olmayıp, hata durumlarında sistemi geriye döndürme (Rollback) senaryolarını da kapsayacak şekilde yapılandırılmıştır.

### 4.1. GKE Cluster ve Alt Yapı Kurulumu
Bulut ortamının komut satırı üzerinden standartize edilmiş kurulum adımları:
```bash
# Google Cloud kimlik doğrulaması ve proje seçimi
gcloud auth login
gcloud config set project YOUR_PROJECT_ID

# 3 Node'lu, e2-medium makine tipli Cluster oluşturulması
gcloud container clusters create techcorp-cluster \
  --num-nodes=3 \
  --zone=europe-west1-b \
  --machine-type=e2-medium

# `kubectl` aracı için kimlik bilgilerinin alınması
gcloud container clusters get-credentials techcorp-cluster --zone=europe-west1-b
### 4.2. K8s Obje Dağıtımı (Deployment)
Sistem manifestolarının sırasıyla Cluster'a uygulanması:
```bash
# Proje ID'sini YAML dosyasında dinamik olarak güncelleme
sed -i 's/YOUR_PROJECT_ID/your-actual-project-id/g' k8s/deployment.yaml

# 1. Kalıcı depolama birimlerinin oluşturulması
kubectl apply -f k8s/pv.yaml
kubectl apply -f k8s/pvc.yaml

# 2. Temel iş yükü ve dışa açılım
kubectl apply -f k8s/deployment.yaml
kubectl apply -f k8s/service.yaml

# 3. Güvenlik ve Ölçeklendirme katmanları
kubectl apply -f k8s/networkpolicy.yaml
kubectl apply -f k8s/hpa.yaml

### 4.3. Versiyon Geri Alma (Rollback) Stratejisi
# Dağıtım geçmişini ve revizyonları görüntüleme
kubectl rollout history deployment/techcorp-deployment

# Bir önceki stabil versiyona anında geri dönme
kubectl rollout undo deployment/techcorp-deployment

# Belirli bir revizyona (örn: versiyon 2) manuel geri dönme
kubectl rollout undo deployment/techcorp-deployment --to-revision=2

KategoriTeknoloji / AraçSürümTercih NedeniUygulama DiliPython3.11Hızlı geliştirme süreci ve geniş kütüphane desteğiWeb FrameworkFlask & Gunicorn3.0.3 / 22.0.0Mikroservis mimarisine uygun hafif yapı ve asenkron WSGI desteğiKonteynerizasyonDockerLatestİşletim sisteminden bağımsız, taşınabilir (portable) uygulama ortamı yaratmakOrkestrasyonKubernetes (GKE)1.28+High Availability, Load Balancing ve Self-healing gereksinimlerini karşılamakCI/CD HattıCloud Build-Geliştirme süreçlerini otomatikleştirerek insan hatasını minimize etmekİmaj YönetimiGCR-Docker imajlarını güvenli ve versiyonlanmış olarak bulutta saklamak
