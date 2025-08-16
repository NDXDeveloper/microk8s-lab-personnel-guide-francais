🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 10.2 Registry privé et gestion d'images

## Introduction : Le cœur de votre pipeline de déploiement

Un registry d'images Docker est comparable à un entrepôt central où sont stockées toutes les images de conteneurs de vos applications. Dans votre lab MicroK8s, disposer de votre propre registry privé transforme radicalement votre capacité à gérer, déployer et sécuriser vos applications. C'est le pont essentiel entre votre code source et vos déploiements Kubernetes.

## Comprendre les registries Docker

### Qu'est-ce qu'une image Docker ?

Avant de parler de registry, rappelons ce qu'est une image Docker. Une image est un package autonome qui contient :

- **Le code de votre application**
- **Toutes les dépendances** (bibliothèques, frameworks)
- **L'environnement d'exécution** (runtime, interpréteur)
- **Les configurations système** nécessaires
- **Les métadonnées** (ports, commandes de démarrage)

Ces images sont construites en couches successives, chaque couche représentant une modification par rapport à la précédente. Cette architecture en couches permet l'optimisation du stockage et des transferts réseau.

### Le rôle du registry

Un registry est un serveur qui stocke et distribue ces images Docker. Pensez-y comme :

- **GitHub pour le code** → **Registry pour les images**
- **npm pour JavaScript** → **Registry pour les conteneurs**
- **Maven pour Java** → **Registry pour Docker**

### Types de registries

**Registries publics** :
- **Docker Hub** : Le registry par défaut, héberge des millions d'images publiques
- **GitHub Container Registry** : Intégré avec GitHub
- **Google Container Registry** : Service de Google Cloud
- **Amazon ECR Public** : Registry public d'AWS

**Registries privés** :
- **Registry local MicroK8s** : Intégré directement dans votre cluster
- **Harbor** : Solution enterprise open source avec interface web
- **GitLab Container Registry** : Intégré à GitLab
- **JFrog Artifactory** : Solution commerciale multi-formats
- **Nexus Repository** : Gestionnaire de repository universel

## Pourquoi un registry privé dans votre lab ?

### Avantages techniques

**Performance et rapidité** : Les images sont stockées localement, éliminant les temps de téléchargement depuis Internet. Dans un lab avec plusieurs déploiements quotidiens, cela représente un gain de temps considérable.

**Indépendance réseau** : Votre lab continue de fonctionner même sans connexion Internet. Idéal pour les environnements isolés ou avec une connexion limitée.

**Contrôle des versions** : Vous décidez exactement quelles versions d'images sont disponibles, évitant les surprises dues aux mises à jour automatiques.

### Avantages de sécurité

**Isolation complète** : Vos images propriétaires ne quittent jamais votre infrastructure, éliminant les risques de fuite de données.

**Scan de vulnérabilités** : Avec des solutions comme Harbor, vous pouvez scanner automatiquement vos images pour détecter les failles de sécurité.

**Contrôle d'accès** : Définissez précisément qui peut pousser ou récupérer des images, avec une authentification forte.

### Avantages organisationnels

**Naming cohérent** : Établissez vos propres conventions de nommage sans contraintes externes.

**Cycle de vie maîtrisé** : Gérez la rétention des images, le nettoyage automatique des anciennes versions.

**Audit et traçabilité** : Suivez qui a poussé quelle image et quand.

## Installation du registry MicroK8s

### Activation de l'addon registry

MicroK8s facilite grandement l'installation d'un registry privé avec son addon intégré :

```bash
# Activer le registry avec stockage par défaut (20GB)
microk8s enable registry

# Ou avec une taille personnalisée
microk8s enable registry:size=40Gi
```

Cette commande simple déploie :
- Un pod registry basé sur l'image officielle Docker Registry v2
- Un service Kubernetes exposant le registry
- Un volume persistant pour stocker les images
- Une configuration réseau appropriée

### Architecture du registry MicroK8s

Le registry s'installe dans le namespace `container-registry` avec les composants suivants :

```yaml
# Pod registry
apiVersion: v1
kind: Pod
metadata:
  name: registry
  namespace: container-registry
spec:
  containers:
  - name: registry
    image: registry:2.7
    env:
    - name: REGISTRY_STORAGE_FILESYSTEM_ROOTDIRECTORY
      value: /var/lib/registry
    volumeMounts:
    - name: registry-data
      mountPath: /var/lib/registry
    ports:
    - containerPort: 5000
      protocol: TCP
```

Le registry est accessible via :
- **Interne au cluster** : `registry.container-registry.svc.cluster.local:5000`
- **Depuis l'hôte** : `localhost:32000`

### Configuration de Docker pour utiliser le registry local

Pour que Docker puisse communiquer avec votre registry non-HTTPS :

```bash
# Éditer la configuration Docker
sudo nano /etc/docker/daemon.json

# Ajouter votre registry comme insecure (développement uniquement)
{
  "insecure-registries": ["localhost:32000", "registry.local:32000"]
}

# Redémarrer Docker
sudo systemctl restart docker
```

**Note importante** : L'option "insecure-registries" ne doit être utilisée qu'en développement. En production, configurez toujours HTTPS avec des certificats valides.

## Gestion des images Docker

### Nomenclature des images

Une image Docker suit une structure précise :

```
[REGISTRY_HOST[:PORT]/]NAMESPACE/IMAGE_NAME[:TAG]

Exemples :
localhost:32000/monlab/app1:v1.0.0
registry.local/backend/api-service:latest
harbor.monlab.local/production/frontend:2.3.1-stable
```

**Composants** :
- **Registry host** : Adresse du registry (optionnel, Docker Hub par défaut)
- **Namespace** : Organisation logique (utilisateur ou projet)
- **Image name** : Nom de l'application
- **Tag** : Version ou variant (optionnel, "latest" par défaut)

### Construction d'images pour votre registry

#### Dockerfile optimisé

Créez des Dockerfiles efficaces pour vos applications :

```dockerfile
# Exemple pour une application Node.js
# Utilisation du multi-stage build pour réduire la taille

# Stage 1: Construction
FROM node:18-alpine AS builder
WORKDIR /app
COPY package*.json ./
RUN npm ci --only=production

# Stage 2: Image finale
FROM node:18-alpine
WORKDIR /app

# Créer un utilisateur non-root
RUN addgroup -g 1001 -S nodejs && \
    adduser -S nodejs -u 1001

# Copier les dépendances depuis le builder
COPY --from=builder --chown=nodejs:nodejs /app/node_modules ./node_modules
COPY --chown=nodejs:nodejs . .

# Métadonnées
LABEL maintainer="monlab@example.com"
LABEL version="1.0.0"
LABEL description="Application Node.js pour mon lab"

# Configuration
USER nodejs
EXPOSE 3000
ENV NODE_ENV=production

# Healthcheck
HEALTHCHECK --interval=30s --timeout=3s --start-period=5s --retries=3 \
  CMD node healthcheck.js

# Commande de démarrage
CMD ["node", "server.js"]
```

#### Build et tag

```bash
# Construction de l'image
docker build -t myapp:dev .

# Tag pour votre registry local
docker tag myapp:dev localhost:32000/monlab/myapp:v1.0.0

# Tags multiples pour différents usages
docker tag myapp:dev localhost:32000/monlab/myapp:latest
docker tag myapp:dev localhost:32000/monlab/myapp:$(git rev-parse --short HEAD)
```

### Push vers le registry

```bash
# Pousser l'image vers votre registry
docker push localhost:32000/monlab/myapp:v1.0.0

# Pousser tous les tags
docker push localhost:32000/monlab/myapp --all-tags

# Vérifier que l'image est bien dans le registry
curl http://localhost:32000/v2/_catalog
# Réponse : {"repositories":["monlab/myapp"]}

# Lister les tags disponibles
curl http://localhost:32000/v2/monlab/myapp/tags/list
# Réponse : {"name":"monlab/myapp","tags":["v1.0.0","latest"]}
```

### Pull depuis le registry

```bash
# Depuis une autre machine ou après avoir nettoyé le cache local
docker pull localhost:32000/monlab/myapp:v1.0.0

# Dans un deployment Kubernetes
kubectl create deployment myapp \
  --image=localhost:32000/monlab/myapp:v1.0.0
```

## Organisation et conventions

### Structure des namespaces

Organisez vos images de manière logique et cohérente :

```
localhost:32000/
├── base/                    # Images de base personnalisées
│   ├── node:18-custom
│   ├── python:3.11-custom
│   └── nginx:alpine-custom
├── dev/                     # Images de développement
│   ├── app1:feature-xyz
│   └── app2:dev-latest
├── staging/                 # Images de staging
│   ├── app1:v1.0.0-rc1
│   └── app2:v2.0.0-beta
└── prod/                    # Images de production
    ├── app1:v1.0.0
    └── app2:v2.0.0
```

### Convention de tags

Établissez des conventions claires pour vos tags :

**Environnements** :
- `dev` : Version de développement
- `staging` : Version de pré-production
- `prod` : Version de production
- `latest` : Dernière version stable

**Versions sémantiques** :
- `v1.0.0` : Version majeure.mineure.patch
- `v1.0.0-rc1` : Release candidate
- `v1.0.0-beta` : Version beta
- `v1.0.0-alpha` : Version alpha

**Git-based** :
- `main` : Construit depuis la branche main
- `feature-login` : Construit depuis une branche feature
- `abc1234` : Hash court du commit Git
- `20240115-1430` : Timestamp de build

### Stratégie de versioning

```yaml
# Exemple de stratégie complète
Production:
  - v1.0.0         # Version sémantique stable
  - stable         # Alias vers la dernière stable

Staging:
  - v1.1.0-rc1     # Release candidate
  - staging        # Alias vers la version en staging

Development:
  - main           # Suit la branche principale
  - dev-20240115   # Build quotidien
  - feature-xyz    # Branches de features

Technique:
  - sha-abc1234    # Hash Git pour traçabilité
  - build-456      # Numéro de build CI/CD
```

## Sécurisation du registry

### Configuration HTTPS avec certificats

Pour un environnement plus proche de la production, configurez HTTPS :

```bash
# Générer un certificat auto-signé (développement)
mkdir -p /etc/docker/registry/certs
openssl req -newkey rsa:4096 -nodes -sha256 \
  -keyout /etc/docker/registry/certs/domain.key \
  -x509 -days 365 \
  -out /etc/docker/registry/certs/domain.crt \
  -subj "/CN=registry.monlab.local"

# Ou utiliser cert-manager pour Let's Encrypt
kubectl apply -f - <<EOF
apiVersion: cert-manager.io/v1
kind: Certificate
metadata:
  name: registry-tls
  namespace: container-registry
spec:
  secretName: registry-tls-secret
  issuerRef:
    name: letsencrypt-prod
    kind: ClusterIssuer
  dnsNames:
  - registry.monlab.local
EOF
```

### Authentification basique

Ajoutez une authentification pour protéger votre registry :

```bash
# Créer un fichier htpasswd
docker run --rm --entrypoint htpasswd \
  httpd:2 -Bbn monuser monpassword > htpasswd

# Créer un secret Kubernetes
kubectl create secret generic registry-auth \
  --from-file=htpasswd \
  -n container-registry

# Configurer le registry pour utiliser l'authentification
kubectl apply -f - <<EOF
apiVersion: v1
kind: ConfigMap
metadata:
  name: registry-config
  namespace: container-registry
data:
  config.yml: |
    version: 0.1
    auth:
      htpasswd:
        realm: Registry Realm
        path: /auth/htpasswd
    storage:
      filesystem:
        rootdirectory: /var/lib/registry
    http:
      addr: :5000
EOF
```

### Scanning de vulnérabilités

Intégrez Trivy pour scanner vos images :

```bash
# Installation de Trivy
sudo apt-get update
sudo apt-get install wget apt-transport-https gnupg lsb-release
wget -qO - https://aquasecurity.github.io/trivy-repo/deb/public.key | sudo apt-key add -
echo "deb https://aquasecurity.github.io/trivy-repo/deb $(lsb_release -sc) main" | \
  sudo tee /etc/apt/sources.list.d/trivy.list
sudo apt-get update
sudo apt-get install trivy

# Scanner une image de votre registry
trivy image localhost:32000/monlab/myapp:v1.0.0

# Résultat exemple :
# ✓ Vulnerabilities:
#   Total: 23 (UNKNOWN: 0, LOW: 15, MEDIUM: 6, HIGH: 2, CRITICAL: 0)
```

## Optimisation et bonnes pratiques

### Réduction de la taille des images

**Utilisez des images de base minimales** :

```dockerfile
# ❌ Mauvais : Image complète Ubuntu (70MB+)
FROM ubuntu:22.04

# ✅ Bon : Alpine Linux (5MB)
FROM alpine:3.18

# ✅ Excellent : Distroless (2MB pour static)
FROM gcr.io/distroless/static-debian11
```

**Multi-stage builds** :

```dockerfile
# Build stage avec tous les outils
FROM golang:1.21 AS builder
WORKDIR /app
COPY . .
RUN go build -o myapp

# Runtime stage minimal
FROM gcr.io/distroless/base-debian11
COPY --from=builder /app/myapp /
ENTRYPOINT ["/myapp"]
```

**Optimisation des layers** :

```dockerfile
# ❌ Mauvais : Chaque RUN crée une layer
RUN apt-get update
RUN apt-get install -y curl
RUN apt-get install -y git
RUN apt-get clean

# ✅ Bon : Une seule layer
RUN apt-get update && \
    apt-get install -y \
      curl \
      git && \
    apt-get clean && \
    rm -rf /var/lib/apt/lists/*
```

### Cache des layers

Docker utilise un cache pour accélérer les builds :

```dockerfile
# Optimiser l'ordre pour maximiser le cache

# 1. Dépendances (changent rarement)
COPY package*.json ./
RUN npm ci

# 2. Code source (change souvent)
COPY . .

# Si vous inversez, le cache est invalidé à chaque changement de code
```

### Nettoyage automatique

Configurez une politique de rétention pour votre registry :

```yaml
# garbage-collection-cronjob.yaml
apiVersion: batch/v1
kind: CronJob
metadata:
  name: registry-gc
  namespace: container-registry
spec:
  schedule: "0 2 * * 0"  # Tous les dimanches à 2h
  jobTemplate:
    spec:
      template:
        spec:
          containers:
          - name: registry-gc
            image: registry:2.7
            command:
            - /bin/registry
            - garbage-collect
            - /etc/docker/registry/config.yml
            volumeMounts:
            - name: registry-data
              mountPath: /var/lib/registry
          restartPolicy: OnFailure
```

Script de nettoyage des anciennes images :

```bash
#!/bin/bash
# clean-old-images.sh

REGISTRY="localhost:32000"
DAYS_TO_KEEP=30

# Lister tous les repositories
REPOS=$(curl -s http://$REGISTRY/v2/_catalog | jq -r '.repositories[]')

for repo in $REPOS; do
  echo "Nettoyage de $repo..."

  # Récupérer tous les tags
  TAGS=$(curl -s http://$REGISTRY/v2/$repo/tags/list | jq -r '.tags[]')

  for tag in $TAGS; do
    # Récupérer la date de création
    CREATED=$(curl -s http://$REGISTRY/v2/$repo/manifests/$tag | \
      jq -r '.history[0].v1Compatibility' | \
      jq -r '.created')

    # Calculer l'âge en jours
    AGE=$(( ($(date +%s) - $(date -d "$CREATED" +%s)) / 86400 ))

    if [ $AGE -gt $DAYS_TO_KEEP ]; then
      echo "  Suppression de $repo:$tag (âge: $AGE jours)"
      # Marquer pour suppression
      curl -X DELETE http://$REGISTRY/v2/$repo/manifests/$tag
    fi
  done
done

echo "Exécution du garbage collector..."
docker exec registry bin/registry garbage-collect /etc/docker/registry/config.yml
```

## Intégration avec le CI/CD

### Pipeline de build automatisé

Intégrez la construction et le push d'images dans votre pipeline :

```yaml
# .gitlab-ci.yml exemple
stages:
  - build
  - push
  - deploy

variables:
  REGISTRY: "registry.monlab.local:5000"
  IMAGE_NAME: "$REGISTRY/monlab/myapp"

build:
  stage: build
  script:
    - docker build -t $IMAGE_NAME:$CI_COMMIT_SHORT_SHA .
    - docker tag $IMAGE_NAME:$CI_COMMIT_SHORT_SHA $IMAGE_NAME:latest

push:
  stage: push
  script:
    - docker push $IMAGE_NAME:$CI_COMMIT_SHORT_SHA
    - docker push $IMAGE_NAME:latest
  only:
    - main

deploy:
  stage: deploy
  script:
    - kubectl set image deployment/myapp myapp=$IMAGE_NAME:$CI_COMMIT_SHORT_SHA
    - kubectl rollout status deployment/myapp
```

### GitHub Actions exemple

```yaml
# .github/workflows/docker-build.yml
name: Build and Push Docker Image

on:
  push:
    branches: [ main ]
    tags: [ 'v*' ]

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
    - uses: actions/checkout@v3

    - name: Set up Docker Buildx
      uses: docker/setup-buildx-action@v2

    - name: Login to Registry
      uses: docker/login-action@v2
      with:
        registry: registry.monlab.local:5000
        username: ${{ secrets.REGISTRY_USERNAME }}
        password: ${{ secrets.REGISTRY_PASSWORD }}

    - name: Build and push
      uses: docker/build-push-action@v4
      with:
        context: .
        push: true
        tags: |
          registry.monlab.local:5000/monlab/myapp:latest
          registry.monlab.local:5000/monlab/myapp:${{ github.sha }}
        cache-from: type=registry,ref=registry.monlab.local:5000/monlab/myapp:buildcache
        cache-to: type=registry,ref=registry.monlab.local:5000/monlab/myapp:buildcache,mode=max
```

## Migration vers Harbor (optionnel)

Pour des besoins plus avancés, Harbor offre une interface web et des fonctionnalités enterprise :

### Pourquoi Harbor ?

- **Interface web** intuitive pour naviguer dans les images
- **Scanning de vulnérabilités** intégré (Trivy/Clair)
- **Réplication** entre registries
- **RBAC avancé** avec projets et équipes
- **Notary** pour la signature d'images
- **Webhooks** pour l'automatisation
- **Quotas** par projet

### Installation de Harbor sur MicroK8s

```bash
# Ajouter le repo Helm
helm repo add harbor https://helm.goharbor.io
helm repo update

# Créer les valeurs personnalisées
cat > harbor-values.yaml <<EOF
expose:
  type: nodePort
  tls:
    enabled: false
  nodePort:
    ports:
      http:
        nodePort: 30080
      https:
        nodePort: 30443

externalURL: http://harbor.monlab.local:30080

persistence:
  persistentVolumeClaim:
    registry:
      size: 20Gi
    chartmuseum:
      size: 5Gi
    database:
      size: 5Gi
    redis:
      size: 1Gi

harborAdminPassword: "HarborAdmin123!"
EOF

# Installer Harbor
helm install harbor harbor/harbor \
  --namespace harbor \
  --create-namespace \
  -f harbor-values.yaml
```

## Monitoring du registry

### Métriques Prometheus

Exposez les métriques de votre registry :

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: registry-config
  namespace: container-registry
data:
  config.yml: |
    version: 0.1
    http:
      addr: :5000
      debug:
        addr: :5001
        prometheus:
          enabled: true
          path: /metrics
    storage:
      filesystem:
        rootdirectory: /var/lib/registry
```

### Dashboard Grafana

Créez un dashboard pour surveiller :
- Nombre d'images par repository
- Taille totale du stockage utilisé
- Fréquence des push/pull
- Latence des opérations
- Erreurs HTTP

### Alertes importantes

```yaml
# Alerte Prometheus pour espace disque
groups:
- name: registry
  rules:
  - alert: RegistryDiskSpaceLow
    expr: |
      (node_filesystem_avail_bytes{mountpoint="/var/lib/registry"} /
       node_filesystem_size_bytes{mountpoint="/var/lib/registry"}) < 0.1
    for: 5m
    labels:
      severity: warning
    annotations:
      summary: "Registry disk space low"
      description: "Registry has less than 10% disk space remaining"
```

## Troubleshooting courant

### Problème : "unauthorized: authentication required"

**Cause** : Authentification non configurée
**Solution** :
```bash
# Se connecter au registry
docker login localhost:32000
# Username: monuser
# Password: monpassword

# Ou créer un secret Kubernetes
kubectl create secret docker-registry regcred \
  --docker-server=localhost:32000 \
  --docker-username=monuser \
  --docker-password=monpassword
```

### Problème : "manifest unknown"

**Cause** : L'image ou le tag n'existe pas
**Solution** :
```bash
# Vérifier les images disponibles
curl http://localhost:32000/v2/_catalog

# Vérifier les tags d'une image
curl http://localhost:32000/v2/monlab/myapp/tags/list
```

### Problème : Espace disque insuffisant

**Cause** : Accumulation d'images non utilisées
**Solution** :
```bash
# Nettoyer les images locales Docker
docker system prune -a

# Exécuter le garbage collector du registry
docker exec registry bin/registry garbage-collect \
  /etc/docker/registry/config.yml
```

### Problème : Push très lent

**Cause** : Layers non optimisées ou réseau lent
**Solution** :
- Utiliser des images de base plus petites
- Implémenter le multi-stage build
- Vérifier la bande passante réseau
- Activer la compression

## Conclusion et perspectives

La mise en place d'un registry privé dans votre lab MicroK8s représente un jalon majeur dans votre parcours DevOps. Vous disposez maintenant d'un système complet pour :

- **Stocker** vos images de manière sécurisée et privée
- **Versionner** précisément chaque build de vos applications
- **Distribuer** efficacement vos images dans votre cluster
- **Automatiser** le cycle de vie complet de vos conteneurs
- **Sécuriser** votre chaîne de déploiement

Les compétences acquises avec votre registry local sont directement transposables en entreprise, où la gestion d'images est cruciale pour la sécurité, la conformité et la performance.

Dans la prochaine section, nous verrons comment automatiser complètement le flux de votre code source jusqu'au déploiement en production avec les pipelines CI/CD, en utilisant votre registry comme hub central de distribution.

⏭️
