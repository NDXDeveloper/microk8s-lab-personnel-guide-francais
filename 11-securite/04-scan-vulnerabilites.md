🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 11.4 Scan de vulnérabilités

## Comprendre le scan de vulnérabilités

Imaginez que vous achetez une voiture d'occasion. Avant de l'utiliser, vous voudriez probablement faire une inspection complète : vérifier les freins, l'état du moteur, les airbags, etc. Le scan de vulnérabilités fait la même chose pour vos images de conteneurs : il inspecte chaque composant pour détecter les problèmes de sécurité connus avant que vous ne les déployiez dans votre cluster.

### Pourquoi scanner est crucial

Les images de conteneurs contiennent :
- **Un système d'exploitation de base** (Alpine, Ubuntu, Debian...)
- **Des bibliothèques système** (OpenSSL, glibc...)
- **Des dépendances d'application** (npm packages, Python modules, Java JARs...)
- **Votre code applicatif**

Chacun de ces éléments peut contenir des vulnérabilités. Une seule bibliothèque obsolète avec une faille critique peut compromettre tout votre cluster.

### Le cycle de vie des vulnérabilités

1. **Découverte** : Un chercheur trouve une faille
2. **Publication** : Attribution d'un CVE (Common Vulnerabilities and Exposures)
3. **Patch** : Les mainteneurs créent un correctif
4. **Mise à jour** : Vous devez appliquer le correctif
5. **Vérification** : Scanner pour confirmer la correction

Sans scan régulier, vous naviguez à l'aveugle avec potentiellement des dizaines de failles critiques dans vos conteneurs.

## Types de vulnérabilités

### Classification par sévérité (CVSS)

Les vulnérabilités sont classées selon leur score CVSS (Common Vulnerability Scoring System) :

- **Critical (9.0-10.0)** : Exploitation triviale, impact maximal
  - Exemple : Remote Code Execution sans authentification
  - Action : Patcher immédiatement ou isoler

- **High (7.0-8.9)** : Exploitation facile, impact important
  - Exemple : Élévation de privilèges
  - Action : Patcher sous 7 jours

- **Medium (4.0-6.9)** : Exploitation modérée, impact limité
  - Exemple : Déni de service local
  - Action : Patcher sous 30 jours

- **Low (0.1-3.9)** : Exploitation difficile, impact minimal
  - Exemple : Fuite d'information mineure
  - Action : Patcher lors de la prochaine maintenance

### Types de failles courantes

```yaml
# Exemples de CVEs typiques dans les conteneurs

OS Packages:
  - CVE-2021-44228:  # Log4Shell - Remote Code Execution
    severity: Critical
    package: log4j
    versions: < 2.15.0

  - CVE-2014-0160:   # Heartbleed - Memory disclosure
    severity: High
    package: openssl
    versions: 1.0.1-1.0.1f

Application Dependencies:
  - CVE-2021-3129:   # Laravel RCE
    severity: Critical
    framework: Laravel
    versions: < 8.4.3

  - CVE-2022-22965:  # Spring4Shell
    severity: Critical
    framework: Spring
    versions: < 5.3.18

Container Specific:
  - CVE-2019-5736:   # runC container escape
    severity: High
    component: runc
    impact: Container breakout
```

## Outils de scan populaires

### Trivy - Scanner tout-en-un

Trivy est l'outil le plus populaire pour scanner les images dans Kubernetes :

**Installation dans MicroK8s :**
```bash
# Installation locale de Trivy
sudo snap install trivy

# Ou via apt
sudo apt-get update
sudo apt-get install wget apt-transport-https gnupg lsb-release
wget -qO - https://aquasecurity.github.io/trivy-repo/deb/public.key | sudo apt-key add -
echo "deb https://aquasecurity.github.io/trivy-repo/deb $(lsb_release -sc) main" | sudo tee /etc/apt/sources.list.d/trivy.list
sudo apt-get update
sudo apt-get install trivy
```

**Utilisation basique :**
```bash
# Scanner une image
trivy image nginx:latest

# Scanner avec détails
trivy image --severity CRITICAL,HIGH nginx:latest

# Format JSON pour automation
trivy image -f json nginx:latest > scan-results.json
```

### Grype - Scanner rapide par Anchore

```bash
# Installation
curl -sSfL https://raw.githubusercontent.com/anchore/grype/main/install.sh | sh -s -- -b /usr/local/bin

# Utilisation
grype nginx:latest
grype registry:5000/myapp:v1.0
```

### Clair - Scanner avec API

Clair offre une API pour intégration dans les pipelines :

```yaml
# Déploiement de Clair dans MicroK8s
apiVersion: apps/v1
kind: Deployment
metadata:
  name: clair
  namespace: security
spec:
  replicas: 1
  selector:
    matchLabels:
      app: clair
  template:
    metadata:
      labels:
        app: clair
    spec:
      containers:
      - name: clair
        image: quay.io/coreos/clair:latest
        ports:
        - containerPort: 6060
        - containerPort: 6061
        env:
        - name: CLAIR_CONF
          value: /config/config.yaml
        volumeMounts:
        - name: config
          mountPath: /config
      volumes:
      - name: config
        configMap:
          name: clair-config
```

## Intégration dans MicroK8s

### Scan à la demande avec Trivy

#### Déploiement de Trivy comme Job

```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: trivy-scan
  namespace: security
spec:
  template:
    spec:
      containers:
      - name: trivy
        image: aquasec/trivy:latest
        command:
        - trivy
        - image
        - --severity
        - CRITICAL,HIGH
        - --exit-code
        - "1"  # Fail si vulnérabilités trouvées
        - nginx:latest
      restartPolicy: Never
  backoffLimit: 1
```

#### CronJob pour scan régulier

```yaml
apiVersion: batch/v1
kind: CronJob
metadata:
  name: daily-vulnerability-scan
  namespace: security
spec:
  schedule: "0 2 * * *"  # Tous les jours à 2h du matin
  jobTemplate:
    spec:
      template:
        spec:
          serviceAccountName: scanner-sa
          containers:
          - name: scanner
            image: aquasec/trivy:latest
            command:
            - /bin/sh
            - -c
            - |
              # Scanner toutes les images du cluster
              kubectl get pods --all-namespaces -o jsonpath="{.items[*].spec.containers[*].image}" | \
              tr -s '[[:space:]]' '\n' | \
              sort | uniq | \
              while read image; do
                echo "Scanning $image"
                trivy image --severity CRITICAL,HIGH "$image"
              done
          restartPolicy: OnFailure
```

### Trivy Operator - Scan continu

Le Trivy Operator scanne automatiquement toutes les images déployées :

```bash
# Installation via Helm
microk8s enable helm3
microk8s helm3 repo add aqua https://aquasecurity.github.io/helm-charts/
microk8s helm3 repo update

# Installer l'operator
microk8s helm3 install trivy-operator aqua/trivy-operator \
  --namespace trivy-system \
  --create-namespace \
  --set="trivy.ignoreUnfixed=true"
```

```yaml
# L'operator crée automatiquement des VulnerabilityReports
apiVersion: aquasecurity.github.io/v1alpha1
kind: VulnerabilityReport
metadata:
  name: deployment-nginx
  namespace: default
spec:
  artifact:
    repository: nginx
    tag: latest
  summary:
    criticalCount: 0
    highCount: 2
    mediumCount: 5
    lowCount: 12
```

### Admission Controller avec validation

```yaml
# ValidatingWebhookConfiguration pour bloquer les images vulnérables
apiVersion: admissionregistration.k8s.io/v1
kind: ValidatingWebhookConfiguration
metadata:
  name: vulnerability-scanner
webhooks:
- name: scan.security.io
  clientConfig:
    service:
      name: image-scanner
      namespace: security
      path: /validate
    caBundle: <BASE64_ENCODED_CA>
  rules:
  - apiGroups: ["apps", ""]
    apiVersions: ["v1"]
    resources: ["deployments", "pods"]
    operations: ["CREATE", "UPDATE"]
  admissionReviewVersions: ["v1", "v1beta1"]
  sideEffects: None
  failurePolicy: Fail
```

## Scan dans le pipeline CI/CD

### GitLab CI avec Trivy

```yaml
# .gitlab-ci.yml
stages:
  - build
  - scan
  - deploy

variables:
  IMAGE_NAME: "$CI_REGISTRY_IMAGE:$CI_COMMIT_SHA"

build:
  stage: build
  script:
    - docker build -t $IMAGE_NAME .
    - docker push $IMAGE_NAME

vulnerability-scan:
  stage: scan
  image: aquasec/trivy:latest
  script:
    - trivy image --exit-code 0 --severity LOW,MEDIUM $IMAGE_NAME
    - trivy image --exit-code 1 --severity HIGH,CRITICAL $IMAGE_NAME
  artifacts:
    reports:
      container_scanning: gl-container-scanning-report.json
    paths:
      - gl-container-scanning-report.json
  allow_failure: false

deploy:
  stage: deploy
  script:
    - kubectl set image deployment/myapp app=$IMAGE_NAME
  only:
    - main
  dependencies:
    - vulnerability-scan
```

### GitHub Actions avec Trivy

```yaml
# .github/workflows/security-scan.yml
name: Security Scan

on:
  push:
    branches: [ main ]
  pull_request:
    branches: [ main ]

jobs:
  scan:
    runs-on: ubuntu-latest
    steps:
    - uses: actions/checkout@v2

    - name: Build image
      run: docker build -t myapp:${{ github.sha }} .

    - name: Run Trivy vulnerability scanner
      uses: aquasecurity/trivy-action@master
      with:
        image-ref: myapp:${{ github.sha }}
        format: 'sarif'
        output: 'trivy-results.sarif'
        severity: 'CRITICAL,HIGH'
        exit-code: '1'

    - name: Upload Trivy results to GitHub Security
      uses: github/codeql-action/upload-sarif@v2
      with:
        sarif_file: 'trivy-results.sarif'
```

### Jenkins Pipeline

```groovy
pipeline {
    agent any

    stages {
        stage('Build') {
            steps {
                script {
                    docker.build("myapp:${env.BUILD_ID}")
                }
            }
        }

        stage('Security Scan') {
            steps {
                script {
                    sh """
                        trivy image \
                          --severity HIGH,CRITICAL \
                          --exit-code 1 \
                          myapp:${env.BUILD_ID}
                    """
                }
            }
        }

        stage('Deploy') {
            when {
                branch 'main'
            }
            steps {
                script {
                    sh "kubectl set image deployment/myapp app=myapp:${env.BUILD_ID}"
                }
            }
        }
    }

    post {
        always {
            sh "trivy image -f json myapp:${env.BUILD_ID} > scan-report.json"
            archiveArtifacts artifacts: 'scan-report.json'
        }
    }
}
```

## Registry privé avec scan intégré

### Harbor avec Trivy intégré

Harbor est un registry qui intègre Trivy nativement :

```yaml
# Déploiement de Harbor dans MicroK8s
apiVersion: v1
kind: Namespace
metadata:
  name: harbor
---
# Installation via Helm
# microk8s helm3 repo add harbor https://helm.goharbor.io
# microk8s helm3 install harbor harbor/harbor \
#   --namespace harbor \
#   --set expose.type=ingress \
#   --set expose.ingress.hosts.core=harbor.monlab.local \
#   --set persistence.enabled=true \
#   --set trivy.enabled=true \
#   --set trivy.vulnDB.autoUpdate=true
```

### Configuration du scan automatique

```yaml
# Policy de scan dans Harbor
apiVersion: v1
kind: ConfigMap
metadata:
  name: harbor-scan-policy
  namespace: harbor
data:
  policy.json: |
    {
      "scan_all_on_push": true,
      "auto_scan": true,
      "severity_threshold": "HIGH",
      "prevent_vulnerable_images": true,
      "whitelist_cves": [
        "CVE-2021-12345"
      ]
    }
```

## Gestion des faux positifs

### Fichier .trivyignore

```bash
# .trivyignore dans votre projet
# Ignorer des CVEs spécifiques
CVE-2021-12345
CVE-2022-67890

# Ignorer par package
libssl1.1
openssl

# Ignorer avec expiration
CVE-2023-11111 exp:2024-12-31

# Ignorer avec justification
CVE-2023-22222 # False positive - not applicable to our use case
```

### Annotations Kubernetes pour exceptions

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: legacy-app
  annotations:
    security.io/vulnerability-exceptions: |
      - cve: CVE-2021-12345
        reason: "False positive - using patched version"
        approved_by: "security-team"
        expires: "2024-12-31"
spec:
  template:
    metadata:
      annotations:
        trivy.security.io/skip: "CVE-2021-12345,CVE-2022-67890"
```

## Reporting et visualisation

### Dashboard Grafana pour vulnérabilités

```yaml
# ConfigMap pour dashboard Grafana
apiVersion: v1
kind: ConfigMap
metadata:
  name: vulnerability-dashboard
  namespace: monitoring
data:
  dashboard.json: |
    {
      "dashboard": {
        "title": "Vulnerability Overview",
        "panels": [
          {
            "title": "Critical Vulnerabilities by Namespace",
            "targets": [{
              "expr": "sum by (namespace) (trivy_vulnerability_count{severity=\"CRITICAL\"})"
            }]
          },
          {
            "title": "Top 10 Vulnerable Images",
            "targets": [{
              "expr": "topk(10, sum by (image) (trivy_vulnerability_count))"
            }]
          },
          {
            "title": "Vulnerability Trend",
            "targets": [{
              "expr": "sum(trivy_vulnerability_count) by (severity)"
            }]
          }
        ]
      }
    }
```

### Alertes Prometheus

```yaml
apiVersion: monitoring.coreos.com/v1
kind: PrometheusRule
metadata:
  name: vulnerability-alerts
  namespace: monitoring
spec:
  groups:
  - name: security
    interval: 5m
    rules:
    - alert: CriticalVulnerabilityDetected
      expr: trivy_vulnerability_count{severity="CRITICAL"} > 0
      for: 10m
      annotations:
        summary: "Critical vulnerability in {{ $labels.image }}"
        description: "Image {{ $labels.image }} has {{ $value }} critical vulnerabilities"

    - alert: HighVulnerabilityThreshold
      expr: sum(trivy_vulnerability_count{severity=~"HIGH|CRITICAL"}) by (namespace) > 10
      for: 30m
      annotations:
        summary: "High number of vulnerabilities in {{ $labels.namespace }}"
        description: "Namespace {{ $labels.namespace }} has more than 10 high/critical vulnerabilities"
```

### Export de rapports

```bash
#!/bin/bash
# Script pour générer un rapport hebdomadaire

# Créer le répertoire de rapports
mkdir -p /reports/$(date +%Y-%m-%d)

# Scanner toutes les images et générer un rapport HTML
trivy image --format template \
  --template "@/usr/local/share/trivy/templates/html.tpl" \
  -o /reports/$(date +%Y-%m-%d)/vulnerability-report.html \
  $(kubectl get pods --all-namespaces -o jsonpath="{.items[*].spec.containers[*].image}" | tr -s '[[:space:]]' '\n' | sort | uniq)

# Générer un résumé CSV
echo "Image,Critical,High,Medium,Low" > /reports/$(date +%Y-%m-%d)/summary.csv
for image in $(kubectl get pods --all-namespaces -o jsonpath="{.items[*].spec.containers[*].image}" | tr -s '[[:space:]]' '\n' | sort | uniq); do
  trivy image -f json $image | jq -r '
    .Results[].Vulnerabilities |
    group_by(.Severity) |
    map({(.[0].Severity): length}) |
    add |
    [.CRITICAL // 0, .HIGH // 0, .MEDIUM // 0, .LOW // 0] |
    @csv' >> /reports/$(date +%Y-%m-%d)/summary.csv
done
```

## Stratégies de remédiation

### 1. Mise à jour des images de base

```dockerfile
# AVANT - Image vulnérable
FROM ubuntu:18.04
RUN apt-get update && apt-get install -y nginx

# APRÈS - Image mise à jour
FROM ubuntu:22.04
RUN apt-get update && apt-get install -y nginx

# MIEUX - Image minimale
FROM nginx:alpine
```

### 2. Multi-stage builds pour réduire la surface d'attaque

```dockerfile
# Build stage
FROM node:16 AS builder
WORKDIR /app
COPY package*.json ./
RUN npm ci --only=production
COPY . .
RUN npm run build

# Runtime stage - Image minimale
FROM gcr.io/distroless/nodejs16-debian11
WORKDIR /app
COPY --from=builder /app/dist ./dist
COPY --from=builder /app/node_modules ./node_modules
EXPOSE 3000
CMD ["dist/index.js"]
```

### 3. Mise à jour automatique des dépendances

```yaml
# Renovate bot configuration
apiVersion: v1
kind: ConfigMap
metadata:
  name: renovate-config
data:
  renovate.json: |
    {
      "extends": ["config:base"],
      "packageRules": [
        {
          "matchUpdateTypes": ["patch", "pin", "digest"],
          "automerge": true
        },
        {
          "matchDepTypes": ["devDependencies"],
          "automerge": true
        }
      ],
      "vulnerabilityAlerts": {
        "enabled": true,
        "labels": ["security"]
      }
    }
```

### 4. Politique de patch

```yaml
# Exemple de politique de remediation
apiVersion: security.io/v1
kind: RemediationPolicy
metadata:
  name: vulnerability-patching
spec:
  rules:
  - severity: CRITICAL
    action: block_deployment
    sla_hours: 24
    notification:
      - security-team@company.com
      - ops-team@company.com

  - severity: HIGH
    action: warn
    sla_hours: 168  # 7 jours
    notification:
      - dev-team@company.com

  - severity: MEDIUM
    action: log
    sla_hours: 720  # 30 jours

  - severity: LOW
    action: ignore
    review_cycle: quarterly
```

## Optimisation des images

### Réduire les vulnérabilités par design

```dockerfile
# 1. Utiliser des images distroless
FROM gcr.io/distroless/java:11

# 2. Utiliser des images slim
FROM python:3.9-slim

# 3. Utiliser Alpine quand possible
FROM node:16-alpine

# 4. Supprimer les packages non nécessaires
RUN apt-get update && \
    apt-get install -y --no-install-recommends nginx && \
    apt-get clean && \
    rm -rf /var/lib/apt/lists/*

# 5. Ne pas installer les outils de build en production
FROM golang:1.19 AS builder
# Build...
FROM scratch
COPY --from=builder /app/binary /binary
```

### Scanner les images de base

```bash
#!/bin/bash
# Script pour comparer les vulnérabilités des images de base

BASE_IMAGES=(
  "ubuntu:20.04"
  "ubuntu:22.04"
  "debian:11"
  "debian:12"
  "alpine:3.17"
  "alpine:3.18"
  "gcr.io/distroless/base"
)

echo "Image,Size,Vulnerabilities" > base-images-comparison.csv

for image in "${BASE_IMAGES[@]}"; do
  # Pull image
  docker pull $image

  # Get size
  size=$(docker images $image --format "{{.Size}}")

  # Count vulnerabilities
  vulns=$(trivy image -f json $image | jq '[.Results[].Vulnerabilities[]] | length')

  echo "$image,$size,$vulns" >> base-images-comparison.csv
done

cat base-images-comparison.csv
```

## Intégration avec les outils de sécurité

### SIEM Integration

```yaml
# Fluentd configuration pour envoyer les scans à un SIEM
apiVersion: v1
kind: ConfigMap
metadata:
  name: fluentd-config
  namespace: logging
data:
  fluent.conf: |
    <source>
      @type tail
      path /var/log/trivy/*.json
      pos_file /var/log/trivy.pos
      tag trivy.scan
      <parse>
        @type json
      </parse>
    </source>

    <filter trivy.scan>
      @type record_transformer
      <record>
        environment "#{ENV['CLUSTER_NAME']}"
        severity_score ${record["Results"].map{|r| r["Vulnerabilities"].map{|v| v["Severity"] == "CRITICAL" ? 10 : v["Severity"] == "HIGH" ? 7 : 3}.sum}.sum}
      </record>
    </filter>

    <match trivy.scan>
      @type elasticsearch
      host siem.company.com
      port 9200
      index_name vulnerability-scans
      type_name scan-result
    </match>
```

### Integration avec Falco

```yaml
# Règles Falco pour détecter l'exploitation de vulnérabilités
apiVersion: v1
kind: ConfigMap
metadata:
  name: falco-rules
  namespace: security
data:
  custom-rules.yaml: |
    - rule: Possible CVE-2021-44228 Exploitation
      desc: Detect potential Log4Shell exploitation attempts
      condition: >
        spawned_process and
        (proc.cmdline contains "jndi:ldap" or
         proc.cmdline contains "jndi:rmi" or
         proc.cmdline contains "${jndi:")
      output: >
        Possible Log4Shell exploitation attempt
        (user=%user.name command=%proc.cmdline container=%container.name)
      priority: CRITICAL
      tags: [vulnerability, log4shell]
```

## Compliance et standards

### CIS Benchmark scanning

```bash
# Scanner avec les benchmarks CIS
trivy image --compliance cis-1.5 nginx:latest

# Résultats formatés
trivy image --compliance cis-1.5 --format table nginx:latest
```

### NIST Framework mapping

```yaml
# Mapping des vulnérabilités au framework NIST
apiVersion: v1
kind: ConfigMap
metadata:
  name: nist-mapping
data:
  mapping.yaml: |
    vulnerability_categories:
      - category: "ID.RA - Risk Assessment"
        vulnerabilities:
          - severity: [CRITICAL, HIGH]
            action: immediate_assessment

      - category: "PR.IP - Information Protection"
        vulnerabilities:
          - type: [data_exposure, information_disclosure]
            action: review_data_classification

      - category: "DE.CM - Security Continuous Monitoring"
        tools:
          - trivy
          - falco
          - prometheus
```

## Bonnes pratiques

### 1. Scan à plusieurs niveaux

```yaml
# Pipeline de scan complet
scanning_pipeline:
  1_pre_build:
    - scan: dependencies
    - tool: npm audit, safety, bundler-audit

  2_build_time:
    - scan: dockerfile
    - tool: hadolint, dockerfile-lint

  3_post_build:
    - scan: image
    - tool: trivy, grype, clair

  4_runtime:
    - scan: running_containers
    - tool: falco, tracee

  5_continuous:
    - scan: deployed_workloads
    - tool: trivy-operator
```

### 2. Politique de sécurité claire

```yaml
# security-policy.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: security-policy
  namespace: security
data:
  policy.yaml: |
    scanning_requirements:
      - all_images_must_be_scanned: true
      - block_on_critical: true
      - max_high_vulnerabilities: 5
      - max_medium_vulnerabilities: 20
      - scan_frequency: daily
      - scan_on_admission: true

    exceptions:
      require_approval: true
      max_duration_days: 30
      require_justification: true
      approvers:
        - security-team
        - platform-team
```

### 3. Automatisation et intégration

```bash
#!/bin/bash
# Script d'automatisation du scan

# Configuration
NAMESPACE=${1:-default}
SEVERITY_THRESHOLD="HIGH,CRITICAL"
SLACK_WEBHOOK="https://hooks.slack.com/services/XXX"

# Fonction de notification
notify_slack() {
    local message=$1
    curl -X POST -H 'Content-type: application/json' \
        --data "{\"text\":\"${message}\"}" \
        $SLACK_WEBHOOK
}

# Scanner toutes les images du namespace
for pod in $(kubectl get pods -n $NAMESPACE -o name); do
    images=$(kubectl get $pod -n $NAMESPACE -o jsonpath='{.spec.containers[*].image}')

    for image in $images; do
        echo "Scanning $image..."

        # Scan avec Trivy
        if trivy image --severity $SEVERITY_THRESHOLD --exit-code 1 $image > /dev/null 2>&1; then
            echo "✅ $image - No critical vulnerabilities"
        else
            echo "❌ $image - Vulnerabilities found!"
            notify_slack "⚠️ Vulnerabilities found in $image (namespace: $NAMESPACE)"

            # Générer rapport détaillé
            trivy image --severity $SEVERITY_THRESHOLD -f json $image > "/tmp/scan-$NAMESPACE-$(date +%s).json"
        fi
    done
done
```

## Métriques et KPIs

### Indicateurs clés

```yaml
# Prometheus metrics pour le suivi
vulnerability_metrics:
  - metric: vulnerability_count_by_severity
    query: sum(trivy_vulnerability_count) by (severity)

  - metric: images_with_critical_vulns
    query: count(trivy_vulnerability_count{severity="CRITICAL"} > 0)

  - metric: mean_time_to_remediation
    query: avg(vulnerability_patch_time_seconds)

  - metric: scan_coverage_percentage
    query: (count(images_scanned) / count(images_total)) * 100

  - metric: false_positive_rate
    query: (count(vulnerabilities_ignored) / count(vulnerabilities_total)) * 100
```

### Dashboard de suivi

```yaml
# Grafana dashboard configuration
apiVersion: v1
kind: ConfigMap
metadata:
  name: vulnerability-kpi-dashboard
data:
  dashboard.json: |
    {
      "panels": [
        {
          "title": "Vulnerability Trend (30 days)",
          "type": "graph",
          "targets": [{
            "expr": "sum(rate(trivy_vulnerability_count[30d])) by (severity)"
          }]
        },
        {
          "title": "Top Vulnerable Namespaces",
          "type": "table",
          "targets": [{
            "expr": "topk(5, sum(trivy_vulnerability_count) by (namespace))"
          }]
        },
        {
          "title": "Remediation SLA Compliance",
          "type": "stat",
          "targets": [{
            "expr": "(count(vulnerabilities_patched_within_sla) / count(vulnerabilities_total)) * 100"
          }]
        }
      ]
    }
```

## Résumé et points clés

Le scan de vulnérabilités est votre système d'alarme incendie pour les conteneurs :

1. **Scannez tôt et souvent** : Build time, admission time, et runtime
2. **Automatisez tout** : Intégrez dans CI/CD et admission controllers
3. **Priorisez par sévérité** : Focus sur CRITICAL et HIGH d'abord
4. **Gérez les faux positifs** : Documentez et justifiez les exceptions
5. **Réduisez la surface d'attaque** : Images minimales, distroless quand possible
6. **Mesurez et améliorez** : KPIs pour suivre les progrès
7. **Intégrez avec l'écosystème** : SIEM, alerting, compliance tools

Le scan de vulnérabilités n'est pas une option mais une nécessité. Commencez simple avec Trivy en ligne de commande, puis progressez vers une intégration complète dans votre pipeline.

---

*Prochain sujet : 11.5 Secrets management avancé - Gérer les données sensibles de manière sécurisée*

⏭️
