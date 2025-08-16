🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 10.5 Helm Charts

## Introduction à Helm

Helm est souvent décrit comme le "gestionnaire de paquets pour Kubernetes", similaire à apt pour Ubuntu ou npm pour Node.js. Il simplifie considérablement le déploiement et la gestion d'applications complexes sur Kubernetes en regroupant tous les manifestes YAML nécessaires dans un seul package réutilisable appelé "Chart".

### Pourquoi utiliser Helm ?

Imaginez que vous devez déployer une application WordPress sur Kubernetes. Sans Helm, vous devriez créer et gérer manuellement :
- Un Deployment pour WordPress
- Un Service pour exposer WordPress
- Un Deployment pour MySQL
- Un Service pour MySQL
- Des PersistentVolumeClaims pour le stockage
- Des ConfigMaps pour la configuration
- Des Secrets pour les mots de passe

Avec Helm, tout cela est packagé dans un seul Chart que vous pouvez installer avec une simple commande.

## Installation de Helm sur MicroK8s

### Méthode 1 : Via l'addon MicroK8s (Recommandé)

La façon la plus simple d'installer Helm sur MicroK8s est d'utiliser l'addon intégré :

```bash
# Activer l'addon Helm3
microk8s enable helm3

# Vérifier l'installation
microk8s helm3 version

# Créer un alias pour simplifier l'utilisation
echo "alias helm='microk8s helm3'" >> ~/.bashrc
source ~/.bashrc
```

### Méthode 2 : Installation manuelle

Si vous préférez installer Helm séparément :

```bash
# Télécharger et installer Helm
curl https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash

# Configurer Helm pour utiliser kubeconfig de MicroK8s
export KUBECONFIG=/var/snap/microk8s/current/credentials/client.config
```

## Concepts fondamentaux

### Chart
Un Chart est un ensemble de fichiers qui décrit une application Kubernetes. Il contient :
- **Chart.yaml** : métadonnées du chart (nom, version, description)
- **values.yaml** : valeurs de configuration par défaut
- **templates/** : dossier contenant les templates de manifestes Kubernetes
- **charts/** : dépendances (autres charts requis)

### Release
Une Release est une instance d'un Chart installé dans votre cluster. Vous pouvez installer le même Chart plusieurs fois avec différentes configurations, créant ainsi plusieurs releases.

### Repository
Un Repository est un serveur qui héberge des Charts. Le plus connu est le Helm Hub, mais vous pouvez créer votre propre repository privé.

## Utilisation basique de Helm

### Ajouter un repository

Les repositories contiennent des collections de Charts prêts à l'emploi :

```bash
# Ajouter le repository officiel Bitnami
helm repo add bitnami https://charts.bitnami.com/bitnami

# Ajouter d'autres repositories populaires
helm repo add stable https://charts.helm.sh/stable
helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx
helm repo add jetstack https://charts.jetstack.io

# Mettre à jour la liste des charts disponibles
helm repo update

# Lister les repositories configurés
helm repo list
```

### Rechercher des Charts

```bash
# Rechercher un chart spécifique
helm search repo wordpress

# Rechercher avec plus de détails
helm search repo nginx --versions

# Voir tous les charts d'un repository
helm search repo bitnami/
```

### Installer un Chart

```bash
# Installation simple avec valeurs par défaut
helm install mon-wordpress bitnami/wordpress

# Installation dans un namespace spécifique
helm install mon-wordpress bitnami/wordpress --namespace production --create-namespace

# Installation avec des valeurs personnalisées
helm install mon-nginx bitnami/nginx \
  --set service.type=LoadBalancer \
  --set replicaCount=3
```

### Personnaliser l'installation avec un fichier values

Créez un fichier `mes-valeurs.yaml` :

```yaml
# mes-valeurs.yaml
replicaCount: 2

image:
  repository: nginx
  tag: "1.21.0"
  pullPolicy: IfNotPresent

service:
  type: ClusterIP
  port: 80

ingress:
  enabled: true
  className: nginx
  hostname: monapp.mondomaine.fr
  tls: true

resources:
  limits:
    cpu: 100m
    memory: 128Mi
  requests:
    cpu: 50m
    memory: 64Mi
```

Puis installer avec ce fichier :

```bash
helm install mon-app bitnami/nginx -f mes-valeurs.yaml
```

### Gérer les releases

```bash
# Lister toutes les releases installées
helm list

# Lister les releases dans tous les namespaces
helm list --all-namespaces

# Voir le statut d'une release
helm status mon-wordpress

# Voir l'historique des déploiements
helm history mon-wordpress

# Voir les valeurs utilisées pour une release
helm get values mon-wordpress

# Voir tous les manifestes générés
helm get manifest mon-wordpress
```

### Mettre à jour une release

```bash
# Mise à jour avec de nouvelles valeurs
helm upgrade mon-wordpress bitnami/wordpress \
  --set wordpressPassword=NouveauMotDePasse

# Mise à jour avec un nouveau fichier de valeurs
helm upgrade mon-app bitnami/nginx -f nouvelles-valeurs.yaml

# Mise à jour vers une version spécifique du chart
helm upgrade mon-wordpress bitnami/wordpress --version 15.2.0
```

### Rollback (retour en arrière)

```bash
# Revenir à la version précédente
helm rollback mon-wordpress

# Revenir à une révision spécifique
helm rollback mon-wordpress 3

# Voir ce qui a changé entre deux révisions
helm diff revision mon-wordpress 2 3
```

### Désinstaller une release

```bash
# Désinstaller une application
helm uninstall mon-wordpress

# Désinstaller et garder l'historique
helm uninstall mon-wordpress --keep-history
```

## Créer votre propre Chart

### Initialiser un nouveau Chart

```bash
# Créer la structure de base d'un chart
helm create mon-application

# Structure créée :
# mon-application/
# ├── Chart.yaml
# ├── values.yaml
# ├── charts/
# ├── templates/
# │   ├── NOTES.txt
# │   ├── _helpers.tpl
# │   ├── deployment.yaml
# │   ├── hpa.yaml
# │   ├── ingress.yaml
# │   ├── service.yaml
# │   ├── serviceaccount.yaml
# │   └── tests/
# └── .helmignore
```

### Structure d'un Chart personnalisé

#### Chart.yaml
```yaml
apiVersion: v2
name: mon-application
description: Une application exemple pour mon lab
type: application
version: 0.1.0
appVersion: "1.0"
keywords:
  - application
  - lab
home: https://github.com/moncompte/mon-application
maintainers:
  - name: Votre Nom
    email: email@exemple.com
```

#### values.yaml
```yaml
# Valeurs par défaut pour mon-application

replicaCount: 1

image:
  repository: mon-registry/mon-application
  pullPolicy: IfNotPresent
  tag: "latest"

nameOverride: ""
fullnameOverride: ""

service:
  type: ClusterIP
  port: 8080

ingress:
  enabled: false
  className: "nginx"
  annotations: {}
  hosts:
    - host: monapp.local
      paths:
        - path: /
          pathType: ImplementationSpecific
  tls: []

resources: {}
  # limits:
  #   cpu: 100m
  #   memory: 128Mi
  # requests:
  #   cpu: 100m
  #   memory: 128Mi

autoscaling:
  enabled: false
  minReplicas: 1
  maxReplicas: 10
  targetCPUUtilizationPercentage: 80

nodeSelector: {}
tolerations: []
affinity: {}
```

#### Template Deployment (templates/deployment.yaml)
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: {{ include "mon-application.fullname" . }}
  labels:
    {{- include "mon-application.labels" . | nindent 4 }}
spec:
  {{- if not .Values.autoscaling.enabled }}
  replicas: {{ .Values.replicaCount }}
  {{- end }}
  selector:
    matchLabels:
      {{- include "mon-application.selectorLabels" . | nindent 6 }}
  template:
    metadata:
      labels:
        {{- include "mon-application.selectorLabels" . | nindent 8 }}
    spec:
      containers:
      - name: {{ .Chart.Name }}
        image: "{{ .Values.image.repository }}:{{ .Values.image.tag | default .Chart.AppVersion }}"
        imagePullPolicy: {{ .Values.image.pullPolicy }}
        ports:
        - name: http
          containerPort: {{ .Values.service.port }}
          protocol: TCP
        {{- if .Values.resources }}
        resources:
          {{- toYaml .Values.resources | nindent 10 }}
        {{- end }}
```

### Tester votre Chart

```bash
# Vérifier la syntaxe du chart
helm lint mon-application/

# Voir les manifestes qui seront générés (dry-run)
helm install mon-app ./mon-application --dry-run --debug

# Générer les manifestes sans installer
helm template mon-app ./mon-application > manifests.yaml

# Installer le chart local
helm install mon-app ./mon-application

# Packager le chart pour distribution
helm package mon-application/
# Crée : mon-application-0.1.0.tgz
```

## Helm et les environnements

### Gestion multi-environnements

Créez différents fichiers de valeurs pour chaque environnement :

**values-dev.yaml**
```yaml
replicaCount: 1
image:
  tag: "develop"
ingress:
  enabled: true
  hostname: app-dev.monlab.local
resources:
  limits:
    memory: 256Mi
    cpu: 100m
```

**values-prod.yaml**
```yaml
replicaCount: 3
image:
  tag: "v1.2.3"
ingress:
  enabled: true
  hostname: app.mondomaine.fr
  tls: true
resources:
  limits:
    memory: 1Gi
    cpu: 500m
```

Déploiement par environnement :
```bash
# Environnement de développement
helm install app-dev ./mon-application -f values-dev.yaml -n dev

# Environnement de production
helm install app-prod ./mon-application -f values-prod.yaml -n production
```

## Bonnes pratiques

### 1. Versionnement des Charts

Toujours versionner vos Charts et les images Docker associées :
- Utilisez le versionnement sémantique (SemVer) : MAJOR.MINOR.PATCH
- Mettez à jour `version` dans Chart.yaml à chaque modification du chart
- Mettez à jour `appVersion` quand l'application elle-même change

### 2. Utilisation des values

- Gardez les secrets hors des fichiers values (utilisez Kubernetes Secrets)
- Documentez toutes les valeurs configurables dans le README
- Fournissez des valeurs par défaut sensées
- Utilisez des valeurs explicites plutôt que des abréviations

### 3. Templates

- Utilisez les helpers (`_helpers.tpl`) pour éviter la duplication
- Validez toujours avec `helm lint` et `--dry-run`
- Ajoutez des annotations pour la traçabilité
- Incluez des NOTES.txt utiles pour l'utilisateur

### 4. Sécurité

```bash
# Ne jamais mettre de secrets directement dans les values
# Mauvais exemple :
helm install app ./chart --set database.password=MonMotDePasse

# Bon exemple : utiliser un Secret Kubernetes existant
helm install app ./chart --set existingSecret=mon-secret-db
```

### 5. Organisation des Charts

Pour un lab personnel, organisez vos charts ainsi :
```
helm-charts/
├── charts/
│   ├── application-web/
│   ├── base-donnees/
│   └── monitoring/
├── releases/
│   ├── dev/
│   │   └── values.yaml
│   ├── staging/
│   │   └── values.yaml
│   └── prod/
│       └── values.yaml
└── scripts/
    ├── deploy.sh
    └── rollback.sh
```

## Intégration avec MicroK8s

### Utilisation du registry local

Si vous avez activé le registry MicroK8s :

```bash
# Activer le registry local
microk8s enable registry

# Dans vos values.yaml
image:
  repository: localhost:32000/mon-app
  tag: "v1.0.0"
```

### Avec l'Ingress Controller

```yaml
# values.yaml pour utiliser l'ingress MicroK8s
ingress:
  enabled: true
  className: "public"  # ou "nginx" selon votre configuration
  annotations:
    cert-manager.io/cluster-issuer: "letsencrypt-prod"
  hosts:
    - host: app.mondomaine.fr
      paths:
        - path: /
          pathType: Prefix
  tls:
    - secretName: app-tls
      hosts:
        - app.mondomaine.fr
```

## Dépannage courant

### Problèmes fréquents et solutions

**Le Chart ne s'installe pas**
```bash
# Vérifier les erreurs de syntaxe
helm lint ./mon-chart

# Voir exactement ce qui sera créé
helm install mon-app ./mon-chart --dry-run --debug

# Vérifier les permissions du namespace
kubectl auth can-i create deployments -n mon-namespace
```

**Les valeurs ne sont pas appliquées**
```bash
# Vérifier les valeurs effectives
helm get values mon-release

# Vérifier la priorité des valeurs
# Ordre : --set > -f values.yaml > valeurs par défaut du chart
```

**Mise à jour bloquée**
```bash
# Forcer la recréation des pods
helm upgrade mon-app ./mon-chart --recreate-pods

# En cas d'échec, revenir en arrière
helm rollback mon-app
```

## Ressources utiles

### Charts populaires pour votre lab

- **bitnami/postgresql** : Base de données PostgreSQL
- **bitnami/redis** : Cache Redis
- **bitnami/rabbitmq** : Message broker
- **prometheus-community/kube-prometheus-stack** : Stack de monitoring complète
- **gitlab/gitlab** : GitLab complet pour CI/CD
- **jenkinsci/jenkins** : Jenkins pour CI/CD
- **elastic/elasticsearch** : Elasticsearch pour logs et recherche

### Commandes de référence rapide

```bash
# Commandes essentielles
helm repo add [name] [url]     # Ajouter un repository
helm search repo [keyword]      # Rechercher des charts
helm install [name] [chart]     # Installer un chart
helm upgrade [name] [chart]     # Mettre à jour
helm rollback [name] [revision] # Revenir en arrière
helm uninstall [name]           # Désinstaller
helm list                       # Lister les releases
helm status [name]              # Statut d'une release
helm get values [name]          # Voir les valeurs
helm create [name]              # Créer un nouveau chart
helm package [chart-path]       # Packager un chart
helm lint [chart-path]          # Valider un chart
```

## Conclusion

Helm transforme la gestion d'applications Kubernetes d'une tâche complexe en quelques commandes simples. Pour votre lab MicroK8s personnel, il vous permet de :
- Déployer rapidement des applications complexes
- Maintenir plusieurs environnements facilement
- Partager vos configurations
- Revenir en arrière en cas de problème
- Standardiser vos déploiements

Commencez par utiliser des Charts existants depuis les repositories publics, puis progressivement créez vos propres Charts adaptés à vos besoins spécifiques. Avec la pratique, Helm deviendra un outil indispensable dans votre boîte à outils Kubernetes.

⏭️
