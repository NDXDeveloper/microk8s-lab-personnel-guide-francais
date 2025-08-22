🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 8.06 - Installation et configuration de Grafana

## Introduction à Grafana

### Qu'est-ce que Grafana ?

Grafana est une plateforme open source de visualisation et d'analyse de métriques. C'est l'interface graphique qui transforme vos données Prometheus en dashboards interactifs et compréhensibles.

**Analogie simple :**
```
Prometheus = Base de données de métriques
PromQL = Langage de requête
Grafana = Excel avec des graphiques en temps réel
```

### Pourquoi Grafana avec Prometheus ?

- **Visualisations riches** : Plus de 20 types de graphiques
- **Dashboards dynamiques** : Variables, filtres, drill-down
- **Alerting visuel** : Configuration intuitive des seuils
- **Templating** : Dashboards réutilisables
- **Communauté** : Des milliers de dashboards prêts à l'emploi
- **Multi-datasources** : Combine Prometheus avec d'autres sources

### Architecture Grafana + Prometheus

```
┌────────────────────────────────────────────────────┐
│                   Utilisateur                       │
│                        │                            │
│                        ▼                            │
│                 [Navigateur Web]                    │
└────────────────────────────────────────────────────┘
                         │
                         ▼ HTTPS
┌────────────────────────────────────────────────────┐
│                    Grafana                          │
│  ┌──────────────────────────────────────────┐     │
│  │            Interface Web                  │     │
│  │  ┌──────────┐  ┌──────────┐  ┌────────┐ │     │
│  │  │Dashboard │  │Dashboard │  │ Alert  │ │     │
│  │  │    1     │  │    2     │  │ Rules  │ │     │
│  │  └──────────┘  └──────────┘  └────────┘ │     │
│  └──────────────────────────────────────────┘     │
│                        │                            │
│  ┌──────────────────────────────────────────┐     │
│  │           Data Source Layer               │     │
│  │         (Prometheus connector)            │     │
│  └──────────────────────────────────────────┘     │
└────────────────────────────────────────────────────┘
                         │
                         ▼ PromQL
┌────────────────────────────────────────────────────┐
│                   Prometheus                        │
│                  [Métriques]                        │
└────────────────────────────────────────────────────┘
```

## Options d'installation sur MicroK8s

### Comparaison des méthodes

| Méthode | Avantages | Inconvénients | Recommandé pour |
|---------|-----------|---------------|-----------------|
| **Addon observability** | Installation en 1 commande | Configuration limitée | Débutants, tests rapides |
| **Helm Chart** | Très personnalisable | Plus complexe | Production, personnalisation |
| **Manifests YAML** | Contrôle total | Maintenance manuelle | Apprentissage, cas spéciaux |
| **Docker (hors K8s)** | Simple, isolé | Pas intégré à K8s | Tests locaux uniquement |

## Méthode 1 : Via l'addon observability (Le plus simple)

### Installation automatique

Si vous avez déjà activé l'addon observability pour Prometheus :

```bash
# L'addon installe Grafana automatiquement
microk8s enable observability

# Vérifier que Grafana est installé
kubectl get pods -n observability | grep grafana

# Sortie attendue :
# grafana-7d966d6c67-xxxxx         1/1     Running   0          5m
```

### Accès à Grafana

```bash
# Méthode 1 : Port-forward (accès local)
kubectl port-forward -n observability svc/grafana 3000:3000

# Ouvrir dans le navigateur : http://localhost:3000

# Méthode 2 : Récupérer l'IP du service
kubectl get svc -n observability grafana
```

### Récupération des credentials par défaut

```bash
# Username par défaut : admin
# Password : récupérer depuis le secret
kubectl get secret -n observability grafana -o jsonpath="{.data.admin-password}" | base64 --decode

# Ou voir directement :
echo "Username: admin"
echo "Password: $(kubectl get secret -n observability grafana -o jsonpath='{.data.admin-password}' | base64 --decode)"
```

## Méthode 2 : Installation via Helm (Recommandé pour personnalisation)

### Préparation

```bash
# Ajouter le repository Grafana
helm repo add grafana https://grafana.github.io/helm-charts
helm repo update

# Voir les versions disponibles
helm search repo grafana/grafana --versions
```

### Configuration personnalisée

Créer un fichier `grafana-values.yaml` :

```yaml
# grafana-values.yaml
# Configuration de base
replicas: 1

# Credentials admin
adminUser: admin
adminPassword: MySecurePassword123!  # Changez ceci !

# Persistence pour sauvegarder dashboards et config
persistence:
  enabled: true
  size: 5Gi
  accessModes:
    - ReadWriteOnce

# Service
service:
  type: ClusterIP  # ou NodePort, LoadBalancer
  port: 80
  targetPort: 3000

# Ingress (si vous avez un ingress controller)
ingress:
  enabled: true
  ingressClassName: nginx
  annotations:
    cert-manager.io/cluster-issuer: "letsencrypt-prod"
    nginx.ingress.kubernetes.io/ssl-redirect: "true"
  hosts:
    - grafana.votredomaine.com
  tls:
    - secretName: grafana-tls
      hosts:
        - grafana.votredomaine.com

# Resources
resources:
  requests:
    cpu: 100m
    memory: 128Mi
  limits:
    cpu: 500m
    memory: 512Mi

# Configuration Grafana
grafana.ini:
  server:
    root_url: https://grafana.votredomaine.com
  auth:
    disable_login_form: false
  auth.anonymous:
    enabled: true
    org_role: Viewer
  security:
    admin_user: admin
    admin_password: MySecurePassword123!

# Datasources pré-configurées
datasources:
  datasources.yaml:
    apiVersion: 1
    datasources:
    - name: Prometheus
      type: prometheus
      url: http://prometheus-server.monitoring.svc.cluster.local:9090
      access: proxy
      isDefault: true
      editable: true

# Dashboards providers
dashboardProviders:
  dashboardproviders.yaml:
    apiVersion: 1
    providers:
    - name: 'default'
      orgId: 1
      folder: ''
      type: file
      disableDeletion: false
      updateIntervalSeconds: 10
      options:
        path: /var/lib/grafana/dashboards/default

# Dashboards à installer automatiquement
dashboards:
  default:
    kubernetes-cluster:
      gnetId: 7249  # ID depuis grafana.com
      revision: 1
      datasource: Prometheus
    node-exporter:
      gnetId: 1860
      revision: 27
      datasource: Prometheus
```

### Installation avec Helm

```bash
# Créer le namespace si nécessaire
kubectl create namespace monitoring

# Installer Grafana
helm install grafana grafana/grafana \
  --namespace monitoring \
  --values grafana-values.yaml

# Suivre le déploiement
kubectl rollout status deployment/grafana -n monitoring

# Vérifier les pods
kubectl get pods -n monitoring -l app.kubernetes.io/name=grafana
```

## Méthode 3 : Installation manuelle avec manifests

### Structure complète des manifests

```yaml
# grafana-deployment.yaml
apiVersion: v1
kind: Namespace
metadata:
  name: monitoring
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: grafana-pvc
  namespace: monitoring
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 5Gi
---
apiVersion: v1
kind: ConfigMap
metadata:
  name: grafana-datasources
  namespace: monitoring
data:
  prometheus.yaml: |
    apiVersion: 1
    datasources:
    - name: Prometheus
      type: prometheus
      access: proxy
      url: http://prometheus:9090
      isDefault: true
      editable: true
---
apiVersion: v1
kind: ConfigMap
metadata:
  name: grafana-dashboard-providers
  namespace: monitoring
data:
  dashboards.yaml: |
    apiVersion: 1
    providers:
    - name: 'default'
      orgId: 1
      folder: ''
      type: file
      disableDeletion: false
      updateIntervalSeconds: 10
      allowUiUpdates: true
      options:
        path: /var/lib/grafana/dashboards
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: grafana
  namespace: monitoring
spec:
  replicas: 1
  selector:
    matchLabels:
      app: grafana
  template:
    metadata:
      labels:
        app: grafana
    spec:
      securityContext:
        fsGroup: 472
        runAsUser: 472
      containers:
      - name: grafana
        image: grafana/grafana:10.2.0
        imagePullPolicy: IfNotPresent
        ports:
        - containerPort: 3000
          name: http-grafana
        env:
        - name: GF_SECURITY_ADMIN_USER
          value: admin
        - name: GF_SECURITY_ADMIN_PASSWORD
          value: admin123  # CHANGEZ CECI !
        - name: GF_SERVER_ROOT_URL
          value: http://localhost:3000
        - name: GF_INSTALL_PLUGINS
          value: grafana-piechart-panel,grafana-clock-panel
        resources:
          requests:
            cpu: 100m
            memory: 128Mi
          limits:
            cpu: 500m
            memory: 512Mi
        volumeMounts:
        - name: grafana-storage
          mountPath: /var/lib/grafana
        - name: grafana-datasources
          mountPath: /etc/grafana/provisioning/datasources
        - name: grafana-dashboard-providers
          mountPath: /etc/grafana/provisioning/dashboards
        - name: dashboards
          mountPath: /var/lib/grafana/dashboards
        livenessProbe:
          httpGet:
            path: /api/health
            port: 3000
          initialDelaySeconds: 30
          timeoutSeconds: 5
        readinessProbe:
          httpGet:
            path: /api/health
            port: 3000
          initialDelaySeconds: 5
          timeoutSeconds: 3
      volumes:
      - name: grafana-storage
        persistentVolumeClaim:
          claimName: grafana-pvc
      - name: grafana-datasources
        configMap:
          name: grafana-datasources
      - name: grafana-dashboard-providers
        configMap:
          name: grafana-dashboard-providers
      - name: dashboards
        emptyDir: {}
---
apiVersion: v1
kind: Service
metadata:
  name: grafana
  namespace: monitoring
spec:
  selector:
    app: grafana
  type: ClusterIP
  ports:
  - port: 3000
    targetPort: 3000
    name: http
```

### Application et vérification

```bash
# Appliquer les manifests
kubectl apply -f grafana-deployment.yaml

# Vérifier le déploiement
kubectl get all -n monitoring -l app=grafana

# Voir les logs
kubectl logs -n monitoring deployment/grafana
```

## Configuration post-installation

### Premier login et changement de mot de passe

1. Accéder à Grafana :
```bash
kubectl port-forward -n monitoring svc/grafana 3000:3000
```

2. Ouvrir http://localhost:3000
3. Login avec admin/admin (ou vos credentials)
4. Changer le mot de passe immédiatement

### Configuration de la datasource Prometheus

Si pas configurée automatiquement :

1. **Aller dans Configuration → Data Sources**
2. **Cliquer sur "Add data source"**
3. **Sélectionner "Prometheus"**
4. **Configurer :**
   - Name: `Prometheus`
   - URL: `http://prometheus:9090` (ou l'adresse de votre Prometheus)
   - Access: `Server (default)`
   - Scrape interval: `15s`
   - HTTP Method: `POST`
5. **Cliquer sur "Save & Test"**

### Configuration via API

```bash
# Créer une datasource via API
curl -X POST -H "Content-Type: application/json" \
  -u admin:admin123 \
  http://localhost:3000/api/datasources \
  -d '{
    "name": "Prometheus",
    "type": "prometheus",
    "url": "http://prometheus:9090",
    "access": "proxy",
    "isDefault": true,
    "jsonData": {
      "httpMethod": "POST",
      "keepCookies": []
    }
  }'
```

## Configuration de l'accès externe

### Via Ingress (Recommandé)

```yaml
# grafana-ingress.yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: grafana-ingress
  namespace: monitoring
  annotations:
    nginx.ingress.kubernetes.io/backend-protocol: "HTTP"
    cert-manager.io/cluster-issuer: "letsencrypt-prod"
spec:
  ingressClassName: nginx
  tls:
  - hosts:
    - grafana.votredomaine.com
    secretName: grafana-tls
  rules:
  - host: grafana.votredomaine.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: grafana
            port:
              number: 3000
```

### Via NodePort

```yaml
# grafana-nodeport.yaml
apiVersion: v1
kind: Service
metadata:
  name: grafana-nodeport
  namespace: monitoring
spec:
  type: NodePort
  selector:
    app: grafana
  ports:
  - port: 3000
    targetPort: 3000
    nodePort: 30300  # Port entre 30000-32767
```

## Configuration de la sécurité

### HTTPS et certificats SSL

```yaml
# Configuration avec cert-manager
apiVersion: cert-manager.io/v1
kind: Certificate
metadata:
  name: grafana-cert
  namespace: monitoring
spec:
  secretName: grafana-tls
  issuerRef:
    name: letsencrypt-prod
    kind: ClusterIssuer
  dnsNames:
  - grafana.votredomaine.com
```

### Authentification et autorisation

#### Configuration LDAP/AD
```ini
# grafana.ini ou via ConfigMap
[auth.ldap]
enabled = true
config_file = /etc/grafana/ldap.toml
allow_sign_up = true

[auth]
disable_login_form = false
```

#### OAuth2 (GitHub, Google, etc.)
```ini
[auth.github]
enabled = true
allow_sign_up = true
client_id = YOUR_GITHUB_CLIENT_ID
client_secret = YOUR_GITHUB_CLIENT_SECRET
scopes = user:email,read:org
auth_url = https://github.com/login/oauth/authorize
token_url = https://github.com/login/oauth/access_token
api_url = https://api.github.com/user
allowed_organizations = your-org
```

### Sécurisation des secrets

```yaml
# Utiliser des secrets Kubernetes
apiVersion: v1
kind: Secret
metadata:
  name: grafana-secrets
  namespace: monitoring
type: Opaque
stringData:
  admin-user: admin
  admin-password: "SuperSecurePassword123!"
  database-password: "DbPassword456!"
---
# Dans le deployment, référencer les secrets
env:
- name: GF_SECURITY_ADMIN_USER
  valueFrom:
    secretKeyRef:
      name: grafana-secrets
      key: admin-user
- name: GF_SECURITY_ADMIN_PASSWORD
  valueFrom:
    secretKeyRef:
      name: grafana-secrets
      key: admin-password
```

## Configuration des dashboards

### Import de dashboards communautaires

1. **Depuis l'interface :**
   - Aller dans Dashboards → Browse
   - Cliquer sur "New" → "Import"
   - Entrer l'ID du dashboard (depuis grafana.com)
   - Sélectionner la datasource Prometheus
   - Cliquer sur "Import"

2. **IDs de dashboards populaires :**

| Dashboard | ID | Description |
|-----------|-----|-------------|
| Node Exporter Full | 1860 | Métriques système complètes |
| Kubernetes Cluster | 7249 | Vue d'ensemble du cluster |
| NGINX Ingress | 9614 | Métriques Ingress Controller |
| Kubernetes Pods | 6417 | Détails des pods |
| Container Metrics | 10000 | Métriques des containers |

### Création d'un dashboard simple

```json
{
  "dashboard": {
    "title": "Mon Premier Dashboard",
    "panels": [
      {
        "id": 1,
        "title": "CPU Usage",
        "type": "graph",
        "targets": [
          {
            "expr": "100 - (avg(rate(node_cpu_seconds_total{mode=\"idle\"}[5m])) * 100)",
            "refId": "A"
          }
        ],
        "gridPos": {
          "h": 8,
          "w": 12,
          "x": 0,
          "y": 0
        }
      },
      {
        "id": 2,
        "title": "Memory Usage",
        "type": "gauge",
        "targets": [
          {
            "expr": "(1 - node_memory_MemAvailable_bytes / node_memory_MemTotal_bytes) * 100",
            "refId": "A"
          }
        ],
        "gridPos": {
          "h": 8,
          "w": 12,
          "x": 12,
          "y": 0
        }
      }
    ]
  }
}
```

### Provisioning automatique de dashboards

```yaml
# ConfigMap avec dashboard JSON
apiVersion: v1
kind: ConfigMap
metadata:
  name: grafana-dashboards
  namespace: monitoring
data:
  kubernetes-dashboard.json: |
    {
      "dashboard": {
        "title": "Kubernetes Overview",
        "panels": [...],
        "version": 1
      }
    }
```

## Plugins Grafana essentiels

### Installation de plugins

```bash
# Via variable d'environnement au démarrage
env:
- name: GF_INSTALL_PLUGINS
  value: "grafana-piechart-panel,grafana-worldmap-panel,grafana-clock-panel"

# Ou après installation
kubectl exec -n monitoring grafana-xxx -- grafana-cli plugins install grafana-piechart-panel

# Redémarrer Grafana
kubectl rollout restart deployment/grafana -n monitoring
```

### Plugins recommandés pour Kubernetes

| Plugin | Description | Commande |
|--------|-------------|----------|
| Pie Chart | Graphiques en camembert | `grafana-piechart-panel` |
| World Map | Carte géographique | `grafana-worldmap-panel` |
| Clock | Horloge temps réel | `grafana-clock-panel` |
| Polystat | Hexagones colorés | `grafana-polystat-panel` |
| Status Panel | Panneau d'état | `vonage-status-panel` |

## Optimisation pour un lab

### Configuration minimale pour ressources limitées

```yaml
# Ressources réduites
resources:
  requests:
    cpu: 50m
    memory: 64Mi
  limits:
    cpu: 200m
    memory: 256Mi

# Désactiver les fonctionnalités non essentielles
env:
- name: GF_ANALYTICS_REPORTING_ENABLED
  value: "false"
- name: GF_ANALYTICS_CHECK_FOR_UPDATES
  value: "false"
- name: GF_USERS_ALLOW_SIGN_UP
  value: "false"
- name: GF_ALERTING_ENABLED
  value: "false"  # Si vous n'utilisez pas les alertes Grafana
```

### Cache et performance

```ini
# grafana.ini optimisations
[database]
# Utiliser SQLite pour un lab (par défaut)
type = sqlite3

[dataproxy]
# Timeout pour les requêtes
timeout = 30
keep_alive_seconds = 30

[dashboards]
# Mise à jour moins fréquente
min_refresh_interval = 30s

[caching]
# Activer le cache
enabled = true
```

## Backup et restauration

### Backup des dashboards et configuration

```bash
# Export de tous les dashboards
for dashboard_uid in $(curl -s -u admin:admin123 http://localhost:3000/api/search | jq -r '.[] | .uid'); do
  curl -s -u admin:admin123 "http://localhost:3000/api/dashboards/uid/${dashboard_uid}" | \
    jq '.dashboard' > "backup_${dashboard_uid}.json"
done

# Backup de la base de données (SQLite)
kubectl exec -n monitoring grafana-xxx -- tar czf - /var/lib/grafana/grafana.db > grafana-backup.tar.gz

# Backup complet du PVC
kubectl exec -n monitoring grafana-xxx -- tar czf - /var/lib/grafana > grafana-full-backup.tar.gz
```

### Restauration

```bash
# Restaurer la base de données
kubectl cp grafana-backup.tar.gz monitoring/grafana-xxx:/tmp/
kubectl exec -n monitoring grafana-xxx -- tar xzf /tmp/grafana-backup.tar.gz -C /

# Importer un dashboard
curl -X POST -H "Content-Type: application/json" \
  -u admin:admin123 \
  http://localhost:3000/api/dashboards/db \
  -d @backup_dashboard.json
```

## Troubleshooting

### Problèmes courants et solutions

#### Grafana ne démarre pas
```bash
# Vérifier les logs
kubectl logs -n monitoring deployment/grafana

# Problèmes courants :
# - Permissions sur le PVC (fsGroup: 472)
# - Mauvaise configuration dans grafana.ini
# - Ressources insuffisantes
```

#### Connexion à Prometheus échoue
```bash
# Tester la connectivité
kubectl exec -n monitoring grafana-xxx -- curl http://prometheus:9090/api/v1/query?query=up

# Vérifier la résolution DNS
kubectl exec -n monitoring grafana-xxx -- nslookup prometheus
```

#### Dashboards ne se chargent pas
```bash
# Vérifier les datasources
curl -u admin:admin123 http://localhost:3000/api/datasources

# Tester une requête
curl -u admin:admin123 "http://localhost:3000/api/datasources/proxy/1/api/v1/query?query=up"
```

### Commandes de diagnostic

```bash
# État général
kubectl describe deployment grafana -n monitoring

# Utilisation des ressources
kubectl top pod -n monitoring -l app=grafana

# Events
kubectl get events -n monitoring --sort-by='.lastTimestamp' | grep grafana

# Configuration active
kubectl exec -n monitoring grafana-xxx -- cat /etc/grafana/grafana.ini
```

## Intégration avec l'écosystème

### Alerting avec Grafana

```yaml
# Configuration des canaux de notification
apiVersion: v1
kind: ConfigMap
metadata:
  name: grafana-notifiers
  namespace: monitoring
data:
  notifiers.yaml: |
    notifiers:
    - name: Slack
      type: slack
      uid: slack-1
      settings:
        url: https://hooks.slack.com/services/YOUR/SLACK/WEBHOOK
        channel: '#alerts'
```

### Intégration avec d'autres datasources

```yaml
# Ajouter Loki pour les logs
datasources:
  datasources.yaml:
    apiVersion: 1
    datasources:
    - name: Prometheus
      type: prometheus
      url: http://prometheus:9090
    - name: Loki
      type: loki
      url: http://loki:3100
    - name: Elasticsearch
      type: elasticsearch
      url: http://elasticsearch:9200
```

## Checklist de validation

- [ ] Grafana est accessible via navigateur
- [ ] Login avec credentials admin fonctionne
- [ ] Mot de passe admin changé
- [ ] Datasource Prometheus configurée et testée
- [ ] Au moins un dashboard importé et fonctionnel
- [ ] Persistance configurée (PVC)
- [ ] Accès externe sécurisé (HTTPS si exposé)
- [ ] Backup strategy en place
- [ ] Ressources adaptées au lab
- [ ] Monitoring de Grafana lui-même

## Préparation pour la suite

Avec Grafana installé et configuré, vous êtes maintenant prêt à :
- Connecter Grafana à Prometheus (section 8.07)
- Importer et créer des dashboards Kubernetes (section 8.08)
- Créer des dashboards personnalisés (section 8.09)
- Utiliser les variables et le templating (section 8.10)

Grafana est votre fenêtre sur vos métriques. Prenez le temps de bien le configurer et de comprendre son interface avant de créer des dashboards complexes.

⏭️
