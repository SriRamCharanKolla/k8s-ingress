# 🚀 Kubernetes Ingress & AWS ALB Controller — Mistakes & Learnings

ఈ డాక్యుమెంట్‌లో మనం `k8s-ingress` ప్రాజెక్ట్ డెవలప్‌మెంట్ మరియు AWS Load Balancer Controller (ALB) సెటప్ చేసే సమయంలో ఎదుర్కొన్న ముఖ్యమైన తప్పులు (Mistakes), ఎర్రర్స్ (Errors), వాటికి గల కారణాలు (Root Causes), మరియు భవిష్యత్ రిఫరెన్స్ కోసం పరిష్కారాలు (Solutions & Best Practices) స్పష్టంగా పొందుపరచబడ్డాయి.

---

## 📑 విషయ సూచిక (Table of Contents)
1. [Mistake 1: Deployment లో Pod Template Indentation లోపం](#-mistake-1-deployment-లో-pod-template-indentation-లోపం)
2. [Mistake 2: Ingress `rules.http.paths` Hierarchy & Indentation లోపం](#-mistake-2-ingress-ruleshttppaths-hierarchy--indentation-లోపం)
3. [Mistake 3: ACM Certificate `Pending validation` & ALB 0 Listeners Error](#-mistake-3-acm-certificate-pending-validation--alb-0-listeners-error)
4. [Mistake 4: సర్టిఫికెట్ `ISSUED` అయినా Ingress లో పాత ఎర్రర్ కనిపించడం & Reconciliation Lag](#-mistake-4-సర్టిఫికెట్-issued-అయినా-ingress-లో-పాత-ఎర్రర్-కనిపించడం--reconciliation-lag)
5. [Mistake 5: Route 53 Record Name లో Typo & Domain Duplication (`DNS_PROBE_FINISHED_NXDOMAIN`)](#-mistake-5-route-53-record-name-లో-typo--domain-duplication-dns_probe_finished_nxdomain)
6. [Route 53 Alias Record Setup (బ్రౌజర్‌లో యాక్సెస్ చేయడానికి సరైన పద్ధతి)](#-route-53-alias-record-setup-బ్రౌజర్‌లో-యాక్సెస్-చేయడానికి-సరైన-పద్ధతి)
7. [AWS Load Balancer Controller & Ingress Debugging Cheat Sheet](#-aws-load-balancer-controller--ingress-debugging-cheat-sheet)
8. [Resource Cleanup & AWS Pricing Guide (ACM, Route 53, CloudFront & Teardown)](#-resource-cleanup--aws-pricing-guide-acm-route-53-cloudfront--teardown)

---

## 📌 Mistake 1: Deployment లో Pod Template Indentation లోపం

### 🛑 Error Message
```text
$ kubectl apply -f manifest.yaml
service/app1 created
Error from server (BadRequest): error when creating "manifest.yaml": Deployment in version "v1" cannot be handled as a Deployment: strict decoding error: unknown field "spec.selector.template"
```

### 🔍 Root Cause (ఎందుకు వచ్చింది?)
[manifest.yaml](file:///Users/sriramcharankolla/Desktop/DevOps/k8s-ingress/app1/manifest.yaml#L9-L25) లో `spec.selector` మరియు `spec.template` రెండు కూడా `spec` కింద సమానమైన లెవెల్ (Siblings) లో ఉండాలి.
కానీ పొరపాటున `template:` ని `selector:` లోపల (4 spaces) రాయడం వల్ల, కుబెర్‌నెటీస్ దీనిని `spec.selector.template` గా గుర్తించి strict decoding ఎర్రర్ ఇచ్చింది.

### ❌ తప్పు కోడ్ (Wrong Code):
```yaml
spec:
  replicas: 2
  selector:
    matchLabels:
      project: roboshop
      tier: frontend
      component: app1
    template:                 # ❌ తప్పు: selector లోపల ఉంది (4 spaces)
      metadata:
        labels: ...
      spec:
        containers: ...
```

### ✅ సరైన కోడ్ (Fixed Code in [manifest.yaml:L9-L25](file:///Users/sriramcharankolla/Desktop/DevOps/k8s-ingress/app1/manifest.yaml#L9-L25)):
```yaml
spec:
  replicas: 2
  selector:                   # Sibling 1 (2 spaces)
    matchLabels:
      project: roboshop
      tier: frontend
      component: app1
  template:                   # ✅ సరిగ్గా: spec కింద నేరుగా ఉంది (2 spaces)
    metadata:
      labels:
        project: roboshop
        tier: frontend
        component: app1
    spec:
      containers:
      - name: app1
        image: ramcharankola2/app1:1.0.0
```

---

## 📌 Mistake 2: Ingress `rules.http.paths` Hierarchy & Indentation లోపం

### 🛑 Error Message
```text
$ kubectl apply -f manifest.yaml
deployment.apps/app1 unchanged
service/app1 unchanged
Error from server (BadRequest): error when creating "manifest.yaml": Ingress in version "v1" cannot be handled as a Ingress: strict decoding error: unknown field "spec.rules[0].backend", unknown field "spec.rules[0].path", unknown field "spec.rules[0].paths"
```

### 🔍 Root Cause (ఎందుకు వచ్చింది?)
Kubernetes `networking.k8s.io/v1` Ingress నియమాల ప్రకారం:
* `spec.rules` కింద కేవలం `host` మరియు `http` మాత్రమే నేరుగా ఉంటాయి.
* `paths`, `path`, `pathType`, మరియు `backend` ఫీల్డ్స్ అన్నీ కూడా తప్పనిసరిగా `http:` లోపల నెస్టెడ్ అయి ఉండాలి.
* ఇండింటేషన్ తేడా వల్ల `paths`, `path`, మరియు `backend` ఫీల్డ్స్ అన్నీ `rules[0]` కింద డైరెక్ట్ కీస్ గా పార్స్ అయ్యాయి.

### ❌ తప్పు కోడ్ (Wrong Code):
```yaml
spec:
  rules:
  -  host: "app1.aitechapp.fun"
     http:                     # ❌ కింద ఏమీ లేదు
     paths:                    # ❌ http కి సమానమైన లెవెల్‌లో ఉంది
     - pathType: Prefix
     path: "/"                 # ❌ http కి సమానమైన లెవెల్‌లో ఉంది
     backend:                  # ❌ http కి సమానమైన లెవెల్‌లో ఉంది
       service:
         name: app1
         port:
           number: 80 
```

### ✅ సరైన కోడ్ (Fixed Code in [manifest.yaml:L52-L64](file:///Users/sriramcharankolla/Desktop/DevOps/k8s-ingress/app1/manifest.yaml#L52-L64)):
```yaml
spec:
  ingressClassName: alb        # ✅ AWS Load Balancer Controller కోసం తప్పనిసరి
  rules:
  - host: "app1.aitechapp.fun"
    http:
      paths:
      - pathType: Prefix
        path: "/"
        backend:
          service:
            name: app1
            port:
              number: 80
```

---

## 📌 Mistake 3: ACM Certificate `Pending validation` & ALB 0 Listeners Error

### 🛑 Symptoms & Error Message
AWS EC2 Console లో Application Load Balancer (`k8s-roboshop-...`) క్రియేట్ అయ్యి **Active** స్టేట్‌లో ఉంటుంది, కానీ లోపల చూస్తే:
> **Listeners and rules (0):** *"No listener. You don't have a listener associated with this load balancer."*

మరియు `kubectl describe ingress app1` రన్ చేసినప్పుడు Warning ఈవెంట్ కనిపిస్తుంది:
```text
Warning  FailedDeployModel  ingress  Failed deploy model due to operation error Elastic Load Balancing v2: CreateListener, 
https response error StatusCode: 400, api error UnsupportedCertificate: 
The certificate 'arn:aws:acm:us-east-1:069416262340:certificate/e09fdd96-80c6-4814-a235-89c17501f08f' 
must have a fully-qualified domain name, a supported signature, and a supported key size.
```

### 🔍 Root Cause (ఎందుకు వచ్చింది?)
1. [manifest.yaml:L46-L47](file:///Users/sriramcharankolla/Desktop/DevOps/k8s-ingress/app1/manifest.yaml#L46-L47) లో కేవలం HTTPS Port 443 మాత్రమే కాన్ఫిగర్ చేయబడింది:
   ```yaml
   alb.ingress.kubernetes.io/listen-ports: '[{"HTTPS": 443}]'
   alb.ingress.kubernetes.io/certificate-arn: 'arn:aws:acm:us-east-1:069416262340:certificate/e09fdd96-80c6-4814-a235-89c17501f08f'
   ```
2. డొమైన్ పేరు `*.aitechapp.fun` మరియు సర్టిఫికెట్ ARN సరైనవే అయినప్పటికీ, AWS Certificate Manager (ACM) లో ఆ సర్టిఫికెట్ ఇంకా **`Pending validation`** స్టేటస్‌లో ఉంది.
3. AWS Elastic Load Balancing నియమాల ప్రకారం, **`Issued`** స్టేటస్‌లో లేని సర్టిఫికెట్‌ను ఏ లోడ్ బ్యాలెన్సర్‌కు అటాచ్ చేయలేము. అందుకే AWS `CreateListener` రిక్వెస్ట్‌ను రిజెక్ట్ చేసింది.
4. ఫలితంగా ALB క్రియేట్ అయినా కూడా 0 లిజినర్లతో ఆగిపోయింది.

### 🛠️ పరిష్కారం (Fix Steps):

#### Step 1: DNS Validation పూర్తి చేయడం (Route 53 లేదా DNS Registrar లో)
1. AWS Console లో **Certificate Manager (ACM)** కి వెళ్ళండి.
2. సంబంధిత Certificate ID (`e09fdd96-80c6-4814-a235-89c17501f08f`) పై క్లిక్ చేయండి.
3. **Domains** సెక్షన్‌లో:
   * Route 53 లో డొమైన్ ఉంటే: **"Create records in Route 53"** బటన్ క్లిక్ చేయండి.
   * వేరే ప్రొవైడర్ (GoDaddy, Namecheap) అయితే: CNAME Name & CNAME Value ని మీ DNS లో యాడ్ చేయండి.
4. 2-5 నిమిషాల్లో స్టేటస్ **`Issued`** (Green Checkmark) గా మారుతుంది.

#### Step 2: Ingress ని Re-apply చేయడం
సర్టిఫికెట్ `Issued` అయిన తర్వాత:
```bash
kubectl apply -f app1/manifest.yaml
```
ఇప్పుడు ALB లో Port 443 (HTTPS) లిజినర్ మరియు రూల్స్ ఆటోమేటిక్‌గా క్రియేట్ అవుతాయి.

#### 💡 ప్రో-టిప్ (టెస్టింగ్ కోసం క్విక్ ట్రిక్):
సర్టిఫికెట్ వాలిడేషన్ ఇంకా పూర్తి కానప్పుడు ట్రాఫిక్ టెస్ట్ చేయాలనుకుంటే, [manifest.yaml:L46-L47](file:///Users/sriramcharankolla/Desktop/DevOps/k8s-ingress/app1/manifest.yaml#L46-L47) లో తాత్కాలికంగా HTTP Port 80 ని వాడవచ్చు:
```yaml
alb.ingress.kubernetes.io/listen-ports: '[{"HTTP": 80}]'
# alb.ingress.kubernetes.io/certificate-arn: ...
```

---

## 📌 Mistake 4: సర్టిఫికెట్ `ISSUED` అయినా Ingress లో పాత ఎర్రర్ కనిపించడం & Reconciliation Lag

### 🛑 Error Message / Confusion
Route 53 లో DNS CNAME రికార్డ్ యాడ్ చేసి, ACM లో సర్టిఫికెట్ **`Issued`** అయినప్పటికీ, టెర్మినల్ లో `kubectl describe ingress app1` రన్ చేస్తే ఇంకా పాత ఎర్రర్ చూపిస్తుంది:
```text
Warning  FailedDeployModel  5m14s  ingress  Failed deploy model due to operation error Elastic Load Balancing v2: CreateListener...
api error UnsupportedCertificate: The certificate '...' must have a fully-qualified domain name...
```
మరియు AWS Console లో ALB లో ఇంకా 0 Listeners కనిపిస్తాయి.

### 🔍 Root Cause (ఎందుకు ఇలా జరుగుతుంది?)
1. **Kubernetes Event Retention Policy:**
   * కుబెర్‌నెటీస్‌లో `kubectl describe` చూపించే `Events` హిస్టారికల్ లాగ్స్ మాత్రమే. ఇవి డిఫాల్ట్‌గా **1 గంట (60 నిమిషాలు)** పాటు etcd లో స్టోర్ అయ్యి ఉంటాయి (`--event-ttl=1h`).
   * ఎర్రర్ వచ్చిన సమయాన్ని గమనించండి (`Age: 5m14s`). సర్టిఫికెట్ AWS ACM లో `ISSUED` అయిన సమయానికి సరిగ్గా **కొన్ని సెకన్ల ముందు** కంట్రోలర్ చివరిసారిగా ట్రై చేసి ఫెయిల్ అయిన ఈవెంట్ అది.
2. **Kubernetes Controller Watch Mechanism (No Real-Time AWS Sync):**
   * AWS Load Balancer Controller అనేది కుబెర్‌నెటీస్ ఇన్గ్రెస్ ఆబ్జెక్ట్‌లలో మార్పులను (Events) మాత్రమే గమనిస్తుంది (Watch).
   * AWS క్లౌడ్‌లో (ACM లో) సర్టిఫికెట్ స్టేటస్ మారిన వెంటనే కుబెర్‌నెటీస్ కంట్రోలర్‌కు నోటిఫికేషన్ రాదు.
   * కుబెర్‌నెటీస్ లో ఇన్గ్రెస్ రిసోర్స్ అప్‌డేట్ అయ్యే వరకు లేదా బ్యాక్‌గ్రౌండ్ లాంగ్ పీరియాడిక్ సింక్ లూప్ జరిగే వరకు కంట్రోలర్ రీ-ట్రై చేయదు.

### 🛠️ పరిష్కారం (Controller ని రీ-ట్రిగ్గర్ చేయడం ఎలా?):
సర్టిఫికెట్ `ISSUED` అయిన వెంటనే కంట్రోలర్ కొత్త స్టేట్‌తో ALB లిజినర్‌ను రీ-క్రియేట్ చేయాలంటే:

#### Option 1: Ingress ని Annotate చేసి Reconcile ట్రిగ్గర్ చేయడం (Recommended):
```bash
kubectl annotate ingress app1 reconcile="$(date +%s)" --overwrite
```

#### Option 2: Ingress మ్యానిఫెస్ట్‌ను Re-apply చేయడం:
```bash
kubectl apply -f app1/manifest.yaml
```

#### Option 3: Controller Pod ని Restart చేయడం:
```bash
kubectl rollout restart deployment/aws-load-balancer-controller -n kube-system
```
> **ఫలితం:** కంట్రోలర్ వెంటనే ACM నుండి `ISSUED` సర్టిఫికెట్‌ను ఫెచ్ చేసి ALB లో **HTTPS: 443** లిజినర్ మరియు టార్గెట్ గ్రూప్‌ను క్రియేట్ చేస్తుంది. `kubectl get ingress app1` లో `ADDRESS` కాలమ్‌లో ALB DNS నేమ్ కనిపిస్తుంది.

---

---

## 📌 Mistake 5: Route 53 Record Name లో Typo & Domain Duplication (`DNS_PROBE_FINISHED_NXDOMAIN`)

### 🛑 Error Message (in Chrome Browser)
```text
This site can't be reached
Check if there is a typo in app1.aitechapp.fun.
DNS_PROBE_FINISHED_NXDOMAIN
```

### 🔍 Root Cause (ఎందుకు ఇలా జరిగింది?)
Route 53 లో హోస్టెడ్ జోన్ `aitechapp.fun` లోపల కొత్త రికార్డ్ క్రియేట్/ఎడిట్ చేసేటప్పుడు:
1. **Domain Duplication:** Route 53 ఇన్‌పుట్ బాక్స్ పక్కన ఆటోమేటిక్‌గా `.aitechapp.fun` ఉంటుంది. కానీ రికార్డ్ నేమ్ బాక్స్‌లో కేవలం `app1` అని రాయడానికి బదులుగా మొత్తం `app1.aitectapp.fun` అని టైప్ చేయడం జరిగింది.
2. **Typo:** `aitechapp` కి బదులుగా `aitectapp` ('ch' కి బదులుగా 'ct') టైప్ అయ్యింది.
3. ఫలితంగా Route 53 లో క్రియేట్ అయిన రికార్డ్ పేరు:
   ```text
   app1.aitectapp.fun.aitechapp.fun
   ```
4. బ్రౌజర్‌లో మనం `app1.aitechapp.fun` అని కొట్టినప్పుడు, DNS లో ఆ పేరుతో అసలు రికార్డే లేకపోవడం వల్ల Chrome **`DNS_PROBE_FINISHED_NXDOMAIN`** (Non-Existent Domain) ఎర్రర్ ఇచ్చింది.

### 🛠️ పరిష్కారం (Fix):
1. Route 53 `aitechapp.fun` హోస్టెడ్ జోన్‌లోకి వెళ్ళండి.
2. తప్పుగా ఉన్న రికార్డును ఎడిట్ చేయండి లేదా డిలీట్ చేసి కొత్తది క్రియేట్ చేయండి.
3. **Record name** బాక్స్‌లో కేవలం **`app1`** అని మాత్రమే టైప్ చేయండి (మొత్తం డొమైన్ పేరు రాయకూడదు).
4. `app2` కోసం కూడా అలాగే Record name బాక్స్‌లో కేవలం **`app2`** అని టైప్ చేసి సేవ్ చేయండి.

---

## 📌 Route 53 Alias Record Setup (బ్రౌజర్‌లో యాక్సెస్ చేయడానికి సరైన పద్ధతి)

ALB మరియు HTTPS లిజినర్ క్రియేట్ అయిన తర్వాత, బ్రౌజర్‌లో `https://app1.aitechapp.fun` మరియు `https://app2.aitechapp.fun` ఓపెన్ అవ్వాలంటే Route 53 లో డొమైన్‌ను ALB కి మ్యాప్ చేయాలి.

### స్టెప్స్:
1. AWS Route 53 లో **Hosted Zones** $\rightarrow$ `aitechapp.fun` లోకి వెళ్ళండి.
2. **"Create record"** బటన్ క్లిక్ చేయండి.
3. వివరాలు ఇలా నమోదు చేయండి:
   * **Record name:** కేవలం **`app1`** అని మాత్రమే ఇవ్వండి (పక్కన ఆటోమేటిక్‌గా `.aitechapp.fun` ఉంటుంది).
   * **Record type:** `A - Routes traffic to an IPv4 address and some AWS resources`
   * **Alias:** స్విచ్ ఆన్ (Toggle ON) చేయండి.
   * **Route traffic to:** 
     * *Alias to Application and Classic Load Balancer*
     * *Region:* `us-east-1`
     * *Choose load balancer:* మీ ALB అయిన `k8s-roboshop-37213adecc` ని ఎంచుకోండి.
4. **Create records** క్లిక్ చేయండి.
5. బ్రౌజర్‌లో **`https://app1.aitechapp.fun`** ఓపెన్ చేస్తే సురక్షితమైన గ్రీన్ లాక్ సింబల్‌తో మీ `app1` కంటైనర్ లోడ్ అవుతుంది!
6. `app2` కోసం కూడా సేమ్ స్టెప్స్ ఫాలో అయ్యి Record name వద్ద కేవలం **`app2`** అని ఇచ్చి సేవ్ చేయండి (`https://app2.aitechapp.fun`).

---

## 📌 AWS Load Balancer Controller & Ingress Debugging Cheat Sheet

భవిష్యత్తులో ఇలాంటి సమస్యలు వచ్చినప్పుడు క్షణాల్లో గుర్తించడానికి ఈ కమాండ్స్ వాడండి:

```bash
# 1. Ingress స్టేటస్ మరియు వార్నింగ్ ఈవెంట్స్ చూడటానికి:
kubectl describe ingress <ingress-name>

# 2. AWS Load Balancer Controller లైవ్ లాగ్స్ చూడటానికి:
kubectl logs -n kube-system deployment/aws-load-balancer-controller --tail=100 -f

# 3. ACM సర్టిఫికెట్ స్టేటస్ చెక్ చేయడానికి:
aws acm describe-certificate \
  --certificate-arn <YOUR_CERTIFICATE_ARN> \
  --region us-east-1 \
  --query "CertificateDetail.Status"

# 4. K9s ద్వారా చూడటానికి (Interactive CLI):
# Terminal లో 'k9s' టైప్ చేసి ':ingress' లేదా ':pods' (in kube-system) లోకి వెళ్లి 'l' నొక్కి లాగ్స్ చూడవచ్చు.
```

---

## 📌 Resource Cleanup & AWS Pricing Guide (ACM, Route 53, CloudFront & Teardown)

ప్రాజెక్ట్ పూర్తయిన తర్వాత క్లౌడ్ రిసోర్సులను క్లీనప్ చేయడం, ఏవి మాన్యువల్‌గా డిలీట్ చేయాలి, ఏవి Terraform ఆటోమేటిక్‌గా డిలీట్ చేస్తుంది, మరియు భవిష్యత్ మెయింటెనెన్స్ ఖర్చులు (Pricing) ఎలా ఉంటాయో ఇక్కడ క్లియర్‌గా డాక్యుమెంట్ చేయబడింది.

---

### 1. Manual Cleanup vs Terraform Destroy (ఏవి డిలీట్ చేయాలి? ఏవి ఉంచాలి?)

#### 🔹 `terraform destroy` పరిధి:
* `terraform-aws-workstation` లోని `terraform destroy` కేవలం Terraform స్టేట్ ఫైల్‌లో ఉన్న రిసోర్స్‌లను (EC2 Workstation, Security Group, మరియు EKS Cluster via `eksctl delete cluster`) మాత్రమే ఆటోమేటిక్‌గా డిలీట్ చేస్తుంది.
* మనం AWS Console లో మాన్యువల్‌గా క్రియేట్ చేసిన **ACM Certificate** మరియు **Route 53 Records** Terraform కి తెలియవు. కాబట్టి అవి ఆటోమేటిక్‌గా డిలీట్ అవ్వవు.

#### 🔹 ఏవి మాన్యువల్‌గా డిలీట్ చేయాలి?
* **Route 53 లోని `app1` మరియు `app2` (A - Alias Records):**
  * క్లస్టర్ మరియు ALB డిలీట్ అయిపోయిన తర్వాత ఈ రికార్డులు డెడ్ ఎండ్ పాయింట్స్ అవుతాయి కాబట్టి, వీటిని Route 53 లో మాన్యువల్‌గా డిలీట్ చేయాలి.

#### 🔹 ఏవి డిలీట్ చేయకుండా ఉంచుకోవచ్చు?
* **ACM Certificate (`*.aitechapp.fun`) & DNS Validation CNAME Record:**
  * **వీటిని డిలీట్ చేయాల్సిన అవసరం లేదు, అలాగే ఉంచుకోవచ్చు!**
  * ఎందుకంటే ACM పబ్లిక్ సర్టిఫికెట్స్ **100% ఉచితం ($0.00)**.
  * వీటిని అలాగే ఉంచడం వల్ల భవిష్యత్తులో వేరే ప్రాజెక్ట్‌లకు మళ్లీ ఇదే సర్టిఫికెట్‌ను నేరుగా వాడుకోవచ్చు (మళ్లీ డొమైన్ వాలిడేషన్ చేసే పని ఉండదు).

---

### 2. AWS ACM Certificate Pricing (సర్టిఫికెట్స్ ఉచితమా? లిమిట్ ఎంత?)

* **Always 100% Free Forever:** AWS Certificate Manager (ACM) లో పబ్లిక్ SSL/TLS సర్టిఫికెట్స్ పూర్తిగా ఉచితం.
* **ఫ్రీ టైర్ ముగిసిన తర్వాత కూడా:** మీ అకౌంట్ 12-నెలల Free Tier అయిపోయినా, లేదా క్రెడిట్స్ వాడుతున్నా సరే:
  * AWS సర్వీసెస్ (ALB, CloudFront, API Gateway) కోసం వాడే Amazon-issued పబ్లిక్ సర్టిఫికెట్లకు ఎప్పటికీ ఎటువంటి ఛార్జ్ పడదు ($0.00).
  * డిఫాల్ట్‌గా ప్రతి AWS అకౌంట్‌లో **2,500 సర్టిఫికెట్స్** వరకు ఉచితంగా క్రియేట్ చేసుకోవచ్చు.
* ⚠️ **గమనిక:** ఎంటర్‌ప్రైజ్ సంస్థల కోసం ఉండే "AWS Private CA" కి మాత్రమే నెలకు $400 ఛార్జ్ ఉంటుంది. మనం వాడే సాధారణ పబ్లిక్ సర్టిఫికెట్లకు రూపాయి కూడా ఛార్జ్ పడదు.

---

### 3. Route 53 Pricing (Hosted Zone vs DNS Records)

| అంశం | ఖర్చు (Pricing) | వివరణ |
| :--- | :--- | :--- |
| **Hosted Zone (`aitechapp.fun`)** | **$0.50 / నెల** (~₹42) | మీరు మీ డొమైన్ కోసం Route 53 లో క్రియేట్ చేసిన హోస్టెడ్ జోన్‌కు నెలకు 50 సెంట్స్ ఛార్జ్ పడుతుంది. |
| **DNS Records (A, CNAME, etc.)** | **$0.00 (ఉచితం)** | హోస్టెడ్ జోన్ లోపల మీరు ఎన్ని రికార్డులు పెట్టినా అదనపు ఛార్జ్ ఏమీ ఉండదు. |
| **Alias Queries to AWS Resources** | **$0.00 (ఉచితం)** | Route 53 నుండి ALB లేదా CloudFront కి పాయింట్ చేసే Alias క్వెరీలకు ఎలాంటి ఛార్జ్ ఉండదు. |

> 💡 **ఖర్చు ఆదా చేసే ట్రిక్:** మీరు డొమైన్‌ను ప్రస్తుతం అస్సలు వాడకపోతే మరియు ఆ 50 సెంట్స్ కూడా ఆదా చేయాలనుకుంటే మాత్రమే Route 53 Hosted Zone ను డిలీట్ చేయాలి. ప్రాక్టీస్ కోసం వాడుతున్నంత కాలం నెలకు కేవలం 50 సెంట్స్ మాత్రమే పడుతుంది.

---

### 4. CloudFront & SSL సర్టిఫికెట్ Pricing

AWS CloudFront లో **Always Free Tier (ఎప్పటికీ ఉచితం)** ఉంటుంది:
1. **CloudFront ఉచిత కోటా (ప్రతి నెలా):**
   * **1 TB (1024 GB)** డేటా ట్రాన్స్‌ఫర్ అవుట్ ప్రతి నెలా ఉచితం.
   * **10,000,000 (1 కోటి)** HTTP / HTTPS రిక్వెస్ట్‌లు ప్రతి నెలా ఉచితం.
2. **CloudFront తో ACM SSL సర్టిఫికెట్ వాడకం:**
   * CloudFront కి మీ కస్టమ్ డొమైన్ (`*.aitechapp.fun`) అటాచ్ చేసి ACM సర్టిఫికెట్ వాడటం **పూర్తిగా ఉచితం ($0.00)**.
   * *ముఖ్య నియమం:* CloudFront గ్లోబల్ సర్వీస్ కాబట్టి, దాని కోసం వాడే ACM సర్టిఫికెట్ తప్పనిసరిగా **`us-east-1` (N. Virginia)** రీజియన్‌లోనే ఉండాలి (మనం క్రియేట్ చేసిన సర్టిఫికెట్ `us-east-1` లోనే ఉంది కాబట్టి, దీన్నే నేరుగా CloudFront కి వాడుకోవచ్చు).

---

### 5. Ingress Project Cleanup & Teardown Checklist (సరైన ఆర్డర్)

క్లౌడ్‌లో బిల్లు పడకుండా క్లీన్‌గా డిలీట్ చేయడానికి ఈ క్రమాన్ని పాటించాలి:

```bash
# Step 1: Workstation లో Ingress & Workloads డిలీట్ చేయండి (ముందుగా ALB డిలీట్ అవ్వాలి)
kubectl delete -f app1/manifest.yaml
kubectl delete -f app2/manifest.yaml

# ⚠️ వెయిట్ చేయండి: AWS Console లో ALB పూర్తిగా డిలీట్ అయ్యే వరకు 1-2 నిమిషాలు వేచి చూడండి. 
# లేకపోతే 'DependencyViolation' ఎర్రర్ వచ్చి VPC డిలీషన్ ఫెయిల్ అవుతుంది.

# Step 2: Route 53 లో అప్లికేషన్ రికార్డ్స్ డిలీట్ చేయండి
# Route 53 Console కి వెళ్లి కేవలం 'app1' మరియు 'app2' Alias A-Records ని డిలీట్ చేయండి.
# (ACM Certificate మరియు CNAME రికార్డ్ ని ఉచితం కాబట్టి అలాగే ఉంచవచ్చు).

# Step 3: మీ లోకల్ లాప్‌టాప్ లో Terraform Destroy రన్ చేయండి
cd /Users/sriramcharankolla/Desktop/DevOps/terraform-aws-workstation
terraform destroy
```
