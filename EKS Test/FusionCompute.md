
أيوه، **لو المقصود بـ DCS هو Huawei DCS (Huawei Distributed Cloud Solution / DCS Virtualization Platform)** اللي بيشتغل كمنصة Virtualization شبيهة بـ VMware vSphere، فالإجابة هي:

> **نعم، ممكن تشغل EKS Anywhere عليه، لكن ليس كـ "Huawei DCS Provider".**
> 
> هتشغله عن طريق **Virtual Machines** داخل DCS، وكأن DCS عبارة عن Hypervisor أو Virtualization Platform.

يعني المعمارية هتبقى بالشكل ده:

```text
                 Physical Servers
                        │
                        ▼
        Huawei DCS Virtualization Platform
                        │
        ┌───────────────┼────────────────┐
        │               │                │
        ▼               ▼                ▼
      VM-CP1          VM-CP2          VM-CP3
   (Ubuntu/RHEL)   (Ubuntu/RHEL)   (Ubuntu/RHEL)

        ▼               ▼                ▼
        Kubernetes Control Plane (EKS Anywhere)

                        │

        ┌───────────────┼────────────────┐
        ▼               ▼                ▼
     VM-W1           VM-W2            VM-W3
```

---

# السؤال المهم

هل AWS بتدعم Huawei DCS رسميًا؟

الإجابة:

**لا.**

AWS حاليًا بتدعم Providers رسميًا مثل:

- Bare Metal (Tinkerbell)
    
- vSphere
    
- Nutanix
    
- CloudStack
    
- Snow
    
- Docker (للتجارب)
    

ولا يوجد Provider رسمي باسم Huawei DCS.

وده معناه إن EKS Anywhere مش هيعرف يتكلم مع DCS API مباشرة زي ما بيتكلم مع vCenter.

---

# طيب إزاي نشغله؟

فيه طريقتين.

## الطريقة الأولى (الأسهل)

لو DCS مجرد Hypervisor.

اعمل عليه VMs.

مثلاً

```text
Huawei DCS

↓

Create VM

↓

Install Ubuntu

↓

Install Docker

↓

Install kubelet

↓

EKS Anywhere
```

بالنسبة لـ EKS Anywhere، هو شايفها مجرد Linux Machines.

---

## الطريقة الثانية

لو Huawei DCS بيدعم APIs متوافقة مع VMware (وده يعتمد على إصدار DCS)، ممكن تستفيد من طبقة توافق، لكن ده مش السيناريو الرسمي وغالبًا هتحتاج اختبارات.

---

# المطلوب داخل DCS

هتعمل VMs.

مثلاً

|VM|CPU|RAM|Disk|
|---|---|---|---|
|Admin|4|8GB|100GB|
|CP1|4|16GB|120GB|
|CP2|4|16GB|120GB|
|CP3|4|16GB|120GB|
|Worker1|8|32GB|200GB|
|Worker2|8|32GB|200GB|

---

# الشبكة

كل الـ VMs لازم تكون على نفس الشبكة.

مثلاً

```text
192.168.100.10

192.168.100.11

192.168.100.12

192.168.100.13
```

والـ API Server ممكن يبقى:

```text
192.168.100.100
```

VIP أو Load Balancer حسب التصميم.

---

# Storage

تقدر تستخدم Storage اللي DCS بيقدمه للـ VMs.

لكن داخل Kubernetes هتحتاج CSI Driver مناسب لو عايز Dynamic Provisioning. لو مفيش CSI خاص بـ Huawei DCS، تقدر تستخدم حلول مستقلة عن منصة الـ Virtualization زي:

- NFS CSI
    
- OpenEBS
    
- Longhorn
    
- Ceph CSI (لو عندك Ceph)
    

---

# Networking

داخل الـ Cluster هتثبت CNI زي:

- Cilium
    
- Calico
    

أما الـ VM Networking فهي مسؤولية Huawei DCS.

---

# هل هيشتغل؟

طالما الـ VMs عليها:

- Ubuntu أو RHEL
    
- CPU Virtualization
    
- RAM كافية
    
- Network بين الـ Nodes
    
- DNS
    
- NTP
    

فـ Kubernetes نفسه مش هيفرق معاه إذا كان شغال على:

- VMware
    
- Proxmox
    
- Huawei DCS
    
- Hyper-V
    
- KVM
    

لأن في النهاية هو شايف Linux Servers.

---

# هل أستخدم EKS Anywhere؟

هنا لازم نفرق بين حاجتين:

إذا كنت عايز **تجربة AWS EKS Anywhere نفسها** بكل إمكانيات إدارة الـ Lifecycle (إنشاء الـ VMs تلقائيًا، Scale، Upgrade من خلال Provider)، فغياب دعم رسمي لـ Huawei DCS هيخليك تفقد جزءًا من المزايا دي.

أما إذا كان هدفك **تشغيل Kubernetes متوافق مع توزيعة EKS Distro** داخل بيئة Huawei DCS، فيمكنك تشغيل الـ Nodes كـ VMs وإدارة الكلاستر بنفسك أو بأدوات أخرى.

### سؤالي ليك:

**إيه المقصود بـ Huawei DCS عندك بالضبط؟**

- هل هو **Huawei DCS (Distributed Cloud Solution)** المبني على KVM/OpenStack؟
    
- ولا **Huawei FusionCompute / FusionSphere**؟
    
- ولا منصة تانية من هواوي؟
    

لأن طريقة تثبيت EKS Anywhere والـ Provider المناسب هتختلف حسب المنتج الفعلي.

----
بما إنك بتتكلم عن **Huawei FusionCompute**، فالوضع مختلف شوية عن VMware.

الإجابة المختصرة:

> **لا، EKS Anywhere لا يدعم Huawei FusionCompute كـ Infrastructure Provider رسميًا.** ([Huawei Support](https://support.huawei.com/enterprise/en/multilingual-document/fusioncompute-pid-8576912?utm_source=chatgpt.com "Huawei FusionCompute Support Guide, Manuals & PDF – Huawei"))

وده لأن EKS Anywhere بيدعم Providers محددين فقط لإدارة الـ VM Lifecycle (إنشاء الـ VMs، حذفها، ترقيتها،...).

---

## ليه VMware مدعوم و FusionCompute لأ؟

فى VMware بيحصل الآتى:

```text
eksctl anywhere
        │
        ▼
vCenter API
        │
        ▼
Create VM
Power On VM
Clone Template
Attach Network
Attach Disk
Cloud-init
        │
        ▼
Bootstrap Kubernetes
```

يعنى `eksctl anywhere` بيتكلم مباشرة مع **vCenter API**.

أما فى FusionCompute:

```text
eksctl anywhere
        │
        X
        │
FusionCompute API
```

مفيش Provider داخل EKS Anywhere يعرف يتعامل مع FusionCompute APIs، رغم إن FusionCompute نفسه عنده REST APIs وSDK للإدارة. ([Huawei Support](https://support.huawei.com/enterprise/en/multilingual-document/fusioncompute-pid-8576912?utm_source=chatgpt.com "Huawei FusionCompute Support Guide, Manuals & PDF – Huawei"))

---

# هل معنى كده مستحيل؟

**لا.**

قدامك 3 حلول.

## الحل الأول (أنصح بيه)

اعمل الـ VMs يدويًا داخل FusionCompute.

مثلاً:

```text
FusionCompute

↓

VM-CP1
VM-CP2
VM-CP3

VM-W1
VM-W2
VM-W3
```

بعد كده تنزل:

- Ubuntu 22.04 أو RHEL
    
- containerd
    
- kubeadm أو أي توزيعة Kubernetes
    

وهنا عندك اختيارين:

- Kubernetes باستخدام kubeadm.
    
- أو تشغيل توزيعة متوافقة مع EKS Distro، لكن بدون ميزات الـ Provider الخاصة بـ EKS Anywhere.
    

---

## الحل الثانى

لو شركتك عندها VMware أيضًا.

يبقى تعمل:

```text
FusionCompute

↓

VMware

↓

EKS Anywhere
```

وده نادر جدًا ومش منطقى فى أغلب البيئات.

---

## الحل الثالث

تكتب Provider بنفسك لـ FusionCompute.

وده معناه تطوير كامل باستخدام FusionCompute APIs.

وده مشروع كبير جدًا ومش بيعمله إلا Vendor أو شركة عندها فريق تطوير.

---

# لو أنا مكانك فى EGX

لو البنية عندكم كلها Huawei FusionCompute، فأنا **مش هختار EKS Anywhere**.

هختار أحد الحلول التالية:

```text
FusionCompute

↓

Virtual Machines

↓

Ubuntu

↓

containerd

↓

kubeadm

↓

HA Kubernetes Cluster
```

أو

```text
FusionCompute

↓

OpenShift Virtual Machines

↓

OpenShift
```

لو عندكم رخصة OpenShift.

وده هيديك نفس النتيجة تقريبًا بدون الدخول فى مشاكل عدم وجود Provider.

---

# هل فيه طريقة أخلى EKS Anywhere يشتغل؟

نظريًا نعم، لكن هتفقد أهم ميزة فيه.

لأنك هتعمل:

```text
FusionCompute

↓

Create VMs (Manual)

↓

Install Linux (Manual)

↓

Prepare Nodes (Manual)

↓

Run EKS Anywhere
```

بينما EKS Anywhere مصمم إنه هو يعمل إنشاء وإدارة البنية التحتية تلقائيًا على المنصات المدعومة. مع FusionCompute، مش هيقدر يدير دورة حياة الـ VMs بنفس الطريقة.

---

## لو هدفك هو التعلم أو تنفيذ Production؟

بما إنك مسؤول عن Kubernetes/OpenShift Infrastructure، فأنا عندي سؤال هيحدد أفضل حل:

- **هل عندكم Huawei FusionCompute فقط؟**
    
- **ولا عندكم كمان Huawei FusionSphere + VRM + CPS (Cloud Platform Suite)؟**
    

لأن لو عندكم **FusionSphere/VRM**، أقدر أوضح لك هل ممكن نستفيد من الـ APIs الخاصة به لعمل Automation قريب جدًا من تجربة VMware، أو الأفضل نستخدم kubeadm أو OpenShift على FusionCompute مع Terraform وAnsible لإدارة الـ Infrastructure بشكل احترافي.

----
أيوه، **وده في الحقيقة هو الحل الاحترافي** لو عندك **Huawei FusionCompute**.

بل أنا لو عندي FusionCompute في شركة، **مش هستخدم EKS Anywhere أصلًا**، لكن هبني Automation كامل باستخدام **Ansible + FusionCompute API + kubeadm** أو Ansible + RKE2/K3s حسب احتياج الشركة.

## الفكرة العامة

بدل ما EKS Anywhere يعمل:

```text
Create VM
↓
Install OS
↓
Configure OS
↓
Install Kubernetes
↓
Join Cluster
```

أنت هتخلي Ansible يعمل كل الخطوات دي.

---

# السيناريو

```text
                 Admin Machine
                      │
                Ansible Playbook
                      │
      ┌───────────────┴────────────────┐
      │                                │
      ▼                                ▼
FusionCompute API                 Existing VMs
(Create VM)                         (SSH)
      │                                │
      ▼                                ▼
Power On VM                     Configure Linux
      │                                │
      └───────────────┬────────────────┘
                      ▼
             Install containerd
                      ▼
             Install kubeadm
                      ▼
            kubeadm init (CP1)
                      ▼
        kubeadm join (CP2/CP3/Workers)
                      ▼
            Install CNI (Cilium/Calico)
                      ▼
             Kubernetes Ready
```

---

# هل Ansible يقدر يعمل Create VM؟

نعم، **لو FusionCompute عنده API**.

Ansible يقدر يتعامل مع أي API عن طريق:

- `uri` module
    
- أو Module مخصص لو موجود
    
- أو حتى تشغيل CLI لو Huawei موفرة واحدة.
    

يعني ممكن تعمل Playbook بالشكل التالي (كمفهوم):

```yaml
- Create VM
- Attach Network
- Attach Disk
- Power On
```

بعد كده:

```yaml
- Wait for SSH
```

ثم:

```yaml
- Install containerd
- Disable swap
- Configure sysctl
- Install kubelet
- Install kubeadm
- Install kubectl
```

---

# بعد كده

على أول Control Plane:

```bash
kubeadm init
```

Ansible يحفظ الـ Join Command.

مثلاً:

```bash
kubeadm token create --print-join-command
```

ثم يوزعه على باقي الأجهزة.

---

# بعد كده

Control Plane الثاني والثالث:

```bash
kubeadm join \
--control-plane \
--token xxx
```

---

# ثم

Workers

```bash
kubeadm join
```

---

# ثم

ينزل CNI

مثلاً

```bash
helm install cilium
```

أو

```bash
kubectl apply -f calico.yaml
```

---

# ثم

ينزل Addons

مثلاً

- Metrics Server
    
- Ingress Controller
    
- Cert Manager
    
- MetalLB
    
- Prometheus
    
- Grafana
    

كل ده بـ Ansible.

---

# هل الشركات الكبيرة بتعمل كده؟

أيوه، وده شائع جدًا.

مثلاً:

```text
GitLab

↓

Ansible

↓

Virtualization Platform

↓

Create VM

↓

Install OS

↓

Install Kubernetes

↓

Deploy Applications
```

أو:

```text
Jenkins

↓

Terraform

↓

Ansible

↓

Kubernetes
```

---

# الأفضل من Ansible فقط

في البيئات الاحترافية غالبًا بتفصل بين مسؤوليات الأدوات:

```text
Terraform
        │
        ▼
Create VM on FusionCompute

        │
        ▼
Ansible
        │
Configure Linux
Install containerd
Install Kubernetes
Join Cluster

        │
        ▼
ArgoCD

Deploy Applications
```

- **Terraform**: مسؤول عن الـ Infrastructure (VMs، الشبكات، الأقراص) إذا كان فيه Provider مناسب أو تقدر تستخدم APIs.
    
- **Ansible**: مسؤول عن إعداد أنظمة التشغيل وتثبيت Kubernetes.
    
- **Argo CD**: مسؤول عن نشر التطبيقات وإدارتها بطريقة GitOps.
    

---

# بالنسبة لـ FusionCompute

لو مفيش Terraform Provider رسمي لـ FusionCompute، فما زال عندك خيارين:

1. استخدام **Ansible + FusionCompute REST API** لإنشاء الـ VMs.
    
2. أو تطوير Terraform Provider داخلي لو عندكم احتياج طويل المدى (وده مجهود أكبر).
    

---

## لو كنت مكانك

بما إنك شغال كـ Platform/Kubernetes Administrator، أنا هبني Solution بالشكل ده:

```text
GitLab
   │
   ▼
CI/CD Pipeline
   │
   ▼
Ansible
   │
   ├── Create VMs on FusionCompute (REST API)
   ├── Configure Network
   ├── Configure Storage
   ├── Install Ubuntu
   ├── Install containerd
   ├── Install kubeadm
   ├── Bootstrap HA Cluster
   ├── Install Cilium
   ├── Install MetalLB
   ├── Install Ingress
   ├── Install Prometheus
   ├── Install Grafana
   ├── Install Longhorn أو CSI المناسب
   └── Validate Cluster Health
```

ده هيكون قريب جدًا من تجربة EKS Anywhere من ناحية الأتمتة، لكن باستخدام أدوات مرنة تناسب FusionCompute، وبدون الاعتماد على Provider غير مدعوم رسميًا.

-----
سؤال مهم جدًا، والإجابة هي: **Huawei FusionCompute بالفعل عنده APIs**، لكن لازم تتأكد من **الإصدار (Version)** والإمكانيات المفعلة في بيئتكم.

تقدر تتأكد بأكثر من طريقة.

## الطريقة الأولى: من واجهة FusionCompute (الأسهل)

لو عندك صلاحية على FusionCompute أو VRM:

ابحث عن:

- API
    
- REST API
    
- Open API
    
- Developer Guide
    
- SDK
    

في بعض الإصدارات هتلاقي Documentation أو قسم للمطورين.

---

## الطريقة الثانية: جرب الوصول للـ API

لو عنوان الـ VRM عندك مثلًا:

```text
https://10.10.10.100
```

جرب تفتح:

```text
https://10.10.10.100/service/
```

أو

```text
https://10.10.10.100/docs
```

أو

```text
https://10.10.10.100/swagger
```

وجود Swagger ليس مضمونًا، لكنه موجود في بعض المنتجات الحديثة.

---

## الطريقة الثالثة: اسأل مسؤول النظام

اسأله مباشرة:

> "هل FusionCompute عندنا مفعل فيه Northbound REST API؟"

أو

> "هل فيه Open API أقدر أستخدمه للأتمتة؟"

---

## الطريقة الرابعة: من سطر الأوامر

من جهاز يقدر يوصل للـ VRM:

```bash
curl -k https://<VRM-IP>
```

لو رجع Response من السيرفر، يبقى الخدمة شغالة.

ولو جربت Endpoint معروف من الـ Documentation ورجع JSON بدل HTML، يبقى غالبًا الـ API شغال.

---

## الطريقة الخامسة: Documentation

اعرف إصدار FusionCompute أولًا.

مثلاً:

```text
FusionCompute 8.3
```

أو

```text
FusionCompute 8.5
```

بعدها ابحث عن:

```text
Huawei FusionCompute 8.5 REST API Guide
```

أو

```text
Huawei FusionCompute OpenAPI
```

هتلاقي كل الـ Endpoints الخاصة بإنشاء الـ VM، حذفها، تشغيلها، إيقافها... إلخ.

---

# إيه اللي المفروض يكون موجود في الـ API؟

لو هو API كامل، هتلاقي Endpoints شبيهة بـ:

```text
Login
```

↓

```text
Create VM
```

↓

```text
Delete VM
```

↓

```text
Power On
```

↓

```text
Power Off
```

↓

```text
Clone Template
```

↓

```text
Attach Disk
```

↓

```text
Attach Network
```

↓

```text
Take Snapshot
```

↓

```text
Get VM Status
```

لو لقيت العمليات دي، فأنت تقدر تبني Automation باستخدام Ansible أو أي لغة برمجة.

---

# هل Ansible يقدر يتعامل معاه؟

نعم، حتى لو مفيش Module رسمي لـ FusionCompute.

Ansible يقدر يستدعي أي REST API باستخدام `uri` module، على سبيل المثال:

```yaml
- name: Login to FusionCompute
  uri:
    url: https://<vrm-ip>/...
    method: POST
```

وبعدين:

```yaml
- name: Create VM
  uri:
    url: https://<vrm-ip>/...
    method: POST
```

وهكذا.

---

# لو أنا مكانك

أول حاجة هعملها هي معرفة إصدار المنصة.

نفذ أو اسأل عن:

```text
FusionCompute Version
```

مثلاً:

- FusionCompute 8.2
    
- FusionCompute 8.3
    
- FusionCompute 8.5
    
- FusionCompute 8.6
    

**بعد ما تقولّي الإصدار بالضبط**، أقدر أوضح لك:

- هل الـ REST API متاح في هذا الإصدار.
    
- إزاي تعمل Authentication.
    
- إزاي تعمل Create VM وPower On وDelete VM بالـ API.
    
- وإزاي تربط ده كله مع Ansible لبناء Cluster Kubernetes تلقائيًا.
------
