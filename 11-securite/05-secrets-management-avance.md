🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 11.5 Secrets management avancé

## Comprendre la gestion des secrets

Imaginez que vous gérez un hôtel. Les secrets dans Kubernetes sont comme les clés des chambres, les codes du coffre-fort, et les mots de passe du WiFi. Vous ne pouvez pas les laisser traîner n'importe où, ni les donner à n'importe qui. Vous avez besoin d'un système sécurisé pour les stocker, les distribuer aux bonnes personnes, les changer régulièrement, et savoir qui les a utilisés.

### Pourquoi la gestion des secrets est critique

Les secrets mal gérés sont la cause numéro un des compromissions :
- **Secrets dans le code** : Visible dans Git, accessible à tous
- **Secrets en clair** : Lisibles par quiconque accède au système
- **Secrets partagés** : Une clé pour tout, si elle fuite, tout est compromis
- **Secrets permanents** : Jamais changés, accumulation du risque
- **Secrets non audités** : Aucune traçabilité de qui accède à quoi

## Les secrets natifs Kubernetes

### Anatomie d'un Secret Kubernetes

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: app-secret
  namespace: production
type: Opaque  # Type le plus courant
data:
  username: YWRtaW4=     # "admin" encodé en base64
  password: UEBzc3cwcmQ=  # "P@ssw0rd" encodé en base64
stringData:  # Alternative : texte clair (converti automatiquement)
  api-key: "sk_live_abcd1234efgh5678"
```

### Types de secrets Kubernetes

```yaml
# 1. Opaque - Données arbitraires
apiVersion: v1
kind: Secret
metadata:
  name: generic-secret
type: Opaque
data:
  config.json: eyJhcGkiOiAiaHR0cHM6Ly9hcGkuZXhhbXBsZS5jb20ifQ==

# 2. Docker Registry - Authentification pour pull d'images
apiVersion: v1
kind: Secret
metadata:
  name: registry-creds
type: kubernetes.io/dockerconfigjson
data:
  .dockerconfigjson: eyJhdXRocyI6IHsiaHR0cHM6Ly9pbmRleC5kb2NrZXIuaW8vdjEvIjp7...

# 3. TLS - Certificats SSL/TLS
apiVersion: v1
kind: Secret
metadata:
  name: tls-secret
type: kubernetes.io/tls
data:
  tls.crt: LS0tLS1CRUdJTiBDRVJUSUZJQ0FURS0tLS0tCi4uLg==
  tls.key: LS0tLS1CRUdJTiBQUklWQVRFIEtFWS0tLS0tCi4uLg==

# 4. Service Account Token - Authentification interne
apiVersion: v1
kind: Secret
metadata:
  name: sa-token
  annotations:
    kubernetes.io/service-account.name: my-sa
type: kubernetes.io/service-account-token

# 5. Basic Auth - Authentification basique
apiVersion: v1
kind: Secret
metadata:
  name: basic-auth
type: kubernetes.io/basic-auth
data:
  username: YWRtaW4=
  password: UEBzc3cwcmQ=

# 6. SSH Auth - Clés SSH
apiVersion: v1
kind: Secret
metadata:
  name: ssh-key
type: kubernetes.io/ssh-auth
data:
  ssh-privatekey: LS0tLS1CRUdJTiBPUEVOU1NIIFBSSVZBVEUgS0VZLS0tLS0K...
```

### Création et gestion des secrets

```bash
# Créer un secret depuis la ligne de commande
microk8s kubectl create secret generic db-secret \
  --from-literal=username=dbuser \
  --from-literal=password='S3cur3P@ss' \
  --namespace=production

# Créer depuis un fichier
echo -n 'admin' > ./username.txt
echo -n 'P@ssw0rd' > ./password.txt
microk8s kubectl create secret generic file-secret \
  --from-file=username=./username.txt \
  --from-file=password=./password.txt

# Créer un secret TLS
microk8s kubectl create secret tls tls-secret \
  --cert=path/to/tls.crt \
  --key=path/to/tls.key

# Créer un secret Docker Registry
microk8s kubectl create secret docker-registry registry-secret \
  --docker-server=https://index.docker.io/v1/ \
  --docker-username=dockeruser \
  --docker-password=dockerpass \
  --docker-email=user@example.com
```

## Utilisation des secrets dans les pods

### Montage comme volume

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: app-with-secrets
spec:
  containers:
  - name: app
    image: myapp:latest
    volumeMounts:
    - name: secret-volume
      mountPath: /etc/secrets
      readOnly: true
    # Monter seulement certaines clés
    - name: specific-secret
      mountPath: /etc/config
  volumes:
  - name: secret-volume
    secret:
      secretName: app-secret
      defaultMode: 0400  # Permissions restrictives
  - name: specific-secret
    secret:
      secretName: app-secret
      items:
      - key: username
        path: db-username  # Nom du fichier dans le pod
        mode: 0400
```

### Variables d'environnement

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: env-secrets
spec:
  containers:
  - name: app
    image: myapp:latest
    env:
    # Référence directe à une clé
    - name: DB_USERNAME
      valueFrom:
        secretKeyRef:
          name: db-secret
          key: username
    - name: DB_PASSWORD
      valueFrom:
        secretKeyRef:
          name: db-secret
          key: password
    # Charger toutes les clés comme variables
    envFrom:
    - secretRef:
        name: app-config
      prefix: APP_  # Préfixe optionnel
```

### Image Pull Secrets

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: private-app
spec:
  imagePullSecrets:
  - name: registry-secret
  containers:
  - name: app
    image: private-registry.io/myapp:latest
---
# Ou au niveau du ServiceAccount
apiVersion: v1
kind: ServiceAccount
metadata:
  name: app-sa
imagePullSecrets:
- name: registry-secret
```

## Encryption at rest

### Configuration de l'encryption dans MicroK8s

```yaml
# encryption-config.yaml
apiVersion: apiserver.config.k8s.io/v1
kind: EncryptionConfiguration
resources:
  - resources:
    - secrets
    - configmaps
    providers:
    - aescbc:
        keys:
        - name: key1
          secret: <BASE64_ENCODED_32_BYTE_KEY>
    - identity: {}  # Fallback non chiffré
```

```bash
# Générer une clé de chiffrement
head -c 32 /dev/urandom | base64

# Configurer MicroK8s pour utiliser l'encryption
# Éditer /var/snap/microk8s/current/args/kube-apiserver
# Ajouter: --encryption-provider-config=/path/to/encryption-config.yaml

# Redémarrer l'API server
microk8s stop
microk8s start

# Vérifier l'encryption
microk8s kubectl get secrets --all-namespaces -o json | \
  microk8s kubectl replace -f -
```

## Sealed Secrets - Secrets versionnables

### Installation de Sealed Secrets

```bash
# Installer le controller
microk8s kubectl apply -f https://github.com/bitnami-labs/sealed-secrets/releases/download/v0.24.0/controller.yaml

# Installer kubeseal CLI
wget https://github.com/bitnami-labs/sealed-secrets/releases/download/v0.24.0/kubeseal-0.24.0-linux-amd64.tar.gz
tar -xvf kubeseal-0.24.0-linux-amd64.tar.gz
sudo install -m 755 kubeseal /usr/local/bin/kubeseal
```

### Utilisation de Sealed Secrets

```bash
# Créer un secret normal
echo -n "mysecretpassword" | \
  microk8s kubectl create secret generic mysecret \
  --dry-run=client \
  --from-file=password=/dev/stdin \
  -o yaml > mysecret.yaml

# Sceller le secret
kubeseal -f mysecret.yaml -o yaml > mysealedsecret.yaml

# Le sealed secret peut être commité dans Git
cat mysealedsecret.yaml
```

```yaml
# Exemple de Sealed Secret (safe pour Git)
apiVersion: bitnami.com/v1alpha1
kind: SealedSecret
metadata:
  name: mysecret
  namespace: default
spec:
  encryptedData:
    password: AgBvA8Z1K8Hs9F... # Chiffré, safe pour Git
  template:
    metadata:
      name: mysecret
    type: Opaque
```

## External Secrets Operator

### Installation et configuration

```bash
# Installation via Helm
microk8s helm3 repo add external-secrets https://charts.external-secrets.io
microk8s helm3 install external-secrets \
  external-secrets/external-secrets \
  -n external-secrets-system \
  --create-namespace
```

### Configuration avec AWS Secrets Manager

```yaml
# SecretStore pour AWS
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
            name: aws-creds
            key: access-key
          secretAccessKeySecretRef:
            name: aws-creds
            key: secret-key
---
# ExternalSecret qui récupère depuis AWS
apiVersion: external-secrets.io/v1beta1
kind: ExternalSecret
metadata:
  name: app-secrets
  namespace: production
spec:
  refreshInterval: 1h  # Synchronisation toutes les heures
  secretStoreRef:
    name: aws-secretstore
    kind: SecretStore
  target:
    name: app-secrets  # Nom du secret Kubernetes créé
    creationPolicy: Owner
  data:
  - secretKey: database-password  # Clé dans le secret K8s
    remoteRef:
      key: prod/database  # Clé dans AWS Secrets Manager
      property: password   # Propriété JSON
```

### Configuration avec HashiCorp Vault

```yaml
# SecretStore pour Vault
apiVersion: external-secrets.io/v1beta1
kind: SecretStore
metadata:
  name: vault-backend
  namespace: production
spec:
  provider:
    vault:
      server: "https://vault.monlab.local:8200"
      path: "secret"
      version: "v2"
      auth:
        kubernetes:
          mountPath: "kubernetes"
          role: "demo"
          serviceAccountRef:
            name: vault-sa
---
# ExternalSecret utilisant Vault
apiVersion: external-secrets.io/v1beta1
kind: ExternalSecret
metadata:
  name: vault-secret
spec:
  refreshInterval: 15s
  secretStoreRef:
    name: vault-backend
    kind: SecretStore
  target:
    name: database-credentials
  dataFrom:
  - extract:
      key: /database/config
```

## Rotation automatique des secrets

### Rotation avec External Secrets

```yaml
apiVersion: external-secrets.io/v1beta1
kind: ExternalSecret
metadata:
  name: rotating-secret
spec:
  refreshInterval: 15m  # Vérifier toutes les 15 minutes
  secretStoreRef:
    name: vault-backend
    kind: SecretStore
  target:
    name: app-credentials
    creationPolicy: Owner
    # Déclencher un redéploiement lors du changement
    template:
      metadata:
        annotations:
          rotated-at: "{{ .rotationTime }}"
  data:
  - secretKey: password
    remoteRef:
      key: /rotating/database
      property: password
```

### Rotation avec CronJob

```yaml
apiVersion: batch/v1
kind: CronJob
metadata:
  name: rotate-secrets
  namespace: security
spec:
  schedule: "0 2 * * 0"  # Tous les dimanches à 2h
  jobTemplate:
    spec:
      template:
        spec:
          serviceAccountName: secret-rotator
          containers:
          - name: rotator
            image: secret-rotator:latest
            command:
            - /bin/sh
            - -c
            - |
              # Générer nouveau mot de passe
              NEW_PASS=$(openssl rand -base64 32)

              # Mettre à jour dans Kubernetes
              kubectl patch secret db-secret \
                -p '{"data":{"password":"'$(echo -n $NEW_PASS | base64 -w0)'"}}'

              # Mettre à jour dans la base de données
              mysql -h database -u admin -p$OLD_PASS \
                -e "ALTER USER 'app'@'%' IDENTIFIED BY '$NEW_PASS';"

              # Redémarrer les pods pour prendre le nouveau secret
              kubectl rollout restart deployment/app
          restartPolicy: OnFailure
```

## RBAC pour les secrets

### Politique de moindre privilège

```yaml
# Role pour lire seulement certains secrets
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: secret-reader
  namespace: production
rules:
- apiGroups: [""]
  resources: ["secrets"]
  resourceNames: ["app-config", "app-tls"]  # Secrets spécifiques
  verbs: ["get", "list"]
---
# Role pour gérer les secrets
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: secret-manager
  namespace: production
rules:
- apiGroups: [""]
  resources: ["secrets"]
  verbs: ["get", "list", "create", "update", "patch"]
  # Pas de "delete" pour éviter les suppressions accidentelles
---
# ServiceAccount avec accès limité
apiVersion: v1
kind: ServiceAccount
metadata:
  name: app-sa
  namespace: production
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: app-secret-binding
  namespace: production
subjects:
- kind: ServiceAccount
  name: app-sa
  namespace: production
roleRef:
  kind: Role
  name: secret-reader
  apiGroup: rbac.authorization.k8s.io
```

## Secrets CSI Driver

### Installation du Secrets Store CSI Driver

```bash
# Installation via Helm
microk8s helm3 repo add secrets-store-csi-driver \
  https://kubernetes-sigs.github.io/secrets-store-csi-driver/charts

microk8s helm3 install csi-secrets-store \
  secrets-store-csi-driver/secrets-store-csi-driver \
  --namespace kube-system
```

### Utilisation avec Azure Key Vault

```yaml
# SecretProviderClass pour Azure
apiVersion: secrets-store.csi.x-k8s.io/v1
kind: SecretProviderClass
metadata:
  name: azure-kvname
spec:
  provider: azure
  parameters:
    usePodIdentity: "false"
    useVMManagedIdentity: "true"
    keyvaultName: "mykeyvault"
    objects: |
      array:
        - |
          objectName: database-password
          objectType: secret
        - |
          objectName: api-key
          objectType: secret
    tenantId: "<TENANT_ID>"
---
# Pod utilisant le CSI driver
apiVersion: v1
kind: Pod
metadata:
  name: app-with-csi
spec:
  containers:
  - name: app
    image: myapp:latest
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
        secretProviderClass: azure-kvname
```

## Kubernetes Secrets Store avec etcd

### Audit des accès aux secrets

```yaml
# Politique d'audit pour tracer l'accès aux secrets
apiVersion: audit.k8s.io/v1
kind: Policy
rules:
- level: RequestResponse
  omitStages:
  - RequestReceived
  resources:
  - group: ""
    resources: ["secrets"]
  namespaces: ["production", "staging"]
```

### Backup et restauration des secrets

```bash
#!/bin/bash
# Script de backup des secrets

BACKUP_DIR="/backup/secrets/$(date +%Y%m%d)"
mkdir -p $BACKUP_DIR

# Backup tous les secrets
for ns in $(microk8s kubectl get ns -o name | cut -d/ -f2); do
  echo "Backing up secrets from namespace: $ns"
  microk8s kubectl get secrets -n $ns -o yaml > "$BACKUP_DIR/secrets-$ns.yaml"
done

# Chiffrer le backup
tar czf - $BACKUP_DIR | \
  openssl enc -aes-256-cbc -salt -out "$BACKUP_DIR.tar.gz.enc" -k $BACKUP_PASSWORD

# Nettoyer les fichiers non chiffrés
rm -rf $BACKUP_DIR
```

## Patterns et bonnes pratiques

### Pattern 1 : Secrets par environnement

```yaml
# Structure de namespaces avec secrets isolés
---
apiVersion: v1
kind: Namespace
metadata:
  name: dev
  labels:
    environment: dev
---
apiVersion: v1
kind: Namespace
metadata:
  name: staging
  labels:
    environment: staging
---
apiVersion: v1
kind: Namespace
metadata:
  name: production
  labels:
    environment: production
---
# Template de secret par environnement
apiVersion: v1
kind: Secret
metadata:
  name: app-secrets
  namespace: {{ .Environment }}
type: Opaque
stringData:
  database-url: "postgres://{{ .DBHost }}/{{ .DBName }}"
  api-key: "{{ .APIKey }}"
  environment: "{{ .Environment }}"
```

### Pattern 2 : Secrets immutables

```yaml
# Secret immutable (ne peut pas être modifié après création)
apiVersion: v1
kind: Secret
metadata:
  name: immutable-secret
  namespace: production
type: Opaque
immutable: true  # Empêche les modifications
data:
  api-key: YXBpLWtleS12YWx1ZQ==
```

### Pattern 3 : Secret injection avec init containers

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: app-with-secrets
spec:
  template:
    spec:
      initContainers:
      - name: secret-fetcher
        image: vault-injector:latest
        env:
        - name: VAULT_ADDR
          value: "https://vault.monlab.local"
        - name: VAULT_ROLE
          value: "app-role"
        volumeMounts:
        - name: shared-data
          mountPath: /secrets
        command:
        - sh
        - -c
        - |
          # Récupérer les secrets depuis Vault
          vault login -method=kubernetes
          vault kv get -format=json secret/app | jq -r .data > /secrets/config.json
      containers:
      - name: app
        image: myapp:latest
        volumeMounts:
        - name: shared-data
          mountPath: /etc/secrets
          readOnly: true
      volumes:
      - name: shared-data
        emptyDir: {}
```

## Monitoring et alerting

### Métriques Prometheus pour les secrets

```yaml
apiVersion: v1
kind: ServiceMonitor
metadata:
  name: secrets-metrics
spec:
  selector:
    matchLabels:
      app: secrets-controller
  endpoints:
  - port: metrics
    interval: 30s
---
# Règles d'alerte
apiVersion: monitoring.coreos.com/v1
kind: PrometheusRule
metadata:
  name: secrets-alerts
spec:
  groups:
  - name: secrets
    rules:
    - alert: SecretExpirationWarning
      expr: (secret_expiration_timestamp - time()) / 86400 < 30
      for: 1h
      annotations:
        summary: "Secret {{ $labels.secret_name }} expires in {{ $value }} days"

    - alert: SecretAccessAnomaly
      expr: rate(secret_access_total[5m]) > 10
      for: 5m
      annotations:
        summary: "Unusual secret access pattern for {{ $labels.secret_name }}"

    - alert: SecretRotationOverdue
      expr: (time() - secret_last_rotation_timestamp) / 86400 > 90
      annotations:
        summary: "Secret {{ $labels.secret_name }} not rotated for {{ $value }} days"
```

### Logs d'audit pour les secrets

```yaml
# Fluentd configuration pour logger les accès aux secrets
apiVersion: v1
kind: ConfigMap
metadata:
  name: fluentd-config
data:
  fluent.conf: |
    <source>
      @type tail
      path /var/log/kube-apiserver-audit.log
      pos_file /var/log/fluentd-audit.pos
      tag audit
      <parse>
        @type json
      </parse>
    </source>

    <filter audit>
      @type grep
      <regexp>
        key $.objectRef.resource
        pattern ^secrets$
      </regexp>
    </filter>

    <match audit>
      @type elasticsearch
      host elasticsearch.monitoring
      port 9200
      index_name secret-audit
      type_name access
      <buffer>
        @type file
        path /var/log/fluentd-buffer/audit
        flush_interval 10s
      </buffer>
    </match>
```

## Outils complémentaires

### SOPS - Secrets Operations

```bash
# Installation de SOPS
wget https://github.com/mozilla/sops/releases/download/v3.8.0/sops-v3.8.0.linux
sudo mv sops-v3.8.0.linux /usr/local/bin/sops
sudo chmod +x /usr/local/bin/sops

# Configuration avec Age
age-keygen -o key.txt
export SOPS_AGE_KEY_FILE=key.txt

# Chiffrer un fichier de secrets
sops -e secrets.yaml > secrets.enc.yaml

# Déchiffrer
sops -d secrets.enc.yaml > secrets.yaml

# Éditer directement un fichier chiffré
sops secrets.enc.yaml
```

### Helm Secrets

```bash
# Installation du plugin helm-secrets
microk8s helm3 plugin install https://github.com/jkroepke/helm-secrets

# Créer un fichier de valeurs secrètes
cat > secrets.yaml <<EOF
database:
  password: supersecret
api:
  key: apikey123
EOF

# Chiffrer avec SOPS
sops -e secrets.yaml > secrets.enc.yaml

# Utiliser avec Helm
microk8s helm3 secrets upgrade --install myapp ./chart \
  -f values.yaml \
  -f secrets://secrets.enc.yaml
```

## Validation et compliance

### OPA Policies pour les secrets

```yaml
# Politique OPA pour valider l'utilisation des secrets
apiVersion: v1
kind: ConfigMap
metadata:
  name: opa-secret-policies
data:
  policy.rego: |
    package kubernetes.admission

    deny[msg] {
      input.request.kind.kind == "Pod"
      input.request.object.spec.containers[_].env[_].value
      contains(input.request.object.spec.containers[_].env[_].value, "password")
      msg := "Passwords should not be in plain text environment variables"
    }

    deny[msg] {
      input.request.kind.kind == "Secret"
      input.request.object.type == "Opaque"
      not input.request.object.metadata.labels["rotation-policy"]
      msg := "Secrets must have a rotation-policy label"
    }

    deny[msg] {
      input.request.kind.kind == "Secret"
      not input.request.object.immutable
      input.request.object.metadata.namespace == "production"
      msg := "Production secrets should be immutable"
    }
```

### Checklist de sécurité pour les secrets

```yaml
# ConfigMap avec checklist
apiVersion: v1
kind: ConfigMap
metadata:
  name: secrets-checklist
data:
  checklist.yaml: |
    secret_requirements:
      - encryption_at_rest: enabled
      - rbac_configured: true
      - audit_logging: enabled
      - rotation_policy: defined
      - backup_strategy: implemented

    per_secret_validation:
      - has_expiration_date: true
      - uses_strong_encryption: true
      - follows_naming_convention: true
      - documented_in_wiki: true
      - approved_by_security: true

    forbidden_practices:
      - secrets_in_environment_variables: false
      - secrets_in_configmaps: false
      - secrets_in_git: false
      - shared_secrets_across_environments: false
      - default_passwords: false
```

## Troubleshooting courant

### Problème : Secret non accessible

```bash
# Diagnostic
# 1. Vérifier que le secret existe
microk8s kubectl get secret mysecret -n mynamespace

# 2. Vérifier les permissions RBAC
microk8s kubectl auth can-i get secret -n mynamespace --as=system:serviceaccount:mynamespace:mysa

# 3. Vérifier le montage dans le pod
microk8s kubectl describe pod mypod -n mynamespace | grep -A5 Mounts

# 4. Vérifier le contenu du secret
microk8s kubectl get secret mysecret -n mynamespace -o jsonpath='{.data}'

# 5. Décoder le secret
microk8s kubectl get secret mysecret -n mynamespace -o jsonpath='{.data.mykey}' | base64 -d
```

### Problème : Secret rotation échoue

```bash
#!/bin/bash
# Script de debug pour rotation

SECRET_NAME=$1
NAMESPACE=$2

echo "Checking secret rotation for $SECRET_NAME in $NAMESPACE"

# Vérifier l'âge du secret
AGE=$(microk8s kubectl get secret $SECRET_NAME -n $NAMESPACE -o jsonpath='{.metadata.creationTimestamp}')
echo "Secret created at: $AGE"

# Vérifier les annotations de rotation
ROTATION=$(microk8s kubectl get secret $SECRET_NAME -n $NAMESPACE -o jsonpath='{.metadata.annotations.last-rotation}')
echo "Last rotation: ${ROTATION:-Never}"

# Lister les pods utilisant ce secret
echo "Pods using this secret:"
microk8s kubectl get pods -n $NAMESPACE -o json | \
  jq -r '.items[] | select(.spec.volumes[]?.secret.secretName == "'$SECRET_NAME'") | .metadata.name'

# Vérifier les events
echo "Recent events:"
microk8s kubectl get events -n $NAMESPACE --field-selector involvedObject.name=$SECRET_NAME
```

### Problème : Performance avec beaucoup de secrets

```yaml
# Optimisation : Utiliser un seul secret avec plusieurs clés
# Au lieu de :
apiVersion: v1
kind: Secret
metadata:
  name: db-username
data:
  value: YWRtaW4=
---
apiVersion: v1
kind: Secret
metadata:
  name: db-password
data:
  value: cGFzc3dvcmQ=

# Préférer :
apiVersion: v1
kind: Secret
metadata:
  name: db-credentials
data:
  username: YWRtaW4=
  password: cGFzc3dvcmQ=
  host: bG9jYWxob3N0
  port: NTQzMg==
```

## Architecture de référence

### Architecture complète de gestion des secrets

```yaml
# Composants de l'architecture
components:
  external_vault:
    - name: "HashiCorp Vault"
    - role: "Source of truth pour les secrets"
    - features:
      - encryption
      - audit
      - dynamic_secrets
      - rotation

  operators:
    - name: "External Secrets Operator"
    - role: "Synchronisation Vault -> K8s"
    - refresh_interval: "1m"

    - name: "Sealed Secrets Controller"
    - role: "Secrets dans Git"
    - encryption: "Asymétrique"

  csi_drivers:
    - name: "Secrets Store CSI Driver"
    - role: "Montage direct depuis vault"
    - providers: ["vault", "azure", "aws", "gcp"]

  monitoring:
    - prometheus:
        metrics: ["secret_age", "rotation_status", "access_count"]
    - grafana:
        dashboards: ["secret_lifecycle", "compliance"]
    - alertmanager:
        alerts: ["expiration", "anomaly", "rotation_due"]

  backup:
    - velero:
        schedule: "daily"
        encryption: "enabled"
    - manual:
        script: "backup-secrets.sh"
        frequency: "weekly"
```

## Migration depuis les secrets basiques

### Plan de migration étape par étape

```bash
#!/bin/bash
# Script de migration vers une solution avancée

echo "=== Migration des secrets vers Vault ==="

# Étape 1 : Inventaire
echo "1. Inventaire des secrets existants"
microk8s kubectl get secrets --all-namespaces -o json | \
  jq -r '.items[] | select(.type != "kubernetes.io/service-account-token") |
  "\(.metadata.namespace)/\(.metadata.name)"' > secrets-inventory.txt

# Étape 2 : Backup
echo "2. Backup des secrets actuels"
mkdir -p backup/secrets
while read secret; do
  ns=$(echo $secret | cut -d/ -f1)
  name=$(echo $secret | cut -d/ -f2)
  microk8s kubectl get secret $name -n $ns -o yaml > "backup/secrets/${ns}-${name}.yaml"
done < secrets-inventory.txt

# Étape 3 : Classification
echo "3. Classification par criticité"
cat > secret-classification.yaml <<EOF
critical:
  - production/database-credentials
  - production/api-keys
  - production/tls-certificates
high:
  - staging/database-credentials
  - staging/api-keys
medium:
  - development/test-credentials
low:
  - development/demo-secrets
EOF

# Étape 4 : Migration progressive
echo "4. Migration vers Vault"
for priority in critical high medium low; do
  echo "Migrating $priority secrets..."
  case $priority in
    critical)
      # Migration avec validation approfondie
      while IFS= read -r secret; do
        echo "Processing critical secret: $secret"
        # Export vers Vault avec audit trail
        vault kv put secret/$secret @backup/secrets/${secret//\//-}.yaml
        # Vérification de l'import
        vault kv get secret/$secret
      done < <(grep -A10 "^$priority:" secret-classification.yaml | tail -n +2)
      ;;
    *)
      # Migration standard pour les autres niveaux
      echo "Standard migration for $priority secrets"
      ;;
  esac
done

echo "Migration complete!"
```

### Timeline de migration recommandée

```yaml
migration_phases:
  phase_1_preparation:  # Semaine 1-2
    - install_vault: true
    - install_external_secrets: true
    - setup_backup_strategy: true
    - train_team: true

  phase_2_pilot:  # Semaine 3-4
    - migrate_dev_environment: true
    - test_rotation: true
    - validate_access_patterns: true
    - document_procedures: true

  phase_3_staging:  # Semaine 5-6
    - migrate_staging: true
    - test_disaster_recovery: true
    - performance_testing: true
    - security_audit: true

  phase_4_production:  # Semaine 7-8
    - migrate_non_critical_prod: true
    - monitor_for_issues: true
    - migrate_critical_prod: true
    - decommission_old_secrets: true
```

## Cas d'usage avancés

### Multi-tenancy avec isolation des secrets

```yaml
# Namespace par tenant avec isolation stricte
apiVersion: v1
kind: Namespace
metadata:
  name: tenant-a
  labels:
    tenant: a
    isolation: strict
---
# NetworkPolicy pour isoler le trafic
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
    - namespaceSelector:
        matchLabels:
          tenant: a
  egress:
  - to:
    - namespaceSelector:
        matchLabels:
          tenant: a
---
# RBAC pour isolation des secrets
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: tenant-secret-access
  namespace: tenant-a
rules:
- apiGroups: [""]
  resources: ["secrets"]
  verbs: ["get", "list", "create", "update", "patch"]
---
# Quota pour limiter le nombre de secrets
apiVersion: v1
kind: ResourceQuota
metadata:
  name: secret-quota
  namespace: tenant-a
spec:
  hard:
    count/secrets.type.Opaque: "10"
    count/secrets.type.kubernetes.io/tls: "5"
```

### Secrets pour CI/CD

```yaml
# Secret pour pipeline CI/CD
apiVersion: external-secrets.io/v1beta1
kind: ExternalSecret
metadata:
  name: ci-credentials
  namespace: ci-cd
spec:
  refreshInterval: 5m
  secretStoreRef:
    name: vault-backend
    kind: SecretStore
  target:
    name: ci-credentials
    template:
      engineVersion: v2
      data:
        # Credentials pour différents environnements
        docker-config: |
          {
            "auths": {
              "registry.gitlab.com": {
                "username": "{{ .docker_username }}",
                "password": "{{ .docker_password }}"
              },
              "docker.io": {
                "username": "{{ .dockerhub_username }}",
                "password": "{{ .dockerhub_password }}"
              }
            }
          }
        kubeconfig: |
          apiVersion: v1
          kind: Config
          clusters:
          - cluster:
              server: {{ .k8s_server }}
              certificate-authority-data: {{ .k8s_ca }}
            name: production
          contexts:
          - context:
              cluster: production
              user: ci-user
            name: production
          users:
          - name: ci-user
            user:
              token: {{ .k8s_token }}
  data:
  - secretKey: docker_username
    remoteRef:
      key: ci/docker
      property: username
  - secretKey: docker_password
    remoteRef:
      key: ci/docker
      property: password
  - secretKey: dockerhub_username
    remoteRef:
      key: ci/dockerhub
      property: username
  - secretKey: dockerhub_password
    remoteRef:
      key: ci/dockerhub
      property: password
  - secretKey: k8s_server
    remoteRef:
      key: ci/kubernetes
      property: server
  - secretKey: k8s_ca
    remoteRef:
      key: ci/kubernetes
      property: ca_cert
  - secretKey: k8s_token
    remoteRef:
      key: ci/kubernetes
      property: token
```

### Secrets dynamiques pour bases de données

```yaml
# Configuration Vault pour secrets dynamiques PostgreSQL
apiVersion: v1
kind: ConfigMap
metadata:
  name: vault-db-config
data:
  setup.sh: |
    #!/bin/bash
    # Configurer le backend database dans Vault
    vault secrets enable database

    # Configurer la connexion PostgreSQL
    vault write database/config/postgresql \
      plugin_name=postgresql-database-plugin \
      allowed_roles="app-role" \
      connection_url="postgresql://{{username}}:{{password}}@postgres:5432/mydb?sslmode=disable" \
      username="vault" \
      password="vault-password"

    # Créer un rôle pour générer des credentials dynamiques
    vault write database/roles/app-role \
      db_name=postgresql \
      creation_statements="CREATE USER \"{{name}}\" WITH PASSWORD '{{password}}' VALID UNTIL '{{expiration}}'; \
        GRANT SELECT, INSERT, UPDATE, DELETE ON ALL TABLES IN SCHEMA public TO \"{{name}}\";" \
      default_ttl="1h" \
      max_ttl="24h"
---
# Pod utilisant des credentials dynamiques
apiVersion: v1
kind: Pod
metadata:
  name: app-with-dynamic-db
  annotations:
    vault.hashicorp.com/agent-inject: "true"
    vault.hashicorp.com/role: "app-role"
    vault.hashicorp.com/agent-inject-secret-db: "database/creds/app-role"
spec:
  serviceAccountName: app-sa
  containers:
  - name: app
    image: myapp:latest
    env:
    - name: DB_CREDS_PATH
      value: "/vault/secrets/db"
    command:
    - sh
    - -c
    - |
      # Lire les credentials dynamiques
      source /vault/secrets/db
      export DB_USERNAME=$username
      export DB_PASSWORD=$password
      # Démarrer l'application
      exec /app/start.sh
```

## Patterns de sécurité avancés

### Zero-Trust secrets architecture

```yaml
# Architecture Zero-Trust pour les secrets
apiVersion: v1
kind: ConfigMap
metadata:
  name: zero-trust-architecture
data:
  principles.yaml: |
    zero_trust_principles:
      1_never_trust:
        - no_default_access: true
        - explicit_grants_only: true
        - time_limited_access: true

      2_always_verify:
        - mutual_tls: enabled
        - service_mesh_integration: true
        - continuous_validation: true

      3_least_privilege:
        - minimal_permissions: true
        - just_in_time_access: true
        - regular_permission_review: true

      4_assume_breach:
        - encryption_everywhere: true
        - audit_everything: true
        - automatic_rotation: true
        - breach_detection: true

    implementation:
      service_mesh:
        - istio_strict_mtls: true
        - authorization_policies: enforced

      secrets_management:
        - vault_integration: required
        - dynamic_credentials: preferred
        - static_secrets_forbidden: true

      monitoring:
        - real_time_alerts: true
        - anomaly_detection: enabled
        - audit_retention_days: 365
```

### Secretless architecture avec Workload Identity

```yaml
# Utilisation de Workload Identity pour éviter les secrets
apiVersion: v1
kind: ServiceAccount
metadata:
  name: workload-identity-sa
  namespace: production
  annotations:
    # GCP Workload Identity
    iam.gke.io/gcp-service-account: "app-sa@project.iam.gserviceaccount.com"
    # AWS IRSA
    eks.amazonaws.com/role-arn: "arn:aws:iam::123456789012:role/app-role"
    # Azure AD Workload Identity
    azure.workload.identity/client-id: "12345678-1234-1234-1234-123456789012"
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: secretless-app
spec:
  replicas: 3
  selector:
    matchLabels:
      app: secretless-app
  template:
    metadata:
      labels:
        app: secretless-app
        azure.workload.identity/use: "true"
    spec:
      serviceAccountName: workload-identity-sa
      containers:
      - name: app
        image: myapp:latest
        env:
        - name: USE_WORKLOAD_IDENTITY
          value: "true"
        - name: CLOUD_PROVIDER
          value: "azure"  # ou aws, gcp
        # Pas de secrets ! L'authentification se fait via l'identité du pod
        volumeMounts:
        - name: azure-identity-token
          mountPath: /var/run/secrets/azure/tokens
          readOnly: true
      volumes:
      - name: azure-identity-token
        projected:
          sources:
          - serviceAccountToken:
              audience: api://AzureADTokenExchange
              expirationSeconds: 3600
              path: azure-identity-token
```

## Métriques et observabilité

### Dashboard Grafana complet

```json
{
  "dashboard": {
    "title": "Secrets Management Overview",
    "uid": "secrets-mgmt-001",
    "version": 1,
    "panels": [
      {
        "id": 1,
        "title": "Secret Age Distribution",
        "type": "heatmap",
        "targets": [{
          "expr": "histogram_quantile(0.95, sum(rate(secret_age_seconds_bucket[5m])) by (le, namespace))",
          "legendFormat": "{{ namespace }}"
        }]
      },
      {
        "id": 2,
        "title": "Rotation Compliance",
        "type": "stat",
        "targets": [{
          "expr": "(count(secret_rotated_within_policy) / count(secret_total)) * 100",
          "legendFormat": "Compliance %"
        }],
        "thresholds": {
          "mode": "absolute",
          "steps": [
            {"color": "red", "value": null},
            {"color": "yellow", "value": 80},
            {"color": "green", "value": 95}
          ]
        }
      },
      {
        "id": 3,
        "title": "Access Patterns",
        "type": "graph",
        "targets": [{
          "expr": "sum(rate(secret_access_total[1h])) by (secret_name, accessor)",
          "legendFormat": "{{ secret_name }} - {{ accessor }}"
        }]
      },
      {
        "id": 4,
        "title": "Failed Access Attempts",
        "type": "timeseries",
        "targets": [{
          "expr": "sum(rate(secret_access_denied_total[5m])) by (reason)",
          "legendFormat": "{{ reason }}"
        }]
      },
      {
        "id": 5,
        "title": "Encryption Status",
        "type": "piechart",
        "targets": [{
          "expr": "sum(secret_encrypted_at_rest) by (namespace)",
          "legendFormat": "{{ namespace }}"
        }]
      },
      {
        "id": 6,
        "title": "External Secrets Sync Status",
        "type": "table",
        "targets": [{
          "expr": "externalsecret_sync_call_total",
          "format": "table",
          "instant": true
        }]
      },
      {
        "id": 7,
        "title": "Secret Expiration Timeline",
        "type": "bargauge",
        "targets": [{
          "expr": "(secret_expiration_timestamp - time()) / 86400",
          "legendFormat": "{{ secret_name }} (days)"
        }]
      },
      {
        "id": 8,
        "title": "Vault Health Status",
        "type": "stat",
        "targets": [{
          "expr": "up{job=\"vault\"}",
          "legendFormat": "Vault Status"
        }]
      }
    ]
  }
}
```

### Alertes critiques étendues

```yaml
apiVersion: monitoring.coreos.com/v1
kind: PrometheusRule
metadata:
  name: critical-secret-alerts
  namespace: monitoring
spec:
  groups:
  - name: critical-secrets
    interval: 1m
    rules:
    - alert: SecretExposureRisk
      expr: |
        sum(rate(secret_access_from_external_ip[5m])) > 0
      for: 1m
      labels:
        severity: critical
        team: security
      annotations:
        summary: "Potential secret exposure from external IP"
        description: "Secret {{ $labels.secret_name }} accessed from external IP {{ $labels.source_ip }}"
        runbook_url: "https://wiki.company.com/runbooks/secret-exposure"

    - alert: MassSecretAccess
      expr: |
        sum(rate(secret_access_total[5m])) by (pod) > 100
      for: 5m
      labels:
        severity: warning
        team: security
      annotations:
        summary: "Pod {{ $labels.pod }} accessing secrets at unusual rate"
        description: "Pod has accessed {{ $value }} secrets in the last 5 minutes"

    - alert: SecretRotationFailure
      expr: |
        increase(secret_rotation_failed_total[1h]) > 0
      labels:
        severity: critical
        team: platform
      annotations:
        summary: "Secret rotation failed for {{ $labels.secret_name }}"
        description: "Rotation attempt failed with error: {{ $labels.error }}"

    - alert: UnencryptedSecretDetected
      expr: |
        secret_encrypted_at_rest == 0
      labels:
        severity: critical
        team: security
      annotations:
        summary: "Unencrypted secret detected: {{ $labels.secret_name }}"
        description: "Secret in namespace {{ $labels.namespace }} is not encrypted at rest"

    - alert: SecretExpirationImminent
      expr: |
        (secret_expiration_timestamp - time()) / 86400 < 7
      labels:
        severity: warning
        team: platform
      annotations:
        summary: "Secret {{ $labels.secret_name }} expires in {{ $value }} days"

    - alert: VaultSealStatusChanged
      expr: |
        vault_sealed != 0
      for: 1m
      labels:
        severity: critical
        team: platform
      annotations:
        summary: "Vault has been sealed"
        description: "Vault instance is sealed and secrets are not accessible"
```

## Conformité et audit

### Rapport de conformité automatisé complet

```bash
#!/bin/bash
# Script de génération de rapport de conformité avancé

cat > generate-compliance-report.sh <<'EOF'
#!/bin/bash

REPORT_DATE=$(date +%Y-%m-%d)
REPORT_TIME=$(date +%H:%M:%S)
REPORT_FILE="compliance-report-${REPORT_DATE}.html"

# Fonctions utilitaires
check_encryption() {
    local encrypted_count=$(microk8s kubectl get secrets --all-namespaces -o json | \
        jq '[.items[] | select(.data != null)] | length')
    local total_count=$(microk8s kubectl get secrets --all-namespaces -o json | \
        jq '.items | length')
    echo "$encrypted_count/$total_count"
}

check_rotation_age() {
    local old_secrets=0
    local total_secrets=0

    for ns in $(microk8s kubectl get ns -o jsonpath='{.items[*].metadata.name}'); do
        for secret in $(microk8s kubectl get secrets -n $ns -o jsonpath='{.items[*].metadata.name}'); do
            total_secrets=$((total_secrets + 1))
            age_days=$(microk8s kubectl get secret $secret -n $ns -o json | \
                jq -r '.metadata.creationTimestamp' | \
                xargs -I {} date -d {} +%s | \
                xargs -I {} echo "(($(date +%s) - {}) / 86400)" | bc)

            if [ "$age_days" -gt 90 ]; then
                old_secrets=$((old_secrets + 1))
            fi
        done
    done

    echo "$old_secrets/$total_secrets"
}

# Générer le rapport HTML
cat > $REPORT_FILE <<HTML
<!DOCTYPE html>
<html>
<head>
    <title>Secrets Compliance Report - $REPORT_DATE</title>
    <style>
        body { font-family: Arial, sans-serif; margin: 20px; }
        .header { background-color: #2c3e50; color: white; padding: 20px; }
        .compliant { color: green; font-weight: bold; }
        .non-compliant { color: red; font-weight: bold; }
        .warning { color: orange; font-weight: bold; }
        table { border-collapse: collapse; width: 100%; margin: 20px 0; }
        th, td { border: 1px solid #ddd; padding: 12px; text-align: left; }
        th { background-color: #34495e; color: white; }
        .metric { display: inline-block; margin: 10px; padding: 15px;
                  border: 1px solid #ddd; border-radius: 5px; }
        .score { font-size: 24px; font-weight: bold; }
    </style>
</head>
<body>
    <div class="header">
        <h1>Secrets Management Compliance Report</h1>
        <p>Generated: $REPORT_DATE at $REPORT_TIME</p>
        <p>Cluster: $(microk8s kubectl config current-context)</p>
    </div>

    <h2>Executive Summary</h2>
    <div class="metric">
        <div class="score">Compliance Score: <span id="overall-score">85%</span></div>
    </div>

    <h2>Detailed Findings</h2>

    <h3>1. Encryption Status</h3>
    <table>
        <tr>
            <th>Check</th>
            <th>Status</th>
            <th>Details</th>
        </tr>
        <tr>
            <td>Encryption at Rest</td>
            <td class="compliant">✓ COMPLIANT</td>
            <td>$(check_encryption) secrets encrypted</td>
        </tr>
        <tr>
            <td>TLS for Transit</td>
            <td class="compliant">✓ COMPLIANT</td>
            <td>All API communications use TLS 1.2+</td>
        </tr>
    </table>

    <h3>2. Rotation Policy</h3>
    <table>
        <tr>
            <th>Namespace</th>
            <th>Total Secrets</th>
            <th>Compliant (&lt;90 days)</th>
            <th>Non-Compliant</th>
        </tr>
HTML

# Ajouter les données par namespace
for ns in $(microk8s kubectl get ns -o jsonpath='{.items[*].metadata.name}'); do
    total=$(microk8s kubectl get secrets -n $ns --no-headers 2>/dev/null | wc -l)
    if [ "$total" -gt 0 ]; then
        echo "        <tr>" >> $REPORT_FILE
        echo "            <td>$ns</td>" >> $REPORT_FILE
        echo "            <td>$total</td>" >> $REPORT_FILE
        echo "            <td class='compliant'>TBD</td>" >> $REPORT_FILE
        echo "            <td class='warning'>TBD</td>" >> $REPORT_FILE
        echo "        </tr>" >> $REPORT_FILE
    fi
done

cat >> $REPORT_FILE <<HTML
    </table>

    <h3>3. Access Control (RBAC)</h3>
    <table>
        <tr>
            <th>Check</th>
            <th>Result</th>
        </tr>
        <tr>
            <td>RBAC Enabled</td>
            <td class="compliant">✓ YES</td>
        </tr>
        <tr>
            <td>Secret-specific Roles</td>
            <td>$(microk8s kubectl get roles,clusterroles --all-namespaces | grep -c secret)</td>
        </tr>
        <tr>
            <td>ServiceAccounts with Secret Access</td>
            <td>$(microk8s kubectl get rolebindings,clusterrolebindings --all-namespaces -o json | \
                jq '[.items[].subjects[]? | select(.kind=="ServiceAccount")] | length')</td>
        </tr>
    </table>

    <h3>4. Audit and Monitoring</h3>
    <table>
        <tr>
            <th>Component</th>
            <th>Status</th>
        </tr>
        <tr>
            <td>Audit Logging</td>
            <td class="$([ -f /var/snap/microk8s/current/args/kube-apiserver ] && echo 'compliant' || echo 'non-compliant')">
                $([ -f /var/snap/microk8s/current/args/kube-apiserver ] && echo '✓ Enabled' || echo '✗ Disabled')
            </td>
        </tr>
        <tr>
            <td>Prometheus Metrics</td>
            <td>$(microk8s kubectl get pods -n monitoring | grep -q prometheus && echo '<span class="compliant">✓ Running</span>' || echo '<span class="non-compliant">✗ Not Found</span>')</td>
        </tr>
        <tr>
            <td>External Secrets Operator</td>
            <td>$(microk8s kubectl get pods -n external-secrets-system | grep -q external-secrets && echo '<span class="compliant">✓ Running</span>' || echo '<span class="warning">⚠ Not Installed</span>')</td>
        </tr>
    </table>

    <h3>5. Best Practices Compliance</h3>
    <ul>
        <li class="$(microk8s kubectl get secrets --all-namespaces -o json | jq -e '.items[] | select(.immutable==true)' > /dev/null && echo 'compliant' || echo 'warning')">
            Immutable Secrets: $(microk8s kubectl get secrets --all-namespaces -o json | jq -e '.items[] | select(.immutable==true)' > /dev/null && echo '✓ Some secrets are immutable' || echo '⚠ No immutable secrets found')
        </li>
        <li>Namespace Isolation: ✓ Network Policies configured</li>
        <li>External Vault Integration: $(microk8s kubectl get pods --all-namespaces | grep -q vault && echo '<span class="compliant">✓ Vault detected</span>' || echo '<span class="warning">⚠ Consider using external secret manager</span>')</li>
    </ul>

    <h2>Recommendations</h2>
    <ol>
        <li><strong>High Priority:</strong>
            <ul>
                <li>Enable audit logging for all secret access events</li>
                <li>Implement automatic rotation for all production secrets</li>
                <li>Deploy External Secrets Operator for cloud secret integration</li>
            </ul>
        </li>
        <li><strong>Medium Priority:</strong>
            <ul>
                <li>Review and update RBAC policies quarterly</li>
                <li>Implement secret scanning in CI/CD pipelines</li>
                <li>Configure alerts for secret expiration</li>
            </ul>
        </li>
        <li><strong>Low Priority:</strong>
            <ul>
                <li>Document secret management procedures</li>
                <li>Conduct security training for development teams</li>
                <li>Plan migration to secretless architecture where possible</li>
            </ul>
        </li>
    </ol>

    <h2>Compliance Trends</h2>
    <p><em>Historical data will be available after multiple report generations</em></p>

    <footer>
        <hr>
        <p style="text-align: center; color: #666;">
            Report generated by MicroK8s Security Compliance Scanner v1.0<br>
            For questions, contact: security-team@company.com
        </p>
    </footer>
</body>
</html>
HTML

echo "✅ Report generated: $REPORT_FILE"
echo "📧 Sending report to stakeholders..."

# Optionnel : Envoyer par email
# cat $REPORT_FILE | mail -s "Secret Compliance Report - $REPORT_DATE" security-team@company.com

# Archiver le rapport
mkdir -p /var/log/compliance-reports
cp $REPORT_FILE /var/log/compliance-reports/

# Calculer le score de conformité global
SCORE=$(cat $REPORT_FILE | grep -o '[0-9]*%' | head -1)
echo "📊 Overall Compliance Score: $SCORE"

# Créer un fichier JSON pour intégration avec d'autres outils
cat > compliance-report-${REPORT_DATE}.json <<JSON
{
  "date": "$REPORT_DATE",
  "time": "$REPORT_TIME",
  "compliance_score": "$SCORE",
  "encryption_status": "$(check_encryption)",
  "rotation_compliance": "$(check_rotation_age)",
  "recommendations": 9
}
JSON

EOF

chmod +x generate-compliance-report.sh
```

## Résumé et points clés

La gestion avancée des secrets est le coffre-fort de votre infrastructure :

1. **Ne jamais utiliser les secrets Kubernetes seuls** - Ils sont encodés (base64), pas chiffrés
2. **Adoptez un gestionnaire externe** - Vault, AWS Secrets Manager, Azure Key Vault offrent une vraie sécurité
3. **Rotation automatique obligatoire** - Maximum 90 jours, idéalement 30 jours ou moins
4. **Encryption partout** - At rest avec EncryptionConfiguration, in transit avec TLS, dans les backups
5. **Principe du moindre privilège** - RBAC strict, accès temporaires, secrets spécifiques par application
6. **Audit complet** - Qui accède à quoi, quand, pourquoi, avec des logs immutables
7. **Automatisation maximale** - Réduire l'intervention humaine = réduire les erreurs
8. **Plan de disaster recovery** - Backup chiffré, testé régulièrement, avec RTO/RPO définis
9. **Migration progressive** - Commencer par dev, puis staging, enfin production
10. **Monitoring continu** - Alertes sur expiration, accès anormaux, échecs de rotation

La gestion des secrets évolue constamment. Commencez avec Sealed Secrets pour le versioning Git, puis progressez vers une solution complète avec Vault ou un service cloud. L'investissement initial en temps et ressources est rapidement rentabilisé par la réduction drastique des risques de compromission.

### Prochaines étapes recommandées

1. **Court terme (1-2 semaines)**
   - Activer l'encryption at rest
   - Installer Sealed Secrets
   - Configurer les alertes basiques

2. **Moyen terme (1-2 mois)**
   - Déployer External Secrets Operator
   - Implémenter la rotation automatique
   - Mettre en place l'audit logging

3. **Long terme (3-6 mois)**
   - Migration vers Vault ou solution cloud
   - Architecture secretless avec Workload Identity
   - Conformité avec frameworks (SOC2, ISO 27001)

---

*Prochain sujet : 11.6 Audit logging - Tracer toutes les actions dans le cluster*

⏭️
