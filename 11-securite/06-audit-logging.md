🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 11.6 Audit logging

## Comprendre l'audit logging

Imaginez que vous gérez une banque. Chaque transaction, chaque ouverture de coffre, chaque consultation de compte doit être enregistrée : qui a fait quoi, quand, et pourquoi. L'audit logging dans Kubernetes fonctionne exactement de la même manière. Il enregistre toutes les actions effectuées sur votre cluster, créant une piste d'audit complète qui vous permet de savoir précisément ce qui s'est passé, même des mois après les faits.

### Pourquoi l'audit logging est indispensable

Sans audit logging, vous êtes aveugle face aux activités de votre cluster :
- **Investigations impossibles** : Qui a supprimé ce pod critique à 3h du matin ?
- **Conformité non respectée** : Impossibilité de prouver qui a accédé aux données sensibles
- **Détection tardive** : Les intrusions passent inaperçues pendant des semaines
- **Apprentissage limité** : Aucune visibilité sur les erreurs de configuration récurrentes
- **Responsabilité floue** : Impossible de déterminer qui a fait quelle modification

## Architecture de l'audit logging

### Le flux d'audit dans Kubernetes

```
Requête API → API Server → Audit Backend → Stockage/Analyse
                   ↓
            Évaluation Policy
                   ↓
            Décision: Logger?
                   ↓
              Enrichissement
                   ↓
                 Output
```

### Les composants clés

**1. API Server**
- Point d'entrée de toutes les requêtes
- Applique les politiques d'audit
- Enrichit les événements avec des métadonnées

**2. Audit Policy**
- Définit quoi logger
- Détermine le niveau de détail
- Configure les exceptions

**3. Audit Backend**
- Log backend : Fichiers locaux
- Webhook backend : Envoi vers système externe
- Dynamic backend : Configuration à chaud

**4. Storage & Analysis**
- Stockage des logs (fichiers, base de données)
- Outils d'analyse (ELK, Splunk, etc.)
- Systèmes d'alerte

## Configuration de l'audit dans MicroK8s

### Activation de l'audit logging

```bash
# Vérifier le statut actuel
microk8s status

# Créer le répertoire pour les logs d'audit
sudo mkdir -p /var/log/microk8s/audit

# Créer la politique d'audit
sudo nano /var/snap/microk8s/current/args/audit-policy.yaml
```

### Politique d'audit de base

```yaml
# audit-policy.yaml - Politique simple pour commencer
apiVersion: audit.k8s.io/v1
kind: Policy
# Ne pas logger les requêtes en lecture du système
omitStages:
  - RequestReceived
rules:
  # Ne pas logger les requêtes de santé et discovery
  - level: None
    users: ["system:kube-proxy"]
    verbs: ["watch"]
    resources:
      - group: ""
        resources: ["endpoints", "services"]

  - level: None
    users: ["system:unsecured"]
    namespaces: ["kube-system"]
    verbs: ["get"]
    resources:
      - group: ""
        resources: ["configmaps"]

  # Ne pas logger les événements système répétitifs
  - level: None
    users:
      - "system:kube-controller-manager"
      - "system:kube-scheduler"
      - "system:serviceaccount:kube-system:endpoints-controller"
    verbs: ["get", "update"]
    namespaces: ["kube-system"]

  # Logger les secrets avec métadonnées uniquement (pas le contenu)
  - level: Metadata
    resources:
      - group: ""
        resources: ["secrets", "configmaps"]

  # Logger les créations/suppressions de pods avec détails
  - level: RequestResponse
    verbs: ["create", "delete", "patch"]
    resources:
      - group: ""
        resources: ["pods", "deployments", "services"]

  # Logger tout le reste au niveau Metadata
  - level: Metadata
```

### Configuration de l'API Server

```bash
# Éditer la configuration de l'API server
sudo nano /var/snap/microk8s/current/args/kube-apiserver

# Ajouter ces lignes :
--audit-policy-file=/var/snap/microk8s/current/args/audit-policy.yaml
--audit-log-path=/var/log/microk8s/audit/audit.log
--audit-log-maxage=30
--audit-log-maxbackup=10
--audit-log-maxsize=100

# Redémarrer MicroK8s
microk8s stop
microk8s start

# Vérifier que l'audit fonctionne
tail -f /var/log/microk8s/audit/audit.log
```

## Niveaux d'audit

### Les quatre niveaux de détail

**1. None**
- Aucun log pour cette requête
- Utilisé pour filtrer le bruit

**2. Metadata**
- Enregistre les métadonnées de la requête
- Qui, quoi, quand, où
- Pas le contenu de la requête/réponse

**3. Request**
- Métadonnées + corps de la requête
- Utile pour voir ce qui a été demandé
- Pas la réponse

**4. RequestResponse**
- Tout : métadonnées, requête et réponse
- Maximum de détails
- Attention à la taille des logs

### Exemples par niveau

```yaml
# Niveau None - Rien n'est loggé
- level: None
  users: ["system:kube-proxy"]
  verbs: ["watch"]

# Niveau Metadata - Juste les infos essentielles
- level: Metadata
  verbs: ["get", "list"]
  resources:
    - group: ""
      resources: ["pods"]

# Niveau Request - Métadonnées + requête
- level: Request
  verbs: ["create", "update"]
  resources:
    - group: "apps"
      resources: ["deployments"]

# Niveau RequestResponse - Tout
- level: RequestResponse
  verbs: ["delete"]
  resources:
    - group: ""
      resources: ["secrets"]
```

## Politique d'audit avancée

### Politique complète pour production

```yaml
apiVersion: audit.k8s.io/v1
kind: Policy
omitStages:
  - RequestReceived  # Éviter les doublons
rules:
  # === EXCLUSIONS SYSTÈME ===
  # Ignorer les health checks
  - level: None
    nonResourceURLs:
      - /healthz*
      - /livez*
      - /readyz*
      - /version
      - /swagger*

  # Ignorer les requêtes système bruyantes
  - level: None
    users:
      - "system:kube-controller-manager"
      - "system:kube-scheduler"
      - "system:apiserver"
    verbs: ["get", "list", "watch"]

  # Ignorer les ServiceAccounts système dans kube-system
  - level: None
    userGroups: ["system:serviceaccounts:kube-system"]
    namespaces: ["kube-system"]
    verbs: ["get", "list", "watch"]

  # === HAUTE PRIORITÉ ===
  # Logger TOUT sur les secrets (sauf le contenu)
  - level: Metadata
    resources:
      - group: ""
        resources: ["secrets"]
    omitStages:
      - RequestReceived

  # Logger les actions d'authentification/autorisation
  - level: RequestResponse
    resources:
      - group: "authentication.k8s.io"
      - group: "authorization.k8s.io"
      - group: "rbac.authorization.k8s.io"

  # Logger les suppressions avec détails complets
  - level: RequestResponse
    verbs: ["delete", "deletecollection"]

  # === SÉCURITÉ ===
  # Logger les modifications RBAC
  - level: RequestResponse
    resources:
      - group: "rbac.authorization.k8s.io"
        resources:
          - clusterrolebindings
          - clusterroles
          - rolebindings
          - roles

  # Logger les NetworkPolicies
  - level: RequestResponse
    resources:
      - group: "networking.k8s.io"
        resources: ["networkpolicies"]

  # Logger les PodSecurityPolicies
  - level: RequestResponse
    resources:
      - group: "policy"
        resources: ["podsecuritypolicies"]

  # === NAMESPACES CRITIQUES ===
  # Tout logger dans les namespaces sensibles
  - level: RequestResponse
    namespaces: ["production", "kube-system", "security"]
    verbs: ["create", "update", "patch", "delete"]

  # === UTILISATEURS SPÉCIFIQUES ===
  # Logger toutes les actions des admins
  - level: RequestResponse
    users:
      - "admin"
      - "root"
    userGroups:
      - "system:masters"

  # Logger les actions depuis des IPs externes
  - level: RequestResponse
    omitStages: []
    objectRef:
      apiVersion: "*"
    userGroups: ["system:unauthenticated"]

  # === RESSOURCES SENSIBLES ===
  # Logger les modifications de configuration
  - level: Request
    verbs: ["create", "update", "patch"]
    resources:
      - group: ""
        resources: ["configmaps"]
      - group: "apps"
        resources: ["deployments", "daemonsets", "statefulsets"]

  # === DÉFAUT ===
  # Logger tout le reste au niveau Metadata
  - level: Metadata
    omitStages:
      - RequestReceived
```

### Politique par environnement

```yaml
# Développement - Minimal
apiVersion: audit.k8s.io/v1
kind: Policy
rules:
  - level: None
    users: ["system:*"]
  - level: Metadata
    verbs: ["delete"]
  - level: None

---
# Staging - Modéré
apiVersion: audit.k8s.io/v1
kind: Policy
rules:
  - level: None
    nonResourceURLs: ["/healthz*", "/version"]
  - level: Metadata
    resources:
      - group: ""
        resources: ["secrets", "configmaps"]
  - level: Request
    verbs: ["create", "update", "delete"]
  - level: Metadata

---
# Production - Complet
apiVersion: audit.k8s.io/v1
kind: Policy
# [Politique complète ci-dessus]
```

## Formats et structure des logs

### Structure d'un événement d'audit

```json
{
  "kind": "Event",
  "apiVersion": "audit.k8s.io/v1",
  "level": "RequestResponse",
  "auditID": "362c5e6f-857a-4187-9c7d-7b609ff6dbdb",
  "stage": "ResponseComplete",
  "requestURI": "/api/v1/namespaces/default/pods/nginx",
  "verb": "delete",
  "user": {
    "username": "john.doe",
    "uid": "john-uid-123",
    "groups": ["developers", "system:authenticated"]
  },
  "sourceIPs": ["192.168.1.100"],
  "userAgent": "kubectl/v1.28.0 (linux/amd64) kubernetes/abc123",
  "objectRef": {
    "resource": "pods",
    "namespace": "default",
    "name": "nginx",
    "apiVersion": "v1"
  },
  "responseStatus": {
    "code": 200,
    "status": "Success"
  },
  "requestReceivedTimestamp": "2024-01-15T10:30:00.000Z",
  "stageTimestamp": "2024-01-15T10:30:00.500Z",
  "annotations": {
    "authorization.k8s.io/decision": "allow",
    "authorization.k8s.io/reason": "RBAC: allowed by RoleBinding"
  }
}
```

### Champs importants

**Identification**
- `auditID` : Identifiant unique de l'événement
- `stage` : Étape du traitement (RequestReceived, ResponseStarted, ResponseComplete, Panic)

**Qui**
- `user.username` : Identité de l'utilisateur
- `user.groups` : Groupes de l'utilisateur
- `sourceIPs` : Adresse IP source

**Quoi**
- `verb` : Action effectuée
- `objectRef` : Ressource ciblée
- `requestURI` : Endpoint API appelé

**Quand**
- `requestReceivedTimestamp` : Heure de réception
- `stageTimestamp` : Heure de l'étape

**Résultat**
- `responseStatus` : Code et statut de la réponse
- `responseObject` : Objet retourné (si level=RequestResponse)

## Backends d'audit

### Backend fichier (par défaut)

```bash
# Configuration dans kube-apiserver
--audit-log-path=/var/log/kubernetes/audit.log
--audit-log-maxage=30    # Jours de rétention
--audit-log-maxbackup=10  # Nombre de fichiers de backup
--audit-log-maxsize=100   # Taille max en MB par fichier

# Rotation automatique avec logrotate
cat > /etc/logrotate.d/k8s-audit <<EOF
/var/log/kubernetes/audit.log {
    daily
    rotate 30
    compress
    delaycompress
    missingok
    notifempty
    create 0600 root root
}
EOF
```

### Backend webhook

```yaml
# webhook-config.yaml
apiVersion: v1
kind: Config
clusters:
- name: audit-webhook
  cluster:
    server: https://audit-collector.monitoring.svc:9880/audit
    certificate-authority-data: <BASE64_CA_CERT>
contexts:
- name: audit-webhook
  context:
    cluster: audit-webhook
current-context: audit-webhook
```

```bash
# Configuration dans kube-apiserver
--audit-webhook-config-file=/etc/kubernetes/audit-webhook.yaml
--audit-webhook-initial-backoff=10s
--audit-webhook-batch-max-size=400
--audit-webhook-batch-max-wait=30s
```

### Backend dynamique

```yaml
# AuditSink pour configuration dynamique
apiVersion: auditregistration.k8s.io/v1alpha1
kind: AuditSink
metadata:
  name: elastic-audit-sink
spec:
  policy:
    level: Metadata
    stages:
    - ResponseComplete
  webhook:
    clientConfig:
      url: "https://elasticsearch.logging:9200/_doc"
      caBundle: <BASE64_ENCODED_CA>
    throttle:
      qps: 10
      burst: 15
```

## Intégration avec ELK Stack

### Déploiement d'Elasticsearch

```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: elasticsearch
  namespace: logging
spec:
  serviceName: elasticsearch
  replicas: 1
  selector:
    matchLabels:
      app: elasticsearch
  template:
    metadata:
      labels:
        app: elasticsearch
    spec:
      containers:
      - name: elasticsearch
        image: docker.elastic.co/elasticsearch/elasticsearch:8.11.0
        env:
        - name: discovery.type
          value: single-node
        - name: ES_JAVA_OPTS
          value: "-Xms512m -Xmx512m"
        - name: xpack.security.enabled
          value: "false"
        ports:
        - containerPort: 9200
        - containerPort: 9300
        volumeMounts:
        - name: data
          mountPath: /usr/share/elasticsearch/data
  volumeClaimTemplates:
  - metadata:
      name: data
    spec:
      accessModes: ["ReadWriteOnce"]
      resources:
        requests:
          storage: 10Gi
```

### Configuration de Filebeat

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: filebeat-config
  namespace: logging
data:
  filebeat.yml: |
    filebeat.inputs:
    - type: log
      enabled: true
      paths:
        - /var/log/kubernetes/audit.log
      json.keys_under_root: true
      json.add_error_key: true
      multiline.pattern: '^{'
      multiline.negate: true
      multiline.match: after

    processors:
      - add_kubernetes_metadata:
          host: ${NODE_NAME}
          matchers:
          - logs_path:
              logs_path: "/var/log/kubernetes/"
      - decode_json_fields:
          fields: ["message"]
          target: "audit"
          overwrite_keys: true

    output.elasticsearch:
      hosts: ['${ELASTICSEARCH_HOST:elasticsearch}:${ELASTICSEARCH_PORT:9200}']
      index: "k8s-audit-%{+yyyy.MM.dd}"

    setup.template.name: "k8s-audit"
    setup.template.pattern: "k8s-audit-*"
---
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: filebeat
  namespace: logging
spec:
  selector:
    matchLabels:
      app: filebeat
  template:
    metadata:
      labels:
        app: filebeat
    spec:
      serviceAccountName: filebeat
      containers:
      - name: filebeat
        image: docker.elastic.co/beats/filebeat:8.11.0
        args: [
          "-c", "/etc/filebeat.yml",
          "-e",
        ]
        env:
        - name: ELASTICSEARCH_HOST
          value: elasticsearch
        - name: NODE_NAME
          valueFrom:
            fieldRef:
              fieldPath: spec.nodeName
        volumeMounts:
        - name: config
          mountPath: /etc/filebeat.yml
          subPath: filebeat.yml
        - name: audit-logs
          mountPath: /var/log/kubernetes
          readOnly: true
      volumes:
      - name: config
        configMap:
          name: filebeat-config
      - name: audit-logs
        hostPath:
          path: /var/log/microk8s/audit
```

### Dashboard Kibana

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: kibana
  namespace: logging
spec:
  replicas: 1
  selector:
    matchLabels:
      app: kibana
  template:
    metadata:
      labels:
        app: kibana
    spec:
      containers:
      - name: kibana
        image: docker.elastic.co/kibana/kibana:8.11.0
        env:
        - name: ELASTICSEARCH_HOSTS
          value: "http://elasticsearch:9200"
        - name: SERVER_BASEPATH
          value: "/kibana"
        ports:
        - containerPort: 5601
---
apiVersion: v1
kind: Service
metadata:
  name: kibana
  namespace: logging
spec:
  selector:
    app: kibana
  ports:
  - port: 5601
    targetPort: 5601
```

## Requêtes et analyses utiles

### Requêtes Elasticsearch courantes

```json
// Toutes les suppressions dans les dernières 24h
{
  "query": {
    "bool": {
      "must": [
        {"term": {"verb": "delete"}},
        {"range": {"requestReceivedTimestamp": {"gte": "now-24h"}}}
      ]
    }
  }
}

// Actions par un utilisateur spécifique
{
  "query": {
    "term": {"user.username": "john.doe"}
  },
  "sort": [{"requestReceivedTimestamp": "desc"}]
}

// Échecs d'autorisation
{
  "query": {
    "bool": {
      "must": [
        {"term": {"responseStatus.code": 403}},
        {"exists": {"field": "annotations.authorization.k8s.io/reason"}}
      ]
    }
  }
}

// Modifications de secrets
{
  "query": {
    "bool": {
      "must": [
        {"term": {"objectRef.resource": "secrets"}},
        {"terms": {"verb": ["create", "update", "patch", "delete"]}}
      ]
    }
  }
}
```

### Scripts d'analyse

```bash
#!/bin/bash
# analyze-audit-logs.sh - Analyse rapide des logs d'audit

AUDIT_LOG="/var/log/microk8s/audit/audit.log"

echo "=== Analyse des logs d'audit ==="
echo

echo "Top 10 utilisateurs actifs:"
cat $AUDIT_LOG | jq -r '.user.username' | sort | uniq -c | sort -rn | head -10
echo

echo "Verbes les plus utilisés:"
cat $AUDIT_LOG | jq -r '.verb' | sort | uniq -c | sort -rn
echo

echo "Ressources les plus accédées:"
cat $AUDIT_LOG | jq -r '.objectRef.resource' | sort | uniq -c | sort -rn | head -10
echo

echo "Échecs d'autorisation (403):"
cat $AUDIT_LOG | jq 'select(.responseStatus.code == 403) | {user: .user.username, resource: .objectRef.resource, verb: .verb}'
echo

echo "Suppressions récentes:"
cat $AUDIT_LOG | jq 'select(.verb == "delete") | {time: .requestReceivedTimestamp, user: .user.username, resource: .objectRef.resource, name: .objectRef.name}'
```

## Alerting sur les événements d'audit

### Règles Prometheus AlertManager

```yaml
apiVersion: monitoring.coreos.com/v1
kind: PrometheusRule
metadata:
  name: audit-alerts
  namespace: monitoring
spec:
  groups:
  - name: audit
    interval: 1m
    rules:
    - alert: UnauthorizedAccessAttempt
      expr: |
        sum(rate(apiserver_audit_event_total{response_code="403"}[5m])) > 10
      for: 2m
      labels:
        severity: warning
      annotations:
        summary: "Multiple unauthorized access attempts detected"
        description: "{{ $value }} forbidden responses in the last 5 minutes"

    - alert: SecretDeleted
      expr: |
        apiserver_audit_event_total{verb="delete",objectRef_resource="secrets"} > 0
      labels:
        severity: critical
      annotations:
        summary: "Secret deleted"
        description: "Secret {{ $labels.objectRef_name }} deleted by {{ $labels.user_username }}"

    - alert: RBACModification
      expr: |
        sum(rate(apiserver_audit_event_total{
          objectRef_resource=~"roles|rolebindings|clusterroles|clusterrolebindings",
          verb=~"create|update|patch|delete"
        }[5m])) > 0
      labels:
        severity: warning
      annotations:
        summary: "RBAC configuration modified"
        description: "RBAC changes detected"

    - alert: ExternalAccess
      expr: |
        apiserver_audit_event_total{source_ips!~"10\\..*|172\\.(1[6-9]|2[0-9]|3[01])\\..*|192\\.168\\..*"} > 0
      labels:
        severity: high
      annotations:
        summary: "API access from external IP"
        description: "Access from {{ $labels.source_ips }}"
```

### Intégration avec Falco

```yaml
# Règles Falco pour événements d'audit
apiVersion: v1
kind: ConfigMap
metadata:
  name: falco-audit-rules
  namespace: security
data:
  audit-rules.yaml: |
    - rule: Unauthorized API Access
      desc: Detect unauthorized API access attempts
      condition: ka.response_code >= 400 and ka.response_code < 500
      output: >
        Unauthorized API access attempt
        (user=%ka.user.name verb=%ka.verb resource=%ka.target.resource response=%ka.response_code)
      priority: WARNING
      source: k8s_audit
      tags: [k8s, api, security]

    - rule: Sensitive Resource Access
      desc: Detect access to sensitive resources
      condition: >
        ka.target.resource in (secrets, configmaps) and
        ka.verb in (get, list, watch)
      output: >
        Sensitive resource accessed
        (user=%ka.user.name verb=%ka.verb resource=%ka.target.resource name=%ka.target.name)
      priority: INFO
      source: k8s_audit

    - rule: Privilege Escalation Attempt
      desc: Detect potential privilege escalation
      condition: >
        ka.verb in (create, update, patch) and
        ka.target.resource in (clusterrolebindings, rolebindings) and
        not ka.user.name in (system:*, kube-*)
      output: >
        Potential privilege escalation
        (user=%ka.user.name verb=%ka.verb resource=%ka.target.resource)
      priority: CRITICAL
      source: k8s_audit
```

## Conformité et réglementation

### Exigences de conformité

```yaml
# Configuration pour différents standards
compliance_requirements:
  PCI_DSS:
    retention_days: 365
    must_log:
      - user_access
      - privilege_changes
      - system_changes
      - failed_access
    encryption: required

  GDPR:
    retention_days: 90  # Sauf obligation légale
    must_log:
      - data_access
      - data_modification
      - data_deletion
    anonymization: required_after_30_days

  SOC2:
    retention_days: 180
    must_log:
      - authentication
      - authorization
      - configuration_changes
    integrity_verification: required

  HIPAA:
    retention_days: 2190  # 6 ans
    must_log:
      - phi_access
      - user_activity
      - system_changes
    encryption: required
    access_control: strict
```

### Script de vérification de conformité

```bash
#!/bin/bash
# compliance-check.sh

COMPLIANCE_STANDARD=${1:-PCI_DSS}
AUDIT_CONFIG="/var/snap/microk8s/current/args/kube-apiserver"

echo "Checking compliance for: $COMPLIANCE_STANDARD"

case $COMPLIANCE_STANDARD in
  PCI_DSS)
    # Vérifier la rétention
    if grep -q "audit-log-maxage=365" $AUDIT_CONFIG; then
      echo "✓ Retention: 365 days configured"
    else
      echo "✗ Retention: Must be at least 365 days for PCI-DSS"
    fi

    # Vérifier le chiffrement
    if [ -f "/etc/kubernetes/audit-webhook.yaml" ]; then
      echo "✓ Encryption: Webhook configured (assume TLS)"
    else
      echo "⚠ Encryption: Ensure logs are encrypted at rest"
    fi
    ;;

  GDPR)
    # Vérifier l'anonymisation
    echo "⚠ Manual check required: Ensure PII is anonymized after 30 days"

    # Vérifier la politique de suppression
    if grep -q "audit-log-maxage=90" $AUDIT_CONFIG; then
      echo "✓ Retention: 90 days configured"
    else
      echo "✗ Retention: Should be 90 days for GDPR"
    fi
    ;;
esac

# Vérifier que les événements requis sont loggés
echo ""
echo "Checking required event logging..."

# Vérifier la politique d'audit
if [ -f "/var/snap/microk8s/current/args/audit-policy.yaml" ]; then
  echo "✓ Audit policy configured"

  # Vérifier les ressources critiques
  for resource in secrets configmaps roles rolebindings; do
    if grep -q "$resource" /var/snap/microk8s/current/args/audit-policy.yaml; then
      echo "  ✓ $resource logging configured"
    else
      echo "  ✗ $resource logging missing"
    fi
  done
else
  echo "✗ No audit policy found"
fi
```

## Performance et optimisation

### Impact sur les performances

```yaml
# Configuration optimisée pour réduire l'impact
apiVersion: audit.k8s.io/v1
kind: Policy
rules:
  # Utiliser omitManagedFields pour réduire la taille
  - level: Metadata
    omitManagedFields: true

  # Limiter RequestResponse aux actions critiques
  - level: RequestResponse
    verbs: ["delete"]
    resources:
      - group: ""
        resources: ["secrets", "serviceaccounts"]
    omitManagedFields: true

  # Utiliser le batching pour le webhook
  # Dans kube-apiserver:
  # --audit-webhook-batch-max-size=100
  # --audit-webhook-batch-max-wait=5s
```

### Métriques de monitoring

```yaml
# ServiceMonitor pour Prometheus
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: audit-metrics
  namespace: monitoring
spec:
  selector:
    matchLabels:
      component: kube-apiserver
  endpoints:
  - port: https
    interval: 30s
    path: /metrics
    metricRelabelings:
    - sourceLabels: [__name__]
      regex: 'apiserver_audit_.*'
      action: keep
```

### Optimisation du stockage

```bash
#!/bin/bash
# optimize-audit-storage.sh

# Compression des vieux logs
find /var/log/microk8s/audit -name "*.log" -mtime +7 -exec gzip {} \;

# Archivage vers stockage moins cher
find /var/log/microk8s/audit -name "*.gz" -mtime +30 -exec mv {} /archive/audit/ \;

# Nettoyage des logs très anciens (après backup)
find /archive/audit -name "*.gz" -mtime +365 -delete

# Rapport d'utilisation
echo "Espace utilisé par les logs d'audit:"
du -sh /var/log/microk8s/audit
echo "Nombre de fichiers:"
ls -1 /var/log/microk8s/audit | wc -l
```

## Troubleshooting

### Problèmes courants

#### L'audit ne démarre pas

```bash
# 1. Vérifier la syntaxe de la politique
microk8s kubectl apply --dry-run=client -f /var/snap/microk8s/current/args/audit-policy.yaml

# 2. Vérifier les logs de l'API server
journalctl -u snap.microk8s.daemon-apiserver -n 100

# 3. Vérifier les permissions du fichier de log
ls -la /var/log/microk8s/audit/
# Si nécessaire :
sudo mkdir -p /var/log/microk8s/audit
sudo chmod 755 /var/log/microk8s/audit

# 4. Vérifier la configuration de l'API server
cat /var/snap/microk8s/current/args/kube-apiserver | grep audit

# 5. Tester avec une politique minimale
cat > /tmp/test-audit-policy.yaml <<EOF
apiVersion: audit.k8s.io/v1
kind: Policy
rules:
  - level: Metadata
EOF

# Appliquer temporairement
sudo cp /tmp/test-audit-policy.yaml /var/snap/microk8s/current/args/audit-policy.yaml
microk8s stop && microk8s start
```

#### Logs trop volumineux

```bash
# Diagnostic de la taille
du -sh /var/log/microk8s/audit/*
ls -lah /var/log/microk8s/audit/ | head -20

# Identifier les sources de bruit
cat /var/log/microk8s/audit/audit.log | \
  jq -r '.user.username' | sort | uniq -c | sort -rn | head -10

# Ajuster la politique pour réduire le volume
cat >> /var/snap/microk8s/current/args/audit-policy.yaml <<EOF
# Ajouter en début de politique pour filtrer le bruit
rules:
  - level: None
    users: ["system:kube-scheduler", "system:kube-controller-manager"]
    verbs: ["get", "list", "watch"]

  - level: None
    nonResourceURLs: ["/metrics", "/healthz*", "/livez*", "/readyz*"]
EOF

# Augmenter la rotation
sudo sed -i 's/--audit-log-maxsize=100/--audit-log-maxsize=50/' \
  /var/snap/microk8s/current/args/kube-apiserver
```

#### Performance dégradée

```bash
# Mesurer l'impact
# Avant activation de l'audit
kubectl top nodes > before-audit.txt

# Après activation
kubectl top nodes > after-audit.txt

# Comparer
diff before-audit.txt after-audit.txt

# Optimisations possibles
# 1. Utiliser le batching pour webhook
--audit-webhook-batch-max-size=500
--audit-webhook-batch-max-wait=30s
--audit-webhook-batch-buffer-size=10000

# 2. Réduire le niveau de log
# Remplacer RequestResponse par Metadata où possible

# 3. Utiliser omitManagedFields
cat >> audit-policy.yaml <<EOF
rules:
  - level: Metadata
    omitManagedFields: true
EOF

# 4. Désactiver les stages non nécessaires
omitStages:
  - RequestReceived
  - ResponseStarted
```

#### Webhook timeout

```yaml
# webhook-debug.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: webhook-debug
data:
  test.sh: |
    #!/bin/bash
    # Test de connectivité webhook
    WEBHOOK_URL="https://audit-collector.monitoring.svc:9880/audit"

    # Test basique
    curl -v -X POST $WEBHOOK_URL \
      -H "Content-Type: application/json" \
      -d '{"test": "connection"}'

    # Test avec timeout
    timeout 5 curl -X POST $WEBHOOK_URL || echo "Timeout après 5 secondes"

    # Vérifier la résolution DNS
    nslookup audit-collector.monitoring.svc

    # Vérifier la route
    traceroute audit-collector.monitoring.svc
```

```bash
# Augmenter les timeouts
--audit-webhook-initial-backoff=30s
--audit-webhook-batch-throttle-qps=10
--audit-webhook-batch-throttle-burst=15

# Vérifier les métriques de webhook
curl -s http://localhost:10252/metrics | grep audit_webhook
```

### Scripts de diagnostic

```bash
#!/bin/bash
# audit-health-check.sh

echo "=== Audit System Health Check ==="
echo

# 1. Configuration
echo "1. Checking configuration..."
if grep -q "audit-policy-file" /var/snap/microk8s/current/args/kube-apiserver; then
  echo "   ✓ Audit policy configured"
  POLICY_FILE=$(grep -oP 'audit-policy-file=\K[^ ]+' /var/snap/microk8s/current/args/kube-apiserver)
  if [ -f "$POLICY_FILE" ]; then
    echo "   ✓ Policy file exists: $POLICY_FILE"
  else
    echo "   ✗ Policy file not found: $POLICY_FILE"
  fi
else
  echo "   ✗ Audit not configured in API server"
fi

# 2. Logs
echo ""
echo "2. Checking log files..."
LOG_PATH=$(grep -oP 'audit-log-path=\K[^ ]+' /var/snap/microk8s/current/args/kube-apiserver)
if [ -n "$LOG_PATH" ]; then
  if [ -f "$LOG_PATH" ]; then
    echo "   ✓ Log file exists: $LOG_PATH"
    SIZE=$(du -h "$LOG_PATH" | cut -f1)
    echo "   ✓ Current size: $SIZE"
    LINES=$(wc -l < "$LOG_PATH")
    echo "   ✓ Total events: $LINES"
  else
    echo "   ✗ Log file not found: $LOG_PATH"
  fi
else
  echo "   ✗ No log path configured"
fi

# 3. Recent activity
echo ""
echo "3. Recent activity (last 5 minutes)..."
if [ -f "$LOG_PATH" ]; then
  FIVE_MIN_AGO=$(date -d '5 minutes ago' --iso-8601=seconds)
  RECENT=$(grep -c "$FIVE_MIN_AGO" "$LOG_PATH" 2>/dev/null || echo "0")
  echo "   Events in last 5 min: $RECENT"
fi

# 4. API Server health
echo ""
echo "4. API Server status..."
if microk8s kubectl get --raw /healthz > /dev/null 2>&1; then
  echo "   ✓ API Server responding"
else
  echo "   ✗ API Server not responding"
fi

# 5. Disk space
echo ""
echo "5. Disk space..."
df -h $(dirname "$LOG_PATH") | tail -1

# 6. Webhook status (if configured)
echo ""
echo "6. Webhook configuration..."
if grep -q "audit-webhook-config-file" /var/snap/microk8s/current/args/kube-apiserver; then
  echo "   ✓ Webhook configured"
  WEBHOOK_CONFIG=$(grep -oP 'audit-webhook-config-file=\K[^ ]+' /var/snap/microk8s/current/args/kube-apiserver)
  if [ -f "$WEBHOOK_CONFIG" ]; then
    echo "   ✓ Webhook config exists"
    SERVER=$(grep -oP 'server:\s*\K.+' "$WEBHOOK_CONFIG" | head -1)
    echo "   Webhook endpoint: $SERVER"
  fi
else
  echo "   ℹ No webhook configured"
fi

echo ""
echo "=== Health Check Complete ==="
```

## Cas d'usage et exemples pratiques

### Détection d'intrusion

```yaml
# Politique pour détecter les comportements suspects
apiVersion: audit.k8s.io/v1
kind: Policy
rules:
  # Logger toutes les actions des comptes non système
  - level: RequestResponse
    users: ["!system:*"]
    omitStages: ["RequestReceived"]

  # Logger les échecs d'authentification
  - level: Metadata
    responseStatuses:
      - code: 401
      - code: 403

  # Logger les exécutions dans les pods
  - level: RequestResponse
    resources:
      - group: ""
        resources: ["pods/exec", "pods/attach"]

  # Logger les accès aux secrets
  - level: Metadata
    resources:
      - group: ""
        resources: ["secrets"]
    verbs: ["get", "list", "watch"]
```

```bash
#!/bin/bash
# intrusion-detection.sh

# Patterns suspects à rechercher
SUSPICIOUS_PATTERNS=(
  "pods/exec"
  "serviceaccounts/token"
  "401\|403"
  "system:anonymous"
)

AUDIT_LOG="/var/log/microk8s/audit/audit.log"

for pattern in "${SUSPICIOUS_PATTERNS[@]}"; do
  echo "Checking for: $pattern"
  grep "$pattern" $AUDIT_LOG | tail -5
  echo "---"
done

# Détection d'escalade de privilèges
echo "Potential privilege escalation:"
cat $AUDIT_LOG | jq 'select(
  .objectRef.resource == "clusterrolebindings" or
  .objectRef.resource == "rolebindings"
) | {user: .user.username, action: .verb, binding: .objectRef.name}'
```

### Investigation post-incident

```bash
#!/bin/bash
# incident-investigation.sh

INCIDENT_TIME="2024-01-15T10:30:00"
USER_SUSPECT="john.doe"
NAMESPACE="production"

echo "=== Incident Investigation ==="
echo "Time: $INCIDENT_TIME"
echo "User: $USER_SUSPECT"
echo "Namespace: $NAMESPACE"
echo

# Timeline des actions de l'utilisateur
echo "User activity timeline:"
cat /var/log/microk8s/audit/audit.log | \
  jq --arg user "$USER_SUSPECT" \
  'select(.user.username == $user) |
  {time: .requestReceivedTimestamp, verb: .verb, resource: .objectRef.resource, name: .objectRef.name}'

# Actions dans le namespace au moment de l'incident
echo ""
echo "Namespace activity around incident time:"
cat /var/log/microk8s/audit/audit.log | \
  jq --arg ns "$NAMESPACE" --arg time "$INCIDENT_TIME" \
  'select(.objectRef.namespace == $ns and
  (.requestReceivedTimestamp | . >= $time and . <= ($time + "Z" | fromdateiso8601 + 3600 | todateiso8601))) |
  {time: .requestReceivedTimestamp, user: .user.username, action: .verb, resource: .objectRef.resource}'

# Recherche de suppressions
echo ""
echo "Deletions in timeframe:"
cat /var/log/microk8s/audit/audit.log | \
  jq --arg time "$INCIDENT_TIME" \
  'select(.verb == "delete" and
  (.requestReceivedTimestamp | . >= ($time + "Z" | fromdateiso8601 - 3600 | todateiso8601) and
  . <= ($time + "Z" | fromdateiso8601 + 3600 | todateiso8601)))'
```

### Rapport d'activité

```bash
#!/bin/bash
# activity-report.sh

REPORT_DATE=$(date +%Y-%m-%d)
OUTPUT_FILE="audit-report-$REPORT_DATE.html"

cat > $OUTPUT_FILE <<HTML
<!DOCTYPE html>
<html>
<head>
    <title>Kubernetes Audit Report - $REPORT_DATE</title>
    <style>
        body { font-family: Arial; margin: 20px; }
        table { border-collapse: collapse; width: 100%; }
        th, td { border: 1px solid #ddd; padding: 8px; text-align: left; }
        th { background-color: #4CAF50; color: white; }
        .warning { background-color: #fff3cd; }
        .danger { background-color: #f8d7da; }
    </style>
</head>
<body>
    <h1>Kubernetes Cluster Audit Report</h1>
    <p>Generated: $(date)</p>

    <h2>Executive Summary</h2>
    <ul>
        <li>Total API calls: $(cat /var/log/microk8s/audit/audit.log | wc -l)</li>
        <li>Unique users: $(cat /var/log/microk8s/audit/audit.log | jq -r '.user.username' | sort -u | wc -l)</li>
        <li>Failed requests: $(cat /var/log/microk8s/audit/audit.log | jq 'select(.responseStatus.code >= 400)' | wc -l)</li>
    </ul>

    <h2>Top Users</h2>
    <table>
        <tr><th>User</th><th>Actions</th></tr>
HTML

cat /var/log/microk8s/audit/audit.log | \
  jq -r '.user.username' | sort | uniq -c | sort -rn | head -10 | \
  while read count user; do
    echo "        <tr><td>$user</td><td>$count</td></tr>" >> $OUTPUT_FILE
  done

cat >> $OUTPUT_FILE <<HTML
    </table>

    <h2>Sensitive Actions</h2>
    <table>
        <tr><th>Time</th><th>User</th><th>Action</th><th>Resource</th></tr>
HTML

cat /var/log/microk8s/audit/audit.log | \
  jq -r 'select(.objectRef.resource == "secrets" or .objectRef.resource == "configmaps") |
  [.requestReceivedTimestamp, .user.username, .verb, .objectRef.resource] | @tsv' | \
  head -20 | \
  while IFS=$'\t' read -r time user verb resource; do
    echo "        <tr><td>$time</td><td>$user</td><td>$verb</td><td>$resource</td></tr>" >> $OUTPUT_FILE
  done

cat >> $OUTPUT_FILE <<HTML
    </table>

    <h2>Failed Operations</h2>
    <table class="danger">
        <tr><th>Time</th><th>User</th><th>Code</th><th>Resource</th></tr>
HTML

cat /var/log/microk8s/audit/audit.log | \
  jq -r 'select(.responseStatus.code >= 400) |
  [.requestReceivedTimestamp, .user.username, .responseStatus.code, .objectRef.resource] | @tsv' | \
  head -20 | \
  while IFS=$'\t' read -r time user code resource; do
    echo "        <tr><td>$time</td><td>$user</td><td>$code</td><td>$resource</td></tr>" >> $OUTPUT_FILE
  done

echo "    </table></body></html>" >> $OUTPUT_FILE

echo "Report generated: $OUTPUT_FILE"
```

## Intégration avec SIEM

### Splunk Integration

```yaml
# ConfigMap pour Splunk Universal Forwarder
apiVersion: v1
kind: ConfigMap
metadata:
  name: splunk-forwarder-config
  namespace: logging
data:
  inputs.conf: |
    [monitor:///var/log/kubernetes/audit.log]
    disabled = false
    index = kubernetes_audit
    sourcetype = kube:audit

    [monitor:///var/log/kubernetes/audit-*.log]
    disabled = false
    index = kubernetes_audit
    sourcetype = kube:audit

  outputs.conf: |
    [indexAndForward]
    index = false

    [tcpout]
    defaultGroup = splunk-indexers

    [tcpout:splunk-indexers]
    server = splunk-indexer.company.com:9997
    compressed = true

  props.conf: |
    [kube:audit]
    SHOULD_LINEMERGE = false
    TRUNCATE = 0
    TIME_PREFIX = requestReceivedTimestamp":"
    TIME_FORMAT = %Y-%m-%dT%H:%M:%S.%N%Z
    MAX_TIMESTAMP_LOOKAHEAD = 30
    KV_MODE = json
```

### Elastic SIEM Integration

```yaml
# Pipeline Logstash pour enrichissement
apiVersion: v1
kind: ConfigMap
metadata:
  name: logstash-pipeline
  namespace: logging
data:
  pipeline.conf: |
    input {
      beats {
        port => 5044
      }
    }

    filter {
      if [kubernetes][audit] {
        json {
          source => "message"
          target => "audit"
        }

        # Enrichissement GeoIP
        if [audit][sourceIPs][0] {
          geoip {
            source => "[audit][sourceIPs][0]"
            target => "geoip"
          }
        }

        # Catégorisation des événements
        if [audit][verb] in ["delete", "deletecollection"] {
          mutate {
            add_field => { "risk_score" => "high" }
          }
        } else if [audit][objectRef][resource] == "secrets" {
          mutate {
            add_field => { "risk_score" => "medium" }
          }
        } else {
          mutate {
            add_field => { "risk_score" => "low" }
          }
        }

        # Détection d'anomalies
        if [audit][responseStatus][code] >= 400 {
          mutate {
            add_tag => ["authentication_failure"]
          }
        }
      }
    }

    output {
      elasticsearch {
        hosts => ["elasticsearch:9200"]
        index => "k8s-audit-%{+YYYY.MM.dd}"
        template_name => "k8s-audit"
        template => "/usr/share/logstash/templates/k8s-audit.json"
      }
    }
```

## Archivage et rétention

### Stratégie de rétention

```yaml
# Configuration de rétention par type
retention_policy:
  hot_storage:  # SSD rapide
    duration: 7_days
    location: /var/log/kubernetes/audit
    format: json
    compression: none

  warm_storage:  # HDD standard
    duration: 30_days
    location: /mnt/audit-archive/warm
    format: json
    compression: gzip

  cold_storage:  # Object storage
    duration: 365_days
    location: s3://audit-archive/cold
    format: parquet  # Pour requêtes analytiques
    compression: snappy

  deletion:
    after: 365_days
    requires_approval: true
    backup_required: true
```

### Script d'archivage automatique

```bash
#!/bin/bash
# archive-audit-logs.sh

# Configuration
HOT_DIR="/var/log/microk8s/audit"
WARM_DIR="/mnt/audit-archive/warm"
COLD_BUCKET="s3://company-audit-archive/kubernetes"
RETENTION_HOT=7
RETENTION_WARM=30
RETENTION_COLD=365

# Fonction pour archiver vers warm
archive_to_warm() {
  echo "Archiving to warm storage..."
  find $HOT_DIR -name "audit-*.log" -mtime +$RETENTION_HOT -print0 | \
  while IFS= read -r -d '' file; do
    filename=$(basename "$file")
    echo "  Compressing $filename..."
    gzip -c "$file" > "$WARM_DIR/${filename}.gz"
    rm "$file"
  done
}

# Fonction pour archiver vers cold
archive_to_cold() {
  echo "Archiving to cold storage..."
  find $WARM_DIR -name "audit-*.log.gz" -mtime +$RETENTION_WARM -print0 | \
  while IFS= read -r -d '' file; do
    filename=$(basename "$file")
    echo "  Uploading $filename to S3..."

    # Convertir en Parquet pour analyses futures
    zcat "$file" | \
      jq -c '.' | \
      python3 -c "
import sys
import pandas as pd
import pyarrow.parquet as pq
data = [eval(line) for line in sys.stdin]
df = pd.DataFrame(data)
df.to_parquet('/tmp/temp.parquet')
"

    aws s3 cp /tmp/temp.parquet "$COLD_BUCKET/${filename%.gz}.parquet"
    rm "$file"
    rm /tmp/temp.parquet
  done
}

# Fonction pour nettoyer les vieux logs
cleanup_old_logs() {
  echo "Cleaning up old logs..."
  aws s3 ls "$COLD_BUCKET" | \
  while read -r line; do
    file_date=$(echo $line | awk '{print $1}')
    file_name=$(echo $line | awk '{print $4}')

    # Calculer l'âge en jours
    age_days=$(( ($(date +%s) - $(date -d "$file_date" +%s)) / 86400 ))

    if [ $age_days -gt $RETENTION_COLD ]; then
      echo "  Deleting $file_name (age: $age_days days)"
      aws s3 rm "$COLD_BUCKET/$file_name"
    fi
  done
}

# Rapport d'archivage
generate_report() {
  echo ""
  echo "=== Archive Status Report ==="
  echo "Hot storage: $(du -sh $HOT_DIR | cut -f1)"
  echo "Warm storage: $(du -sh $WARM_DIR | cut -f1)"
  echo "Cold storage: $(aws s3 ls $COLD_BUCKET --recursive --summarize | grep "Total Size" | cut -d: -f2)"
  echo ""
  echo "File counts:"
  echo "  Hot: $(ls -1 $HOT_DIR/*.log 2>/dev/null | wc -l) files"
  echo "  Warm: $(ls -1 $WARM_DIR/*.gz 2>/dev/null | wc -l) files"
  echo "  Cold: $(aws s3 ls $COLD_BUCKET --recursive | wc -l) files"
}

# Exécution principale
echo "Starting audit log archival process..."
echo "Time: $(date)"

# Créer les répertoires si nécessaire
mkdir -p $WARM_DIR

# Exécuter l'archivage
archive_to_warm
archive_to_cold
cleanup_old_logs

# Générer le rapport
generate_report

echo "Archival process complete."
```

## Bonnes pratiques

### 1. Politique d'audit équilibrée

```yaml
# DO: Politique progressive
rules:
  # Haute priorité - RequestResponse
  - level: RequestResponse
    verbs: ["delete", "deletecollection"]
    resources:
      - group: ""
        resources: ["secrets", "serviceaccounts"]

  # Moyenne priorité - Request
  - level: Request
    verbs: ["create", "update", "patch"]
    resources:
      - group: "rbac.authorization.k8s.io"

  # Basse priorité - Metadata
  - level: Metadata
    verbs: ["get", "list", "watch"]

  # Ignorer le bruit
  - level: None
    users: ["system:kube-proxy"]
    verbs: ["watch"]

# DON'T: Tout logger en RequestResponse
# Cela génère des logs énormes et impacte les performances
```

### 2. Sécurisation des logs

```bash
# Chiffrement des logs au repos
# 1. Créer une clé de chiffrement
openssl rand -base64 32 > /etc/audit-encryption.key
chmod 600 /etc/audit-encryption.key

# 2. Script pour chiffrer les logs archivés
#!/bin/bash
for file in /var/log/audit/*.log; do
  openssl enc -aes-256-cbc -salt -in "$file" \
    -out "${file}.enc" -pass file:/etc/audit-encryption.key
  shred -u "$file"  # Suppression sécurisée
done

# 3. Intégrité des logs avec checksums
find /var/log/audit -name "*.log" -exec sha256sum {} \; > /var/log/audit/checksums.txt
```

### 3. Monitoring des logs d'audit

```yaml
# Alertes sur l'audit logging lui-même
apiVersion: monitoring.coreos.com/v1
kind: PrometheusRule
metadata:
  name: audit-system-alerts
spec:
  groups:
  - name: audit-health
    rules:
    - alert: AuditLogsStopped
      expr: |
        rate(apiserver_audit_events_total[5m]) == 0
      for: 10m
      labels:
        severity: critical
      annotations:
        summary: "No audit events logged in 10 minutes"

    - alert: AuditLogsVolumeTooHigh
      expr: |
        rate(apiserver_audit_events_total[5m]) > 1000
      for: 5m
      labels:
        severity: warning
      annotations:
        summary: "Audit log volume unusually high"

    - alert: AuditLogDiskSpaceLow
      expr: |
        node_filesystem_avail_bytes{mountpoint="/var/log"} /
        node_filesystem_size_bytes{mountpoint="/var/log"} < 0.1
      labels:
        severity: critical
      annotations:
        summary: "Less than 10% disk space for audit logs"
```

### 4. Documentation et processus

```markdown
# Processus de gestion des logs d'audit

## Responsabilités
- **Security Team**: Définition des politiques, investigation des incidents
- **Platform Team**: Maintenance de l'infrastructure, archivage
- **Compliance Team**: Audits réguliers, rapports de conformité

## Procédures

### Investigation d'incident
1. Identifier la timeframe
2. Exécuter `incident-investigation.sh`
3. Analyser les résultats dans Kibana/Splunk
4. Documenter les findings
5. Créer un rapport post-mortem

### Modification de politique
1. Proposer les changements dans Git
2. Review par Security Team
3. Test en environnement de staging
4. Déploiement progressif
5. Monitoring post-déploiement

### Accès aux logs
- Read-only: Via Kibana dashboard
- Investigation: Via kubectl avec RBAC approprié
- Export: Requiert approbation Security Team
```

## Résumé et points clés

L'audit logging est votre boîte noire pour Kubernetes :

1. **Activez l'audit dès le début** - Même en développement, pour prendre les bonnes habitudes
2. **Politique progressive** - Commencez simple, affinez selon vos besoins
3. **Équilibrez détail et performance** - RequestResponse seulement pour le critique
4. **Automatisez l'archivage** - Hot → Warm → Cold selon l'âge
5. **Intégrez avec votre SIEM** - Pour correlation et alerting avancés
6. **Respectez la conformité** - PCI-DSS, GDPR, SOC2 ont des exigences spécifiques
7. **Protégez les logs** - Chiffrement, intégrité, accès restreint
8. **Testez régulièrement** - Simulations d'incidents pour valider la capture
9. **Documentez les processus** - Qui fait quoi en cas d'incident
10. **Monitorez l'audit lui-même** - Les logs doivent toujours fonctionner

L'audit logging n'est pas optionnel en production. C'est votre assurance en cas d'incident, votre preuve de conformité, et votre outil d'apprentissage pour améliorer la sécurité. Investissez le temps nécessaire pour bien le configurer - vous vous remercierez le jour où vous en aurez besoin.

---

*Prochain sujet : 11.7 Bonnes pratiques sécurité - Synthèse et recommandations finales*


⏭️
