🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 8.12 Notifications (Slack, email, webhook)

## Introduction aux notifications

Les notifications sont le point final de votre chaîne de monitoring. Vous avez configuré Prometheus pour collecter les métriques, créé des règles d'alerte pour détecter les problèmes, et maintenant il faut que quelqu'un soit prévenu quand quelque chose ne va pas. C'est là que les notifications entrent en jeu.

Dans un environnement MicroK8s, vous voulez être informé rapidement si un pod crash, si l'espace disque se remplit, ou si vos applications ne répondent plus. Mais attention : mal configurées, les notifications peuvent devenir du spam qui finit ignoré. L'objectif est d'envoyer les bonnes informations, aux bonnes personnes, au bon moment.

## Vue d'ensemble des canaux de notification

### Canaux disponibles dans Alertmanager

Alertmanager supporte nativement plusieurs canaux de notification :

**Email** : Le classique, idéal pour les alertes non-urgentes et les rapports détaillés. Permet des messages riches en HTML avec tableaux et liens.

**Slack** : Parfait pour les équipes utilisant déjà Slack. Les alertes arrivent directement dans vos canaux, avec formatting et couleurs.

**Webhook** : Le plus flexible. Permet d'intégrer avec n'importe quel système qui accepte des requêtes HTTP.

**PagerDuty** : Pour les alertes critiques nécessitant une escalade et un suivi d'astreinte.

**OpsGenie** : Alternative à PagerDuty avec des fonctionnalités similaires.

**Pushover** : Notifications push sur mobile, simple et efficace.

**VictorOps** : Autre système de gestion d'incidents.

**WeChat** : Populaire en Asie, intégration native disponible.

**Telegram** : Via webhook, populaire pour les projets personnels.

**Microsoft Teams** : Via webhook, pour les environnements Microsoft.

## Configuration Email détaillée

### Prérequis pour l'email

Avant de configurer les notifications email, vous aurez besoin :

1. **Un serveur SMTP** : Soit votre propre serveur, soit un service comme Gmail, SendGrid, ou Mailgun
2. **Des credentials** : Username et password (ou app password pour Gmail)
3. **L'adresse d'envoi** : L'adresse qui apparaîtra comme expéditeur
4. **Les destinataires** : Les adresses qui recevront les alertes

### Configuration SMTP globale

Dans votre `alertmanager.yml`, commencez par la configuration globale :

```yaml
global:
  # Adresse email d'envoi
  smtp_from: 'alertmanager@votredomaine.com'

  # Serveur SMTP et port
  smtp_smarthost: 'smtp.gmail.com:587'

  # Authentification
  smtp_auth_username: 'votre-email@gmail.com'
  smtp_auth_password: 'votre-app-password'  # Pas votre mot de passe Gmail normal!

  # Paramètres de sécurité
  smtp_require_tls: true

  # Headers additionnels (optionnel)
  smtp_headers:
    X-Priority: '1'  # Haute priorité
    X-MSMail-Priority: 'High'
```

### Configuration Gmail spécifique

Pour Gmail, vous devez utiliser un "App Password" et non votre mot de passe normal :

1. Activez la validation en deux étapes sur votre compte Google
2. Allez dans les paramètres de sécurité Google
3. Générez un "App Password" pour Alertmanager
4. Utilisez ce password dans la configuration

```yaml
global:
  smtp_from: 'alertes.k8s@gmail.com'
  smtp_smarthost: 'smtp.gmail.com:587'
  smtp_auth_username: 'alertes.k8s@gmail.com'
  smtp_auth_password: 'abcd efgh ijkl mnop'  # App password à 16 caractères
  smtp_require_tls: true
```

### Configuration SendGrid

Pour un service professionnel comme SendGrid :

```yaml
global:
  smtp_from: 'noreply@votredomaine.com'
  smtp_smarthost: 'smtp.sendgrid.net:587'
  smtp_auth_username: 'apikey'  # Littéralement "apikey"
  smtp_auth_password: 'SG.VotreClefAPI...'  # Votre clef API SendGrid
  smtp_require_tls: true
```

### Receiver email simple

Configuration basique d'un receiver email :

```yaml
receivers:
- name: 'email-ops'
  email_configs:
  - to: 'ops-team@example.com'
    send_resolved: true  # Envoie aussi quand l'alerte est résolue
```

### Receiver email avec plusieurs destinataires

```yaml
receivers:
- name: 'email-multiple'
  email_configs:
  - to: 'admin1@example.com,admin2@example.com,admin3@example.com'
    send_resolved: true
```

### Email avec template HTML personnalisé

```yaml
receivers:
- name: 'email-html'
  email_configs:
  - to: 'team@example.com'
    send_resolved: true
    headers:
      Subject: '🚨 [{{ .Status | toUpper }}] {{ .GroupLabels.alertname }} - MicroK8s Lab'
    html: |
      <!DOCTYPE html>
      <html>
      <head>
        <style>
          body { font-family: Arial, sans-serif; }
          .header {
            background-color: {{ if eq .Status "firing" }}#ff4444{{ else }}#44ff44{{ end }};
            color: white;
            padding: 20px;
            text-align: center;
          }
          .alert-box {
            border: 1px solid #ddd;
            margin: 10px;
            padding: 15px;
            border-radius: 5px;
          }
          .label {
            display: inline-block;
            padding: 2px 8px;
            margin: 2px;
            background: #e0e0e0;
            border-radius: 3px;
            font-size: 12px;
          }
          .critical { background: #ff4444; color: white; }
          .warning { background: #ffaa00; color: white; }
          .info { background: #4444ff; color: white; }
          table { width: 100%; border-collapse: collapse; }
          th, td { padding: 8px; text-align: left; border-bottom: 1px solid #ddd; }
          th { background-color: #f2f2f2; }
        </style>
      </head>
      <body>
        <div class="header">
          <h1>{{ if eq .Status "firing" }}🔥 ALERTES ACTIVES{{ else }}✅ ALERTES RÉSOLUES{{ end }}</h1>
          <p>Cluster: MicroK8s Lab | Nombre d'alertes: {{ .Alerts | len }}</p>
        </div>

        <h2>Résumé</h2>
        <table>
          <tr>
            <th>Alerte</th>
            <th>Sévérité</th>
            <th>Nombre</th>
            <th>Depuis</th>
          </tr>
          {{ range .Alerts }}
          <tr>
            <td><strong>{{ .Labels.alertname }}</strong></td>
            <td><span class="label {{ .Labels.severity }}">{{ .Labels.severity | toUpper }}</span></td>
            <td>1</td>
            <td>{{ .StartsAt.Format "02/01 15:04" }}</td>
          </tr>
          {{ end }}
        </table>

        <h2>Détails des alertes</h2>
        {{ range .Alerts }}
        <div class="alert-box">
          <h3>{{ .Labels.alertname }}</h3>

          <p><strong>Description:</strong> {{ .Annotations.description }}</p>

          <p><strong>Labels:</strong>
          {{ range $key, $value := .Labels }}
            <span class="label">{{ $key }}: {{ $value }}</span>
          {{ end }}
          </p>

          <p><strong>Début:</strong> {{ .StartsAt.Format "02/01/2006 15:04:05" }}</p>
          {{ if .EndsAt }}
          <p><strong>Fin:</strong> {{ .EndsAt.Format "02/01/2006 15:04:05" }}</p>
          {{ end }}

          {{ if .Annotations.runbook_url }}
          <p><strong>Documentation:</strong> <a href="{{ .Annotations.runbook_url }}">Runbook</a></p>
          {{ end }}

          {{ if .GeneratorURL }}
          <p><strong>Source:</strong> <a href="{{ .GeneratorURL }}">Voir dans Prometheus</a></p>
          {{ end }}
        </div>
        {{ end }}

        <hr>
        <p style="text-align: center; color: #666; font-size: 12px;">
          Envoyé par Alertmanager | {{ now.Format "02/01/2006 15:04:05" }}
        </p>
      </body>
      </html>
```

### Configuration avancée email

```yaml
receivers:
- name: 'email-advanced'
  email_configs:
  - to: 'primary@example.com'
    send_resolved: true

    # Email différent pour les alertes critiques
    - to: 'oncall@example.com'
      match:
        severity: critical

    # Configuration TLS personnalisée
    tls_config:
      insecure_skip_verify: false  # Ne jamais mettre true en production!

    # Headers personnalisés
    headers:
      Subject: '[K8S-{{ .GroupLabels.cluster }}] {{ .GroupLabels.alertname }}'
      X-Alert-Status: '{{ .Status }}'
      X-Alert-Severity: '{{ .GroupLabels.severity }}'
      Reply-To: 'no-reply@example.com'
```

## Configuration Slack détaillée

### Prérequis Slack

Pour configurer Slack, vous avez deux options :

**Option 1 : Webhook URL (Simple)**
1. Allez dans votre workspace Slack
2. Ajoutez l'app "Incoming Webhooks"
3. Choisissez un canal
4. Copiez l'URL du webhook

**Option 2 : Slack App (Avancé)**
1. Créez une app Slack
2. Activez les webhooks
3. Installez l'app dans votre workspace
4. Utilisez le token OAuth

### Configuration Slack basique

```yaml
receivers:
- name: 'slack-notifications'
  slack_configs:
  - api_url: 'https://hooks.slack.com/services/T00000000/B00000000/XXXXXXXXXXXXXXXXXXXXXXXX'
    channel: '#alerts'
    send_resolved: true
```

### Configuration Slack avec formatting

```yaml
receivers:
- name: 'slack-formatted'
  slack_configs:
  - api_url: 'YOUR_WEBHOOK_URL'
    channel: '#alerts-k8s'
    send_resolved: true

    # Titre du message
    title: '{{ if eq .Status "firing" }}🚨{{ else }}✅{{ end }} [{{ .Status | toUpper }}] {{ .GroupLabels.alertname }}'
    title_link: '{{ .GeneratorURL }}'  # Lien vers Prometheus

    # Texte principal
    text: |
      {{ range .Alerts }}
      *Alerte:* {{ .Labels.alertname }}
      *Sévérité:* {{ .Labels.severity }}
      *Description:* {{ .Annotations.description }}
      *Namespace:* {{ .Labels.namespace }}
      *Pod:* {{ .Labels.pod }}
      {{ end }}

    # Couleur de la barre latérale
    color: '{{ if eq .Status "firing" }}{{ if eq .GroupLabels.severity "critical" }}danger{{ else }}warning{{ end }}{{ else }}good{{ end }}'

    # Texte du fallback (pour les notifications)
    fallback: '{{ .GroupLabels.alertname }} - {{ .Status }}'

    # Préfixe pour le nom d'utilisateur
    username: 'Alertmanager'

    # Icône du bot
    icon_emoji: ':rotating_light:'

    # Footer
    footer: 'MicroK8s Cluster'
    footer_icon: 'https://microk8s.io/favicon.ico'
```

### Slack avec fields structurés

```yaml
receivers:
- name: 'slack-fields'
  slack_configs:
  - api_url: 'YOUR_WEBHOOK_URL'
    channel: '#alerts'
    send_resolved: true

    title: 'Alerte Kubernetes'

    # Utilisation de fields pour une présentation structurée
    fields:
    - title: 'Alerte'
      value: '{{ .GroupLabels.alertname }}'
      short: true

    - title: 'Sévérité'
      value: '{{ .GroupLabels.severity }}'
      short: true

    - title: 'Cluster'
      value: 'MicroK8s-Lab'
      short: true

    - title: 'Environnement'
      value: '{{ .GroupLabels.env | default "production" }}'
      short: true

    - title: 'Namespace'
      value: '{{ .CommonLabels.namespace | default "N/A" }}'
      short: true

    - title: 'Nombre d\'alertes'
      value: '{{ .Alerts | len }}'
      short: true

    - title: 'Description'
      value: '{{ range .Alerts }}{{ .Annotations.description }}{{ end }}'
      short: false
```

### Slack avec actions (boutons)

```yaml
receivers:
- name: 'slack-actions'
  slack_configs:
  - api_url: 'YOUR_WEBHOOK_URL'
    channel: '#alerts'
    send_resolved: true

    title: '{{ .GroupLabels.alertname }}'
    text: '{{ .Annotations.description }}'

    # Actions (boutons cliquables)
    actions:
    - type: 'button'
      text: '📊 Grafana'
      url: 'http://grafana.example.com/d/k8s-overview?var-namespace={{ .GroupLabels.namespace }}'

    - type: 'button'
      text: '📝 Runbook'
      url: '{{ .Annotations.runbook_url }}'
      style: 'primary'

    - type: 'button'
      text: '🔇 Silence'
      url: 'http://alertmanager.example.com/#/silences/new?filter={alertname="{{ .GroupLabels.alertname }}"}'
      style: 'danger'
```

### Configuration multi-canal Slack

```yaml
route:
  receiver: 'default'
  routes:
  - match:
      severity: critical
    receiver: 'slack-critical'

  - match:
      severity: warning
    receiver: 'slack-warnings'

receivers:
- name: 'slack-critical'
  slack_configs:
  - api_url: 'WEBHOOK_URL_CRITICAL'
    channel: '#alerts-critical'
    username: 'Critical Alert'
    icon_emoji: ':fire:'

- name: 'slack-warnings'
  slack_configs:
  - api_url: 'WEBHOOK_URL_WARNINGS'
    channel: '#alerts-warnings'
    username: 'Warning Alert'
    icon_emoji: ':warning:'
```

## Configuration Webhook détaillée

### Comprendre les webhooks

Un webhook est simplement une requête HTTP POST envoyée à une URL de votre choix. C'est le moyen le plus flexible d'intégrer Alertmanager avec n'importe quel système.

### Structure du payload webhook

Alertmanager envoie un JSON structuré comme ceci :

```json
{
  "version": "4",
  "groupKey": "{}:{alertname=\"HighCPU\"}",
  "status": "firing",
  "receiver": "webhook-receiver",
  "groupLabels": {
    "alertname": "HighCPU"
  },
  "commonLabels": {
    "alertname": "HighCPU",
    "severity": "warning",
    "namespace": "default"
  },
  "commonAnnotations": {
    "description": "CPU usage is above 80%"
  },
  "externalURL": "http://alertmanager.example.com",
  "alerts": [
    {
      "status": "firing",
      "labels": {
        "alertname": "HighCPU",
        "severity": "warning",
        "namespace": "default",
        "pod": "app-xyz"
      },
      "annotations": {
        "description": "CPU usage is above 80%"
      },
      "startsAt": "2024-01-15T10:00:00Z",
      "endsAt": "0001-01-01T00:00:00Z",
      "generatorURL": "http://prometheus.example.com/..."
    }
  ]
}
```

### Configuration webhook basique

```yaml
receivers:
- name: 'webhook-basic'
  webhook_configs:
  - url: 'http://votre-service.com/alerts'
    send_resolved: true
```

### Webhook avec authentification

```yaml
receivers:
- name: 'webhook-auth'
  webhook_configs:
  - url: 'https://api.exemple.com/v1/alerts'
    send_resolved: true

    # Authentification Bearer Token
    http_config:
      bearer_token: 'votre-token-secret'

    # Ou authentification Basic
    http_config:
      basic_auth:
        username: 'alertmanager'
        password: 'mot-de-passe-secret'

    # Headers personnalisés
      headers:
        X-Custom-Header: 'valeur'
        X-Environment: 'production'
```

### Webhook avec retry et timeout

```yaml
receivers:
- name: 'webhook-resilient'
  webhook_configs:
  - url: 'https://api.exemple.com/alerts'
    send_resolved: true

    # Nombre maximum d'alertes par requête
    max_alerts: 10

    http_config:
      # Timeout de la requête
      timeout: '30s'

      # Configuration TLS
      tls_config:
        insecure_skip_verify: false

      # Proxy si nécessaire
      proxy_url: 'http://proxy.company.com:8080'
```

### Webhook vers Microsoft Teams

Microsoft Teams nécessite un format spécifique :

```yaml
receivers:
- name: 'teams-webhook'
  webhook_configs:
  - url: 'https://outlook.office.com/webhook/YOUR-TEAMS-WEBHOOK-URL'
    send_resolved: true
    http_config:
      headers:
        Content-Type: 'application/json'
```

Avec un adaptateur pour transformer le format (vous devrez déployer un service intermédiaire) :

```javascript
// Exemple de service adaptateur Node.js
app.post('/teams-adapter', (req, res) => {
  const alert = req.body;

  const teamsMessage = {
    "@type": "MessageCard",
    "@context": "http://schema.org/extensions",
    "themeColor": alert.status === "firing" ? "FF0000" : "00FF00",
    "summary": `Alert: ${alert.groupLabels.alertname}`,
    "sections": [{
      "activityTitle": alert.groupLabels.alertname,
      "activitySubtitle": `Status: ${alert.status}`,
      "facts": alert.alerts.map(a => ({
        "name": a.labels.namespace,
        "value": a.annotations.description
      })),
      "markdown": true
    }],
    "potentialAction": [{
      "@type": "OpenUri",
      "name": "View in Alertmanager",
      "targets": [{
        "os": "default",
        "uri": alert.externalURL
      }]
    }]
  };

  // Envoyer à Teams
  axios.post(TEAMS_WEBHOOK_URL, teamsMessage);
  res.sendStatus(200);
});
```

### Webhook vers Telegram

Pour Telegram, utilisez un bot et l'API Telegram :

```yaml
receivers:
- name: 'telegram-webhook'
  webhook_configs:
  - url: 'http://telegram-adapter:8080/alert'
    send_resolved: true
```

Service adaptateur pour Telegram :

```python
# Exemple Python avec Flask
from flask import Flask, request
import requests

app = Flask(__name__)

TELEGRAM_BOT_TOKEN = "YOUR_BOT_TOKEN"
TELEGRAM_CHAT_ID = "YOUR_CHAT_ID"

@app.route('/alert', methods=['POST'])
def handle_alert():
    data = request.json

    # Formater le message
    status_emoji = "🔥" if data['status'] == 'firing' else "✅"
    message = f"{status_emoji} *{data['groupLabels']['alertname']}*\n"
    message += f"Status: {data['status']}\n"

    for alert in data['alerts']:
        message += f"\n📍 *{alert['labels'].get('namespace', 'N/A')}/{alert['labels'].get('pod', 'N/A')}*\n"
        message += f"{alert['annotations']['description']}\n"

    # Envoyer à Telegram
    telegram_url = f"https://api.telegram.org/bot{TELEGRAM_BOT_TOKEN}/sendMessage"
    requests.post(telegram_url, json={
        'chat_id': TELEGRAM_CHAT_ID,
        'text': message,
        'parse_mode': 'Markdown'
    })

    return '', 200
```

### Webhook vers Discord

Discord utilise aussi des webhooks mais avec un format spécifique :

```yaml
receivers:
- name: 'discord-webhook'
  webhook_configs:
  - url: 'http://discord-adapter:8080/alert'
    send_resolved: true
```

Adaptateur pour Discord :

```javascript
// Exemple Node.js pour Discord
app.post('/alert', async (req, res) => {
  const alert = req.body;

  const embed = {
    title: alert.groupLabels.alertname,
    color: alert.status === 'firing' ? 0xFF0000 : 0x00FF00,
    fields: alert.alerts.map(a => ({
      name: `${a.labels.namespace}/${a.labels.pod || 'N/A'}`,
      value: a.annotations.description,
      inline: false
    })),
    footer: {
      text: `Alertmanager - ${new Date().toISOString()}`
    }
  };

  await axios.post(DISCORD_WEBHOOK_URL, {
    embeds: [embed]
  });

  res.sendStatus(200);
});
```

## Intégration avec des services tiers

### PagerDuty

Pour les alertes critiques avec escalade :

```yaml
receivers:
- name: 'pagerduty'
  pagerduty_configs:
  - service_key: 'YOUR-PAGERDUTY-SERVICE-KEY'

    # URL personnalisée si vous utilisez PagerDuty EU
    url: 'https://events.eu.pagerduty.com/v2/enqueue'

    # Détails enrichis
    details:
      cluster: 'microk8s-lab'
      environment: '{{ .GroupLabels.env }}'
      namespace: '{{ .CommonLabels.namespace }}'
      severity: '{{ .GroupLabels.severity }}'

    # Composant et groupe
    component: '{{ .GroupLabels.service }}'
    group: '{{ .GroupLabels.cluster }}'

    # Classe d'événement
    class: '{{ .GroupLabels.alertname }}'
```

### OpsGenie

Alternative à PagerDuty :

```yaml
receivers:
- name: 'opsgenie'
  opsgenie_configs:
  - api_key: 'YOUR-OPSGENIE-API-KEY'

    # Message et description
    message: '{{ .GroupLabels.alertname }}'
    description: '{{ range .Alerts }}{{ .Annotations.description }}{{ end }}'

    # Source de l'alerte
    source: 'Prometheus-MicroK8s'

    # Tags
    tags: 'kubernetes,{{ .GroupLabels.severity }},{{ .GroupLabels.namespace }}'

    # Priorité basée sur la sévérité
    priority: |
      {{ if eq .GroupLabels.severity "critical" }}P1{{ else if eq .GroupLabels.severity "warning" }}P3{{ else }}P5{{ end }}

    # Détails additionnels
    details:
      cluster: 'microk8s-lab'
      alertCount: '{{ .Alerts | len }}'
```

### Pushover (notifications mobile)

Pour des notifications push directes sur mobile :

```yaml
receivers:
- name: 'pushover'
  pushover_configs:
  - user_key: 'YOUR-USER-KEY'
    token: 'YOUR-APP-TOKEN'

    # Titre et message
    title: '{{ .GroupLabels.alertname }}'
    message: '{{ .Annotations.description }}'

    # URL pour plus de détails
    url: '{{ .GeneratorURL }}'
    url_title: 'View in Prometheus'

    # Priorité (-2 à 2)
    priority: |
      {{ if eq .GroupLabels.severity "critical" }}2{{ else if eq .GroupLabels.severity "warning" }}0{{ else }}-1{{ end }}

    # Son de notification
    sound: |
      {{ if eq .GroupLabels.severity "critical" }}siren{{ else }}pushover{{ end }}
```

## Templates réutilisables

### Création de templates partagés

Créez un fichier `/etc/alertmanager/templates/notifications.tmpl` :

```go
{{ define "cluster.name" }}MicroK8s-Lab{{ end }}

{{ define "slack.title" }}
[{{ .Status | toUpper }}{{ if eq .Status "firing" }}:{{ .Alerts.Firing | len }}{{ end }}] {{ .GroupLabels.alertname }}
{{ end }}

{{ define "slack.text" }}
{{ range .Alerts }}
*Namespace:* `{{ .Labels.namespace }}`
*Pod:* `{{ .Labels.pod | default "N/A" }}`
*Node:* `{{ .Labels.node | default "N/A" }}`
*Description:* {{ .Annotations.description }}
*Started:* {{ .StartsAt.Format "15:04:05 MST" }}
{{ if .EndsAt }}*Ended:* {{ .EndsAt.Format "15:04:05 MST" }}{{ end }}
{{ end }}
{{ end }}

{{ define "email.subject" }}
[{{ template "cluster.name" . }}] {{ .GroupLabels.alertname }} ({{ .Status }})
{{ end }}

{{ define "webhook.summary" }}
{
  "cluster": "{{ template "cluster.name" . }}",
  "alert": "{{ .GroupLabels.alertname }}",
  "status": "{{ .Status }}",
  "severity": "{{ .GroupLabels.severity }}",
  "count": {{ .Alerts | len }},
  "time": "{{ now.Format "2006-01-02T15:04:05Z07:00" }}"
}
{{ end }}

{{ define "priority.level" }}
{{- if eq .GroupLabels.severity "critical" -}}
1
{{- else if eq .GroupLabels.severity "warning" -}}
2
{{- else -}}
3
{{- end -}}
{{ end }}

{{ define "color.code" }}
{{- if eq .Status "firing" -}}
  {{- if eq .GroupLabels.severity "critical" -}}
    #FF0000
  {{- else if eq .GroupLabels.severity "warning" -}}
    #FFA500
  {{- else -}}
    #808080
  {{- end -}}
{{- else -}}
  #00FF00
{{- end -}}
{{ end }}
```

### Utilisation des templates

```yaml
# Charger les templates
templates:
- '/etc/alertmanager/templates/*.tmpl'

receivers:
- name: 'slack-templated'
  slack_configs:
  - api_url: 'YOUR_WEBHOOK'
    title: '{{ template "slack.title" . }}'
    text: '{{ template "slack.text" . }}'
    color: '{{ template "color.code" . }}'

- name: 'email-templated'
  email_configs:
  - to: 'team@example.com'
    headers:
      Subject: '{{ template "email.subject" . }}'
```

## Stratégies de notification

### Notification progressive

Différents canaux selon l'urgence :

```yaml
route:
  receiver: 'default'
  routes:
  # Critique : Toutes les notifications
  - match:
      severity: critical
    receiver: 'all-channels'

  # Warning : Email seulement en journée
  - match:
      severity: warning
    receiver: 'email-only'

  # Info : Slack uniquement
  - match:
      severity: info
    receiver: 'slack-only'

receivers:
- name: 'all-channels'
  email_configs:
  - to: 'oncall@example.com'
  slack_configs:
  - api_url: 'WEBHOOK_URL'
    channel: '#alerts-critical'
  pagerduty_configs:
  - service_key: 'KEY'

- name: 'email-only'
  email_configs:
  - to: 'team@example.com'

- name: 'slack-only'
  slack_configs:
  - api_url: 'WEBHOOK_URL'
    channel: '#alerts-info'
```

### Escalade temporelle

Augmenter la sévérité si non résolu :

```yaml
route:
  receiver: 'email'
  group_wait: 30s
  group_interval: 5m
  repeat_interval: 4h

  routes:
  # Première notification : Email
  - match:
      severity: warning
    receiver: 'email'
    continue: true

  # Si toujours actif après 30 min : Slack
  - match:
      severity: warning
    receiver: 'slack'
    group_wait: 30m
    continue: true

  # Si toujours actif après 1h : PagerDuty
  - match:
      severity: warning
    receiver: 'pagerduty'
    group_wait: 1h

### Notification par équipe

Routing basé sur les labels :

```yaml
route:
  receiver: 'default'
  routes:
  # Équipe Backend
  - match_re:
      service: '(api|backend|database)'
    receiver: 'backend-team'

  # Équipe Frontend
  - match_re:
      service: '(web|frontend|cdn)'
    receiver: 'frontend-team'

  # Équipe Infrastructure
  - match_re:
      component: '(node|network|storage)'
    receiver: 'infra-team'

receivers:
- name: 'backend-team'
  slack_configs:
  - api_url: 'BACKEND_WEBHOOK'
    channel: '#backend-alerts'
  email_configs:
  - to: 'backend@example.com'

- name: 'frontend-team'
  slack_configs:
  - api_url: 'FRONTEND_WEBHOOK'
    channel: '#frontend-alerts'
  email_configs:
  - to: 'frontend@example.com'

- name: 'infra-team'
  slack_configs:
  - api_url: 'INFRA_WEBHOOK'
    channel: '#infra-alerts'
  email_configs:
  - to: 'infra@example.com'
```

## Gestion des failures

### Retry et fallback

Configuration de la résilience :

```yaml
global:
  # Timeout pour résoudre les alertes
  resolve_timeout: 5m

  # Configuration HTTP globale
  http_config:
    # Suivre les redirections
    follow_redirects: true

    # Timeout des requêtes HTTP
    timeout: 10s

receivers:
- name: 'primary-with-fallback'
  # Canal principal
  slack_configs:
  - api_url: 'PRIMARY_WEBHOOK'
    send_resolved: true

  # Fallback si Slack échoue
  email_configs:
  - to: 'alerts@example.com'
    send_resolved: true
```

### Monitoring des notifications

Créez des alertes pour surveiller Alertmanager lui-même :

```yaml
groups:
- name: alertmanager-health
  rules:
  - alert: NotificationsFailing
    expr: |
      rate(alertmanager_notifications_failed_total[5m]) > 0
    for: 5m
    labels:
      severity: critical
      component: alerting
    annotations:
      summary: "Notifications Alertmanager en échec"
      description: "{{ $value | humanizePercentage }} des notifications échouent"

  - alert: NotificationLatencyHigh
    expr: |
      histogram_quantile(0.99, rate(alertmanager_notification_latency_seconds_bucket[5m])) > 5
    for: 10m
    labels:
      severity: warning
    annotations:
      summary: "Latence élevée des notifications"
      description: "P99 latency: {{ $value }}s"
```

### Dead Letter Queue

Implémenter un système de fallback avec webhook :

```python
# Service de Dead Letter Queue
from flask import Flask, request
import json
import logging
from datetime import datetime

app = Flask(__name__)

# Stockage local des alertes non envoyées
FAILED_ALERTS = []

@app.route('/dlq', methods=['POST'])
def dead_letter_queue():
    alert = request.json

    # Ajouter timestamp et tenter de renvoyer
    alert['dlq_received'] = datetime.now().isoformat()
    FAILED_ALERTS.append(alert)

    # Log pour investigation
    logging.error(f"Alert in DLQ: {json.dumps(alert)}")

    # Tenter un envoi alternatif (email de secours)
    try:
        send_emergency_email(alert)
    except:
        pass  # Ne pas faire échouer le DLQ

    # Sauvegarder sur disque
    with open('/var/log/failed_alerts.json', 'a') as f:
        f.write(json.dumps(alert) + '\n')

    return '', 200

@app.route('/dlq/retry', methods=['POST'])
def retry_failed():
    # Endpoint pour rejouer les alertes échouées
    for alert in FAILED_ALERTS:
        # Logique de retry
        pass
    return json.dumps({"retried": len(FAILED_ALERTS)})
```

## Sécurité des notifications

### Protection des secrets

Ne jamais mettre les secrets en clair dans les ConfigMaps :

```yaml
# Utiliser des Secrets Kubernetes
apiVersion: v1
kind: Secret
metadata:
  name: alertmanager-secrets
  namespace: monitoring
type: Opaque
stringData:
  slack-webhook: "https://hooks.slack.com/services/SECRET"
  email-password: "mot-de-passe-secret"
  pagerduty-key: "service-key-secret"
```

Référencer les secrets dans le deployment :

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: alertmanager
spec:
  template:
    spec:
      containers:
      - name: alertmanager
        env:
        - name: SLACK_WEBHOOK_URL
          valueFrom:
            secretKeyRef:
              name: alertmanager-secrets
              key: slack-webhook
        - name: EMAIL_PASSWORD
          valueFrom:
            secretKeyRef:
              name: alertmanager-secrets
              key: email-password
```

### Chiffrement TLS

Toujours utiliser HTTPS pour les webhooks :

```yaml
receivers:
- name: 'secure-webhook'
  webhook_configs:
  - url: 'https://api.example.com/alerts'  # HTTPS obligatoire
    http_config:
      tls_config:
        # Certificat client si requis
        cert_file: /etc/alertmanager/certs/client.crt
        key_file: /etc/alertmanager/certs/client.key

        # CA pour validation
        ca_file: /etc/alertmanager/certs/ca.crt

        # Ne jamais désactiver en production
        insecure_skip_verify: false
```

### Authentification et autorisation

Protéger vos endpoints webhook :

```javascript
// Middleware d'authentification pour webhook
app.use('/webhook', (req, res, next) => {
  const token = req.headers['x-auth-token'];

  if (!token || token !== process.env.WEBHOOK_SECRET) {
    return res.status(401).json({ error: 'Unauthorized' });
  }

  // Vérifier la signature HMAC si disponible
  const signature = req.headers['x-alertmanager-signature'];
  if (signature) {
    const computed = crypto
      .createHmac('sha256', process.env.HMAC_SECRET)
      .update(JSON.stringify(req.body))
      .digest('hex');

    if (signature !== computed) {
      return res.status(403).json({ error: 'Invalid signature' });
    }
  }

  next();
});
```

## Test des notifications

### Test manuel avec amtool

Tester vos configurations :

```bash
# Installer amtool
wget https://github.com/prometheus/alertmanager/releases/download/v0.26.0/amtool

# Tester une alerte
amtool alert add \
  alertname="TestAlert" \
  severity="warning" \
  namespace="default" \
  --annotation="description=Ceci est un test" \
  --alertmanager.url=http://alertmanager:9093

# Vérifier les alertes actives
amtool alert query --alertmanager.url=http://alertmanager:9093

# Tester un receiver spécifique
amtool config routes test \
  --config.file=/etc/alertmanager/alertmanager.yml \
  severity=critical namespace=production
```

### Test avec curl

Envoyer une alerte de test directement :

```bash
# Créer une alerte de test
curl -X POST http://alertmanager:9093/api/v1/alerts \
  -H "Content-Type: application/json" \
  -d '[
    {
      "labels": {
        "alertname": "TestNotification",
        "severity": "warning",
        "namespace": "default",
        "pod": "test-pod-123"
      },
      "annotations": {
        "description": "Test de notification - veuillez ignorer",
        "summary": "Alerte de test"
      },
      "startsAt": "'$(date -u +%Y-%m-%dT%H:%M:%SZ)'",
      "generatorURL": "http://prometheus:9090/test"
    }
  ]'
```

### Script de test complet

```bash
#!/bin/bash
# test-notifications.sh

ALERTMANAGER_URL="http://alertmanager:9093"

# Fonction pour envoyer une alerte
send_test_alert() {
  local severity=$1
  local alertname=$2
  local description=$3

  echo "Envoi alerte: $alertname (severity: $severity)"

  curl -s -X POST "$ALERTMANAGER_URL/api/v1/alerts" \
    -H "Content-Type: application/json" \
    -d '[{
      "labels": {
        "alertname": "'"$alertname"'",
        "severity": "'"$severity"'",
        "environment": "test",
        "cluster": "microk8s-lab"
      },
      "annotations": {
        "description": "'"$description"'",
        "summary": "Test notification",
        "runbook_url": "https://wiki.example.com/runbooks/test"
      },
      "startsAt": "'$(date -u +%Y-%m-%dT%H:%M:%SZ)'"
    }]'
}

# Test différents niveaux
send_test_alert "info" "TestInfo" "Test notification niveau info"
sleep 2

send_test_alert "warning" "TestWarning" "Test notification niveau warning"
sleep 2

send_test_alert "critical" "TestCritical" "Test notification niveau critical"

echo "Tests envoyés. Vérifiez vos canaux de notification."
```

## Bonnes pratiques

### Formatage des messages

**Soyez concis** : Le titre doit dire l'essentiel
```yaml
title: '🔥 {{ .GroupLabels.alertname }} - {{ .GroupLabels.namespace }}'
# Plutôt que : "Une alerte s'est déclenchée dans le système"
```

**Incluez le contexte** : Qui, quoi, où, quand
```yaml
text: |
  *Quoi:* {{ .Annotations.summary }}
  *Où:* Namespace {{ .Labels.namespace }}, Pod {{ .Labels.pod }}
  *Quand:* {{ .StartsAt.Format "15:04:05" }}
  *Impact:* {{ .Annotations.impact | default "Non spécifié" }}
```

**Actions claires** : Que doit faire la personne ?
```yaml
annotations:
  description: "CPU > 90% sur {{ $labels.pod }}"
  action: "Vérifier les logs, envisager un scale horizontal"
  runbook_url: "https://wiki/runbooks/high-cpu"
```

### Éviter la fatigue d'alerte

**Groupement intelligent** :
```yaml
route:
  group_by: ['alertname', 'cluster', 'namespace']
  group_wait: 30s      # Attendre pour grouper
  group_interval: 5m   # Intervalle entre groupes
  repeat_interval: 12h # Ne pas spammer
```

**Seuils appropriés** : Éviter les faux positifs
```yaml
# Mauvais : Alerte dès 50% CPU
expr: cpu_usage > 0.5

# Mieux : Alerte si > 80% pendant 10 minutes
expr: cpu_usage > 0.8
for: 10m
```

**Priorités claires** :
- Critical : Action immédiate requise (PagerDuty)
- Warning : À traiter dans la journée (Email)
- Info : Pour information seulement (Slack)

### Maintenance et évolution

**Documentation** : Chaque alerte doit avoir :
```yaml
annotations:
  summary: "Résumé court du problème"
  description: "Description détaillée avec contexte"
  impact: "Impact sur les utilisateurs/système"
  action: "Que faire pour résoudre"
  runbook_url: "Lien vers documentation détaillée"
  dashboard_url: "Lien vers dashboard Grafana"
```

**Revue régulière** :
- Analyser les alertes des 30 derniers jours
- Identifier les alertes trop fréquentes
- Ajuster les seuils et les timings
- Supprimer les alertes obsolètes

**Test périodique** :
```yaml
# Alerte de test hebdomadaire
- alert: WeeklyTestAlert
  expr: vector(1)
  for: 1m
  labels:
    severity: info
    test: "true"
  annotations:
    summary: "Test hebdomadaire du système d'alerting"
    description: "Cette alerte teste que les notifications fonctionnent"
```

## Dépannage

### Les notifications ne sont pas envoyées

**Vérifier la configuration** :
```bash
# Valider la syntaxe
amtool check-config /etc/alertmanager/alertmanager.yml

# Vérifier le routage
amtool config routes show --config.file=alertmanager.yml
```

**Vérifier les logs** :
```bash
kubectl logs -n monitoring alertmanager-0 | grep -E "error|fail"
```

**Tester manuellement le canal** :
```bash
# Test Slack
curl -X POST -H 'Content-Type: application/json' \
  -d '{"text":"Test manuel"}' \
  YOUR_SLACK_WEBHOOK_URL

# Test email
echo "Test" | mail -s "Test" email@example.com
```

### Notifications en double

**Vérifier le grouping** :
```yaml
route:
  group_by: ['alertname', 'namespace']  # Ajouter plus de labels
  group_interval: 5m  # Augmenter l'intervalle
```

**Vérifier les routes** : Éviter `continue: true` non nécessaires

### Notifications manquées

**Vérifier les inhibitions** :
```bash
amtool alert query --silenced --inhibited \
  --alertmanager.url=http://alertmanager:9093
```

**Vérifier les silences actifs** :
```bash
amtool silence query --alertmanager.url=http://alertmanager:9093
```

## Conclusion

La configuration des notifications est un aspect crucial mais délicat du monitoring. L'objectif est de trouver le juste équilibre : être alerté quand c'est nécessaire, sans être submergé d'alertes inutiles.

Dans votre lab MicroK8s, commencez simple avec un canal (email ou Slack), testez bien votre configuration, puis ajoutez progressivement d'autres canaux selon vos besoins. N'oubliez pas que chaque environnement est unique - ce qui fonctionne pour une équipe peut ne pas convenir à une autre.

La clé du succès est l'itération continue : surveillez vos alertes, collectez les retours, ajustez les configurations, et répétez. Avec le temps, vous développerez un système de notification qui vous alerte efficacement sans créer de fatigue d'alerte.

⏭️
