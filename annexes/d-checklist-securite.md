🔝 Retour au [Sommaire](/SOMMAIRE.md)

# Annexe D - Checklist de sécurité

## Introduction à la Sécurité Kubernetes

La sécurité dans Kubernetes n'est pas une option, c'est une nécessité. Même pour un lab personnel, adopter de bonnes pratiques de sécurité dès le début vous préparera pour des environnements de production et protégera votre infrastructure domestique.

### Pourquoi Sécuriser un Lab Personnel ?

Même un lab personnel peut être :
- **Exposé accidentellement** à Internet
- **Utilisé comme point d'entrée** vers votre réseau domestique
- **Compromis par des images Docker** malveillantes
- **Exploité pour du minage** de cryptomonnaie
- **Source d'apprentissage** pour de bonnes pratiques

### Les 4 Piliers de la Sécurité Kubernetes

1. **Sécurité de l'Infrastructure** : Le cluster et les nodes
2. **Sécurité des Workloads** : Les applications et conteneurs
3. **Sécurité du Réseau** : Communications et exposition
4. **Sécurité des Données** : Secrets et stockage

## Niveau 1 : Sécurité de Base (Essentiel)

### ✅ 1.1 Sécurisation de l'Accès au Cluster

#### Configuration de kubectl sécurisée

```bash
# Vérifier les permissions du fichier kubeconfig
ls -la ~/.kube/config
# Doit être : -rw------- (600)

# Si incorrect, corriger :
chmod 600 ~/.kube/config

# Ne jamais committer le kubeconfig dans Git
echo ".kube/config" >> .gitignore
echo "*.kubeconfig" >> .gitignore
```

#### Désactiver l'accès anonyme

```bash
# Vérifier l'accès anonyme
kubectl get --raw /api/v1/namespaces/default/pods --as=system:anonymous

# Si accessible, le désactiver dans MicroK8s
microk8s kubectl -n kube-system edit configmap/kubeapi-config
# Ajouter : --anonymous-auth=false
```

#### Limiter l'accès à l'API

```bash
# Configuration firewall pour l'API (port 16443 pour MicroK8s)
sudo ufw allow from 192.168.1.0/24 to any port 16443
sudo ufw deny 16443

# Vérifier les règles
sudo ufw status verbose
```

### ✅ 1.2 Mots de Passe et Secrets

#### Ne jamais mettre de secrets en clair

```yaml
# ❌ MAUVAIS - Secret en clair dans le YAML
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config
data:
  database_password: "SuperSecret123"  # JAMAIS !

# ✅ BON - Utiliser un Secret
apiVersion: v1
kind: Secret
metadata:
  name: app-secret
type: Opaque
stringData:
  database_password: "SuperSecret123"  # Sera encodé automatiquement
```

#### Création sécurisée de secrets

```bash
# Créer un secret depuis la ligne de commande
kubectl create secret generic db-password \
  --from-literal=password='ComplexP@ssw0rd!' \
  --dry-run=client -o yaml | kubectl apply -f -

# Créer depuis un fichier (qui ne sera pas commité)
echo -n 'ComplexP@ssw0rd!' > ./password.txt
kubectl create secret generic db-password --from-file=./password.txt
rm ./password.txt  # Supprimer immédiatement

# Générer un mot de passe aléatoire
kubectl create secret generic random-password \
  --from-literal=password=$(openssl rand -base64 32)
```

### ✅ 1.3 Images de Conteneurs

#### Utiliser des images officielles et à jour

```yaml
# deployment-secure.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: secure-app
spec:
  template:
    spec:
      containers:
      - name: app
        image: nginx:1.21-alpine  # ✅ Version spécifique, image officielle
        # image: nginx:latest     # ❌ Éviter 'latest'
        # image: random/nginx     # ❌ Éviter les images non officielles
        imagePullPolicy: Always   # Toujours vérifier les mises à jour
```

#### Scanner les images

```bash
# Installer Trivy pour scanner les vulnérabilités
wget https://github.com/aquasecurity/trivy/releases/download/v0.48.0/trivy_0.48.0_Linux-64bit.tar.gz
tar zxvf trivy_0.48.0_Linux-64bit.tar.gz
sudo mv trivy /usr/local/bin/

# Scanner une image
trivy image nginx:alpine

# Scanner une image locale
docker save nginx:alpine | trivy image --input -
```

### ✅ 1.4 Mises à Jour

#### Maintenir MicroK8s à jour

```bash
# Vérifier la version actuelle
microk8s version

# Voir les canaux disponibles
snap info microk8s

# Mettre à jour vers la dernière version stable
sudo snap refresh microk8s --channel=1.29/stable

# Activer les mises à jour automatiques
sudo snap set microk8s refresh.timer=daily
```

#### Script de vérification des mises à jour

```bash
#!/bin/bash
# check-updates.sh

echo "=== Vérification des mises à jour de sécurité ==="

# MicroK8s
echo -n "MicroK8s: "
snap list microk8s

# Images des pods
echo -e "\nImages utilisées:"
kubectl get pods --all-namespaces -o jsonpath="{.items[*].spec.containers[*].image}" | \
  tr -s '[[:space:]]' '\n' | sort | uniq

# Suggestion de scan
echo -e "\nPour scanner les vulnérabilités:"
echo "trivy image <nom-image>"
```

## Niveau 2 : Sécurité Intermédiaire (Recommandé)

### ✅ 2.1 RBAC (Role-Based Access Control)

#### Comprendre RBAC

RBAC contrôle qui peut faire quoi dans votre cluster :
- **ServiceAccount** : Identité pour les pods
- **Role/ClusterRole** : Définit les permissions
- **RoleBinding/ClusterRoleBinding** : Attribue les permissions

#### Créer un utilisateur avec permissions limitées

```yaml
# rbac-limited-user.yaml
---
# ServiceAccount pour l'application
apiVersion: v1
kind: ServiceAccount
metadata:
  name: app-reader
  namespace: default
---
# Role avec permissions lecture seule
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: pod-reader
  namespace: default
rules:
- apiGroups: [""]
  resources: ["pods", "services"]
  verbs: ["get", "watch", "list"]
---
# RoleBinding pour lier le ServiceAccount au Role
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: read-pods
  namespace: default
subjects:
- kind: ServiceAccount
  name: app-reader
  namespace: default
roleRef:
  kind: Role
  name: pod-reader
  apiGroup: rbac.authorization.k8s.io
```

#### Utiliser le ServiceAccount dans un pod

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: secure-pod
spec:
  serviceAccountName: app-reader  # Utilise le ServiceAccount limité
  containers:
  - name: app
    image: nginx:alpine
```

#### Auditer les permissions

```bash
# Vérifier ce qu'un utilisateur peut faire
kubectl auth can-i --list --as=system:serviceaccount:default:app-reader

# Vérifier une action spécifique
kubectl auth can-i delete pods --as=system:serviceaccount:default:app-reader
# Réponse : no

# Voir tous les RoleBindings
kubectl get rolebindings --all-namespaces
kubectl get clusterrolebindings
```

### ✅ 2.2 Network Policies

#### Isolation réseau par défaut

```yaml
# network-policy-deny-all.yaml
# Refuse tout le trafic par défaut
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny-all
  namespace: production
spec:
  podSelector: {}  # S'applique à tous les pods
  policyTypes:
  - Ingress
  - Egress
```

#### Autoriser seulement le trafic nécessaire

```yaml
# network-policy-web-app.yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: web-app-policy
  namespace: production
spec:
  podSelector:
    matchLabels:
      app: web
  policyTypes:
  - Ingress
  - Egress
  ingress:
  # Autoriser depuis l'ingress controller
  - from:
    - namespaceSelector:
        matchLabels:
          name: ingress
    ports:
    - protocol: TCP
      port: 8080
  egress:
  # Autoriser vers la base de données
  - to:
    - podSelector:
        matchLabels:
          app: database
    ports:
    - protocol: TCP
      port: 5432
  # Autoriser DNS
  - to:
    - namespaceSelector:
        matchLabels:
          name: kube-system
    - podSelector:
        matchLabels:
          k8s-app: kube-dns
    ports:
    - protocol: UDP
      port: 53
```

#### Tester les Network Policies

```bash
# Créer des pods de test
kubectl run test-pod --image=nicolaka/netshoot -it --rm -- bash

# Dans le pod, tester la connectivité
nc -zv service-name 80
ping another-pod
curl http://service-name
```

### ✅ 2.3 Security Contexts

#### Configurer les Security Contexts

```yaml
# pod-security-context.yaml
apiVersion: v1
kind: Pod
metadata:
  name: secure-pod
spec:
  securityContext:
    runAsNonRoot: true        # Ne pas exécuter en root
    runAsUser: 1000          # UID spécifique
    runAsGroup: 3000         # GID spécifique
    fsGroup: 2000            # GID pour les volumes
    seccompProfile:
      type: RuntimeDefault    # Profil Seccomp
  containers:
  - name: app
    image: nginx:alpine
    securityContext:
      allowPrivilegeEscalation: false  # Pas d'escalade de privilèges
      readOnlyRootFilesystem: true     # Système de fichiers en lecture seule
      capabilities:
        drop:
        - ALL                           # Supprimer toutes les capabilities
        add:
        - NET_BIND_SERVICE             # Ajouter seulement celle nécessaire
    volumeMounts:
    - name: cache
      mountPath: /var/cache/nginx
    - name: run
      mountPath: /var/run
  volumes:
  - name: cache
    emptyDir: {}
  - name: run
    emptyDir: {}
```

### ✅ 2.4 Admission Controllers

#### Pod Security Standards

```bash
# Activer Pod Security Standards pour un namespace
kubectl label namespace production \
  pod-security.kubernetes.io/enforce=restricted \
  pod-security.kubernetes.io/audit=restricted \
  pod-security.kubernetes.io/warn=restricted

# Niveaux disponibles :
# - privileged : Aucune restriction
# - baseline : Restrictions minimales
# - restricted : Restrictions maximales (recommandé)
```

#### Exemple de pod conforme

```yaml
# pod-restricted.yaml
apiVersion: v1
kind: Pod
metadata:
  name: restricted-pod
  namespace: production
spec:
  securityContext:
    runAsNonRoot: true
    runAsUser: 1000
    seccompProfile:
      type: RuntimeDefault
  containers:
  - name: app
    image: nginx:alpine
    securityContext:
      allowPrivilegeEscalation: false
      readOnlyRootFilesystem: true
      capabilities:
        drop:
        - ALL
```

## Niveau 3 : Sécurité Avancée (Production)

### ✅ 3.1 Chiffrement

#### Chiffrement des secrets au repos

```bash
# Créer une clé de chiffrement
echo -n "$(head -c 32 /dev/urandom | base64)" > encryption-key.txt

# Configuration du chiffrement
cat <<EOF > encryption-config.yaml
apiVersion: apiserver.config.k8s.io/v1
kind: EncryptionConfiguration
resources:
  - resources:
    - secrets
    providers:
    - aescbc:
        keys:
        - name: key1
          secret: $(cat encryption-key.txt)
    - identity: {}
EOF

# Appliquer (nécessite redémarrage de l'API server)
# Pour MicroK8s, éditer /var/snap/microk8s/current/args/kube-apiserver
# Ajouter : --encryption-provider-config=/path/to/encryption-config.yaml
```

#### TLS pour tous les services

```yaml
# ingress-tls-mandatory.yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: secure-ingress
  annotations:
    nginx.ingress.kubernetes.io/ssl-redirect: "true"
    nginx.ingress.kubernetes.io/force-ssl-redirect: "true"
    cert-manager.io/cluster-issuer: "letsencrypt-prod"
spec:
  tls:
  - hosts:
    - app.example.com
    secretName: app-tls
  rules:
  - host: app.example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: app-service
            port:
              number: 80
```

### ✅ 3.2 Audit et Logging

#### Activer l'audit logging

```bash
# Configuration de l'audit
cat <<EOF > audit-policy.yaml
apiVersion: audit.k8s.io/v1
kind: Policy
rules:
  # Log tous les accès aux secrets
  - level: RequestResponse
    resources:
    - group: ""
      resources: ["secrets"]
  # Log les créations/suppressions
  - level: Request
    verbs: ["create", "delete", "deletecollection"]
  # Log les échecs d'authentification
  - level: Metadata
    omitStages:
    - RequestReceived
EOF

# Activer dans MicroK8s
# Éditer /var/snap/microk8s/current/args/kube-apiserver
# Ajouter :
# --audit-policy-file=/path/to/audit-policy.yaml
# --audit-log-path=/var/log/kubernetes/audit.log
# --audit-log-maxage=30
# --audit-log-maxbackup=10
# --audit-log-maxsize=100
```

#### Centraliser les logs

```yaml
# fluentd-config.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: fluentd-config
  namespace: kube-system
data:
  fluent.conf: |
    <source>
      @type tail
      path /var/log/containers/*.log
      tag kubernetes.*
      format json
    </source>

    <filter kubernetes.**>
      @type kubernetes_metadata
    </filter>

    <match **>
      @type elasticsearch
      host elasticsearch.logging
      port 9200
      logstash_format true
    </match>
```

### ✅ 3.3 Supply Chain Security

#### Signing d'images avec Cosign

```bash
# Installer Cosign
wget https://github.com/sigstore/cosign/releases/download/v2.0.0/cosign-linux-amd64
sudo mv cosign-linux-amd64 /usr/local/bin/cosign
sudo chmod +x /usr/local/bin/cosign

# Générer une paire de clés
cosign generate-key-pair

# Signer une image
cosign sign --key cosign.key docker.io/myuser/myimage:v1

# Vérifier une signature
cosign verify --key cosign.pub docker.io/myuser/myimage:v1
```

#### Policy as Code avec OPA

```yaml
# opa-policy.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: opa-policies
  namespace: opa
data:
  image-whitelist.rego: |
    package kubernetes.admission

    deny[msg] {
      input.request.kind.kind == "Pod"
      image := input.request.object.spec.containers[_].image
      not starts_with(image, "registry.company.com/")
      msg := sprintf("Image non autorisée: %v", [image])
    }

    deny[msg] {
      input.request.kind.kind == "Pod"
      container := input.request.object.spec.containers[_]
      container.securityContext.runAsRoot == true
      msg := "Les conteneurs ne doivent pas s'exécuter en root"
    }
```

### ✅ 3.4 Scanning de Vulnérabilités

#### Scan automatique avec Trivy Operator

```bash
# Installer Trivy Operator
kubectl apply -f https://raw.githubusercontent.com/aquasecurity/trivy-operator/main/deploy/static/trivy-operator.yaml

# Vérifier les rapports de vulnérabilités
kubectl get vulnerabilityreports --all-namespaces

# Voir les détails d'un rapport
kubectl describe vulnerabilityreport -n default
```

#### Script de scan régulier

```bash
#!/bin/bash
# security-scan.sh

echo "=== Scan de sécurité du cluster ==="

# Scanner toutes les images
for image in $(kubectl get pods --all-namespaces -o jsonpath="{.items[*].spec.containers[*].image}" | tr -s '[[:space:]]' '\n' | sort | uniq); do
    echo "Scanning: $image"
    trivy image --severity HIGH,CRITICAL $image
done

# Vérifier les PSP
echo -e "\n=== Pod Security Policies ==="
kubectl get psp

# Vérifier les ServiceAccounts avec trop de permissions
echo -e "\n=== ServiceAccounts avec cluster-admin ==="
kubectl get clusterrolebindings -o json | jq -r '.items[] | select(.roleRef.name=="cluster-admin") | .subjects[]?.name'

# Vérifier les secrets exposés
echo -e "\n=== Secrets potentiellement exposés ==="
kubectl get pods --all-namespaces -o jsonpath='{range .items[*]}{.metadata.namespace}{"\t"}{.metadata.name}{"\t"}{.spec.containers[*].env[*]}{"\n"}{end}' | grep -i password
```

## Checklist de Sécurité Complète

### 🔒 Infrastructure et Accès

- [ ] **Kubeconfig sécurisé** (permissions 600)
- [ ] **Pas de kubeconfig dans Git**
- [ ] **Firewall configuré** pour l'API
- [ ] **Accès anonyme désactivé**
- [ ] **MicroK8s à jour**
- [ ] **Système d'exploitation à jour**
- [ ] **SSH sécurisé** (clés uniquement, pas de root)
- [ ] **Audit logging activé**

### 🔐 Authentification et Autorisation

- [ ] **RBAC activé et configuré**
- [ ] **ServiceAccounts avec permissions minimales**
- [ ] **Pas de cluster-admin** pour les applications
- [ ] **Tokens avec expiration**
- [ ] **Rotation régulière des credentials**

### 🛡️ Workloads et Conteneurs

- [ ] **Images officielles utilisées**
- [ ] **Versions spécifiques** (pas de :latest)
- [ ] **Images scannées** pour vulnérabilités
- [ ] **Security Context configuré**
- [ ] **Pas d'exécution en root**
- [ ] **ReadOnlyRootFilesystem** quand possible
- [ ] **Capabilities minimales**
- [ ] **Resource limits définies**

### 🔑 Secrets et Configuration

- [ ] **Secrets Kubernetes utilisés** (pas de ConfigMaps)
- [ ] **Secrets chiffrés au repos**
- [ ] **Rotation des secrets**
- [ ] **Pas de secrets dans les logs**
- [ ] **Pas de secrets dans les variables d'environnement** (préférer les volumes)
- [ ] **Secrets avec permissions restrictives**

### 🌐 Réseau

- [ ] **Network Policies en place**
- [ ] **Deny all par défaut**
- [ ] **TLS/HTTPS obligatoire**
- [ ] **Certificats valides** (Let's Encrypt)
- [ ] **Ingress sécurisé**
- [ ] **Pas de NodePort** en production
- [ ] **Service mesh** (Istio/Linkerd) pour zero-trust

### 📊 Monitoring et Compliance

- [ ] **Logs centralisés**
- [ ] **Alertes configurées**
- [ ] **Métriques de sécurité**
- [ ] **Scan régulier des vulnérabilités**
- [ ] **Backup régulier**
- [ ] **Plan de disaster recovery**
- [ ] **Tests de pénétration**

## Scripts d'Audit de Sécurité

### Script d'Audit Complet

```bash
#!/bin/bash
# security-audit.sh

RED='\033[0;31m'
GREEN='\033[0;32m'
YELLOW='\033[1;33m'
NC='\033[0m'

echo "======================================"
echo "   Audit de Sécurité Kubernetes"
echo "======================================"

SCORE=0
TOTAL=0

check() {
    local test_name=$1
    local test_command=$2
    local expected=$3

    TOTAL=$((TOTAL + 1))

    result=$(eval $test_command 2>/dev/null)

    if [[ "$result" == *"$expected"* ]]; then
        echo -e "${GREEN}✓${NC} $test_name"
        SCORE=$((SCORE + 1))
    else
        echo -e "${RED}✗${NC} $test_name"
    fi
}

echo -e "\n=== Accès et Authentification ==="

check "Accès anonyme désactivé" \
    "kubectl get --raw /api/v1 --as=system:anonymous 2>&1" \
    "Forbidden"

check "RBAC activé" \
    "kubectl api-versions | grep rbac.authorization.k8s.io" \
    "rbac.authorization.k8s.io"

check "Permissions kubeconfig" \
    "stat -c %a ~/.kube/config" \
    "600"

echo -e "\n=== Workloads ==="

check "Pas de pods en root" \
    "kubectl get pods --all-namespaces -o jsonpath='{.items[*].spec.securityContext.runAsUser}' | grep -c '^0$'" \
    "0"

check "Pas de conteneurs privileged" \
    "kubectl get pods --all-namespaces -o jsonpath='{.items[*].spec.containers[*].securityContext.privileged}' | grep -c true" \
    "0"

echo -e "\n=== Secrets ==="

check "Pas de secrets dans les env vars" \
    "kubectl get pods --all-namespaces -o jsonpath='{.items[*].spec.containers[*].env[*].value}' | grep -i password | wc -l" \
    "0"

echo -e "\n=== Réseau ==="

check "Network Policies présentes" \
    "kubectl get networkpolicies --all-namespaces | wc -l" \
    "0"

check "Ingress avec TLS" \
    "kubectl get ingress --all-namespaces -o jsonpath='{.items[*].spec.tls}' | grep -c secretName" \
    "0"

echo -e "\n=== Images ==="

# Liste des images à risque
RISKY_IMAGES=("latest" "alpine:edge" "ubuntu:latest")
for image in "${RISKY_IMAGES[@]}"; do
    check "Pas d'image $image" \
        "kubectl get pods --all-namespaces -o jsonpath=\"{.items[*].spec.containers[*].image}\" | grep -c $image" \
        "0"
done

echo -e "\n======================================"
echo -e "Score: $SCORE/$TOTAL"

if [ $SCORE -eq $TOTAL ]; then
    echo -e "${GREEN}Excellent ! Toutes les vérifications passent.${NC}"
elif [ $SCORE -ge $((TOTAL * 75 / 100)) ]; then
    echo -e "${YELLOW}Bon niveau de sécurité, mais des améliorations sont possibles.${NC}"
else
    echo -e "${RED}Attention ! Plusieurs problèmes de sécurité détectés.${NC}"
fi
```

### Script de Hardening Automatique

```bash
#!/bin/bash
# auto-hardening.sh

echo "=== Hardening automatique du cluster ==="

# 1. Appliquer les Network Policies par défaut
echo "Application des Network Policies..."
for ns in $(kubectl get namespaces -o jsonpath='{.items[*].metadata.name}'); do
    if [[ "$ns" != "kube-system" && "$ns" != "kube-public" && "$ns" != "kube-node-lease" ]]; then
        cat <<EOF | kubectl apply -f -
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny-all
  namespace: $ns
spec:
  podSelector: {}
  policyTypes:
  - Ingress
  - Egress
EOF
    fi
done

# 2. Appliquer les Pod Security Standards
echo "Application des Pod Security Standards..."
for ns in $(kubectl get namespaces -o jsonpath='{.items[*].metadata.name}'); do
    if [[ "$ns" != "kube-system" && "$ns" != "kube-public" ]]; then
        kubectl label namespace $ns \
            pod-security.kubernetes.io/enforce=baseline \
            pod-security.kubernetes.io/warn=restricted \
            --overwrite
    fi
done

# 3. Créer des ResourceQuotas
echo "Création des ResourceQuotas..."
for ns in $(kubectl get namespaces -o jsonpath='{.items[*].metadata.name}'); do
    if [[ "$ns" != "kube-system" && "$ns" != "kube-public" && "$ns" != "kube-node-lease" ]]; then
        cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: ResourceQuota
metadata:
  name: default-quota
  namespace: $ns
spec:
  hard:
    requests.cpu: "4"
    requests.memory: 8Gi
    limits.cpu: "8"
    limits.memory: 16Gi
    persistentvolumeclaims: "5"
EOF
    fi
done

# 4. Créer des LimitRanges
echo "Création des LimitRanges..."
for ns in $(kubectl get namespaces -o jsonpath='{.items[*].metadata.name}'); do
    if [[ "$ns" != "kube-system" && "$ns" != "kube-public" && "$ns" != "kube-node-lease" ]]; then
        cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: LimitRange
metadata:
  name: default-limits
  namespace: $ns
spec:
  limits:
  - default:
      memory: 512Mi
      cpu: 500m
    defaultRequest:
      memory: 256Mi
      cpu: 100m
    type: Container
EOF
    fi
done

echo "✓ Hardening terminé !"
```

## Guide de Réponse aux Incidents

### En Cas de Compromission Suspectée

```bash
#!/bin/bash
# incident-response.sh

echo "=== Réponse à incident de sécurité ==="

# 1. Isoler les pods suspects
NAMESPACE=$1
POD=$2

if [ -z "$NAMESPACE" ] || [ -z "$POD" ]; then
    echo "Usage: ./incident-response.sh <namespace> <pod>"
    exit 1
fi

echo "Isolation du pod $POD dans $NAMESPACE..."

# Créer une NetworkPolicy d'isolation
cat <<EOF | kubectl apply -f -
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: isolate-$POD
  namespace: $NAMESPACE
spec:
  podSelector:
    matchLabels:
      name: $POD
  policyTypes:
  - Ingress
  - Egress
EOF

# 2. Capturer les logs
echo "Capture des logs..."
kubectl logs $POD -n $NAMESPACE --all-containers > incident-$POD-$(date +%Y%m%d-%H%M%S).log

# 3. Capturer l'état du pod
echo "Capture de l'état..."
kubectl describe pod $POD -n $NAMESPACE > incident-$POD-describe-$(date +%Y%m%d-%H%M%S).txt
kubectl get pod $POD -n $NAMESPACE -o yaml > incident-$POD-yaml-$(date +%Y%m%d-%H%M%S).yaml

# 4. Capturer une image forensique
echo "Création d'une image forensique..."
kubectl exec $POD -n $NAMESPACE -- tar czf /tmp/forensic.tar.gz / 2>/dev/null || true
kubectl cp $NAMESPACE/$POD:/tmp/forensic.tar.gz ./forensic-$POD-$(date +%Y%m%d-%H%M%S).tar.gz

# 5. Suspendre le pod (ne pas supprimer pour l'investigation)
echo "Suspension du pod..."
kubectl scale deployment ${POD%-*} --replicas=0 -n $NAMESPACE 2>/dev/null || \
kubectl delete pod $POD -n $NAMESPACE --grace-period=999999

echo "✓ Pod isolé et preuves collectées"
echo "Fichiers créés :"
ls -la incident-$POD-* forensic-$POD-*
```

## Best Practices de Sécurité

### 1. Principe du Moindre Privilège

- **Toujours commencer avec zéro permission** et ajouter au besoin
- **Un ServiceAccount par application**
- **Des namespaces séparés** pour isoler les workloads
- **RBAC granulaire** plutôt que cluster-admin

### 2. Défense en Profondeur

- **Plusieurs couches de sécurité** (réseau, RBAC, policies)
- **Pas de single point of failure**
- **Segmentation** des environnements (dev/staging/prod)
- **Monitoring** à chaque niveau

### 3. Sécurité par Design

- **Security as Code** : Tout en YAML versionné
- **Shift Left** : Tests de sécurité dès le développement
- **Immutable Infrastructure** : Pas de modifications en production
- **GitOps** : Déploiements tracés et auditables

### 4. Gestion des Secrets

```bash
# Bonnes pratiques pour les secrets

# 1. Rotation régulière
cat <<'EOF' > rotate-secrets.sh
#!/bin/bash
# Rotation mensuelle des secrets
OLD_SECRET=$(kubectl get secret app-secret -o jsonpath='{.data.password}' | base64 -d)
NEW_SECRET=$(openssl rand -base64 32)

# Créer nouveau secret
kubectl create secret generic app-secret-new \
  --from-literal=password="$NEW_SECRET" \
  --dry-run=client -o yaml | kubectl apply -f -

# Mettre à jour les deployments
kubectl set env deployment/app PASSWORD_SECRET=app-secret-new

# Supprimer ancien secret après validation
echo "Ancien secret à supprimer après validation : app-secret"
EOF

# 2. Utiliser un gestionnaire de secrets externe
# Exemple avec Sealed Secrets
kubectl apply -f https://github.com/bitnami-labs/sealed-secrets/releases/download/v0.24.0/controller.yaml

# Créer un secret scellé
echo -n "mypassword" | kubectl create secret generic mysecret \
  --dry-run=client --from-file=password=/dev/stdin -o yaml | \
  kubeseal -o yaml > sealed-secret.yaml

# Le sealed-secret.yaml peut être commité dans Git
```

## Monitoring de Sécurité

### Dashboard de Sécurité

```yaml
# security-dashboard.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: security-dashboard
  namespace: monitoring
data:
  dashboard.json: |
    {
      "dashboard": {
        "title": "Security Dashboard",
        "panels": [
          {
            "title": "Failed Auth Attempts",
            "query": "sum(rate(apiserver_authentication_attempts{result=\"error\"}[5m]))"
          },
          {
            "title": "Privileged Containers",
            "query": "count(kube_pod_container_status_running{pod=~\".*\"} * on(pod) group_left kube_pod_container_info{privileged=\"true\"})"
          },
          {
            "title": "Pods Running as Root",
            "query": "count(kube_pod_container_info{run_as_user=\"0\"})"
          },
          {
            "title": "Network Policy Coverage",
            "query": "count(kube_networkpolicy_info) by (namespace)"
          }
        ]
      }
    }
```

### Alertes de Sécurité

```yaml
# security-alerts.yaml
apiVersion: monitoring.coreos.com/v1
kind: PrometheusRule
metadata:
  name: security-alerts
  namespace: monitoring
spec:
  groups:
  - name: security
    interval: 30s
    rules:
    # Alerte : Pod privileged créé
    - alert: PrivilegedPodCreated
      expr: kube_pod_container_info{privileged="true"} > 0
      for: 1m
      labels:
        severity: warning
      annotations:
        summary: "Pod privileged détecté"
        description: "Pod {{ $labels.pod }} dans {{ $labels.namespace }} s'exécute en mode privileged"

    # Alerte : Trop d'échecs d'authentification
    - alert: HighAuthFailureRate
      expr: rate(apiserver_authentication_attempts{result="error"}[5m]) > 0.1
      for: 5m
      labels:
        severity: critical
      annotations:
        summary: "Taux élevé d'échecs d'authentification"
        description: "{{ $value }} échecs/sec détectés"

    # Alerte : Secret accédé fréquemment
    - alert: SecretAccessSpike
      expr: rate(apiserver_audit_event_total{verb="get",objectRef_resource="secrets"}[5m]) > 1
      for: 2m
      labels:
        severity: warning
      annotations:
        summary: "Accès anormal aux secrets"
        description: "{{ $value }} accès/sec aux secrets"

    # Alerte : Image non scannée
    - alert: UnscannedImage
      expr: |
        kube_pod_container_info{image!~".*sha256.*"}
        unless on(image)
        trivy_image_vulnerabilities_info
      for: 10m
      labels:
        severity: info
      annotations:
        summary: "Image non scannée"
        description: "Image {{ $labels.image }} n'a pas été scannée"
```

### Script de Monitoring Continu

```bash
#!/bin/bash
# security-monitor.sh

# Monitoring de sécurité en temps réel

watch_security() {
    while true; do
        clear
        echo "=== Security Monitor - $(date) ==="

        echo -e "\n📊 Pods à Risque:"
        kubectl get pods --all-namespaces -o json | jq -r '
            .items[] |
            select(.spec.securityContext.runAsUser == 0 or
                   .spec.containers[].securityContext.privileged == true or
                   .spec.hostNetwork == true) |
            "\(.metadata.namespace)/\(.metadata.name)"
        '

        echo -e "\n🔓 Services Exposés:"
        kubectl get services --all-namespaces -o json | jq -r '
            .items[] |
            select(.spec.type == "NodePort" or .spec.type == "LoadBalancer") |
            "\(.metadata.namespace)/\(.metadata.name) - \(.spec.type)"
        '

        echo -e "\n🔑 Secrets Récemment Modifiés:"
        kubectl get secrets --all-namespaces \
            --sort-by=.metadata.creationTimestamp \
            -o jsonpath='{range .items[*]}{.metadata.namespace}{"\t"}{.metadata.name}{"\t"}{.metadata.creationTimestamp}{"\n"}{end}' | \
            tail -5

        echo -e "\n⚠️  Events de Sécurité:"
        kubectl get events --all-namespaces \
            --field-selector type=Warning \
            --sort-by='.lastTimestamp' | \
            grep -E "(Failed|Invalid|Unauthorized|Forbidden)" | \
            tail -5

        sleep 10
    done
}

watch_security
```

## Conformité et Standards

### CIS Kubernetes Benchmark

```bash
#!/bin/bash
# cis-benchmark-check.sh

echo "=== CIS Kubernetes Benchmark Check ==="

# Installer kube-bench
wget https://github.com/aquasecurity/kube-bench/releases/download/v0.7.0/kube-bench_0.7.0_linux_amd64.tar.gz
tar -xvf kube-bench_0.7.0_linux_amd64.tar.gz

# Exécuter le benchmark
./kube-bench run --targets master,node,etcd,policies

# Générer un rapport
./kube-bench run --json > cis-report-$(date +%Y%m%d).json

# Résumé des résultats
echo -e "\n=== Résumé ==="
./kube-bench run | grep -E "PASS|FAIL|WARN|INFO" | tail -20
```

### Checklist OWASP pour Kubernetes

```markdown
## OWASP Kubernetes Security Top 10

### ✅ K01: Insecure Workload Configurations
- [ ] Security contexts configurés
- [ ] Pas de conteneurs privileged
- [ ] Capabilities minimales

### ✅ K02: Supply Chain Vulnerabilities
- [ ] Images scannées
- [ ] Registre privé sécurisé
- [ ] Signatures d'images

### ✅ K03: Overly Permissive RBAC
- [ ] Principe du moindre privilège
- [ ] Pas de wildcard dans les roles
- [ ] Audit régulier des permissions

### ✅ K04: Lack of Centralized Policy Enforcement
- [ ] OPA ou Kyverno déployé
- [ ] Pod Security Standards activés
- [ ] Network Policies en place

### ✅ K05: Inadequate Logging and Monitoring
- [ ] Audit logging activé
- [ ] Logs centralisés
- [ ] Alertes configurées

### ✅ K06: Broken Authentication
- [ ] Pas d'accès anonyme
- [ ] MFA pour l'accès admin
- [ ] Rotation des tokens

### ✅ K07: Missing Network Segmentation
- [ ] Network Policies par défaut
- [ ] Isolation des namespaces
- [ ] Service mesh pour zero-trust

### ✅ K08: Secrets Management Failures
- [ ] Secrets chiffrés au repos
- [ ] Pas de secrets dans le code
- [ ] Rotation régulière

### ✅ K09: Misconfigured Cluster Components
- [ ] etcd sécurisé
- [ ] API server durci
- [ ] kubelet protégé

### ✅ K10: Outdated and Vulnerable Components
- [ ] Kubernetes à jour
- [ ] Images à jour
- [ ] Dépendances patchées
```

## Plans de Réponse

### Plan de Réponse aux Vulnérabilités

```bash
#!/bin/bash
# vulnerability-response.sh

respond_to_vulnerability() {
    SEVERITY=$1
    CVE=$2
    AFFECTED_IMAGE=$3

    case $SEVERITY in
        CRITICAL)
            echo "🚨 CRITIQUE: $CVE dans $AFFECTED_IMAGE"
            # 1. Identifier les pods affectés
            kubectl get pods --all-namespaces -o json | \
                jq -r ".items[] | select(.spec.containers[].image == \"$AFFECTED_IMAGE\") | \
                \"\(.metadata.namespace)/\(.metadata.name)\""

            # 2. Isoler immédiatement
            # 3. Planifier patch d'urgence
            ;;
        HIGH)
            echo "⚠️  ÉLEVÉ: $CVE dans $AFFECTED_IMAGE"
            # Planifier patch dans 24h
            ;;
        MEDIUM)
            echo "📌 MOYEN: $CVE dans $AFFECTED_IMAGE"
            # Planifier patch dans la semaine
            ;;
        LOW)
            echo "📝 FAIBLE: $CVE dans $AFFECTED_IMAGE"
            # Documenter pour prochain cycle
            ;;
    esac
}

# Scanner et répondre
for image in $(kubectl get pods --all-namespaces -o jsonpath="{.items[*].spec.containers[*].image}" | tr -s '[[:space:]]' '\n' | sort | uniq); do
    trivy image --format json $image | jq -r '.Results[].Vulnerabilities[] | "\(.Severity) \(.VulnerabilityID)"' | while read severity cve; do
        respond_to_vulnerability $severity $cve $image
    done
done
```

### Plan de Backup Sécurisé

```bash
#!/bin/bash
# secure-backup.sh

BACKUP_DIR="/secure/backups/$(date +%Y%m%d-%H%M%S)"
ENCRYPTION_KEY="/secure/keys/backup.key"

# Créer le répertoire de backup
mkdir -p $BACKUP_DIR

# 1. Backup des ressources (sans les secrets)
for resource in deployments services configmaps ingresses; do
    kubectl get $resource --all-namespaces -o yaml > $BACKUP_DIR/$resource.yaml
done

# 2. Backup chiffré des secrets
kubectl get secrets --all-namespaces -o yaml | \
    openssl enc -aes-256-cbc -salt -pass file:$ENCRYPTION_KEY > $BACKUP_DIR/secrets.yaml.enc

# 3. Backup des certificats
kubectl get certificates --all-namespaces -o yaml > $BACKUP_DIR/certificates.yaml

# 4. Backup de la configuration RBAC
kubectl get clusterroles,clusterrolebindings,roles,rolebindings --all-namespaces -o yaml > $BACKUP_DIR/rbac.yaml

# 5. Créer une archive chiffrée
tar czf - $BACKUP_DIR | openssl enc -aes-256-cbc -salt -pass file:$ENCRYPTION_KEY > backup-$(date +%Y%m%d).tar.gz.enc

# 6. Nettoyer
rm -rf $BACKUP_DIR

echo "✓ Backup sécurisé créé : backup-$(date +%Y%m%d).tar.gz.enc"
```

## Templates de Sécurité

### Template de Namespace Sécurisé

```yaml
# secure-namespace-template.yaml
---
apiVersion: v1
kind: Namespace
metadata:
  name: secure-app
  labels:
    pod-security.kubernetes.io/enforce: restricted
    pod-security.kubernetes.io/audit: restricted
    pod-security.kubernetes.io/warn: restricted
---
# Network Policy par défaut
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny-all
  namespace: secure-app
spec:
  podSelector: {}
  policyTypes:
  - Ingress
  - Egress
  egress:
  # Autoriser DNS uniquement
  - to:
    - namespaceSelector:
        matchLabels:
          name: kube-system
    ports:
    - protocol: UDP
      port: 53
---
# ResourceQuota
apiVersion: v1
kind: ResourceQuota
metadata:
  name: resource-quota
  namespace: secure-app
spec:
  hard:
    requests.cpu: "2"
    requests.memory: 4Gi
    limits.cpu: "4"
    limits.memory: 8Gi
    persistentvolumeclaims: "2"
---
# LimitRange
apiVersion: v1
kind: LimitRange
metadata:
  name: limit-range
  namespace: secure-app
spec:
  limits:
  - default:
      cpu: "500m"
      memory: "512Mi"
    defaultRequest:
      cpu: "100m"
      memory: "128Mi"
    type: Container
---
# ServiceAccount par défaut
apiVersion: v1
kind: ServiceAccount
metadata:
  name: default
  namespace: secure-app
automountServiceAccountToken: false
```

### Template de Deployment Sécurisé

```yaml
# secure-deployment-template.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: secure-app
  namespace: secure-app
  labels:
    app: secure-app
    version: v1.0.0
spec:
  replicas: 2
  selector:
    matchLabels:
      app: secure-app
  template:
    metadata:
      labels:
        app: secure-app
      annotations:
        # Force le re-déploiement si le ConfigMap change
        checksum/config: {{ include (print $.Template.BasePath "/configmap.yaml") . | sha256sum }}
    spec:
      serviceAccountName: app-service-account
      automountServiceAccountToken: false
      securityContext:
        runAsNonRoot: true
        runAsUser: 10001
        runAsGroup: 10001
        fsGroup: 10001
        seccompProfile:
          type: RuntimeDefault
      containers:
      - name: app
        image: myapp:v1.0.0-sha256@sha256:abc123...  # Image avec digest
        imagePullPolicy: Always
        ports:
        - containerPort: 8080
          name: http
          protocol: TCP
        env:
        - name: PORT
          value: "8080"
        # Secrets montés comme fichiers, pas en env
        volumeMounts:
        - name: secrets
          mountPath: /etc/secrets
          readOnly: true
        - name: tmp
          mountPath: /tmp
        - name: cache
          mountPath: /app/cache
        securityContext:
          allowPrivilegeEscalation: false
          readOnlyRootFilesystem: true
          runAsNonRoot: true
          runAsUser: 10001
          capabilities:
            drop:
            - ALL
            add:
            - NET_BIND_SERVICE  # Seulement si nécessaire
        resources:
          requests:
            memory: "128Mi"
            cpu: "100m"
          limits:
            memory: "256Mi"
            cpu: "500m"
        livenessProbe:
          httpGet:
            path: /healthz
            port: http
            scheme: HTTP
          initialDelaySeconds: 30
          periodSeconds: 10
          timeoutSeconds: 5
          failureThreshold: 3
        readinessProbe:
          httpGet:
            path: /ready
            port: http
            scheme: HTTP
          initialDelaySeconds: 10
          periodSeconds: 5
          timeoutSeconds: 3
      volumes:
      - name: secrets
        secret:
          secretName: app-secrets
          defaultMode: 0400  # Lecture seule pour l'utilisateur
      - name: tmp
        emptyDir: {}
      - name: cache
        emptyDir: {}
      affinity:
        podAntiAffinity:
          preferredDuringSchedulingIgnoredDuringExecution:
          - weight: 100
            podAffinityTerm:
              labelSelector:
                matchExpressions:
                - key: app
                  operator: In
                  values:
                  - secure-app
              topologyKey: kubernetes.io/hostname
```

## Maintenance de la Sécurité

### Planning de Maintenance

```bash
#!/bin/bash
# security-maintenance.sh

echo "=== Maintenance de Sécurité Hebdomadaire ==="

# Lundi : Scan de vulnérabilités
if [ $(date +%u) -eq 1 ]; then
    echo "Lundi : Scan de vulnérabilités"
    ./security-scan.sh > reports/scan-$(date +%Y%m%d).txt
fi

# Mardi : Revue des accès
if [ $(date +%u) -eq 2 ]; then
    echo "Mardi : Revue RBAC"
    kubectl get clusterrolebindings -o json | \
        jq '.items[] | select(.roleRef.name=="cluster-admin")' > reports/admin-access.json
fi

# Mercredi : Mise à jour des images
if [ $(date +%u) -eq 3 ]; then
    echo "Mercredi : Vérification des mises à jour"
    for image in $(kubectl get pods --all-namespaces -o jsonpath="{.items[*].spec.containers[*].image}" | tr -s '[[:space:]]' '\n' | sort | uniq); do
        echo "Checking updates for: $image"
        # Logique de vérification des nouvelles versions
    done
fi

# Jeudi : Backup
if [ $(date +%u) -eq 4 ]; then
    echo "Jeudi : Backup sécurisé"
    ./secure-backup.sh
fi

# Vendredi : Audit et rapport
if [ $(date +%u) -eq 5 ]; then
    echo "Vendredi : Audit complet"
    ./security-audit.sh > reports/audit-$(date +%Y%m%d).txt

    # Générer le rapport hebdomadaire
    cat > reports/weekly-$(date +%Y%m%d).md <<EOF
# Rapport de Sécurité Hebdomadaire

## Résumé
- Date : $(date)
- Cluster : MicroK8s Lab

## Vulnérabilités Détectées
$(grep CRITICAL reports/scan-*.txt | wc -l) critiques
$(grep HIGH reports/scan-*.txt | wc -l) élevées

## Actions Effectuées
- Scans : ✓
- Backups : ✓
- Mises à jour : En cours

## Recommandations
1. Mettre à jour les images avec vulnérabilités critiques
2. Revoir les accès cluster-admin
3. Implémenter les Network Policies manquantes

---
Généré automatiquement
EOF
fi
```

## Conclusion

La sécurité de votre cluster Kubernetes est un processus continu qui nécessite :

### Points Clés à Retenir

1. **Commencez par les bases** : Sécurisez l'accès, utilisez des secrets, mettez à jour
2. **Progressez méthodiquement** : RBAC, Network Policies, Security Contexts
3. **Automatisez** : Scripts d'audit, scans réguliers, alertes
4. **Documentez** : Gardez trace des configurations et incidents
5. **Apprenez continuellement** : La sécurité évolue constamment

### Prochaines Étapes

1. Exécutez le script d'audit pour évaluer votre niveau actuel
2. Implémentez les corrections niveau par niveau
3. Mettez en place le monitoring de sécurité
4. Planifiez des revues régulières
5. Restez informé des nouvelles vulnérabilités

### Ressources Additionnelles

- [CIS Kubernetes Benchmark](https://www.cisecurity.org/benchmark/kubernetes)
- [OWASP Kubernetes Security](https://owasp.org/www-project-kubernetes-top-ten/)
- [Kubernetes Security Documentation](https://kubernetes.io/docs/concepts/security/)
- [NIST Cybersecurity Framework](https://www.nist.gov/cyberframework)

---

*Note : Cette checklist est un guide pour un lab personnel. Pour un environnement de production, consultez un expert en sécurité et adaptez selon vos besoins spécifiques de conformité et de régulation.*

⏭️
