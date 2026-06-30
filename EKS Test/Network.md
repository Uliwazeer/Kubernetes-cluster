
دى من أهم مراحل التصميم، وأغلب الناس بتختار IPs عشوائى، وده غلط. **الـ IP Plan بيتعمل قبل ما تعمل أول VM**، لأن بعد كده تغييره بيبقى متعب.

خلينا نصممها كأننا فى شركة Enterprise.

---

# أول حاجة لازم تعرفها

فى Kubernetes عندك **4 أنواع من الـ IPs**، والناس بتخلط بينهم.

```
1- VM IPs (Management Network)

2- VIPs

3- Pod CIDR

4- Service CIDR
```

كل واحد دول شبكة مختلفة.

---

# 1) Management Network (دى أهم واحدة)

دى IPs بتاعة الـ VMs نفسها.

مثلاً

```
CP1

172.30.10.11
```

```
CP2

172.30.10.12
```

وهكذا.

دى اللى هتطلبها من الـ Security.

---

# 2) VIP

مثلاً

```
172.30.10.10
```

ده مش مربوط على VM.

ده Virtual IP.

```
VIP

↓

HAProxy

↓

CP1
CP2
CP3
```

كل kubectl هيكلم الـ VIP.

---

# 3) Pod Network

مثلاً

```
10.244.0.0/16
```

دى خاصة بالـ Pods.

مش Team Security هيفتحها.

---

# 4) Service Network

مثلاً

```
10.96.0.0/12
```

دى برضه داخل Kubernetes فقط.

---

# طيب اختار رينج الـ VM ازاى؟

## أول قاعدة

ما تستخدمش نفس رينج الـ Production.

إنت قلت الإنتاج عندكم:

```
172.21.x.x
```

يبقى ابعد عنه.

---

## ثانى قاعدة

اختار Subnet جديدة بالكامل.

مثلاً

```
172.30.10.0/24
```

ليه؟

لأنها:

- سهلة
    
- فيها 254 IP
    
- مفيش تعارض مع الإنتاج
    

---

# مثال أنا هعمله

```
Network

172.30.10.0/24
```

Gateway

```
172.30.10.1
```

DNS

```
172.30.10.2
```

NTP

```
172.30.10.3
```

---

بعدها

```
VIP API

172.30.10.10
```

---

```
VIP Ingress

172.30.10.20
```

---

Harbor

```
172.30.10.30
```

---

Admin

```
172.30.10.40
```

---

CP

```
172.30.10.101

172.30.10.102

172.30.10.103
```

---

Workers

```
172.30.10.111

172.30.10.112

172.30.10.113

172.30.10.114

172.30.10.115

172.30.10.116
```

---

Monitoring

```
172.30.10.130
```

---

Logging

```
172.30.10.140
```

---

Registry Backup

```
172.30.10.150
```

---

# ليه سبت فراغات؟

عشان بعد سنة.

لو زودت

```
Worker7
```

مش هيبقى

```
172.30.10.117
```

ممكن يبقى

```
172.30.10.120
```

لو فيه Nodes خرجت.

وده Design أحسن.

---

# طيب لو بعدين بقى عندى Cluster تانى؟

أعمل

```
Production

172.21.0.0/24

----------------

Testing

172.30.10.0/24

----------------

Development

172.30.20.0/24

----------------

DR

172.30.30.0/24
```

يبقى كل Cluster ليه Subnet.

---

# هطلب إيه من Security؟

هقولهم:

> نحتاج تخصيص VLAN أو Subnet جديدة لبيئة Kubernetes Test.

مثلاً:

```
172.30.10.0/24
```

Gateway

```
172.30.10.1
```

ويتم حجز IPs التالية:

|الاستخدام|IP|
|---|---|
|Gateway|172.30.10.1|
|DNS|172.30.10.2|
|NTP|172.30.10.3|
|API VIP|172.30.10.10|
|Ingress VIP|172.30.10.20|
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

# نقطة مهمة جدًا

أنت قلت إن الإنتاج بيستخدم **VIP + HAProxy + Cilium**.

إذن أنصحك تعمل **نفس التصميم** في التست، لكن **برينج مختلف**.

يعني:

- **Production**
    
    - Management: `172.21.x.x`
        
    - Pod CIDR: `10.244.0.0/16` (أو حسب الإنتاج)
        
    - Service CIDR: `10.96.0.0/12`
        
- **Test**
    
    - Management: `172.30.10.0/24`
        
    - Pod CIDR: يفضل يكون مختلف لو فيه أي ربط بين البيئتين، مثل `10.250.0.0/16`
        
    - Service CIDR: مثل `10.97.0.0/16` أو أي مدى لا يتعارض مع الإنتاج.
        

**لو بيئة التست والإنتاج ممكن يتوصلوا ببعض (Routing أو VPN)، فلا تستخدم نفس Pod CIDR أو Service CIDR** حتى لا يحصل تعارض في التوجيه.

---

## أفضل ممارسة في الشركات الكبيرة

الـ **IP Plan** لا يحدده فريق Kubernetes وحده.

عادةً يتم الاتفاق مع:

- **Network Team**: يخصص VLAN وSubnet ويضمن عدم وجود تعارض مع أي شبكة أخرى.
    
- **Security Team**: يراجع سياسات الـ Firewall والـ ACLs.
    
- **Platform Team**: يحدد احتياجات الكلاستر (VIPs، عدد العقد، الخدمات).
    

بالتالي بدل ما تقول لهم: _"ادوني أي رينج فاضي"_، الأفضل تقول:

> "أحتاج VLAN مخصصة لبيئة Kubernetes Test، مع Subnet /24، وIPين محجوزين للـ VIP، وStatic IPs للعقد، ولا تتعارض مع أي شبكات حالية أو مستقبلية."

وده الأسلوب اللي بيستخدم في أغلب البيئات المؤسسية.

----
معلومة مهمة جدًا، وخلت الصورة أوضح.

لكن فيه نقطة لازم نتأكد منها لأن فيه خلط فى المصطلحات.

أنت قلت:

> **"HAProxy موجود على الكلاستر واللود بلنسر هو VIP."**

الـ **VIP مش Load Balancer**.

الـ **VIP مجرد IP**، ولازم فيه حاجة "تمسك" الـ IP ده وتعمل Load Balancing.

يعنى الـ VIP لوحده مش بيعمل حاجة.

---

## غالبًا عندكم Architecture شبه ده

```text
                    Clients
                       │
                       ▼
              172.21.x.x (VIP)
                       │
        ┌──────────────┴──────────────┐
        │                             │
   HAProxy Pod 1                HAProxy Pod 2
        │                             │
        └──────────────┬──────────────┘
                       │
                  Kubernetes Services
                       │
                     Pods
```

أو

```text
               VIP
                │
                ▼
        Service Type=LoadBalancer
                │
          HAProxy Pods
                │
          Backend Services
```

---

# بالنسبة لـ API Server

السؤال الأهم:

**أنت لما تعمل:**

```bash
kubectl config view
```

هتلاقى:

```yaml
server: https://172.21.x.x:6443
```

ولا

```yaml
server: https://cp1:6443
```

ولا

```yaml
server: https://api.cluster.local:6443
```

لأن ده هيحدد هل الـ API Server كمان وراه VIP ولا لأ.

---

# لو هتعمل Test Cluster

أنا أنصحك بنفس التصميم ده.

## VIP 1

```
172.30.10.10
```

للـ API Server.

---

## VIP 2

```
172.30.10.20
```

للـ Ingress / HAProxy.

---

## VIP 3

```
172.30.10.200-220
```

للـ Services من نوع `LoadBalancer`.

ولو هتستخدم Cilium، يبقى **Cilium LoadBalancer IPAM** هو اللي يوزع الـ IPs من الـ Pool ده.

---

# بس هنا فيه سؤال مهم جدًا

أنت قلت:

> **"HAProxy موجود على الكلاستر."**

طيب **مين ماسك الـ VIP؟**

لأن الـ HAProxy Pod **لوحده ميقدرش يمتلك VIP على الشبكة الخارجية**.

لازم يكون فيه حاجة من دول:

- **Keepalived**
    
- **Cilium L2 Announcements**
    
- **MetalLB (L2 أو BGP)**
    
- **F5**
    
- **A10**
    
- **Router BGP**
    
- **ARP Announcement**
    

---

## أنا شاكك بنسبة كبيرة إنكم بتستخدموا Cilium L2 Announcements

لأنك قلت:

- Cilium
    
- VIP
    
- HAProxy
    

وده تصميم منتشر جدًا:

```text
VIP
        │
        ▼
Cilium L2 Announcement
        │
        ▼
HAProxy Service
        │
        ▼
HAProxy Pods
```

هنا Cilium هو اللى بيعلن على الشبكة:

> "أنا صاحب الـ VIP ده"

عن طريق Gratuitous ARP (أو NDP فى IPv6)، والـ Switch يوجه الترافيك للعقدة الصحيحة.

---

# أنا عايز منك تعمل حاجة بسيطة على الـ Production

ابعتلى ناتج الأوامر دى (بعد إخفاء أى بيانات حساسة لو لزم الأمر):

```bash
kubectl get svc -A
```

```bash
kubectl get pods -A | grep -i haproxy
```

```bash
kubectl get pods -A | grep -i cilium
```

```bash
kubectl get ippools -A
```

ولو الأمر الأخير مش موجود:

```bash
kubectl get crd | grep -i cilium
```

**من الأربع أوامر دول هعرف بنسبة 99% إزاى الـ VIP متنفذ عندكم، وساعتها أقدر أخليك تعمل نفس التصميم حرفيًا على Huawei FusionCompute بدون اختلاف عن الـ Production.**

-------

دى أحسن خطوة ممكن تعملها. 👍

بدل ما نبنى Test Cluster من الصفر، **هنعمل Reverse Engineering للـ Production** ونخلى الـ Test شبهه بنسبة 95%.

أنا هقسمها زى ما الـ Platform Engineers بيعملوا.

---

# المرحلة الأولى: معلومات الـ Cluster

```bash
kubectl version
```

```bash
kubectl cluster-info
```

```bash
kubectl get nodes -o wide
```

```bash
kubectl get node -o yaml
```

---

# المرحلة الثانية: Control Plane

```bash
kubectl get pods -n kube-system -o wide
```

---

اعرف الـ API Server

```bash
kubectl config view
```

---

اعرف الـ Endpoint

```bash
kubectl get endpoints kubernetes -o yaml
```

---

# المرحلة الثالثة: Networking

### CNI

```bash
kubectl get pods -A | grep -Ei "cilium|calico|flannel|weave|ovn"
```

---

لو Cilium

```bash
kubectl get ciliumnodes
```

---

```bash
kubectl get ciliumendpoints -A
```

---

```bash
kubectl get crd | grep cilium
```

---

```bash
kubectl get cm -n kube-system cilium-config -o yaml
```

---

# المرحلة الرابعة

اعرف

Pod CIDR

```bash
kubectl get nodes -o jsonpath='{range .items[*]}{.metadata.name}{" "}{.spec.podCIDR}{"\n"}{end}'
```

---

Service CIDR

(من الـ API Server)

```bash
kubectl -n kube-system get pod -l component=kube-apiserver -o yaml
```

وابحث عن

```text
service-cluster-ip-range
```

---

# المرحلة الخامسة

Ingress

```bash
kubectl get ingressclass
```

---

```bash
kubectl get pods -A | grep ingress
```

---

```bash
kubectl get svc -A | grep LoadBalancer
```

---

# المرحلة السادسة

HAProxy

```bash
kubectl get pods -A | grep haproxy
```

---

```bash
kubectl get svc -A | grep haproxy
```

---

```bash
kubectl get deploy -A | grep haproxy
```

---

# المرحلة السابعة

VIP

```bash
kubectl get svc -A
```

---

```bash
kubectl get svc -A -o wide
```

---

# المرحلة الثامنة

Cilium LB

```bash
kubectl get ippools
```

---

```bash
kubectl get l2announcementpolicies
```

---

```bash
kubectl get ciliumloadbalancerippools
```

---

```bash
kubectl get ciliuml2announcementpolicies
```

---

# المرحلة التاسعة

Storage

```bash
kubectl get sc
```

---

```bash
kubectl get pv
```

---

```bash
kubectl get pvc -A
```

---

# المرحلة العاشرة

Monitoring

```bash
kubectl get pods -A | grep prometheus
```

---

```bash
kubectl get pods -A | grep grafana
```

---

# المرحلة الحادية عشر

Metrics

```bash
kubectl top nodes
```

---

```bash
kubectl top pods -A
```

---

# المرحلة الثانية عشر

Cert Manager

```bash
kubectl get pods -A | grep cert
```

---

# المرحلة الثالثة عشر

DNS

```bash
kubectl get pods -A | grep dns
```

---

```bash
kubectl get svc -n kube-system
```

---

# المرحلة الرابعة عشر

LoadBalancer

```bash
kubectl get svc -A
```

---

# المرحلة الخامسة عشر

Namespaces

```bash
kubectl get ns
```

---

# المرحلة السادسة عشر

Versions

```bash
kubectl get deploy -A
```

---

# المرحلة السابعة عشر

Resources

```bash
kubectl describe node
```

---

# المرحلة الثامنة عشر

Admission

```bash
kubectl get validatingwebhookconfigurations
```

---

```bash
kubectl get mutatingwebhookconfigurations
```

---

# المرحلة التاسعة عشر

Runtime

على أى Node

```bash
crictl info
```

---

أو

```bash
containerd --version
```

---

# المرحلة العشرون

OS

على كل Node

```bash
cat /etc/os-release
```

---

```bash
uname -r
```

---

# المرحلة الحادية والعشرون

kubeadm

```bash
kubeadm config view
```

---

# المرحلة الثانية والعشرون

Certificates

```bash
kubeadm certs check-expiration
```

---

# المرحلة الثالثة والعشرون

Etcd

```bash
kubectl -n kube-system get pods | grep etcd
```

---

```bash
etcdctl endpoint health
```

---

# المرحلة الرابعة والعشرون

الكوماند ده مهم جدًا

```bash
kubectl get all -A -o wide
```

---

# المرحلة الخامسة والعشرون

أهم كوماند بالنسبة ليا

```bash
kubectl get svc -A -o wide
```

---

# ولو عندك صلاحية SSH على الـ Control Plane

عايز كمان الملفات دى:

```bash
/etc/kubernetes/manifests/
```

خصوصًا:

```text
kube-apiserver.yaml

kube-controller-manager.yaml

kube-scheduler.yaml

etcd.yaml
```

---

# والأهم من كل ده

أنا عايز معلومات عن **البنية التحتية** نفسها، مش Kubernetes بس.

## من فريق الـ Infrastructure أو الـ Hypervisor:

- عدد الـ Control Plane VMs ومواصفاتها (vCPU/RAM/Disk).
    
- عدد الـ Worker VMs ومواصفاتها.
    
- هل الـ HAProxy داخل الكلاستر فعلًا ولا فيه Load Balancer خارجي؟
    
- هل فيه Keepalived أو Cilium L2 Announcements أو MetalLB؟
    
- الـ VIPs المستخدمة وعناوينها (بعد إخفاء الأرقام الحقيقية إذا لزم).
    
- الـ VLAN ID الخاصة بالكلاستر.
    
- الـ Subnet وقناع الشبكة والـ Gateway.
    
- الـ DNS Servers.
    
- الـ NTP Servers.
    
- نوع الـ Storage (Local، SAN، NFS، CSI...).
    

---

# أنصحك بحاجة هتوفر علينا وقت كبير جدًا

بدل ما تنفذ **30 أمر واحد واحد**، شغّل السكربت التالي على الـ Production (لو مسموح بسياسات الشركة)، وهو هيجمع أغلب المعلومات في ملفات داخل مجلد واحد. **راجع المخرجات واحذف أو أخفِ أي معلومات حساسة قبل مشاركتها.**

```bash
mkdir -p cluster-audit

kubectl version > cluster-audit/version.txt
kubectl cluster-info > cluster-audit/cluster-info.txt
kubectl get nodes -o wide > cluster-audit/nodes.txt
kubectl get ns > cluster-audit/namespaces.txt
kubectl get pods -A -o wide > cluster-audit/pods.txt
kubectl get svc -A -o wide > cluster-audit/services.txt
kubectl get ingressclass > cluster-audit/ingressclass.txt
kubectl get sc > cluster-audit/storageclass.txt
kubectl get pv > cluster-audit/pv.txt
kubectl get pvc -A > cluster-audit/pvc.txt
kubectl get crd > cluster-audit/crds.txt
kubectl get validatingwebhookconfigurations > cluster-audit/validating-webhooks.txt
kubectl get mutatingwebhookconfigurations > cluster-audit/mutating-webhooks.txt
kubectl get cm -n kube-system > cluster-audit/kube-system-configmaps.txt
kubectl get svc -A -o yaml > cluster-audit/services.yaml
kubectl config view --minify > cluster-audit/kubeconfig-minify.yaml
```

بعد ما تجمع المعلومات (مع إزالة أي بيانات حساسة مثل الشهادات أو الـ Tokens أو عناوين IP الحقيقية إذا كانت سياسة الشركة تمنع مشاركتها)، ابعتها لي، وسأبني لك **Test Cluster مطابق للـ Production** يشمل:

- نفس الـ Networking.
    
- نفس الـ VIP Design.
    
- نفس Cilium.
    
- نفس HAProxy.
    
- نفس Ingress.
    
- نفس Monitoring.
    
- نفس Storage.
    
- نفس Security Settings.
    
- ونفس الـ Deployment Workflow على Huawei FusionCompute باستخدام Ansible.