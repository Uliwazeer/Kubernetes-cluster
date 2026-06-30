


https://computingforgeeks.com/install-harbor-registry-centos-debian-ubuntu/


-----

https://www.youtube.com/watch?v=Cpi27cWhYp8


|   |
|---|
|# Switch to root user|
|sudo su|
||
|# Update and upgrade the system|
|apt update && sudo apt -y full-upgrade|
||
|# Install required packages for HTTPS and software properties|
|sudo apt install -y apt-transport-https ca-certificates curl software-properties-common|
||
|# Ensure certificates and curl are installed|
|sudo apt-get update|
|sudo apt-get install -y ca-certificates curl|
||
|# Create directory for Docker GPG key|
|sudo install -m 0755 -d /etc/apt/keyrings|
||
|# Download and add Docker GPG key|
|sudo curl -fsSL https://download.docker.com/linux/ubuntu/gpg -o /etc/apt/keyrings/docker.asc|
|sudo chmod a+r /etc/apt/keyrings/docker.asc|
||
|# Add Docker repository|
|echo "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.asc] \|
|https://download.docker.com/linux/ubuntu \|
|$(. /etc/os-release && echo "${UBUNTU_CODENAME:-$VERSION_CODENAME}") stable" \| \|
|sudo tee /etc/apt/sources.list.d/docker.list > /dev/null|
||
|# Update package list|
|sudo apt-get update|
||
|# Install Docker Engine, CLI, containerd, buildx, and compose plugin|
|sudo apt-get install -y docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin|
||
|# Download Harbor offline installer|
|curl -LO https://github.com/goharbor/harbor/releases/download/v2.12.3/harbor-offline-installer-v2.12.3.tgz|
||
|# Extract the Harbor installer|
|tar zxvf harbor-offline-installer-v2.12.3.tgz|
|cd harbor|
|ls -al|
||
|# Copy and edit Harbor configuration template|
|cp harbor.yml.tmpl harbor.yml|
|nano harbor.yml|
||
|# Run Harbor installation|
|./install.sh|
||
|# Login to Harbor registry|
|docker login 192.168.0.12:8080|
||
|# Pull a test Docker image|
|docker pull hello-world|
||
|# Tag the image for Harbor registry|
|docker tag hello-world 192.168.0.12:8080/library/hello-world|
||
|# Push the image to Harbor|
|docker push 192.168.0.12:8080/library/hello-world|


---
أنت بدأت تفكر بالطريقة الصح، لأن في الشركات الكبيرة **أول حاجة بتتكلم مع Security قبل ما تعمل أي VM لها Internet**.

لكن خليني أوضح حاجة مهمة:

> **ما تطلبش من الـ Security "افتحلي Internet".**
> 
> اطلب **Whitelist** محددة جدًا (Egress فقط) من VM الـ Harbor، ولمدة مؤقتة لو أمكن.

وده هو الـ Best Practice.

---

# السيناريو الكامل

هنفترض إنك لسه فى البداية.

## المرحلة 1

تعمل VM

مثلاً

```text
Hostname:
harbor-test

IP:
172.30.0.70

OS:
Ubuntu 24.04

CPU:
4 vCPU

RAM:
8 GB

Disk:
250 GB
```

---

## المرحلة 2

تعمل لها

- Static IP
    
- DNS
    
- NTP
    
- SSH
    

لكن **لسه مفيش Internet**.

---

## المرحلة 3

تبعت Request للـ Security Team.

الطلب هيكون تقريبًا بالشكل ده:

> نحتاج السماح للـ VM التالية:
> 
> IP: **172.30.0.70**
> 
> بخروج (Outbound/Egress) عبر HTTPS إلى مجموعة محددة من المواقع المطلوبة لتحميل صور Kubernetes ومكونات الـ Platform.
> 
> لا نحتاج أي Inbound من الإنترنت.

لاحظ:

- **Outbound فقط.**
    
- **لا تطلب فتح RDP أو SSH من الإنترنت.**
    
- **لا تطلب Any/Any.**
    

---

# هيحتاج يطلع على إيه؟

يعتمد على اللى هتنزله.

لو هتبنى Cluster زى الإنتاج (kubeadm + Cilium + Ingress + Monitoring)، فغالبًا هتحتاج:

## Kubernetes Images

```text
registry.k8s.io
```

---

## Cilium

```text
quay.io
```

---

## GitHub Releases

```text
github.com
```

و

```text
raw.githubusercontent.com
```

و

```text
objects.githubusercontent.com
```

و

```text
release-assets.githubusercontent.com
```

---

## Helm

لو هتستخدم Helm

```text
ghcr.io
```

وأحيانًا مستودعات الـ Helm Charts حسب الأدوات اللي هتنزلها.

---

## Ubuntu

لأن Harbor VM Ubuntu

```text
archive.ubuntu.com
security.ubuntu.com
```

أو لو عندكم Local APT Repository، يبقى **مش هتحتاج دول**.

---

## Docker

لو هتستخدم Docker Engine

```text
download.docker.com
```

لكن لو هتثبت Docker من الـ Local Repo أو بطريقة داخلية، يبقى مش محتاج.

---

# البورتات

تقريبًا كلها:

|البروتوكول|البورت|
|---|---|
|HTTPS|443|
|HTTP|80 (لو فيه Redirect أو Repository قديم)|

يعني غالبًا يكفي:

```text
TCP 443
```

وHTTP لو عندكم احتياج.

---

# بعد ما Security يوافق

الخطوات هتكون:

```
Install Ubuntu

↓

Update Packages

↓

Install Docker

↓

Install Harbor

↓

Harbor جاهز

↓

Login

↓

Create Project

↓

Pull Images

↓

Tag Images

↓

Push Images

↓

Close Internet Access (اختياري لكنه أفضل)

↓

Nodes Pull from Harbor فقط
```

---

# بعد ما Harbor يشتغل

أنت مش هتخلي الـ Nodes تطلع Internet.

الـ Workflow هيكون:

```
Internet

↓

Harbor VM فقط

↓

docker pull

↓

docker tag

↓

docker push

↓

Harbor

↓

Worker1
Worker2
CP1
CP2
CP3
```

---

# طيب الـ Nodes هيحتاجوا إيه؟

تقريبًا:

```
Node

↓

HTTPS

↓

172.30.0.70
```

يعني كل الـ Nodes محتاجة توصل لـ Harbor.

لو Harbor هيشتغل بـ HTTPS يبقى افتح:

```
TCP 443
```

ولو هتشغله بدون TLS للتجارب (مش أنصح بيه)، يبقى غالبًا:

```
TCP 80
```

أو منفذ Registry غير قياسي لو اخترته.

---

# أنا لو مكانك

هعمل السياسة دى.

```
Harbor VM

Outbound:

443

↓

registry.k8s.io

↓

quay.io

↓

github.com

↓

raw.githubusercontent.com

↓

ghcr.io

↓

archive.ubuntu.com

↓

security.ubuntu.com
```

وبعد ما أخلص تحميل كل الصور:

**أبعت للـ Security يقفل الـ Internet تانى.**

ويبقى:

```
Harbor

↓

Workers فقط
```

---

# لكن...

**دى مش الطريقة اللى الشركات الكبيرة كلها بتعملها.**

اللى بيحصل فى المؤسسات الكبيرة (خصوصًا البنوك، البورصات، والجهات الحكومية) هو إنهم غالبًا **ما بيفتحوش Harbor على الإنترنت مباشرة**.

بدل كده بيبقى فيه:

```
Internet
      │
      ▼
Corporate Proxy
      │
      ▼
Harbor
```

أو بيبقى فيه **Repository Management Server** (مثل Nexus أو Artifactory) هو اللى يطلع للإنترنت، وHarbor يعمل منه Replication أو يتم تغذيته داخليًا.

## قبل ما تطلب من الـ Security أي حاجة، اسألهم سؤالين مهمين:

1. **هل عند الشركة Proxy للخروج للإنترنت؟** (زي Squid أو Proxy مؤسسي).
    
2. **هل فيه بالفعل Nexus أو Artifactory أو Harbor داخلي؟**
    

لو الإجابة "أيوه"، فغالبًا **مش هتحتاج Internet على Harbor أصلًا**، وهتبقى الخطة مختلفة وأقرب للـ Production.

### نصيحتي

**ما تبعتش طلب فتح URLs دلوقتي.**  
ابدأ بسؤال فريق الـ Infrastructure أو الـ Security:

> "هل عندنا Enterprise Proxy أو Internal Container Registry (Harbor/Nexus/Artifactory) أقدر أستخدمه بدل فتح Internet مباشرة؟"

الإجابة على السؤال ده ممكن توفر عليك إنشاء Harbor جديد أو تقلل جدًا عدد الاستثناءات الأمنية المطلوبة.

---
فكرتك صح، لكن هعدلها شوية لأن فيه **مفهوم مهم في الـ Firewall Rules**.

أنت كتبت:

> Source = جهازي 192.168.11.64

وده هيستخدم **بس لو جهازك هو اللي هيطلع يحمل كل حاجة**.

لكن فى السيناريو اللى اتفقنا عليه، المفروض يبقى:

```text
Internet
      │
      ▼
Harbor VM
      │
      ▼
Kubernetes Nodes
```

يعنى **الـ Harbor VM هى الوحيدة اللى هتطلع Internet**.

وليس جهازك.

جهازك هيبقى:

- SSH
    
- Ansible
    
- kubectl
    

بس.

---

# السيناريو اللى أنصح بيه

## VM

|Machine|Internet|
|---|---|
|Admin Machine|❌ لا|
|Harbor VM|✅ نعم (Outbound فقط)|
|Control Plane|❌ لا|
|Worker|❌ لا|

وده اللى بيتعمل فى معظم الشركات.

---

# Request تبع الـ Security

نفترض

```text
Harbor VM

IP

172.30.0.70
```

هيبقى الجدول بالشكل ده

|Source|Destination|Protocol|Port|Purpose|
|---|---|---|---|---|
|172.30.0.70|registry.k8s.io|TCP|443|Kubernetes Images|
|172.30.0.70|quay.io|TCP|443|Cilium / Prometheus / Images|
|172.30.0.70|ghcr.io|TCP|443|Helm OCI Charts / Images|
|172.30.0.70|github.com|TCP|443|Releases|
|172.30.0.70|raw.githubusercontent.com|TCP|443|YAML Files|
|172.30.0.70|objects.githubusercontent.com|TCP|443|GitHub Assets|
|172.30.0.70|release-assets.githubusercontent.com|TCP|443|GitHub Releases|
|172.30.0.70|download.docker.com|TCP|443|Docker Packages (لو هتستخدم Docker)|
|172.30.0.70|archive.ubuntu.com|TCP|80/443|Ubuntu Packages _(لو مفيش Local APT Repo)_|
|172.30.0.70|security.ubuntu.com|TCP|80/443|Ubuntu Security Updates _(لو مفيش Local APT Repo)_|
|172.30.0.70|get.helm.sh|TCP|443|Helm Binary|
|172.30.0.70|charts.jetstack.io|TCP|443|cert-manager|
|172.30.0.70|charts.bitnami.com|TCP|443|Bitnami Charts (اختياري)|

---

# لو هتنزل Cilium

هيحتاج غالباً

```text
quay.io
```

فقط.

---

# لو هتنزل Prometheus

هيحتاج

```text
quay.io
ghcr.io
```

---

# لو هتنزل Grafana

هيحتاج

```text
ghcr.io
docker.io (أحيانًا حسب الصورة المستخدمة)
```

---

# لو هتنزل Metrics Server

هيحتاج

```text
registry.k8s.io
```

---

# لو هتنزل ingress-nginx

هيحتاج

```text
registry.k8s.io
```

أو حسب إصدار الـ Chart قد يسحب صورًا من Registries أخرى، لكن الأشهر حاليًا `registry.k8s.io`.

---

# لو هتنزل HAProxy Ingress

غالبًا

```text
docker.io

أو

ghcr.io
```

حسب الصورة التى ستختارها.

---

# لو هتنزل Cert Manager

هيحتاج

```text
quay.io

charts.jetstack.io
```

---

# لو هتنزل Harbor نفسه

هتحتاج

```text
github.com

release-assets.githubusercontent.com
```

علشان تحمل الـ Offline Installer.

---

# طيب الـ Nodes؟

لا.

مش محتاجين يطلعوا Internet.

هيعملوا كده

```text
Worker

↓

containerd

↓

HTTPS

↓

Harbor
```

بس.

---

# يعنى الـ Firewall الداخلى

هيبقى

|Source|Destination|Port|
|---|---|---|
|CP1|Harbor|443|
|CP2|Harbor|443|
|CP3|Harbor|443|
|Worker1|Harbor|443|
|Worker2|Harbor|443|
|Worker3|Harbor|443|
|Worker4|Harbor|443|
|Worker5|Harbor|443|
|Worker6|Harbor|443|

بس.

---

# وAdmin Machine؟

تحتاج

|Source|Destination|Port|
|---|---|---|
|Admin|Harbor|443|
|Admin|CP1|22|
|Admin|CP2|22|
|Admin|CP3|22|
|Admin|Worker1-6|22|
|Admin|Kubernetes API VIP|6443|

---

# أهم حاجة معظم الناس بتنساها

لو هتستخدم **Harbor كـ Private Registry**، فبعد ما تحمل الصور، **اقفل الإنترنت عن Harbor** لو سياسة الشركة تسمح.

ويبقى الـ Flow

```text
Internet

↓

Harbor

↓

Pull Images

↓

Push Images

↓

Close Internet

↓

Nodes

↓

Pull Images From Harbor
```

وده هو الـ Best Practice.

---

# لكن لو أنا بعمل Enterprise Platform (وده اللى أنصحك بيه)

أنا **مش هطلب فتح Domains**.

هطلب من الـ Security **فتح Proxy Server فقط**.

يعنى بدل 15 URL

يبقى:

```text
Harbor VM

↓

Corporate Proxy

↓

Internet
```

ليه؟

لأن بعد شهر هتحتاج Image جديدة، وبعد شهرين Helm Chart جديد، وبعدها Kubernetes Version جديد، وكل مرة هترجع تطلب فتح Domain جديد.

أما لو عندكم **Enterprise Proxy**، فالـ Security Team هيطبق الـ URL Filtering والـ Logging والـ Malware Inspection على الـ Proxy، وأنت مش هتحتاج تعدل الـ Firewall كل شوية.

**سؤالي الأخير ليك:**  
هل الشركة عندكم فيها **Proxy Server** (مثل Blue Coat أو Zscaler أو Squid أو أي Corporate Proxy)، ولا الخروج للإنترنت بيكون مباشرة من الـ Firewall؟ لأن الإجابة هتغير التصميم اللى أنصحك بيه بالكامل.

-----
هجاوبك فى سيناريوهين لأن ده مهم.

---

# السيناريو الأول (قبل ما تعمل Harbor)

يعنى لسه عندك Internet.

مثلاً عايز تنزل Image

```text
nginx
```

هتعمل

لو بتستخدم Docker

```bash
docker pull nginx:latest
```

أو لو بتستخدم containerd

```bash
ctr -n k8s.io images pull docker.io/library/nginx:latest
```

أو

```bash
nerdctl pull nginx:latest
```

هيحصل فى الخلفية:

```text
Docker/Containerd
        │
        ▼
DNS Lookup
        │
        ▼
docker.io
        │
        ▼
Manifest
        │
        ▼
Layer 1
Layer 2
Layer 3
...
        │
        ▼
Stored Locally
```

---

# السيناريو الثانى (بعد ما تعمل Harbor)

وده اللى هيكون عندك.

نفترض

Harbor

```text
harbor.company.local
```

وعملت Project

```text
kubernetes
```

يبقى

```text
harbor.company.local/kubernetes
```

---

## أول مرة

تنزل الصورة من الإنترنت

```bash
docker pull nginx:latest
```

---

بعدها

Tag

```bash
docker tag nginx:latest harbor.company.local/kubernetes/nginx:latest
```

---

بعدها

Login

```bash
docker login harbor.company.local
```

---

بعدها

Push

```bash
docker push harbor.company.local/kubernetes/nginx:latest
```

هيحصل

```text
Docker

↓

HTTPS

↓

Harbor

↓

PostgreSQL

↓

Registry Storage

↓

Image Saved
```

---

بعد كده

أى Node

بدل

```bash
docker pull nginx
```

هتكتب

```bash
ctr -n k8s.io images pull harbor.company.local/kubernetes/nginx:latest
```

أو

فى الـ Pod

```yaml
image: harbor.company.local/kubernetes/nginx:latest
```

هيحصل

```text
Worker

↓

DNS

↓

harbor.company.local

↓

HTTPS

↓

Image Download

↓

containerd

↓

Run Container
```

---

# طب لو Kubernetes نفسه

مثلاً

CoreDNS

بدل

```text
registry.k8s.io/coredns
```

هتخليها

```text
harbor.company.local/kubernetes/coredns
```

---

# لو هتنزل Image جديدة بعد شهر

مثلاً

Redis

هتكرر نفس الخطوات

```bash
docker pull redis:7
```

↓

```bash
docker tag redis:7 harbor.company.local/kubernetes/redis:7
```

↓

```bash
docker push harbor.company.local/kubernetes/redis:7
```

خلاص.

---

# لو عايز تنزل كل صور Kubernetes مرة واحدة

ممكن تستخدم `kubeadm` لمعرفة الصور المطلوبة للإصدار الذي ستستخدمه:

```bash
kubeadm config images list --kubernetes-version <VERSION>
```

ثم تسحب كل صورة وتعيد وسمها (tag) وترفعها إلى Harbor.

---

## نصيحة مهمة

بما إنك هتبنى Cluster Enterprise، **ما ترفعش كل الصور فى Project واحد**.

اعمل Projects منفصلة فى Harbor، مثلاً:

```text
kubernetes/
cilium/
monitoring/
ingress/
logging/
cert-manager/
applications/
```

مثال:

```text
harbor.company.local/kubernetes/kube-apiserver:v1.xx.x

harbor.company.local/cilium/cilium:v1.xx.x

harbor.company.local/monitoring/prometheus:v3.x

harbor.company.local/monitoring/grafana:12.x

harbor.company.local/ingress/haproxy-ingress:v0.xx
```

وده بيسهل جدًا:

- إدارة الصلاحيات (RBAC).
    
- معرفة الصور الخاصة بكل مكون.
    
- عمل Cleanup وRetention Policies.
    
- التحديثات لاحقًا.
    

لو هتبنى البيئة من الصفر، أنا أنصح من البداية تبنى Harbor بنفس التنظيم ده بدل ما تحط كل الصور فى Project واحد.

-----

السؤال ده ممتاز، وفيه أكتر من سيناريو. لكن **لو بنتكلم عن Enterprise Best Practice**، فالسيناريو اللى أنصحك بيه هو إن **Harbor VM هي اللي تنزل الصور من الإنترنت وترفعها لنفسها**، وليس الـ Admin Machine.

## السيناريو الأول (أفضل Practice)

```text
                    Internet
                        │
                        ▼
                  Harbor VM
      (Docker / containerd / nerdctl)
                        │
        docker pull / ctr pull / nerdctl pull
                        │
                        ▼
               Harbor Registry Storage
                        │
                        ▼
        ----------------------------------
        │               │               │
      CP1             Worker1        Worker2
          (Pull from Harbor فقط)
```

يعني:

1. تعمل Login على Harbor (الـ Registry نفسه).
    
2. تنزل الصورة على **Harbor VM**.
    
3. تعمل Tag.
    
4. تعمل Push إلى Harbor.
    
5. الـ Nodes تسحب من Harbor.
    

مثال:

```bash
docker pull registry.k8s.io/kube-apiserver:v1.34.0

docker tag registry.k8s.io/kube-apiserver:v1.34.0 \
harbor.company.local/kubernetes/kube-apiserver:v1.34.0

docker push harbor.company.local/kubernetes/kube-apiserver:v1.34.0
```

الصورة لم تخرج من الـ Harbor VM، وإنما نزلت عليها ثم اتخزنت في الـ Registry الموجود عليها.

---

# السيناريو الثاني

ممكن تعملها من الـ Admin Machine.

```text
Internet

↓

Admin Machine

↓

docker pull

↓

docker push

↓

Harbor

↓

Nodes
```

وده شغال ومفيش فيه مشكلة، لكن له عيوب:

- لازم الـ Admin Machine يبقى عندها Internet.
    
- لازم الـ Admin Machine يبقى عندها Docker أو containerd.
    
- كل تحديث هتحتاج تعمله من الـ Admin.
    

---

# السيناريو الثالث (اللي الشركات الكبيرة بتعمله)

وده الأكثر احترافية.

```text
Internet
      │
      ▼
Corporate Proxy
      │
      ▼
Harbor VM
      │
      ▼
Harbor Registry
      │
      ▼
Kubernetes Nodes
```

يعني:

- الـ Harbor فقط هو اللي يطلع الإنترنت (أو عبر Proxy).
    
- الـ Admin Machine **مفيهاش Internet**.
    
- الـ Nodes **مفيهاش Internet**.
    

---

# طب لو بعد شهر نزل Kubernetes 1.35؟

هتعمل:

على Harbor VM:

```bash
docker pull registry.k8s.io/kube-apiserver:v1.35.x
docker tag ...
docker push ...
```

خلاص.

ولا Admin ولا Workers هيطلعوا الإنترنت.

---

# طب لو نزلت Cilium جديد؟

برضه على Harbor VM:

```bash
docker pull quay.io/cilium/cilium:v1.xx.x
docker tag ...
docker push ...
```

---

# طب لو Prometheus؟

نفس الفكرة.

---

# إذًا مين اللي عليه Docker؟

أنا بعمل كده:

|Machine|Docker/containerd|
|---|---|
|Harbor VM|✅ نعم|
|Admin Machine|❌ لا (إلا إذا احتجت لاختبارات)|
|Control Plane|containerd فقط لتشغيل Kubernetes|
|Workers|containerd فقط|

يعني Harbor VM تبقى هي "Image Management Server".

---

# فيه حاجة أحسن من docker pull؟

أيوه.

Harbor نفسه بيدعم ميزة اسمها **Proxy Cache**.

يعني بدل ما تعمل:

```text
docker pull

↓

tag

↓

push
```

كل مرة،

Harbor يعملها أوتوماتيك لأول طلب للصورة ويحتفظ بنسخة محلية. لكن:

- الميزة دي مناسبة لما يكون Harbor عنده اتصال مستمر بالإنترنت أو بالـ Proxy.
    
- **أنا لا أنصح تستخدمها في بيئة Production معزولة (Air-Gapped)**، لأنك ساعتها عايز تتحكم بنفسك في الصور والإصدارات الموجودة داخل الشركة.
    

---

## أنا لو ببني البيئة عندكم هخليها كده

```text
                  Internet
                      │
               (TCP 443 فقط)
                      │
                 Harbor VM
      Ubuntu 24.04 + Docker + Harbor
                      │
     Pull → Tag → Push → Verify Images
                      │
              يغلق Internet بعد الانتهاء
                      │
      ┌───────────────┼────────────────┐
      ▼               ▼                ▼
    CP1             CP2              CP3
      │
      ▼
  Workers (1-6)
      │
      ▼
 containerd يسحب من Harbor فقط

Admin Machine
      │
      ├── SSH
      ├── Ansible
      └── kubectl
```

وده هو التصميم اللي أنصح بيه لبيئتكم لأنه:

- يعزل الإنترنت في VM واحدة فقط.
    
- يخلي كل الصور تحت سيطرتك داخل Harbor.
    
- يقلل جدًا الطلبات اللي هتحتاجها من فريق الـ Security بعد تجهيز البيئة.