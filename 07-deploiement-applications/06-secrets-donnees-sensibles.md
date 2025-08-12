🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 7.6 - Secrets et données sensibles

## Introduction

Imaginez que vous confiez les clés de votre maison à quelqu'un. Vous ne les laisseriez pas traîner n'importe où, vous les donneriez discrètement et vous vous assureriez qu'elles sont en sécurité. C'est exactement le rôle des Secrets dans Kubernetes : gérer de manière sécurisée les informations sensibles comme les mots de passe, les clés API, les certificats SSL, et toute autre donnée confidentielle.

Dans cette section, nous allons explorer en profondeur comment Kubernetes gère les secrets, comment les utiliser correctement, et surtout comment éviter les erreurs de sécurité courantes.

## Pourquoi les Secrets sont critiques

### Le danger des données sensibles exposées

**Ce qu'il ne faut JAMAIS faire :**
```yaml
# ❌ MAUVAIS - Secret en clair dans le code
apiVersion: apps/v1
kind: Deployment
metadata:
  name: bad-practice-app
spec:
  template:
    spec:
      containers:
      - name: app
        image: myapp:v1
        env:
        - name: DATABASE_PASSWORD
          value: "SuperSecret123!"  # JAMAIS faire ça !
        - name: API_KEY
          value: "sk_live_4242424242424242"  # Exposé dans Git !
```

**Conséquences d'une mauvaise gestion :**
- 🔓 **Fuites de données** : Mots de passe dans les logs, Git, images Docker
- 💰 **Pertes financières** : Utilisation frauduleuse d'API payantes
- 📊 **Non-conformité** : Violation RGPD, PCI-DSS, HIPAA
- 🎯 **Cible d'attaques** : Accès non autorisé aux systèmes
- 🔄 **Rotation impossible** : Changement de secrets = rebuild complet

### La solution Kubernetes : Les Secrets

Les Secrets Kubernetes offrent :
- 🔒 **Séparation** : Données sensibles hors du code
- 🔐 **Contrôle d'accès** : RBAC pour limiter qui peut voir quoi
- 📝 **Audit** : Traçabilité des accès
- 🔄 **Rotation** : Mise à jour sans redéploiement
- 🔑 **Chiffrement** : Protection au repos (avec configuration)

## Comprendre les Secrets Kubernetes

### Qu'est-ce qu'un Secret ?

Un Secret est un objet Kubernetes qui contient une petite quantité de données sensibles. Contrairement aux ConfigMaps :

| Aspect | ConfigMap | Secret |
|--------|-----------|---------|
| **Usage** | Configuration non-sensible | Données sensibles |
| **Encodage** | Texte brut | Base64 (pas chiffré !) |
| **Taille max** | 1 MB | 1 MB |
| **Montage** | Normal | tmpfs (RAM) par défaut |
| **Logs** | Apparaît dans les logs | Masqué dans les logs |
| **RBAC** | Permissions standards | Permissions restrictives recommandées |

### Types de Secrets

Kubernetes propose plusieurs types de Secrets pour différents usages :

```yaml
# 1. Opaque (générique) - Le plus courant
apiVersion: v1
kind: Secret
metadata:
  name: generic-secret
type: Opaque
data:
  username: YWRtaW4=  # "admin" en base64
  password: UEBzc3cwcmQ=  # "P@ssw0rd" en base64

---
# 2. kubernetes.io/service-account-token
# Créé automatiquement pour les ServiceAccounts

---
# 3. kubernetes.io/dockerconfigjson - Pour Docker registry
apiVersion: v1
kind: Secret
metadata:
  name: docker-registry-secret
type: kubernetes.io/dockerconfigjson
data:
  .dockerconfigjson: eyJhdXRocyI6ey...  # Config Docker en base64

---
# 4. kubernetes.io/tls - Pour certificats TLS
apiVersion: v1
kind: Secret
metadata:
  name: tls-secret
type: kubernetes.io/tls
data:
  tls.crt: LS0tLS1CRUdJTi...  # Certificat en base64
  tls.key: LS0tLS1CRUdJTi...  # Clé privée en base64

---
# 5. kubernetes.io/basic-auth - Pour authentification basique
apiVersion: v1
kind: Secret
metadata:
  name: basic-auth-secret
type: kubernetes.io/basic-auth
stringData:
  username: admin
  password: t0p-S3cr3t

---
# 6. kubernetes.io/ssh-auth - Pour clés SSH
apiVersion: v1
kind: Secret
metadata:
  name: ssh-secret
type: kubernetes.io/ssh-auth
data:
  ssh-privatekey: LS0tLS1CRUdJTi...  # Clé SSH privée en base64
```

### Base64 n'est PAS du chiffrement !

**Point critique à comprendre :**
```bash
# Encoder en base64
echo -n "MonMotDePasse" | base64
# Résultat : TW9uTW90RGVQYXNzZQ==

# Décoder depuis base64
echo "TW9uTW90RGVQYXNzZQ==" | base64 -d
# Résultat : MonMotDePasse

# ⚠️ N'importe qui peut décoder !
```

Le base64 est juste un **encodage**, pas du **chiffrement**. Les Secrets Kubernetes sont :
- Encodés en base64 dans les manifestes
- Stockés en clair dans etcd (sauf si encryption at rest activé)
- Transmis en clair aux Pods (via API sécurisée)

## Création et gestion des Secrets

### Méthode 1 : Création déclarative (YAML)

```yaml
# secret-declarative.yaml
apiVersion: v1
kind: Secret
metadata:
  name: app-secrets
  namespace: production
  labels:
    app: myapp
    environment: production
  annotations:
    description: "Secrets de l'application principale"
    rotation-policy: "90-days"
type: Opaque
# Utiliser 'data' pour du base64
data:
  database-username: cHJvZHVzZXI=  # produser
  database-password: U3VwM3JTM2NyM3QhMjAyNA==  # Sup3rS3cr3t!2024

# Ou utiliser 'stringData' pour du texte clair (plus lisible)
stringData:
  api-key: "sk_live_4242424242424242"
  jwt-secret: "my-super-long-jwt-secret-key-that-should-be-random"
  encryption-key: "AES256-encryption-key-32-bytes!!"

  # Fichier de configuration avec secrets
  database-url: "postgresql://produser:Sup3rS3cr3t!2024@db.prod:5432/myapp"

  # Configuration JSON avec secrets
  config.json: |
    {
      "auth": {
        "clientId": "abc123def456",
        "clientSecret": "super-secret-client-key",
        "redirectUrl": "https://app.example.com/callback"
      },
      "api": {
        "key": "api_key_production_2024",
        "secret": "api_secret_production_2024"
      }
    }
```

### Méthode 2 : Création impérative (kubectl)

```bash
# Créer un secret depuis des valeurs littérales
kubectl create secret generic app-secrets \
  --from-literal=username=admin \
  --from-literal=password='S3cr3t!P@ssw0rd' \
  --namespace=production

# Créer depuis des fichiers
kubectl create secret generic app-secrets \
  --from-file=ssh-privatekey=/path/to/.ssh/id_rsa \
  --from-file=ssh-publickey=/path/to/.ssh/id_rsa.pub \
  --namespace=production

# Créer un secret Docker Registry
kubectl create secret docker-registry docker-hub-secret \
  --docker-server=docker.io \
  --docker-username=myusername \
  --docker-password=mypassword \
  --docker-email=myemail@example.com \
  --namespace=production

# Créer un secret TLS
kubectl create secret tls webapp-tls \
  --cert=path/to/tls.crt \
  --key=path/to/tls.key \
  --namespace=production

# Créer depuis un fichier .env
kubectl create secret generic app-secrets \
  --from-env-file=prod.env \
  --namespace=production
```

### Méthode 3 : Génération sécurisée

```bash
#!/bin/bash
# generate-secrets.sh

# Générer des mots de passe aléatoires
generate_password() {
  openssl rand -base64 32
}

# Générer une clé de chiffrement
generate_key() {
  openssl rand -hex 32
}

# Créer le secret avec des valeurs générées
kubectl create secret generic generated-secrets \
  --from-literal=db-password="$(generate_password)" \
  --from-literal=api-key="$(generate_key)" \
  --from-literal=jwt-secret="$(generate_password)" \
  --from-literal=encryption-key="$(generate_key)" \
  --namespace=production \
  --dry-run=client -o yaml > generated-secrets.yaml

echo "Secrets générés et sauvegardés dans generated-secrets.yaml"
echo "⚠️  Stocker ce fichier de manière sécurisée !"
```

## Utilisation des Secrets dans les applications

### 1. Comme variables d'environnement

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: secure-app
  namespace: production
spec:
  replicas: 3
  selector:
    matchLabels:
      app: secure-app
  template:
    metadata:
      labels:
        app: secure-app
    spec:
      containers:
      - name: app
        image: myapp:v1.0.0
        env:
        # Variable unique depuis un Secret
        - name: DATABASE_PASSWORD
          valueFrom:
            secretKeyRef:
              name: app-secrets
              key: database-password

        - name: API_KEY
          valueFrom:
            secretKeyRef:
              name: app-secrets
              key: api-key
              optional: false  # Fail si le secret n'existe pas

        # Construire une URL avec username et password
        - name: DATABASE_USERNAME
          valueFrom:
            secretKeyRef:
              name: app-secrets
              key: database-username

        - name: DATABASE_URL
          value: "postgresql://$(DATABASE_USERNAME):$(DATABASE_PASSWORD)@db.prod:5432/myapp"

        # Charger tous les secrets comme variables
        envFrom:
        - secretRef:
            name: app-secrets
            optional: false

        # Combiner avec ConfigMap
        - configMapRef:
            name: app-config
```

### 2. Comme fichiers montés

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: app-with-secret-files
  namespace: production
spec:
  template:
    spec:
      containers:
      - name: app
        image: myapp:v1.0.0
        volumeMounts:
        # Monter tous les secrets dans un répertoire
        - name: secrets-volume
          mountPath: /etc/secrets
          readOnly: true

        # Monter un secret spécifique comme fichier
        - name: database-password
          mountPath: /etc/secrets/database-password
          subPath: database-password
          readOnly: true

        # Certificats TLS
        - name: tls-certs
          mountPath: /etc/tls
          readOnly: true

        # Clés SSH
        - name: ssh-keys
          mountPath: /root/.ssh
          readOnly: true

      volumes:
      # Volume depuis Secret complet
      - name: secrets-volume
        secret:
          secretName: app-secrets
          defaultMode: 0400  # Lecture seule pour le propriétaire

      # Volume avec sélection de clés
      - name: database-password
        secret:
          secretName: app-secrets
          items:
          - key: database-password
            path: database-password
            mode: 0400

      # Volume TLS
      - name: tls-certs
        secret:
          secretName: webapp-tls
          items:
          - key: tls.crt
            path: cert.pem
          - key: tls.key
            path: key.pem
            mode: 0400

      # Volume SSH
      - name: ssh-keys
        secret:
          secretName: ssh-secret
          defaultMode: 0400
          items:
          - key: ssh-privatekey
            path: id_rsa
          - key: ssh-publickey
            path: id_rsa.pub
            mode: 0444  # Public key peut être lisible
```

### 3. Pour l'authentification Docker Registry

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: private-image-app
spec:
  template:
    spec:
      # Référencer le secret pour pull l'image
      imagePullSecrets:
      - name: docker-registry-secret

      containers:
      - name: app
        image: private-registry.example.com/myapp:v1.0.0
        # L'image sera pullée avec les credentials du secret
```

### 4. Pour ServiceAccount

```yaml
# ServiceAccount avec Secret automatique
apiVersion: v1
kind: ServiceAccount
metadata:
  name: app-service-account
  namespace: production
automountServiceAccountToken: true

---
# Deployment utilisant le ServiceAccount
apiVersion: apps/v1
kind: Deployment
metadata:
  name: app-with-sa
spec:
  template:
    spec:
      serviceAccountName: app-service-account
      containers:
      - name: app
        image: myapp:v1.0.0
        # Le token du SA est monté automatiquement dans:
        # /var/run/secrets/kubernetes.io/serviceaccount/
```

## Sécurité avancée des Secrets

### 1. Encryption at Rest

Par défaut, les Secrets sont stockés en clair dans etcd. Activez le chiffrement :

```yaml
# encryption-config.yaml (sur le master node)
apiVersion: apiserver.config.k8s.io/v1
kind: EncryptionConfiguration
resources:
  - resources:
    - secrets
    providers:
    # Utiliser AES-CBC avec PKCS#7 padding
    - aescbc:
        keys:
        - name: key1
          secret: <32-byte-key-base64-encoded>
    # Fallback pour les anciens secrets
    - identity: {}
```

Configuration MicroK8s :
```bash
# Activer le chiffrement des secrets
microk8s enable rbac
microk8s kubectl create secret generic encryption-key \
  --from-literal=key="$(head -c 32 /dev/urandom | base64)" \
  -n kube-system

# Vérifier
microk8s kubectl get secrets --all-namespaces
```

### 2. RBAC pour les Secrets

Limitez strictement qui peut accéder aux Secrets :

```yaml
# Role avec accès limité aux secrets
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: secret-reader
  namespace: production
rules:
# Lecture seule sur certains secrets
- apiGroups: [""]
  resources: ["secrets"]
  resourceNames: ["app-config-secret", "app-public-secret"]
  verbs: ["get", "list"]

---
# Role sans accès aux secrets
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: developer-role
  namespace: production
rules:
- apiGroups: [""]
  resources: ["pods", "services", "configmaps"]
  verbs: ["*"]
# Pas d'accès aux secrets !

---
# RoleBinding
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: secret-reader-binding
  namespace: production
subjects:
- kind: User
  name: trusted-user
  apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: Role
  name: secret-reader
  apiGroup: rbac.authorization.k8s.io
```

### 3. Network Policies pour les Secrets

Limitez quels Pods peuvent accéder aux services avec secrets :

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: secret-access-policy
  namespace: production
spec:
  podSelector:
    matchLabels:
      access-secrets: "true"
  policyTypes:
  - Ingress
  - Egress
  egress:
  # Autoriser uniquement l'accès à la DB
  - to:
    - podSelector:
        matchLabels:
          app: postgresql
    ports:
    - protocol: TCP
      port: 5432
  # DNS toujours autorisé
  - to:
    - namespaceSelector:
        matchLabels:
          name: kube-system
      podSelector:
        matchLabels:
          k8s-app: kube-dns
    ports:
    - protocol: UDP
      port: 53
```

### 4. Pod Security Standards

Appliquez des politiques de sécurité strictes :

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: secure-namespace
  labels:
    # Enforce Pod Security Standards
    pod-security.kubernetes.io/enforce: restricted
    pod-security.kubernetes.io/audit: restricted
    pod-security.kubernetes.io/warn: restricted

---
# Pod avec sécurité renforcée
apiVersion: v1
kind: Pod
metadata:
  name: secure-pod
  namespace: secure-namespace
spec:
  securityContext:
    runAsNonRoot: true
    runAsUser: 1000
    fsGroup: 2000
    seccompProfile:
      type: RuntimeDefault

  containers:
  - name: app
    image: myapp:v1.0.0
    securityContext:
      allowPrivilegeEscalation: false
      readOnlyRootFilesystem: true
      capabilities:
        drop:
        - ALL

    volumeMounts:
    - name: secrets
      mountPath: /etc/secrets
      readOnly: true

    # Utiliser un répertoire temporaire en RAM
    - name: tmp
      mountPath: /tmp

  volumes:
  - name: secrets
    secret:
      secretName: app-secrets
      defaultMode: 0400
  - name: tmp
    emptyDir:
      medium: Memory
      sizeLimit: 100Mi
```

## Gestion externe des Secrets

### 1. Sealed Secrets

Sealed Secrets permet de stocker des secrets chiffrés dans Git :

```bash
# Installer Sealed Secrets Controller
kubectl apply -f https://github.com/bitnami-labs/sealed-secrets/releases/download/v0.18.0/controller.yaml

# Installer kubeseal CLI
wget https://github.com/bitnami-labs/sealed-secrets/releases/download/v0.18.0/kubeseal-0.18.0-linux-amd64.tar.gz
tar -xvf kubeseal-0.18.0-linux-amd64.tar.gz
sudo install -m 755 kubeseal /usr/local/bin/kubeseal
```

Utilisation :
```yaml
# secret.yaml - NE PAS commiter
apiVersion: v1
kind: Secret
metadata:
  name: my-secret
  namespace: production
stringData:
  password: "SuperSecret123!"
```

```bash
# Créer un Sealed Secret
kubeseal --format yaml < secret.yaml > sealed-secret.yaml

# sealed-secret.yaml - PEUT être commité dans Git
```

```yaml
apiVersion: bitnami.com/v1alpha1
kind: SealedSecret
metadata:
  name: my-secret
  namespace: production
spec:
  encryptedData:
    password: AgA5kF3F4Z... # Chiffré, safe pour Git
```

### 2. External Secrets Operator

Synchronise les secrets depuis des systèmes externes :

```yaml
# Installation
helm repo add external-secrets https://charts.external-secrets.io
helm install external-secrets external-secrets/external-secrets -n external-secrets-system --create-namespace

---
# SecretStore pour AWS Secrets Manager
apiVersion: external-secrets.io/v1beta1
kind: SecretStore
metadata:
  name: aws-secretstore
  namespace: production
spec:
  provider:
    aws:
      service: SecretsManager
      region: eu-west-1
      auth:
        secretRef:
          accessKeyIDSecretRef:
            name: aws-credentials
            key: access-key-id
          secretAccessKeySecretRef:
            name: aws-credentials
            key: secret-access-key

---
# ExternalSecret qui synchronise depuis AWS
apiVersion: external-secrets.io/v1beta1
kind: ExternalSecret
metadata:
  name: app-secrets
  namespace: production
spec:
  refreshInterval: 1h
  secretStoreRef:
    name: aws-secretstore
    kind: SecretStore
  target:
    name: app-secrets
    creationPolicy: Owner
  data:
  - secretKey: database-password
    remoteRef:
      key: prod/database/password
  - secretKey: api-key
    remoteRef:
      key: prod/api/key
```

### 3. HashiCorp Vault

Intégration avec Vault pour gestion centralisée :

```yaml
# Vault Agent Injector
apiVersion: apps/v1
kind: Deployment
metadata:
  name: app-with-vault
spec:
  template:
    metadata:
      annotations:
        vault.hashicorp.com/agent-inject: "true"
        vault.hashicorp.com/role: "app-role"
        vault.hashicorp.com/agent-inject-secret-database: "secret/data/database/config"
        vault.hashicorp.com/agent-inject-template-database: |
          {{- with secret "secret/data/database/config" -}}
          export DB_PASSWORD="{{ .Data.data.password }}"
          export DB_USERNAME="{{ .Data.data.username }}"
          {{- end }}
    spec:
      serviceAccountName: app-vault
      containers:
      - name: app
        image: myapp:v1.0.0
        command: ['sh', '-c']
        args: ['source /vault/secrets/database && exec app']
```

### 4. SOPS (Secrets OPerationS)

Chiffrement de secrets dans les fichiers YAML :

```bash
# Installer SOPS
brew install sops  # ou télécharger depuis GitHub

# Créer un fichier de secrets
cat > secrets.yaml <<EOF
apiVersion: v1
kind: Secret
metadata:
  name: app-secrets
stringData:
  password: SuperSecret123!
  api_key: sk_live_xxxxx
EOF

# Chiffrer avec SOPS (utilise AWS KMS, GCP KMS, ou Age)
sops -e secrets.yaml > secrets.enc.yaml

# Le fichier chiffré peut être stocké dans Git
cat secrets.enc.yaml
```

## Patterns et bonnes pratiques

### Pattern 1 : Rotation des Secrets

```yaml
# Secrets versionnés pour rotation
apiVersion: v1
kind: Secret
metadata:
  name: db-password-v1
  annotations:
    rotation-date: "2024-01-01"
    expires: "2024-04-01"
stringData:
  password: "OldPassword123!"
---
apiVersion: v1
kind: Secret
metadata:
  name: db-password-v2
  annotations:
    rotation-date: "2024-04-01"
    expires: "2024-07-01"
stringData:
  password: "NewPassword456!"
---
# Deployment avec référence versionnée
apiVersion: apps/v1
kind: Deployment
metadata:
  name: app
spec:
  template:
    spec:
      containers:
      - name: app
        env:
        - name: DB_PASSWORD
          valueFrom:
            secretKeyRef:
              name: db-password-v2  # Référence à la nouvelle version
              key: password
```

Script de rotation automatique :
```bash
#!/bin/bash
# rotate-secrets.sh

NAMESPACE="production"
SECRET_NAME="db-password"
NEW_PASSWORD=$(openssl rand -base64 32)

# Créer nouveau secret versionné
VERSION=$(date +%Y%m%d%H%M%S)
kubectl create secret generic "${SECRET_NAME}-${VERSION}" \
  --from-literal=password="${NEW_PASSWORD}" \
  --namespace="${NAMESPACE}"

# Mettre à jour le deployment
kubectl set env deployment/app \
  DB_PASSWORD="${NEW_PASSWORD}" \
  --namespace="${NAMESPACE}"

# Attendre que le rollout soit terminé
kubectl rollout status deployment/app --namespace="${NAMESPACE}"

# Supprimer l'ancien secret après 24h
echo "kubectl delete secret ${SECRET_NAME}-old --namespace=${NAMESPACE}" | at now + 24 hours
```

### Pattern 2 : Secrets par environnement

```yaml
# Structure organisée par environnement
---
# secrets-dev.yaml
apiVersion: v1
kind: Secret
metadata:
  name: app-secrets
  namespace: development
stringData:
  database_url: "postgresql://dev:devpass@localhost:5432/devdb"
  api_endpoint: "https://api-dev.example.com"
  debug_mode: "true"
---
# secrets-staging.yaml
apiVersion: v1
kind: Secret
metadata:
  name: app-secrets
  namespace: staging
stringData:
  database_url: "postgresql://staging:stagingpass@db-staging:5432/stagingdb"
  api_endpoint: "https://api-staging.example.com"
  debug_mode: "false"
---
# secrets-prod.yaml
apiVersion: v1
kind: Secret
metadata:
  name: app-secrets
  namespace: production
stringData:
  database_url: "postgresql://prod:prodpass@db-prod:5432/proddb"
  api_endpoint: "https://api.example.com"
  debug_mode: "false"
  monitoring_key: "monitoring_key_production"
```

### Pattern 3 : Secrets immutables

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: immutable-secret
  namespace: production
immutable: true  # Ne peut plus être modifié après création
stringData:
  api_key: "sk_live_immutable_key_2024"
  certificate: |
    -----BEGIN CERTIFICATE-----
    MIIDXTCCAkWgAwIBAgIJAKl...
    -----END CERTIFICATE-----
```

### Pattern 4 : Validation des Secrets

```yaml
# InitContainer pour valider les secrets
apiVersion: apps/v1
kind: Deployment
metadata:
  name: app-with-validation
spec:
  template:
    spec:
      initContainers:
      - name: validate-secrets
        image: busybox
        command: ['sh', '-c']
        args:
        - |
          echo "Validating secrets..."

          # Vérifier que le mot de passe est assez long
          if [ ${#DB_PASSWORD} -lt 12 ]; then
            echo "ERROR: Database password too short"
            exit 1
          fi

          # Vérifier que l'API key a le bon format
          if ! echo "$API_KEY" | grep -q "^sk_live_"; then
            echo "ERROR: Invalid API key format"
            exit 1
          fi

          echo "Secrets validation passed"
        env:
        - name: DB_PASSWORD
          valueFrom:
            secretKeyRef:
              name: app-secrets
              key: database-password
        - name: API_KEY
          valueFrom:
            secretKeyRef:
              name: app-secrets
              key: api-key

      containers:
      - name: app
        image: myapp:v1.0.0
```

## Monitoring et audit

### Audit des accès aux Secrets

```yaml
# Audit Policy pour tracer l'accès aux secrets
apiVersion: audit.k8s.io/v1
kind: Policy
rules:
# Log tous les accès aux secrets
- level: RequestResponse
  omitStages:
  - RequestReceived
  resources:
  - group: ""
    resources: ["secrets"]
  namespaces: ["production", "staging"]
```

### Alertes Prometheus

```yaml
apiVersion: monitoring.coreos.com/v1
kind: PrometheusRule
metadata:
  name: secret-alerts
spec:
  groups:
  - name: secrets
    rules:
    - alert: SecretAccessed
      expr: |
        rate(apiserver_audit_event_total{verb="get",objectRef_resource="secrets"}[5m]) > 0.1
      for: 5m
      labels:
        severity: warning
      annotations:
        summary: "Unusual secret access pattern detected"
        description: "Secret {{ $labels.objectRef_name }} accessed {{ $value }} times/sec"

    - alert: SecretModified
      expr: |
        apiserver_audit_event_total{verb=~"create|update|patch|delete",objectRef_resource="secrets"}
      for: 1m
      labels:
        severity: critical
      annotations:
        summary: "Secret modified"
        description: "Secret {{ $labels.objectRef_name }} was {{ $labels.verb }}d"
```

### Scanning des Secrets exposés

```bash
#!/bin/bash
# scan-exposed-secrets.sh

echo "Scanning for exposed secrets..."

# Vérifier les variables d'environnement dans les Pods
kubectl get pods --all-namespaces -o json | \
  jq -r '.items[].spec.containers[].env[]? |
    select(.value != null) |
    select(.value | test("password|secret|key|token"; "i")) |
    "WARNING: Potential secret in env var: \(.name)"'

# Vérifier les ConfigMaps pour des secrets
kubectl get configmaps --all-namespaces -o json | \
  jq -r '.items[].data | to_entries[] |
    select(.value | test("password|secret|key|token"; "i")) |
    "WARNING: Potential secret in ConfigMap"'

# Vérifier les logs pour des secrets
kubectl logs --all-containers --all-namespaces --since=1h 2>/dev/null | \
  grep -iE "(password|secret|token|key).*=" | \
  head -10 && echo "WARNING: Potential secrets in logs"

echo "Scan complete. Review warnings above."
```

## Troubleshooting des Secrets

### Problèmes courants et solutions

#### Secret introuvable

```bash
# Erreur : secret "app-secrets" not found

# Diagnostic
kubectl get secrets -n production
kubectl describe secret app-secrets -n production

# Vérifier le namespace
kubectl get secrets --all-namespaces | grep app-secrets

# Créer si manquant
kubectl create secret generic app-secrets \
  --from-literal=key=value \
  -n production
```

#### Permission refusée

```yaml
# Erreur : User "system:serviceaccount:production:default" cannot get resource "secrets"

# Solution : Créer un Role et RoleBinding
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: secret-reader
  namespace: production
rules:
- apiGroups: [""]
  resources: ["secrets"]
  verbs: ["get", "list"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: secret-reader-binding
  namespace: production
subjects:
- kind: ServiceAccount
  name: default
  namespace: production
roleRef:
  kind: Role
  name: secret-reader
  apiGroup: rbac.authorization.k8s.io
```

#### Secret monté vide

```bash
# Diagnostic
kubectl exec -it <pod-name> -n production -- ls -la /etc/secrets/
kubectl exec -it <pod-name> -n production -- cat /etc/secrets/password

# Vérifier le montage
kubectl describe pod <pod-name> -n production | grep -A 10 "Mounts:"

# Vérifier que la clé existe dans le Secret
kubectl get secret app-secrets -n production -o jsonpath='{.data}'
```

#### Décodage base64

```bash
# Voir le contenu décodé d'un Secret
kubectl get secret app-secrets -n production -o json | \
  jq -r '.data | to_entries[] | "\(.key): \(.value | @base64d)"'

# Ou pour une clé spécifique
kubectl get secret app-secrets -n production \
  -o jsonpath='{.data.password}' | base64 -d
```

### Commandes de diagnostic utiles

```bash
# Lister tous les Secrets
kubectl get secrets --all-namespaces

# Voir les détails d'un Secret (sans les valeurs)
kubectl describe secret app-secrets -n production

# Exporter un Secret
kubectl get secret app-secrets -n production -o yaml > secret-backup.yaml

# Comparer deux Secrets
diff <(kubectl get secret secret1 -o yaml) \
     <(kubectl get secret secret2 -o yaml)

# Voir quels Pods utilisent un Secret
kubectl get pods -n production -o json | \
  jq -r '.items[] | select(.spec.volumes[]?.secret.secretName == "app-secrets") | .metadata.name'

# Audit trail des Secrets
kubectl get events -n production --field-selector involvedObject.kind=Secret
```

## Cas d'usage avancés

### Multi-tenancy avec Secrets

```yaml
# Namespace par tenant avec secrets isolés
---
apiVersion: v1
kind: Namespace
metadata:
  name: tenant-a
  labels:
    tenant: a
---
apiVersion: v1
kind: Secret
metadata:
  name: tenant-secrets
  namespace: tenant-a
stringData:
  api_key: "tenant_a_api_key"
  database_url: "postgresql://tenant_a:pass@db/tenant_a"
---
# NetworkPolicy pour isolation
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: tenant-isolation
  namespace: tenant-a
spec:
  podSelector: {}
  policyTypes:
  - Ingress
  - Egress
  ingress:
  - from:
    - podSelector: {}  # Seulement depuis le même namespace
  egress:
  - to:
    - podSelector: {}  # Seulement vers le même namespace
  - to:  # DNS
    - namespaceSelector:
        matchLabels:
          name: kube-system
    ports:
    - protocol: UDP
      port: 53
```

### Secrets pour CI/CD

```yaml
# Secrets pour pipeline GitLab CI
apiVersion: v1
kind: Secret
metadata:
  name: ci-secrets
  namespace: gitlab-runner
stringData:
  DOCKER_REGISTRY_USER: "gitlab-ci"
  DOCKER_REGISTRY_PASSWORD: "registry-password"
  SONAR_TOKEN: "sonar-analysis-token"
  DEPLOY_KEY: |
    -----BEGIN RSA PRIVATE KEY-----
    MIIEowIBAAKCAQEA...
    -----END RSA PRIVATE KEY-----
---
# Runner GitLab avec secrets
apiVersion: apps/v1
kind: Deployment
metadata:
  name: gitlab-runner
spec:
  template:
    spec:
      containers:
      - name: runner
        image: gitlab/gitlab-runner:latest
        envFrom:
        - secretRef:
            name: ci-secrets
```

### Secrets dynamiques avec Init Containers

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: app-with-dynamic-secrets
spec:
  template:
    spec:
      initContainers:
      # Générer des secrets au démarrage
      - name: secret-generator
        image: hashicorp/vault:latest
        command: ['sh', '-c']
        args:
        - |
          # Authentification Vault
          export VAULT_TOKEN=$(vault write -field=token auth/kubernetes/login \
            role=myapp \
            jwt=$(cat /var/run/secrets/kubernetes.io/serviceaccount/token))

          # Récupérer les secrets dynamiques
          vault read -format=json secret/data/myapp | \
            jq -r '.data.data | to_entries[] | "\(.key)=\(.value)"' > /shared/secrets

          echo "Secrets generated successfully"
        volumeMounts:
        - name: shared
          mountPath: /shared

      containers:
      - name: app
        image: myapp:v1.0.0
        command: ['sh', '-c']
        args:
        - |
          # Charger les secrets dynamiques
          export $(cat /shared/secrets | xargs)
          exec app
        volumeMounts:
        - name: shared
          mountPath: /shared

      volumes:
      - name: shared
        emptyDir:
          medium: Memory  # En RAM pour sécurité
```

### Secrets pour bases de données

```yaml
# Secret pour PostgreSQL
apiVersion: v1
kind: Secret
metadata:
  name: postgres-secret
  namespace: database
stringData:
  POSTGRES_USER: "admin"
  POSTGRES_PASSWORD: "SuperSecurePassword123!"
  POSTGRES_DB: "production"

  # Utilisateurs supplémentaires
  init.sql: |
    CREATE USER app_user WITH PASSWORD 'AppPassword456!';
    GRANT CONNECT ON DATABASE production TO app_user;
    GRANT USAGE ON SCHEMA public TO app_user;
    GRANT CREATE ON SCHEMA public TO app_user;

    CREATE USER readonly_user WITH PASSWORD 'ReadOnly789!';
    GRANT CONNECT ON DATABASE production TO readonly_user;
    GRANT USAGE ON SCHEMA public TO readonly_user;
    GRANT SELECT ON ALL TABLES IN SCHEMA public TO readonly_user;
---
# StatefulSet PostgreSQL avec secrets
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: postgres
spec:
  serviceName: postgres
  replicas: 1
  template:
    spec:
      containers:
      - name: postgres
        image: postgres:14
        envFrom:
        - secretRef:
            name: postgres-secret
        volumeMounts:
        - name: init-script
          mountPath: /docker-entrypoint-initdb.d
        - name: data
          mountPath: /var/lib/postgresql/data
      volumes:
      - name: init-script
        secret:
          secretName: postgres-secret
          items:
          - key: init.sql
            path: init.sql
  volumeClaimTemplates:
  - metadata:
      name: data
    spec:
      accessModes: ["ReadWriteOnce"]
      resources:
        requests:
          storage: 10Gi
```

## Outils et écosystème

### 1. Kubeseal (Sealed Secrets)

```bash
# Workflow complet avec Sealed Secrets
# 1. Créer un secret local
cat > secret.yaml <<EOF
apiVersion: v1
kind: Secret
metadata:
  name: myapp
  namespace: production
stringData:
  username: admin
  password: SecretPassword123!
EOF

# 2. Sceller le secret
kubeseal --format yaml < secret.yaml > sealed-secret.yaml

# 3. Le sealed-secret.yaml peut être commité dans Git
git add sealed-secret.yaml
git commit -m "Add sealed secret for myapp"

# 4. Appliquer le sealed secret
kubectl apply -f sealed-secret.yaml

# 5. Le controller déchiffre et crée le Secret réel
kubectl get secret myapp -n production
```

### 2. Mozilla SOPS

```yaml
# .sops.yaml - Configuration SOPS
creation_rules:
  - path_regex: .*secret.*\.yaml$
    kms: 'arn:aws:kms:us-east-1:1234567890:key/xxxxx'
  - path_regex: .*\.yaml$
    age: 'age1yyy...'
```

```bash
# Chiffrer
sops -e secret.yaml > secret.enc.yaml

# Déchiffrer et appliquer
sops -d secret.enc.yaml | kubectl apply -f -

# Éditer un secret chiffré
sops secret.enc.yaml  # Ouvre dans l'éditeur
```

### 3. Secrets Store CSI Driver

```yaml
# Installation
helm repo add secrets-store-csi-driver https://kubernetes-sigs.github.io/secrets-store-csi-driver/charts
helm install csi-secrets-store secrets-store-csi-driver/secrets-store-csi-driver -n kube-system

---
# SecretProviderClass pour Azure Key Vault
apiVersion: secrets-store.csi.x-k8s.io/v1
kind: SecretProviderClass
metadata:
  name: azure-kvstore
  namespace: production
spec:
  provider: azure
  parameters:
    usePodIdentity: "true"
    keyvaultName: "mykeyvault"
    tenantId: "tenant-id"
    objects: |
      array:
        - |
          objectName: database-password
          objectType: secret
        - |
          objectName: api-key
          objectType: secret
---
# Pod utilisant CSI
apiVersion: v1
kind: Pod
metadata:
  name: app-with-csi
spec:
  containers:
  - name: app
    image: myapp:v1.0.0
    volumeMounts:
    - name: secrets-store
      mountPath: "/mnt/secrets"
      readOnly: true
  volumes:
  - name: secrets-store
    csi:
      driver: secrets-store.csi.k8s.io
      readOnly: true
      volumeAttributes:
        secretProviderClass: azure-kvstore
```

## Checklist de sécurité

### Avant le déploiement

- [ ] **Jamais de secrets dans le code** : Vérifier Git, images Docker
- [ ] **Base64 n'est pas du chiffrement** : Comprendre la différence
- [ ] **Rotation planifiée** : Stratégie de rotation définie
- [ ] **RBAC configuré** : Accès minimal aux Secrets
- [ ] **Encryption at rest** : Activé sur le cluster
- [ ] **Audit logging** : Traçabilité des accès

### Configuration des Secrets

- [ ] **Type approprié** : Opaque, TLS, Docker, etc.
- [ ] **Namespace correct** : Isolation par environnement
- [ ] **Labels cohérents** : Pour organisation et sélection
- [ ] **Immutable si possible** : Protection contre modifications
- [ ] **Taille < 1MB** : Limite Kubernetes
- [ ] **Pas dans ConfigMaps** : Séparation config/secrets

### Utilisation dans les Pods

- [ ] **ReadOnly montage** : Protection en écriture
- [ ] **Mode 0400** : Permissions restrictives
- [ ] **tmpfs pour montage** : En RAM, pas sur disque
- [ ] **Variables vs fichiers** : Choisir selon le cas d'usage
- [ ] **Pas dans les logs** : Masquer les valeurs sensibles
- [ ] **InitContainers pour validation** : Vérifier avant démarrage

### Monitoring et maintenance

- [ ] **Alertes configurées** : Accès suspects, modifications
- [ ] **Backup des Secrets** : Stratégie de sauvegarde
- [ ] **Scan régulier** : Recherche de secrets exposés
- [ ] **Rotation automatique** : Scripts ou outils en place
- [ ] **Documentation** : Processus documentés
- [ ] **Formation équipe** : Bonnes pratiques connues

## Erreurs à éviter absolument

### ❌ Ne JAMAIS faire

```yaml
# 1. Secrets en clair dans les manifestes
env:
- name: PASSWORD
  value: "MonMotDePasse"  # ❌ JAMAIS

# 2. Secrets dans les ConfigMaps
kind: ConfigMap
data:
  password: "secret123"  # ❌ Utiliser Secret

# 3. Logs avec secrets
echo "Connecting with password: $PASSWORD"  # ❌

# 4. Secrets dans les images Docker
FROM alpine
ENV API_KEY=sk_live_xxxxx  # ❌

# 5. Commits Git avec secrets
git add secret.yaml  # ❌ Utiliser Sealed Secrets

# 6. Large secrets (> 1MB)
data:
  huge_file: <2MB base64>  # ❌ Limite dépassée

# 7. Permissions trop larges
defaultMode: 0777  # ❌ Trop permissif
```

### ✅ Toujours faire

```yaml
# 1. Utiliser les Secrets pour données sensibles
kind: Secret
stringData:
  password: "$(generated_password)"  # ✅

# 2. Permissions restrictives
defaultMode: 0400  # ✅ Read-only owner

# 3. RBAC minimal
verbs: ["get"]  # ✅ Pas "list" ou "*"

# 4. Rotation régulière
annotations:
  rotation-date: "2024-01-01"  # ✅

# 5. Audit et monitoring
audit: RequestResponse  # ✅ Tracer les accès

# 6. Chiffrement pour Git
kind: SealedSecret  # ✅ Safe pour Git

# 7. Validation des secrets
initContainers:
- name: validate  # ✅ Vérifier avant usage
```

## Conclusion

La gestion des Secrets est un aspect critique de la sécurité Kubernetes. Une mauvaise gestion peut compromettre l'ensemble de votre infrastructure.

**Points clés à retenir :**

1. **Les Secrets ne sont pas magiques** : Base64 n'est pas du chiffrement
   - Comprendre les limites
   - Implémenter des couches de sécurité supplémentaires

2. **Défense en profondeur** : Plusieurs couches de sécurité
   - RBAC pour limiter l'accès
   - Encryption at rest
   - Network policies
   - Audit logging

3. **Outils externes** : Pour une sécurité renforcée
   - Sealed Secrets pour Git
   - Vault pour gestion centralisée
   - CSI drivers pour cloud providers

4. **Rotation et maintenance** : Sécurité continue
   - Rotation régulière des secrets
   - Monitoring des accès
   - Scans de sécurité

5. **Formation et processus** : Facteur humain
   - Former les équipes
   - Documenter les processus
   - Automatiser au maximum

Dans la prochaine section (7.7), nous verrons comment gérer le stockage persistant pour les applications qui ont besoin de conserver des données au-delà du cycle de vie des Pods.

N'oubliez jamais : **La sécurité des Secrets est une responsabilité partagée** entre Kubernetes, vos outils, et vos pratiques. Un seul maillon faible peut compromettre l'ensemble.

⏭️
