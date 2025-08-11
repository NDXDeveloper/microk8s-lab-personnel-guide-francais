🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 5.6 Renouvellement automatique

## Pourquoi le renouvellement automatique est-il crucial ?

Le renouvellement automatique des certificats SSL/TLS est l'une des fonctionnalités les plus importantes d'une infrastructure moderne. Sans lui, vous risquez des interruptions de service catastrophiques lorsque vos certificats expirent, souvent au moment le moins opportun.

### Analogie simple

Imaginez que votre permis de conduire expire automatiquement tous les 90 jours. Sans système de renouvellement automatique, vous devriez vous souvenir de faire la démarche manuellement tous les trois mois. Un oubli et vous ne pouvez plus conduire légalement. Le renouvellement automatique, c'est comme avoir un assistant qui s'occupe de toutes ces démarches administratives sans que vous y pensiez.

### Conséquences d'un certificat expiré

**Interruption de service**
- Sites web inaccessibles avec erreurs SSL
- APIs qui refusent les connexions
- Applications mobiles qui perdent la connectivité
- Perte de confiance des utilisateurs

**Impact opérationnel**
- Intervention d'urgence souvent en dehors des heures ouvrables
- Stress et pression sur les équipes techniques
- Possible perte de données ou transactions
- Coût de l'interruption de service

**Réputation et confiance**
- Avertissements de sécurité dans les navigateurs
- Perception d'un manque de professionnalisme
- Impact possible sur le référencement (SEO)
- Perte de confiance des partenaires techniques

## Principe de fonctionnement avec Cert-Manager

### Architecture du renouvellement

Cert-manager surveille continuellement tous les certificats qu'il gère et déclenche automatiquement leur renouvellement selon des critères prédéfinis. Ce processus s'appuie sur plusieurs composants qui travaillent ensemble :

**Controller de surveillance**
- Vérification périodique de l'état des certificats
- Calcul des dates d'expiration
- Déclenchement des processus de renouvellement

**Gestionnaire de challenges**
- Résolution automatique des défis ACME
- Gestion des validations DNS ou HTTP
- Nettoyage automatique après validation

**Gestionnaire de secrets**
- Mise à jour atomique des secrets Kubernetes
- Préservation des anciens certificats pendant la transition
- Notification des services affectés

### Cycle de renouvellement

```
1. Surveillance continue
   ↓
2. Détection d'expiration proche
   ↓
3. Création d'une nouvelle CertificateRequest
   ↓
4. Résolution du challenge ACME
   ↓
5. Obtention du nouveau certificat
   ↓
6. Mise à jour du secret Kubernetes
   ↓
7. Redémarrage des pods si nécessaire
```

## Configuration du renouvellement

### Paramètres de base

Chaque certificat peut être configuré avec des paramètres spécifiques de renouvellement :

```yaml
# Configuration de renouvellement dans un Certificate
apiVersion: cert-manager.io/v1
kind: Certificate
metadata:
  name: example-cert
  namespace: default
spec:
  secretName: example-tls
  issuerRef:
    name: letsencrypt-prod
    kind: ClusterIssuer

  # Durée de validité du certificat
  duration: 2160h    # 90 jours (maximum Let's Encrypt)

  # Délai avant expiration pour déclencher le renouvellement
  renewBefore: 720h  # 30 jours (renouvellement anticipé)

  dnsNames:
  - example.com
  - www.example.com
```

### Calcul des délais de renouvellement

**Durée de validité (duration)**
- Let's Encrypt : maximum 90 jours (2160 heures)
- CA commerciales : généralement 1 à 2 ans
- Certificats auto-signés : durée personnalisable

**Délai de renouvellement (renewBefore)**
- Recommandation générale : 1/3 de la durée de validité
- Let's Encrypt (90j) : 30 jours avant expiration
- Certificats 1 an : 120 jours avant expiration
- Permet de résoudre les problèmes avant l'urgence

**Exemples de configuration optimale**

```yaml
# Configuration pour Let's Encrypt (durée courte)
duration: 2160h     # 90 jours
renewBefore: 720h   # 30 jours (33% de la durée)

# Configuration pour CA commerciale (durée longue)
duration: 8760h     # 1 an
renewBefore: 2160h  # 90 jours (25% de la durée)

# Configuration pour développement (durée personnalisée)
duration: 720h      # 30 jours
renewBefore: 168h   # 7 jours (23% de la durée)
```

## Stratégies de renouvellement

### Renouvellement anticipé

La stratégie recommandée consiste à renouveler les certificats bien avant leur expiration :

**Avantages**
- Temps suffisant pour résoudre les problèmes éventuels
- Évite les interventions d'urgence
- Permet les tests et validations
- Réduit le stress opérationnel

**Configuration anticipée**
```yaml
# Renouvellement très anticipé pour les services critiques
apiVersion: cert-manager.io/v1
kind: Certificate
metadata:
  name: critical-service-cert
spec:
  secretName: critical-service-tls
  duration: 2160h     # 90 jours
  renewBefore: 1440h  # 60 jours (renouvellement à mi-parcours)

  # Forcer un nouveau certificat à chaque renouvellement
  privateKey:
    rotationPolicy: Always
```

### Renouvellement échelonné

Pour éviter que tous vos certificats se renouvellent en même temps :

```yaml
# Service A - renouvellement le 1er du mois
apiVersion: cert-manager.io/v1
kind: Certificate
metadata:
  name: service-a-cert
spec:
  duration: 2160h
  renewBefore: 720h    # 30 jours
---
# Service B - renouvellement le 15 du mois
apiVersion: cert-manager.io/v1
kind: Certificate
metadata:
  name: service-b-cert
spec:
  duration: 2520h      # 105 jours (décalage)
  renewBefore: 720h    # 30 jours
```

### Renouvellement par environnement

Stratégie différenciée selon l'importance de l'environnement :

```yaml
# Production - renouvellement très anticipé
apiVersion: cert-manager.io/v1
kind: Certificate
metadata:
  name: prod-cert
  namespace: production
spec:
  duration: 2160h
  renewBefore: 1080h   # 45 jours (très anticipé)
---
# Staging - renouvellement standard
apiVersion: cert-manager.io/v1
kind: Certificate
metadata:
  name: staging-cert
  namespace: staging
spec:
  duration: 2160h
  renewBefore: 720h    # 30 jours (standard)
---
# Développement - renouvellement minimal
apiVersion: cert-manager.io/v1
kind: Certificate
metadata:
  name: dev-cert
  namespace: development
spec:
  duration: 720h       # 30 jours
  renewBefore: 168h    # 7 jours (minimal)
```

## Gestion des clés privées

### Rotation des clés

```yaml
# Configuration avec rotation automatique des clés
apiVersion: cert-manager.io/v1
kind: Certificate
metadata:
  name: rotating-key-cert
spec:
  secretName: rotating-key-tls
  privateKey:
    # Nouvelle clé privée à chaque renouvellement
    rotationPolicy: Always
    algorithm: RSA
    size: 2048

  # Renouvellement fréquent pour rotation régulière
  duration: 1440h      # 60 jours
  renewBefore: 360h    # 15 jours
```

### Conservation des clés

```yaml
# Configuration avec conservation des clés
apiVersion: cert-manager.io/v1
kind: Certificate
metadata:
  name: stable-key-cert
spec:
  secretName: stable-key-tls
  privateKey:
    # Réutilisation de la même clé privée
    rotationPolicy: Never
    algorithm: ECDSA
    size: 256
```

### Choix de la stratégie

**Rotation (Always) - Recommandée pour**
- Services critiques exposés publiquement
- Environnements avec exigences de sécurité élevées
- Conformité à certains standards (PCI-DSS, etc.)

**Conservation (Never) - Appropriée pour**
- Services internes avec contraintes de performance
- Applications avec mise en cache agressive des certificats
- Environnements de développement

## Monitoring du renouvellement

### Métriques Cert-Manager

Cert-manager expose automatiquement des métriques Prometheus pour surveiller les renouvellements :

```yaml
# ServiceMonitor pour collecter les métriques cert-manager
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: cert-manager-metrics
  namespace: cert-manager
spec:
  selector:
    matchLabels:
      app.kubernetes.io/name: cert-manager
  endpoints:
  - port: tcp-prometheus-servicemonitor
    interval: 60s
    path: /metrics
```

### Alertes Prometheus

Configuration d'alertes pour le renouvellement des certificats :

```yaml
# Règles d'alerte pour les certificats
groups:
- name: cert-manager-alerts
  rules:

  # Certificat expire bientôt
  - alert: CertificateExpiringSoon
    expr: cert_manager_certificate_expiration_timestamp_seconds - time() < 7 * 24 * 3600
    for: 1h
    labels:
      severity: warning
    annotations:
      summary: "Certificat {{ $labels.name }} expire dans moins de 7 jours"
      description: "Le certificat {{ $labels.name }} dans le namespace {{ $labels.namespace }} expire le {{ $value | humanizeTimestamp }}"

  # Échec de renouvellement
  - alert: CertificateRenewalFailed
    expr: cert_manager_certificate_ready_status{condition="False"} == 1
    for: 15m
    labels:
      severity: critical
    annotations:
      summary: "Échec de renouvellement du certificat {{ $labels.name }}"
      description: "Le certificat {{ $labels.name }} dans le namespace {{ $labels.namespace }} n'a pas pu être renouvelé"

  # Certificat non prêt
  - alert: CertificateNotReady
    expr: cert_manager_certificate_ready_status{condition="True"} == 0
    for: 10m
    labels:
      severity: warning
    annotations:
      summary: "Certificat {{ $labels.name }} non prêt"
      description: "Le certificat {{ $labels.name }} dans le namespace {{ $labels.namespace }} n'est pas dans un état prêt"
```

### Dashboard Grafana

Visualisation des métriques de certificats :

```json
{
  "dashboard": {
    "title": "Cert-Manager - Certificats SSL/TLS",
    "panels": [
      {
        "title": "Délai avant expiration",
        "type": "stat",
        "targets": [
          {
            "expr": "(cert_manager_certificate_expiration_timestamp_seconds - time()) / 86400",
            "legendFormat": "{{ name }} ({{ namespace }})"
          }
        ]
      },
      {
        "title": "État des certificats",
        "type": "piechart",
        "targets": [
          {
            "expr": "cert_manager_certificate_ready_status",
            "legendFormat": "{{ condition }}"
          }
        ]
      }
    ]
  }
}
```

## Diagnostic des échecs de renouvellement

### Commandes de diagnostic

```bash
# Vérifier l'état de tous les certificats
kubectl get certificates --all-namespaces -o wide

# Détails d'un certificat spécifique
kubectl describe certificate mon-certificat -n mon-namespace

# Voir les événements liés aux certificats
kubectl get events --field-selector involvedObject.kind=Certificate

# Vérifier les CertificateRequest en échec
kubectl get certificaterequests --all-namespaces | grep -v "True"

# Logs cert-manager pour investigation
kubectl logs -n cert-manager deployment/cert-manager -f
```

### Causes communes d'échec

**Limite de débit atteinte**
```bash
# Vérifier dans les logs
kubectl logs -n cert-manager deployment/cert-manager | grep -i "rate limit"

# Solution : attendre ou utiliser l'environnement staging
```

**Problème de validation DNS/HTTP**
```bash
# Vérifier les challenges en cours
kubectl get challenges --all-namespaces

# Détails d'un challenge spécifique
kubectl describe challenge nom-du-challenge
```

**Configuration Issuer incorrecte**
```bash
# Vérifier l'état des issuers
kubectl get clusterissuers -o wide

# Détails d'un issuer
kubectl describe clusterissuer nom-issuer
```

### Renouvellement forcé

En cas de problème, vous pouvez forcer le renouvellement :

```bash
# Méthode 1 : Annotation de renouvellement forcé
kubectl annotate certificate mon-certificat cert-manager.io/force-renewal=$(date +%s)

# Méthode 2 : Suppression de la CertificateRequest
kubectl delete certificaterequest -l cert-manager.io/certificate-name=mon-certificat

# Méthode 3 : Suppression complète du secret (recréation totale)
kubectl delete secret mon-certificat-tls
```

## Automatisation avancée

### Scripts de surveillance

```bash
#!/bin/bash
# Script de surveillance des certificats

NAMESPACE_LIST="default production staging"
ALERT_DAYS=7

for ns in $NAMESPACE_LIST; do
  echo "=== Namespace: $ns ==="

  # Récupérer les certificats du namespace
  kubectl get certificates -n $ns -o json | jq -r '
    .items[] |
    select(.status.notAfter != null) |
    [
      .metadata.name,
      .status.notAfter,
      ((.status.notAfter | fromdateiso8601) - now) / 86400 | floor
    ] | @tsv
  ' | while IFS=$'\t' read name expiry days_left; do

    if [ "$days_left" -lt "$ALERT_DAYS" ]; then
      echo "⚠️  ALERTE: $name expire dans $days_left jours ($expiry)"
    else
      echo "✅ OK: $name expire dans $days_left jours"
    fi
  done
  echo
done
```

### CronJob de vérification

```yaml
# CronJob pour vérification quotidienne
apiVersion: batch/v1
kind: CronJob
metadata:
  name: cert-health-check
  namespace: cert-manager
spec:
  schedule: "0 9 * * *"  # Tous les jours à 9h
  jobTemplate:
    spec:
      template:
        spec:
          serviceAccountName: cert-health-checker
          containers:
          - name: checker
            image: bitnami/kubectl:latest
            command:
            - /bin/bash
            - -c
            - |
              echo "=== Vérification des certificats ==="
              kubectl get certificates --all-namespaces \
                -o custom-columns="NAMESPACE:.metadata.namespace,NAME:.metadata.name,READY:.status.conditions[0].status,EXPIRES:.status.notAfter" \
                --no-headers | while read ns name ready expires; do

                if [ "$ready" != "True" ]; then
                  echo "❌ $ns/$name - État: $ready"
                fi
              done
          restartPolicy: OnFailure
```

## Intégration avec les applications

### Redémarrage automatique des pods

Certaines applications nécessitent un redémarrage pour prendre en compte les nouveaux certificats :

```yaml
# Deployment avec annotation de redémarrage automatique
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web-app
spec:
  template:
    metadata:
      annotations:
        # Redémarrage lors du changement de certificat
        cert-manager.io/restart-on-certificate-change: "true"
    spec:
      containers:
      - name: web-app
        image: nginx:latest
        volumeMounts:
        - name: tls-certs
          mountPath: /etc/ssl/certs
          readOnly: true
      volumes:
      - name: tls-certs
        secret:
          secretName: web-app-tls
```

### Configuration avec reloader

Utilisation de reloader pour automatiser les redémarrages :

```yaml
# Installation de reloader
apiVersion: apps/v1
kind: Deployment
metadata:
  name: reloader
  namespace: kube-system
spec:
  replicas: 1
  selector:
    matchLabels:
      app: reloader
  template:
    metadata:
      labels:
        app: reloader
    spec:
      containers:
      - name: reloader
        image: stakater/reloader:latest
        imagePullPolicy: IfNotPresent
---
# Deployment avec annotation reloader
apiVersion: apps/v1
kind: Deployment
metadata:
  name: mon-app
  annotations:
    reloader.stakater.com/auto: "true"  # Redémarrage automatique
spec:
  template:
    spec:
      containers:
      - name: app
        image: mon-app:latest
        volumeMounts:
        - name: tls
          mountPath: /etc/certs
      volumes:
      - name: tls
        secret:
          secretName: mon-app-tls
```

## Bonnes pratiques

### Configuration optimale

**Délais de renouvellement**
- Production : 33% de la durée de validité minimum
- Staging : 25% de la durée de validité
- Développement : peut être plus court pour les tests

**Gestion des environnements**
- Certificats séparés par environnement
- Stratégies de renouvellement adaptées à la criticité
- Tests de renouvellement en staging avant production

**Monitoring et alerting**
- Alertes configurées pour les échecs de renouvellement
- Dashboard de visualisation des statuts
- Notifications proactives avant expiration

### Sécurité

**Rotation des clés**
- Activée pour les services critiques
- Évaluée au cas par cas selon les performances requises
- Documentée dans la politique de sécurité

**Gestion des secrets**
- RBAC approprié sur les secrets contenant les certificats
- Chiffrement au repos activé
- Audit des accès aux certificats

**Sauvegarde**
- Sauvegarde des certificats et clés privées
- Procédures de restauration testées
- Plan de continuité en cas de défaillance massive

## Préparation pour la section suivante

Le renouvellement automatique constitue le pilier de la fiabilité de votre infrastructure SSL/TLS. Avec ces mécanismes en place, vous êtes maintenant prêt pour la dernière section de ce chapitre : le dépannage des certificats, qui vous donnera les outils pour diagnostiquer et résoudre rapidement tous les problèmes que vous pourriez rencontrer.

Une infrastructure de certificats avec renouvellement automatique bien configuré vous permettra de vous concentrer sur le développement de vos applications plutôt que sur la maintenance des certificats.

⏭️
