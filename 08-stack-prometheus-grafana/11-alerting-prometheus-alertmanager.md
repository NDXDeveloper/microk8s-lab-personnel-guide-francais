🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 8.11 Alerting avec Prometheus Alertmanager

## Introduction à l'alerting

L'alerting est un composant essentiel de toute stratégie de monitoring. Avoir de beaux dashboards Grafana c'est bien, mais personne ne peut les regarder 24h/24. C'est là qu'intervient Alertmanager : il surveille constamment vos métriques et vous prévient quand quelque chose ne va pas.

Dans un environnement Kubernetes sur MicroK8s, l'alerting devient crucial. Vos pods peuvent redémarrer, manquer de mémoire, ou vos volumes peuvent se remplir - autant de situations où vous voulez être prévenu immédiatement plutôt que de découvrir le problème quand les utilisateurs se plaignent.

## Architecture de l'alerting Prometheus

### Vue d'ensemble du système

Le système d'alerting Prometheus fonctionne en deux parties distinctes mais complémentaires :

**Prometheus** évalue les règles d'alerte et génère des alertes quand les conditions sont remplies. Il ne gère pas l'envoi des notifications lui-même.

**Alertmanager** reçoit les alertes de Prometheus, les déduplique, les groupe, les route vers les bons destinataires et gère les silences. C'est le chef d'orchestre des notifications.

Cette séparation permet une grande flexibilité : vous pouvez avoir plusieurs Prometheus qui envoient vers un seul Alertmanager, ou l'inverse selon vos besoins.

### Flux de données

```
Métriques → Prometheus → Règles d'alerte → Alertes → Alertmanager → Notifications
                ↑                                           ↓
            Recording                                   Email/Slack/
              Rules                                     PagerDuty/etc.
```

## Installation d'Alertmanager sur MicroK8s

### Prérequis

Avant d'installer Alertmanager, assurez-vous que Prometheus est déjà installé et fonctionnel dans votre cluster MicroK8s. Alertmanager a besoin de Prometheus pour recevoir les alertes.

### Déploiement via manifeste

Créez d'abord un namespace dédié si ce n'est pas déjà fait :

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: monitoring
```

### ConfigMap pour la configuration

La configuration d'Alertmanager se fait via un ConfigMap :

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: alertmanager-config
  namespace: monitoring
data:
  alertmanager.yml: |
    global:
      resolve_timeout: 5m

    route:
      group_by: ['alertname', 'cluster', 'service']
      group_wait: 10s
      group_interval: 10s
      repeat_interval: 12h
      receiver: 'default-receiver'

    receivers:
    - name: 'default-receiver'
      # Configuration du receiver (email, slack, etc.)
```

### Déploiement d'Alertmanager

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: alertmanager
  namespace: monitoring
spec:
  replicas: 1
  selector:
    matchLabels:
      app: alertmanager
  template:
    metadata:
      labels:
        app: alertmanager
    spec:
      containers:
      - name: alertmanager
        image: prom/alertmanager:latest
        ports:
        - containerPort: 9093
        volumeMounts:
        - name: config
          mountPath: /etc/alertmanager
        - name: alertmanager-storage
          mountPath: /alertmanager
      volumes:
      - name: config
        configMap:
          name: alertmanager-config
      - name: alertmanager-storage
        emptyDir: {}
```

### Service pour exposer Alertmanager

```yaml
apiVersion: v1
kind: Service
metadata:
  name: alertmanager
  namespace: monitoring
spec:
  selector:
    app: alertmanager
  ports:
    - port: 9093
      targetPort: 9093
  type: ClusterIP
```

## Configuration de Prometheus pour Alertmanager

### Connexion à Alertmanager

Dans votre configuration Prometheus, ajoutez la section alerting :

```yaml
alerting:
  alertmanagers:
  - static_configs:
    - targets:
      - alertmanager.monitoring.svc.cluster.local:9093
```

### Chargement des règles d'alerte

```yaml
rule_files:
  - '/etc/prometheus/rules/*.yml'
```

## Création de règles d'alerte

### Anatomie d'une règle d'alerte

Une règle d'alerte Prometheus suit cette structure :

```yaml
groups:
- name: example-alerts
  interval: 30s
  rules:
  - alert: HighMemoryUsage
    expr: (node_memory_MemAvailable_bytes / node_memory_MemTotal_bytes) < 0.1
    for: 5m
    labels:
      severity: warning
      component: infrastructure
    annotations:
      summary: "Mémoire disponible faible sur {{ $labels.instance }}"
      description: "Moins de 10% de mémoire disponible depuis 5 minutes"
```

### Composants d'une règle

**alert** : Le nom de l'alerte, unique et descriptif

**expr** : La requête PromQL qui définit la condition d'alerte. Si elle retourne un résultat, l'alerte se déclenche

**for** : Durée pendant laquelle la condition doit être vraie avant de déclencher l'alerte. Évite les faux positifs

**labels** : Métadonnées additionnelles attachées à l'alerte. Utilisées pour le routage et le groupement

**annotations** : Informations descriptives pour enrichir l'alerte. Peuvent utiliser des templates

## Règles d'alerte essentielles pour Kubernetes

### Alertes sur les Pods

```yaml
groups:
- name: kubernetes-pods
  rules:
  - alert: PodCrashLooping
    expr: rate(kube_pod_container_status_restarts_total[15m]) > 0
    for: 5m
    labels:
      severity: critical
    annotations:
      summary: "Pod {{ $labels.namespace }}/{{ $labels.pod }} redémarre en boucle"
      description: "Le pod a redémarré {{ $value }} fois dans les 15 dernières minutes"

  - alert: PodNotReady
    expr: sum by (namespace, pod) (kube_pod_status_phase{phase=~"Pending|Unknown"}) > 0
    for: 15m
    labels:
      severity: warning
    annotations:
      summary: "Pod {{ $labels.namespace }}/{{ $labels.pod }} n'est pas prêt"
      description: "Le pod est en état {{ $labels.phase }} depuis 15 minutes"

  - alert: ContainerOOMKilled
    expr: kube_pod_container_status_last_terminated_reason{reason="OOMKilled"} > 0
    for: 1m
    labels:
      severity: warning
    annotations:
      summary: "Container tué pour manque de mémoire"
      description: "{{ $labels.namespace }}/{{ $labels.pod }}/{{ $labels.container }} a été tué (OOM)"
```

### Alertes sur les Nodes

```yaml
groups:
- name: kubernetes-nodes
  rules:
  - alert: NodeNotReady
    expr: kube_node_status_condition{condition="Ready",status="true"} == 0
    for: 5m
    labels:
      severity: critical
    annotations:
      summary: "Node {{ $labels.node }} n'est pas prêt"
      description: "Le node Kubernetes n'est pas dans un état Ready"

  - alert: NodeMemoryPressure
    expr: kube_node_status_condition{condition="MemoryPressure",status="true"} == 1
    for: 5m
    labels:
      severity: warning
    annotations:
      summary: "Pression mémoire sur le node {{ $labels.node }}"
      description: "Le node manque de mémoire disponible"

  - alert: NodeDiskPressure
    expr: kube_node_status_condition{condition="DiskPressure",status="true"} == 1
    for: 5m
    labels:
      severity: warning
    annotations:
      summary: "Pression disque sur le node {{ $labels.node }}"
      description: "Le node manque d'espace disque"
```

### Alertes sur les Deployments

```yaml
groups:
- name: kubernetes-deployments
  rules:
  - alert: DeploymentReplicasMismatch
    expr: |
      kube_deployment_spec_replicas != kube_deployment_status_replicas_available
    for: 15m
    labels:
      severity: warning
    annotations:
      summary: "Deployment {{ $labels.namespace }}/{{ $labels.deployment }} replicas mismatch"
      description: "Nombre de replicas désiré: {{ $labels.spec_replicas }}, disponible: {{ $value }}"

  - alert: DeploymentGenerationMismatch
    expr: |
      kube_deployment_status_observed_generation != kube_deployment_metadata_generation
    for: 15m
    labels:
      severity: warning
    annotations:
      summary: "Deployment {{ $labels.namespace }}/{{ $labels.deployment }} generation mismatch"
      description: "Le deployment a une configuration non appliquée"
```

### Alertes sur les ressources

```yaml
groups:
- name: resource-usage
  rules:
  - alert: HighCPUUsage
    expr: |
      sum(rate(container_cpu_usage_seconds_total[5m])) by (pod, namespace) > 0.8
    for: 10m
    labels:
      severity: warning
    annotations:
      summary: "Utilisation CPU élevée pour {{ $labels.namespace }}/{{ $labels.pod }}"
      description: "CPU à {{ $value | humanizePercentage }} depuis 10 minutes"

  - alert: HighMemoryUsage
    expr: |
      sum(container_memory_working_set_bytes) by (pod, namespace) /
      sum(container_spec_memory_limit_bytes) by (pod, namespace) > 0.8
    for: 10m
    labels:
      severity: warning
    annotations:
      summary: "Utilisation mémoire élevée pour {{ $labels.namespace }}/{{ $labels.pod }}"
      description: "Mémoire à {{ $value | humanizePercentage }} de la limite"

  - alert: PersistentVolumeFillingUp
    expr: |
      kubelet_volume_stats_available_bytes / kubelet_volume_stats_capacity_bytes < 0.1
    for: 5m
    labels:
      severity: critical
    annotations:
      summary: "PersistentVolume {{ $labels.persistentvolumeclaim }} presque plein"
      description: "Moins de 10% d'espace disponible"
```

## Configuration d'Alertmanager

### Structure du fichier de configuration

Le fichier `alertmanager.yml` est structuré en plusieurs sections :

```yaml
global:
  # Configuration globale
  resolve_timeout: 5m
  smtp_from: 'alertmanager@example.com'
  smtp_smarthost: 'smtp.gmail.com:587'
  smtp_auth_username: 'alertmanager@example.com'
  smtp_auth_password: 'your-app-password'

templates:
  # Chemins vers les templates personnalisés
  - '/etc/alertmanager/templates/*.tmpl'

route:
  # Routage des alertes
  group_by: ['alertname', 'cluster', 'service']
  group_wait: 10s
  group_interval: 10s
  repeat_interval: 12h
  receiver: 'team-devops'

  routes:
  - match:
      severity: critical
    receiver: 'team-oncall'
    continue: true

  - match:
      severity: warning
    receiver: 'team-devops'

receivers:
  # Définition des receivers
  - name: 'team-devops'
  - name: 'team-oncall'

inhibit_rules:
  # Règles d'inhibition
  - source_match:
      severity: 'critical'
    target_match:
      severity: 'warning'
    equal: ['alertname', 'cluster', 'service']
```

### Concepts clés de routage

**group_by** : Regroupe les alertes similaires en une seule notification. Évite le spam

**group_wait** : Temps d'attente avant d'envoyer la première notification d'un nouveau groupe

**group_interval** : Temps d'attente avant d'envoyer les alertes suivantes du même groupe

**repeat_interval** : Intervalle de répétition pour les alertes non résolues

## Configuration des receivers

### Email

Configuration pour envoyer des emails via Gmail :

```yaml
receivers:
- name: 'email-notifications'
  email_configs:
  - to: 'devops-team@example.com'
    from: 'alertmanager@example.com'
    smarthost: 'smtp.gmail.com:587'
    auth_username: 'alertmanager@example.com'
    auth_password: 'app-specific-password'
    headers:
      Subject: '[{{ .GroupLabels.severity | toUpper }}] {{ .GroupLabels.alertname }}'
    html: |
      <h2>🚨 Alerte Kubernetes</h2>
      <p><b>Cluster:</b> MicroK8s Lab</p>
      {{ range .Alerts }}
      <hr>
      <p><b>Alerte:</b> {{ .Labels.alertname }}</p>
      <p><b>Sévérité:</b> {{ .Labels.severity }}</p>
      <p><b>Description:</b> {{ .Annotations.description }}</p>
      <p><b>Depuis:</b> {{ .StartsAt.Format "2006-01-02 15:04:05" }}</p>
      {{ end }}
```

### Slack

Configuration pour Slack :

```yaml
receivers:
- name: 'slack-notifications'
  slack_configs:
  - api_url: 'YOUR_SLACK_WEBHOOK_URL'
    channel: '#alerts'
    title: 'Alerte Kubernetes'
    text: '{{ range .Alerts }}{{ .Annotations.summary }}{{ end }}'
    send_resolved: true
    color: '{{ if eq .Status "firing" }}danger{{ else }}good{{ end }}'
    fields:
    - title: 'Sévérité'
      value: '{{ .GroupLabels.severity }}'
      short: true
    - title: 'Cluster'
      value: 'MicroK8s Lab'
      short: true
```

### Webhook générique

Pour intégration avec d'autres systèmes :

```yaml
receivers:
- name: 'webhook-receiver'
  webhook_configs:
  - url: 'http://example.com/alerts'
    send_resolved: true
    http_config:
      bearer_token: 'your-auth-token'
    max_alerts: 10
```

### PagerDuty

Pour les alertes critiques nécessitant une escalade :

```yaml
receivers:
- name: 'pagerduty-critical'
  pagerduty_configs:
  - service_key: 'YOUR_PAGERDUTY_SERVICE_KEY'
    description: '{{ .GroupLabels.alertname }}'
    details:
      severity: '{{ .GroupLabels.severity }}'
      namespace: '{{ .CommonLabels.namespace }}'
      pod: '{{ .CommonLabels.pod }}'
```

## Routage avancé des alertes

### Routage par sévérité

```yaml
route:
  receiver: 'default'
  group_by: ['alertname']

  routes:
  - match:
      severity: critical
    receiver: 'pagerduty-oncall'
    group_wait: 30s
    repeat_interval: 4h

  - match:
      severity: warning
    receiver: 'slack-warnings'
    group_wait: 5m
    repeat_interval: 12h

  - match_re:
      namespace: kube-.*
    receiver: 'k8s-admin-team'
```

### Routage par namespace

```yaml
route:
  receiver: 'default'

  routes:
  - match:
      namespace: production
    receiver: 'prod-team'
    routes:
    - match:
        severity: critical
      receiver: 'prod-oncall'

  - match:
      namespace: staging
    receiver: 'staging-team'

  - match:
      namespace: development
    receiver: 'dev-team'
```

### Routage temporel (heures de bureau)

Bien qu'Alertmanager ne supporte pas nativement les horaires, vous pouvez utiliser des labels dans Prometheus :

```yaml
groups:
- name: business-hours
  rules:
  - alert: NonCriticalIssue
    expr: some_metric > threshold
    labels:
      severity: warning
      time_sensitivity: business_hours
    annotations:
      summary: "Problème non critique détecté"
```

Puis router différemment :

```yaml
routes:
- match:
    time_sensitivity: business_hours
  receiver: 'email-only'

- match:
    time_sensitivity: '24x7'
  receiver: 'pagerduty'
```

## Inhibition et silence

### Règles d'inhibition

Les inhibitions empêchent certaines alertes d'être envoyées si d'autres sont déjà actives :

```yaml
inhibit_rules:
- source_match:
    severity: 'critical'
    alertname: 'NodeDown'
  target_match:
    severity: 'warning'
  equal: ['node']
  # Si un node est down, on n'envoie pas les alertes warning pour ce node

- source_match:
    alertname: 'DeploymentDown'
  target_match:
    alertname: 'PodDown'
  equal: ['namespace', 'deployment']
  # Si un deployment est down, on n'envoie pas les alertes pour ses pods
```

### Création de silences

Les silences peuvent être créées via l'interface web d'Alertmanager ou via l'API :

```bash
# Via amtool (CLI d'Alertmanager)
amtool silence add alertname="TestAlert" --duration="2h" --comment="Maintenance planifiée"

# Via l'API REST
curl -X POST http://alertmanager:9093/api/v1/silences \
  -H "Content-Type: application/json" \
  -d '{
    "matchers": [
      {"name": "alertname", "value": "DiskSpaceLow"},
      {"name": "instance", "value": "node1"}
    ],
    "startsAt": "2024-01-01T00:00:00Z",
    "endsAt": "2024-01-01T02:00:00Z",
    "createdBy": "admin",
    "comment": "Maintenance disque"
  }'
```

## Templates personnalisés

### Création de templates

Créez un fichier `/etc/alertmanager/templates/custom.tmpl` :

```go
{{ define "custom.title" }}
[{{ .Status | toUpper }}{{ if eq .Status "firing" }}:{{ .Alerts.Firing | len }}{{ end }}] {{ .GroupLabels.alertname }}
{{ end }}

{{ define "custom.slack.text" }}
{{ range .Alerts }}
*Alerte:* {{ .Labels.alertname }}
*Namespace:* {{ .Labels.namespace }}
*Pod:* {{ .Labels.pod }}
*Description:* {{ .Annotations.description }}
*Graphique:* <http://grafana.example.com/d/dashboard-id?var-namespace={{ .Labels.namespace }}|Voir dans Grafana>
{{ end }}
{{ end }}

{{ define "custom.email.html" }}
<!DOCTYPE html>
<html>
<head>
    <style>
        .alert-box {
            border: 2px solid #ff6b6b;
            padding: 10px;
            margin: 10px 0;
            border-radius: 5px;
        }
        .resolved {
            border-color: #51cf66;
        }
    </style>
</head>
<body>
    <h1>{{ template "custom.title" . }}</h1>
    {{ range .Alerts }}
    <div class="alert-box {{ if eq .Status "resolved" }}resolved{{ end }}">
        <h3>{{ .Labels.alertname }}</h3>
        <p><strong>Status:</strong> {{ .Status }}</p>
        <p><strong>Début:</strong> {{ .StartsAt.Format "02/01/2006 15:04:05" }}</p>
        {{ if .EndsAt }}
        <p><strong>Fin:</strong> {{ .EndsAt.Format "02/01/2006 15:04:05" }}</p>
        {{ end }}
        <p>{{ .Annotations.description }}</p>
    </div>
    {{ end }}
</body>
</html>
{{ end }}
```

### Utilisation des templates

```yaml
receivers:
- name: 'custom-email'
  email_configs:
  - to: 'team@example.com'
    html: '{{ template "custom.email.html" . }}'
    headers:
      Subject: '{{ template "custom.title" . }}'

- name: 'custom-slack'
  slack_configs:
  - api_url: 'YOUR_WEBHOOK_URL'
    title: '{{ template "custom.title" . }}'
    text: '{{ template "custom.slack.text" . }}'
```

## Intégration avec Grafana

### Configuration de Grafana pour Alertmanager

Dans Grafana, ajoutez Alertmanager comme source de données :

1. Configuration → Data Sources → Add data source
2. Sélectionnez "Alertmanager"
3. URL : `http://alertmanager.monitoring.svc.cluster.local:9093`
4. Sauvegardez et testez

### Visualisation des alertes dans Grafana

Grafana peut afficher les alertes actives d'Alertmanager :

1. Créez un nouveau panel
2. Choisissez la visualisation "Alert list"
3. Sélectionnez votre datasource Alertmanager
4. Configurez les filtres si nécessaire

### Annotations d'alertes sur les graphiques

Ajoutez des annotations pour voir quand les alertes se sont déclenchées :

```json
{
  "datasource": "Prometheus",
  "enable": true,
  "expr": "ALERTS{alertname=\"HighCPUUsage\"}",
  "iconColor": "red",
  "name": "High CPU Alerts",
  "step": "60s",
  "tagKeys": "severity,namespace",
  "textFormat": "{{ alertname }}"
}
```

## Monitoring d'Alertmanager

### Métriques d'Alertmanager

Alertmanager expose ses propres métriques :

```promql
# Nombre d'alertes reçues
rate(alertmanager_alerts_received_total[5m])

# Nombre de notifications envoyées
rate(alertmanager_notifications_sent_total[5m])

# Erreurs d'envoi de notifications
rate(alertmanager_notifications_failed_total[5m])

# Nombre de silences actifs
alertmanager_silences
```

### Alertes sur Alertmanager lui-même

```yaml
groups:
- name: alertmanager-monitoring
  rules:
  - alert: AlertmanagerNotificationsFailing
    expr: rate(alertmanager_notifications_failed_total[5m]) > 0
    for: 5m
    labels:
      severity: critical
      component: alerting
    annotations:
      summary: "Alertmanager n'arrive pas à envoyer les notifications"
      description: "{{ $value }} notifications ont échoué dans les 5 dernières minutes"

  - alert: AlertmanagerConfigReloadFailed
    expr: alertmanager_config_last_reload_successful == 0
    for: 5m
    labels:
      severity: warning
    annotations:
      summary: "Rechargement de la configuration Alertmanager échoué"
      description: "La dernière tentative de rechargement a échoué"
```

## Bonnes pratiques

### Conception des alertes

**Évitez le bruit** : Utilisez le paramètre `for` pour éviter les alertes transitoires
```yaml
for: 5m  # L'alerte doit être vraie pendant 5 minutes
```

**Soyez spécifique** : Incluez suffisamment de labels pour identifier le problème
```yaml
labels:
  namespace: "{{ $labels.namespace }}"
  pod: "{{ $labels.pod }}"
  container: "{{ $labels.container }}"
```

**Messages actionnables** : Les annotations doivent dire quoi faire
```yaml
annotations:
  summary: "Espace disque faible sur {{ $labels.node }}"
  description: "Il reste {{ $value | humanize }}% d'espace"
  runbook_url: "https://wiki.example.com/runbooks/disk-space"
```

### Organisation des règles

**Groupez logiquement** : Organisez vos règles par composant
```
rules/
├── kubernetes-pods.yml
├── kubernetes-nodes.yml
├── kubernetes-storage.yml
├── application-specific.yml
└── infrastructure.yml
```

**Versionnez** : Gardez vos règles dans Git pour tracer les changements

**Testez** : Validez vos règles avant de les déployer
```bash
promtool check rules rules/*.yml
```

### Gestion de la fatigue d'alerte

**Groupement intelligent** : Groupez les alertes liées
```yaml
group_by: ['alertname', 'namespace', 'deployment']
```

**Intervalles appropriés** : Ajustez selon la criticité
```yaml
routes:
- match:
    severity: critical
  repeat_interval: 4h
- match:
    severity: warning
  repeat_interval: 24h
```

**Inhibitions pertinentes** : Évitez les alertes redondantes
```yaml
inhibit_rules:
- source_match:
    alertname: ClusterDown
  target_match_re:
    alertname: .*
  equal: ['cluster']
```

## Dépannage courant

### Les alertes ne se déclenchent pas

**Vérifiez la règle** : Testez la requête PromQL dans Prometheus
```
http://prometheus:9090/graph
```

**Vérifiez la configuration** : Les règles sont-elles chargées ?
```bash
curl http://prometheus:9090/api/v1/rules | jq
```

**Vérifiez les logs** : Prometheus indique les erreurs d'évaluation
```bash
kubectl logs -n monitoring prometheus-0 | grep error
```

### Les notifications ne sont pas envoyées

**Vérifiez Alertmanager** : L'alerte arrive-t-elle ?
```
http://alertmanager:9093/#/alerts
```

**Vérifiez le routage** : La route est-elle correcte ?
```bash
amtool config routes --config.file=alertmanager.yml
```

**Vérifiez les logs** : Erreurs d'envoi ?
```bash
kubectl logs -n monitoring alertmanager-0 | grep error
```

### Trop d'alertes (spam)

**Augmentez le `for`** : Attendez plus longtemps avant d'alerter
```yaml
for: 10m  # Au lieu de 1m
```

**Ajustez les seuils** : Sont-ils trop sensibles ?
```yaml
expr: cpu_usage > 0.9  # Au lieu de > 0.7
```

**Améliorez le groupement** : Regroupez mieux les alertes
```yaml
group_by: ['alertname', 'namespace', 'severity']
group_interval: 5m
```

## Conclusion

L'alerting avec Prometheus et Alertmanager est un pilier fondamental de l'observabilité dans Kubernetes. Dans votre lab MicroK8s, un système d'alerting bien configuré vous permet de détecter et résoudre les problèmes avant qu'ils n'impactent vos applications.

Commencez avec quelques alertes essentielles (pods crashant, nodes down, espace disque) puis enrichissez progressivement selon vos besoins. N'oubliez pas que trop d'alertes est aussi problématique que pas assez - visez la qualité plutôt que la quantité.

L'objectif final est d'avoir un système qui vous alerte uniquement quand une action humaine est nécessaire, avec suffisamment de contexte pour résoudre rapidement le problème. Avec le temps et l'expérience, vous affinerez vos règles pour atteindre cet équilibre optimal.

⏭️
