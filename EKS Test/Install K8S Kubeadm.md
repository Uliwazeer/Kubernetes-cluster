
Setup K8S Kubeadm

https://gist.github.com/manojkumarponnaluri/e48d46a265cd9bad9903e2aa8eef33c2 

https://lucaberton.com/blog/install-kubernetes-rocky-linux-9/

----
ده بالضبط الأسلوب الصح. قبل ما تكتب أول أمر `kubeadm init`، فيه شغل Design وInfrastructure لازم يخلص. لو مشيت بالترتيب ده، مش هترجع خطوة لورا.

أنا هعتبر إننا **Platform Engineers** وبنبنى Test Environment من الصفر على **Huawei FusionCompute** بحيث يبقى قريب جدًا من الـ Production.

---

# Phase 0 - Planning (ممنوع تعمل VM قبلها)

أول حاجة اعمل Document فيه:

## 1. Architecture

مثلاً:

```text
Huawei Server 1
        │
        ├── CP1
        ├── CP2
        ├── Worker1
        ├── Worker2
        ├── Harbor

Huawei Server 2
        │
        ├── CP3
        ├── Worker3
        ├── Worker4
        ├── Worker5
        ├── Worker6
        ├── Admin
```

لاحظ إني وزعت الـ VMs على الـ Hostين بحيث لو Host وقع، الكلاستر يفضل شغال.

---

## 2. IP Plan

مثلاً

|VM|IP|
|---|---|
|Harbor|172.30.10.30|
|Admin|172.30.10.40|
|CP1|172.30.10.101|
|CP2|172.30.10.102|
|CP3|172.30.10.103|
|Worker1|172.30.10.111|
|Worker2|172.30.10.112|
|Worker3|172.30.10.113|
|Worker4|172.30.10.114|
|Worker5|172.30.10.115|
|Worker6|172.30.10.116|

---

## 3. DNS Names

|Hostname|IP|
|---|---|
|harbor.lab.local|172.30.10.30|
|admin.lab.local|172.30.10.40|
|cp1.lab.local|172.30.10.101|
|cp2.lab.local|172.30.10.102|
|cp3.lab.local|172.30.10.103|

---

## 4. VIPs

|VIP|الاستخدام|
|---|---|
|172.30.10.10|Kubernetes API|
|172.30.10.20|Ingress|
|172.30.10.200-220|LoadBalancer Pool|

---

# Phase 1 - تجهيز FusionCompute

قبل إنشاء أي VM تأكد من:

- Datastore موجود وفيه مساحة كافية.
    
- Virtual Switch / Port Group أو VLAN مخصصة للكلاستر.
    
- الشبكة متصلة بالـ Gateway.
    
- الـ DNS وNTP Servers معروفين.
    
- عندك ISO لـ Ubuntu Server وRocky Linux.
    

---

# Phase 2 - إنشاء أول VM (Harbor)

ابدأ بالـ Harbor لأنه هيبقى مصدر الصور لباقي البيئة.

### المواصفات المقترحة

- 4 vCPU
    
- 8 GB RAM
    
- 250 GB Disk (أو أكثر حسب عدد الصور)
    
- Ubuntu Server 24.04 LTS
    
- NIC واحدة على VLAN الكلاستر
    
- Thin Provisioning للتست (لو سياسة الشركة تسمح)
    

بعد تثبيت النظام:

- Static IP.
    
- Hostname.
    
- DNS.
    
- NTP.
    
- تحديث النظام.
    
- تثبيت Docker.
    
- تثبيت Harbor.
    

---

# Phase 3 - إنشاء Admin VM

المواصفات:

- Ubuntu Server.
    
- 4 vCPU.
    
- 8 GB RAM.
    
- 80-100 GB Disk.
    

عليها:

- Ansible.
    
- kubectl.
    
- Helm.
    
- Git.
    
- ssh.
    
- jq.
    
- yq.
    
- k9s (اختياري).
    

---

# Phase 4 - إنشاء Control Plane Template

بدل ما تعمل كل VM من الصفر.

اعمل VM واحدة:

```text
Rocky Linux 9
```

وجهزها:

- آخر Updates.
    
- hostname مؤقت.
    
- SSH.
    
- chrony.
    
- firewalld حسب السياسة.
    
- SELinux حسب التصميم.
    
- تثبيت containerd.
    
- تثبيت kubeadm.
    
- تثبيت kubelet.
    
- تثبيت kubectl.
    
- إعداد kernel modules.
    
- إعداد sysctl.
    

لكن **لا تعمل `kubeadm init`**.

بعدها:

Shutdown.

ثم اعمل منها **Template** أو Clone.

---

# Phase 5 - إنشاء Control Planes

اعمل 3 Clones:

```text
cp1
cp2
cp3
```

وغير فقط:

- Hostname.
    
- Static IP.
    

---

# Phase 6 - إنشاء Worker Template

ممكن نفس الـ Template السابق.

اعمل منه 6 Clones.

---

# Phase 7 - مراجعة البنية

قبل Kubernetes، تأكد أن:

- كل VM تعمل Ping على الباقي (حسب سياسة الشبكة).
    
- الـ DNS يعمل.
    
- الـ NTP متزامن.
    
- الـ SSH يعمل من الـ Admin.
    
- كل VM تصل إلى Harbor.
    
- Harbor يمكنه الوصول للإنترنت (إذا كان مسموحًا مؤقتًا).
    

---

# Phase 8 - إعداد Ansible

من الـ Admin:

- Inventory.
    
- SSH Keys.
    
- اختبار الاتصال بـ `ansible all -m ping`.
    
- Playbooks لإعداد الأنظمة.
    

---

# Phase 9 - تجهيز Harbor بالصور

- تنزيل الصور المطلوبة.
    
- Tag.
    
- Push إلى Harbor.
    
- التأكد من ظهورها.
    
- إعداد `containerd` على جميع الـ Nodes لاستخدام Harbor.
    

---

# Phase 10 - بناء الكلاستر

بعدها فقط:

- إنشاء الـ API VIP.
    
- `kubeadm init`.
    
- ضم الـ Control Planes.
    
- ضم الـ Workers.
    
- تثبيت Cilium.
    
- إعداد الـ LoadBalancer/VIPs.
    
- تثبيت HAProxy Ingress.
    
- تثبيت Prometheus.
    
- تثبيت Grafana.
    

---

# نقطة مهمة جدًا بخصوص FusionCompute

أنا **لا أنصح** تعمل 11 VM يدويًا واحدة واحدة.

الترتيب الاحترافي هو:

```text
Ubuntu ISO
        │
        ▼
Harbor VM
        │
        ▼
Admin VM
        │
        ▼
Rocky Base VM
        │
        ▼
Template
        │
        ├── CP1
        ├── CP2
        ├── CP3
        ├── Worker1
        ├── Worker2
        ├── Worker3
        ├── Worker4
        ├── Worker5
        └── Worker6
```

ده يوفر وقت كبير ويضمن إن كل الـ Nodes متطابقة.

---

## اقتراح مهم قبل التنفيذ

بما إن هدفك إن البيئة تبقى قريبة جدًا من الـ Production، فأنا أقترح نبنيها على **مراحل (Milestones)**، وما ننتقلش للمرحلة التالية إلا بعد نجاح السابقة:

1. **Infrastructure**: FusionCompute + الشبكة + IP Plan + إنشاء الـ VMs.
    
2. **Base OS**: إعداد Rocky وUbuntu، الـ DNS، الـ NTP، الـ SSH، والـ Repository.
    
3. **Platform Services**: Harbor، ثم Ansible، ثم تجهيز الـ Nodes.
    
4. **Kubernetes**: kubeadm HA + Cilium + VIPs + HAProxy.
    
5. **Observability**: Prometheus + Grafana + Alerting.
    

ده هيخليك لو حصلت مشكلة تعرف تحددها بسرعة، بدل ما تكون بتبني كل حاجة مرة واحدة ويصعب تتبع سبب أي خطأ.

----
بص يا علي، **الدوكيومنت اللي أنت جايبه ده ممتاز لتجربة Lab صغيرة**، لكنه **مش مناسب لبيئة HA Enterprise** زي اللي أنت عايز تبنيها. لو مشيت عليه حرفيًا، هتعمل Cluster شغال، لكن **مش هيبقى شبه الـ Production**.

أنا هجاوب على أسئلتك واحدة واحدة.

---

# أولاً: الـ DNS هحطه فين؟

أنت عندك مستويين من الـ DNS.

## 1) DNS الخاص بالـ Operating System (مهم جدًا)

كل VM لازم تعرف:

- DNS Server
    
- Search Domain
    

مثلاً:

```text
DNS Server:
172.30.10.2

Search Domain:
lab.company.local
```

وده بيتحط أثناء إعداد الـ OS أو بعد التثبيت (حسب NetworkManager أو netplan أو systemd-resolved).

مثلاً على Rocky:

```bash
nmcli connection modify ens160 ipv4.dns "172.30.10.2"
nmcli connection modify ens160 ipv4.dns-search "lab.company.local"
nmcli connection up ens160
```

---

## 2) CoreDNS

ده بيتنزل بعد ما Kubernetes يشتغل.

يعنى **إنت مش هتعمله دلوقتى**.

---

# ثانياً: الـ NTP

كل الـ Nodes لازم تكون synchronized.

لو عندكم NTP Server فى الشركة:

مثلاً

```text
172.21.0.5
```

يبقى كل VM تستخدمه.

لو مفيش.

ممكن تعمل VM صغيرة

```text
ntp.lab.local
```

وتستخدم Chrony.

أنا شخصيًا لو الشركة عندها NTP داخلى، **مش هعمل VM جديدة**.

---

# ثالثاً: MAC Address

فى الدوكيومنت مكتوب

> Unique hostname, MAC, product_uuid

وده صحيح.

لكن على FusionCompute

**متعملش MAC بنفسك.**

سيب FusionCompute يولده.

لأن كل Clone هيطلع له:

- MAC مختلف
    
- UUID مختلف
    

وده المطلوب.

---

# رابعاً: Static IP

أنت قلت:

> "أنا هحطه وأنا بكريت الـ VM"

لا.

وده فرق مهم.

الـ Hypervisor **مش بيحدد الـ IP**.

هو بيعمل:

- vCPU
    
- RAM
    
- Disk
    
- NIC
    
- VLAN
    
- MAC
    

لكن الـ IP بيتحط داخل نظام التشغيل (أو عبر DHCP لو كنت تستخدمه).

---

# خامساً: iBMC

أنت قلت

> عندى 2 سيرفر والمانجمنت بتاعهم iBMC.

الـ iBMC زيه زى iLO أو iDRAC.

هو لإدارة **السيرفر الفيزيائى**:

- تشغيل وإيقاف.
    
- Console.
    
- مراقبة الهاردوير.
    

**مش هتنشئ منه الـ VMs.**

---

# إنشاء الـ VMs

هيتم من واجهة **FusionCompute / DCS**.

وليس من الـ iBMC.

يعنى:

```text
Huawei Physical Server
        │
        ▼
FusionCompute
        │
        ▼
Create VM
```

الـ iBMC تستخدمه لو السيرفر نفسه وقع أو محتاج تدخل Out-of-Band.

---

# هل أروح على كل Host وأعمل VMs؟

لا.

فى FusionCompute المفروض إنك تدخل على الـ Management (FusionCompute أو DCS Portal)، ومنه تختار الـ Host اللى هتنشئ عليه الـ VM.

مثلاً:

```text
Host1

↓

CP1

CP2

Worker1

Worker2

Harbor
```

```text
Host2

↓

CP3

Worker3

Worker4

Worker5

Worker6

Admin
```

يعنى التوزيع بيكون من خلال الـ Management UI.

---

# سادساً: الـ VIP

وده أهم سؤال.

إنت قلت:

> "مشفتش فى Documentation أى VIP"

طبيعى جدًا.

لأن الدوكيومنت ده بيشرح:

```text
Single Control Plane
```

وليس

```text
High Availability Cluster
```

---

## لو هتعمل HA

إنت **مش هتكتب**:

```bash
kubeadm init
```

زى ما فى الدوكيومنت.

هتكتب حاجة شبه كده (كمثال):

```bash
kubeadm init \
--control-plane-endpoint api.lab.local:6443 \
--upload-certs
```

لاحظ

```text
api.lab.local
```

ده

VIP

أو DNS

يشاور على VIP.

مش

CP1.

---

# يبقى الـ VIP بيتعمل إمتى؟

**قبل `kubeadm init`.**

وده اللى ناس كتير بتغلط فيه.

الترتيب الصحيح:

```text
VMs

↓

Network

↓

DNS

↓

NTP

↓

Harbor

↓

HAProxy / VIP

↓

API VIP يعمل

↓

kubeadm init

↓

Join CP2

↓

Join CP3
```

وليس العكس.

---

# طيب مين هيعمل الـ VIP؟

دى حسب التصميم.

لو هتستخدم

HAProxy

داخل Kubernetes

يبقى **مش هينفع تعتمد عليه للـ API VIP**، لأن الكلاستر لسه ما قامش.

يعنى عند لحظة إنشاء الكلاستر، مفيش Pods ولا Services.

علشان كده فى بيئات kubeadm HA فيه حلول مثل:

- Keepalived + HAProxy على VMs خارج الكلاستر.
    
- أو حلول أخرى توفر الـ API Endpoint قبل تشغيل Kubernetes.
    

أما **الـ HAProxy Ingress داخل الكلاستر** فهو يشتغل بعد قيام الكلاستر، وليس قبله.

---

# يبقى عندك نوعين

## قبل الكلاستر

```text
API VIP
```

---

## بعد الكلاستر

```text
Ingress VIP

LoadBalancer VIP
```

---

# أنا عندى إحساس إن البرودكشن عندكم فيه حاجة ناقصة فى الصورة

لأنك قلت

> HAProxy داخل الكلاستر

وده منطقى للـ Ingress.

لكن

الـ API Server

لازم حد يوزع عليه الطلبات قبل قيام الكلاستر.

وده غالبًا واحد من:

- Keepalived + HAProxy خارج الكلاستر.
    
- جهاز Load Balancer خارجى.
    
- أو خدمة مشابهة.
    

---

# أنا عايز منك معلومتين فقط من البرودكشن

دول هيكشفوا التصميم كله.

## أول حاجة

على أى جهاز عنده kubectl:

```bash
kubectl config view --minify
```

أنا عايز فقط السطر:

```yaml
server:
```

(تقدر تخفى الـ IP لو سياسة الشركة تمنع مشاركته).

---

## ثانى حاجة

على أى Control Plane:

```bash
cat /etc/kubernetes/manifests/kube-apiserver.yaml
```

وابعت فقط الجزء اللى فيه:

```text
--advertise-address
```

و

```text
--control-plane-endpoint
```

إن وجد.

---

## بعد المعلومتين دول

هقدر أرسم لك **نفس الـ Production Architecture بنسبة 100%**، وننفذه على Huawei FusionCompute، بما فى ذلك:

- تصميم الـ VIPs.
    
- ترتيب إنشاء الـ VMs.
    
- توقيت إنشاء كل VIP.
    
- إعدادات DNS وNTP.
    
- والخطوات الصحيحة لبناء الـ Test Cluster دون الاعتماد على دليل مخصص لـ Single Control Plane.