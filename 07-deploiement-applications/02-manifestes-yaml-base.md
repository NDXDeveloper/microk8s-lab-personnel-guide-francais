🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 7.2 - Manifestes YAML de base

## Introduction

Les manifestes YAML sont le langage que vous utilisez pour communiquer avec Kubernetes. C'est comme écrire une recette de cuisine : vous décrivez ce que vous voulez, et Kubernetes s'occupe de le préparer. Dans cette section, nous allons apprendre à écrire des manifestes YAML clairs, maintenables et conformes aux bonnes pratiques.

## Qu'est-ce que YAML ?

YAML (Yet Another Markup Language) est un format de sérialisation de données lisible par l'humain. Kubernetes l'a choisi car il est :
- **Lisible** : Structure claire avec indentation
- **Simple** : Pas de balises complexes comme XML
- **Expressif** : Peut représenter des structures complexes
- **Portable** : Format texte standard

### Règles fondamentales de YAML

**1. L'indentation est cruciale**
```yaml
# CORRECT - indentation cohérente avec 2 espaces
metadata:
  name: mon-app
  labels:
    app: webapp

# INCORRECT - mélange d'espaces et de tabs
metadata:
	name: mon-app  # Tab ici = erreur
  labels:  # Espaces ici
    app: webapp
```

**2. Les types de données**
```yaml
# Chaîne de caractères (string)
name: mon-application
version: "1.0"  # Guillemets optionnels sauf cas spéciaux

# Nombre (number)
replicas: 3
port: 8080

# Booléen (boolean)
enabled: true
debug: false

# Null
middleware: null
# ou
middleware: ~

# Liste (array)
ports:
  - 80
  - 443
# ou format inline
ports: [80, 443]

# Dictionnaire (map/object)
labels:
  app: webapp
  version: v1
# ou format inline
labels: {app: webapp, version: v1}
```

**3. Les chaînes multi-lignes**
```yaml
# Préserve les retours à la ligne (literal)
description: |
  Ceci est une application
  sur plusieurs lignes.
  Chaque ligne est préservée.

# Fusionne en une seule ligne (folded)
summary: >
  Ce texte sera
  fusionné en une
  seule ligne.
```

**4. Les commentaires**
```yaml
# Ceci est un commentaire
apiVersion: v1  # Commentaire en fin de ligne
kind: Pod
# metadata:  # Ligne commentée = ignorée
```

## Structure d'un manifeste Kubernetes

Tout manifeste Kubernetes suit une structure standard avec quatre sections principales :

```yaml
apiVersion: apps/v1           # 1. Version de l'API
kind: Deployment              # 2. Type de ressource
metadata:                     # 3. Métadonnées
  name: webapp-deployment
  namespace: production
  labels:
    app: webapp
spec:                        # 4. Spécification
  replicas: 3
  selector:
    matchLabels:
      app: webapp
  template:
    # ... configuration du Pod
```

### 1. apiVersion : La version de l'API

L'`apiVersion` indique quelle version de l'API Kubernetes utiliser. Chaque type de ressource a ses versions supportées.

**Versions courantes :**
```yaml
# Ressources de base
apiVersion: v1                    # Pod, Service, ConfigMap, Secret, PVC

# Workloads
apiVersion: apps/v1                # Deployment, StatefulSet, DaemonSet, ReplicaSet

# Réseau
apiVersion: networking.k8s.io/v1   # Ingress, NetworkPolicy

# Batch/Jobs
apiVersion: batch/v1               # Job, CronJob

# Autoscaling
apiVersion: autoscaling/v2         # HorizontalPodAutoscaler

# RBAC
apiVersion: rbac.authorization.k8s.io/v1  # Role, ClusterRole, RoleBinding
```

### 2. kind : Le type de ressource

Le `kind` spécifie quel type d'objet Kubernetes vous créez.

**Types essentiels pour les applications :**
- `Pod` : Unité de base contenant les conteneurs
- `Deployment` : Gestion des Pods avec réplication
- `Service` : Exposition réseau des Pods
- `Ingress` : Routage HTTP/HTTPS externe
- `ConfigMap` : Configuration non-sensible
- `Secret` : Données sensibles
- `PersistentVolumeClaim` : Demande de stockage

### 3. metadata : Les métadonnées

La section `metadata` contient les informations d'identification et d'organisation.

```yaml
metadata:
  name: webapp                    # Nom unique dans le namespace
  namespace: production           # Namespace (défaut: default)
  labels:                        # Labels pour organiser/sélectionner
    app: webapp
    version: v1.2.3
    environment: production
    team: backend
  annotations:                   # Métadonnées additionnelles
    description: "Application web principale"
    documentation: "https://wiki.internal/webapp"
    contact: "team-backend@company.com"
```

**Règles de nommage :**
- Caractères autorisés : lettres minuscules, chiffres, tirets (-)
- Doit commencer et finir par une lettre ou un chiffre
- Maximum 253 caractères (63 pour certaines ressources)
- Exemples valides : `webapp`, `backend-api-v2`, `cache-redis-01`

**Labels vs Annotations :**
- **Labels** : Pour sélectionner et organiser (max 63 caractères)
- **Annotations** : Pour stocker des métadonnées (pas de limite de taille)

### 4. spec : La spécification

La section `spec` contient la configuration spécifique à chaque type de ressource.

## Manifestes pour chaque type de ressource

### Pod : Le manifeste le plus simple

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: nginx-pod
  labels:
    app: nginx
spec:
  containers:
  - name: nginx
    image: nginx:1.21
    ports:
    - containerPort: 80
      name: http
    env:
    - name: ENVIRONMENT
      value: "development"
    resources:
      requests:
        memory: "64Mi"
        cpu: "250m"
      limits:
        memory: "128Mi"
        cpu: "500m"
```

### Deployment : Le manifeste de production

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: webapp-deployment
  labels:
    app: webapp
    version: v1.0.0
spec:
  replicas: 3
  revisionHistoryLimit: 5  # Garder 5 anciennes versions
  selector:
    matchLabels:
      app: webapp
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 1
  template:
    metadata:
      labels:
        app: webapp
        version: v1.0.0
    spec:
      containers:
      - name: webapp
        image: myregistry/webapp:v1.0.0
        imagePullPolicy: IfNotPresent
        ports:
        - containerPort: 8080
          name: http
          protocol: TCP
        env:
        - name: PORT
          value: "8080"
        - name: LOG_LEVEL
          value: "info"
        livenessProbe:
          httpGet:
            path: /health
            port: 8080
          initialDelaySeconds: 30
          periodSeconds: 10
          timeoutSeconds: 5
          failureThreshold: 3
        readinessProbe:
          httpGet:
            path: /ready
            port: 8080
          initialDelaySeconds: 10
          periodSeconds: 5
          timeoutSeconds: 3
          failureThreshold: 3
        resources:
          requests:
            memory: "128Mi"
            cpu: "100m"
          limits:
            memory: "256Mi"
            cpu: "500m"
        volumeMounts:
        - name: config
          mountPath: /etc/webapp
          readOnly: true
      volumes:
      - name: config
        configMap:
          name: webapp-config
```

### Service : L'exposition réseau

```yaml
apiVersion: v1
kind: Service
metadata:
  name: webapp-service
  labels:
    app: webapp
spec:
  type: ClusterIP  # ou NodePort, LoadBalancer
  selector:
    app: webapp
  ports:
  - name: http
    port: 80         # Port du Service
    targetPort: 8080 # Port du conteneur
    protocol: TCP
  - name: https
    port: 443
    targetPort: 8443
    protocol: TCP
  sessionAffinity: None  # ou ClientIP pour sticky sessions
```

### ConfigMap : La configuration

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: webapp-config
  labels:
    app: webapp
data:
  # Valeurs simples
  database_host: "db.default.svc.cluster.local"
  database_port: "5432"
  cache_ttl: "3600"

  # Fichier de configuration complet
  application.yaml: |
    server:
      port: 8080
      timeout: 30s
    database:
      host: db.default.svc.cluster.local
      port: 5432
      name: webapp_db
      pool_size: 10
    cache:
      enabled: true
      ttl: 3600
    features:
      new_ui: true
      beta_features: false

  # Script d'initialisation
  init.sh: |
    #!/bin/bash
    echo "Initializing application..."
    mkdir -p /data/uploads
    chmod 755 /data/uploads
    echo "Initialization complete"
```

### Secret : Les données sensibles

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: webapp-secret
  labels:
    app: webapp
type: Opaque  # Type générique
data:
  # Valeurs en base64
  username: YWRtaW4=     # echo -n "admin" | base64
  password: UEBzc3cwcmQ=  # echo -n "P@ssw0rd" | base64
---
# Alternative avec stringData (plus lisible)
apiVersion: v1
kind: Secret
metadata:
  name: webapp-secret-clear
type: Opaque
stringData:  # Kubernetes encode automatiquement en base64
  username: admin
  password: P@ssw0rd
  api_key: "sk_live_abcd1234efgh5678"
```

### Ingress : Le routage externe

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: webapp-ingress
  labels:
    app: webapp
  annotations:
    # Annotations spécifiques à NGINX
    nginx.ingress.kubernetes.io/rewrite-target: /
    nginx.ingress.kubernetes.io/ssl-redirect: "true"
    nginx.ingress.kubernetes.io/proxy-body-size: "10m"
    # Cert-manager pour SSL automatique
    cert-manager.io/cluster-issuer: "letsencrypt-prod"
spec:
  ingressClassName: nginx  # Classe d'Ingress à utiliser
  tls:
  - hosts:
    - webapp.example.com
    - www.webapp.example.com
    secretName: webapp-tls
  rules:
  - host: webapp.example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: webapp-service
            port:
              number: 80
  - host: www.webapp.example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: webapp-service
            port:
              number: 80
```

### PersistentVolumeClaim : Le stockage

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: webapp-storage
  labels:
    app: webapp
spec:
  accessModes:
    - ReadWriteOnce  # RWO: un seul nœud en lecture/écriture
    # - ReadOnlyMany   # ROX: plusieurs nœuds en lecture seule
    # - ReadWriteMany  # RWX: plusieurs nœuds en lecture/écriture
  resources:
    requests:
      storage: 10Gi
  storageClassName: microk8s-hostpath  # Classe de stockage MicroK8s
  # volumeMode: Filesystem  # ou Block
```

## Techniques avancées

### 1. Références entre ressources

**Référencer un ConfigMap dans un Deployment :**
```yaml
spec:
  containers:
  - name: app
    env:
    # Une seule valeur du ConfigMap
    - name: DATABASE_HOST
      valueFrom:
        configMapKeyRef:
          name: webapp-config
          key: database_host

    # Toutes les valeurs du ConfigMap comme variables
    envFrom:
    - configMapRef:
        name: webapp-config

    # Monter comme fichiers
    volumeMounts:
    - name: config-volume
      mountPath: /etc/config
  volumes:
  - name: config-volume
    configMap:
      name: webapp-config
```

**Référencer un Secret :**
```yaml
spec:
  containers:
  - name: app
    env:
    - name: DB_PASSWORD
      valueFrom:
        secretKeyRef:
          name: webapp-secret
          key: password

    # Monter comme fichiers
    volumeMounts:
    - name: secret-volume
      mountPath: /etc/secrets
      readOnly: true
  volumes:
  - name: secret-volume
    secret:
      secretName: webapp-secret
      defaultMode: 0400  # Permissions restrictives
```

### 2. Multi-documents YAML

Vous pouvez définir plusieurs ressources dans un seul fichier avec `---` :

```yaml
# configmap.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: webapp-config
data:
  environment: "production"
---
apiVersion: v1
kind: Secret
metadata:
  name: webapp-secret
stringData:
  password: "secret123"
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: webapp
spec:
  replicas: 3
  # ...
---
apiVersion: v1
kind: Service
metadata:
  name: webapp-service
spec:
  selector:
    app: webapp
  # ...
```

### 3. Templates et variables

Kubernetes ne supporte pas nativement les variables, mais vous pouvez utiliser des outils :

**Avec envsubst (substitution de variables shell) :**
```yaml
# deployment-template.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: ${APP_NAME}
spec:
  replicas: ${REPLICAS}
  template:
    spec:
      containers:
      - name: app
        image: ${IMAGE_REGISTRY}/${APP_NAME}:${VERSION}
```

```bash
# Utilisation
export APP_NAME=webapp
export REPLICAS=3
export IMAGE_REGISTRY=myregistry.io
export VERSION=v1.0.0

envsubst < deployment-template.yaml | kubectl apply -f -
```

### 4. Validation des manifestes

**Validation syntaxique :**
```bash
# Vérifier la syntaxe YAML
kubectl apply --dry-run=client -f manifest.yaml

# Validation côté serveur
kubectl apply --dry-run=server -f manifest.yaml

# Voir le résultat sans appliquer
kubectl apply --dry-run=client -o yaml -f manifest.yaml
```

**Outils de validation :**
```bash
# yamllint pour la syntaxe YAML
yamllint manifest.yaml

# kubeval pour la conformité Kubernetes
kubeval manifest.yaml

# kubectl explain pour la documentation
kubectl explain deployment.spec.template.spec.containers
```

## Organisation des manifestes

### Structure de répertoires recommandée

```
my-app/
├── base/                    # Configurations de base
│   ├── deployment.yaml
│   ├── service.yaml
│   ├── configmap.yaml
│   └── kustomization.yaml
├── overlays/               # Variations par environnement
│   ├── development/
│   │   ├── deployment-patch.yaml
│   │   └── kustomization.yaml
│   ├── staging/
│   │   ├── deployment-patch.yaml
│   │   └── kustomization.yaml
│   └── production/
│       ├── deployment-patch.yaml
│       ├── resources.yaml
│       └── kustomization.yaml
└── scripts/
    ├── deploy.sh
    └── rollback.sh
```

### Conventions de nommage des fichiers

```bash
# Format : [numéro]-[type]-[nom].yaml
01-namespace.yaml
02-configmap-webapp.yaml
03-secret-webapp.yaml
04-deployment-webapp.yaml
05-service-webapp.yaml
06-ingress-webapp.yaml

# Ou par type de ressource
namespaces/
  production.yaml
deployments/
  webapp.yaml
  worker.yaml
services/
  webapp.yaml
  worker.yaml
```

## Bonnes pratiques

### 1. Toujours spécifier les ressources

```yaml
resources:
  requests:    # Minimum garanti
    memory: "128Mi"
    cpu: "100m"
  limits:      # Maximum autorisé
    memory: "512Mi"
    cpu: "1000m"
```

### 2. Utiliser des tags d'image spécifiques

```yaml
# MAUVAIS - peut changer sans prévenir
image: nginx:latest
image: nginx

# BON - version fixe
image: nginx:1.21.6
image: myapp:v1.2.3
image: myapp@sha256:45b23dee...  # Encore plus précis
```

### 3. Définir des health checks

```yaml
livenessProbe:
  httpGet:
    path: /health
    port: 8080
  initialDelaySeconds: 30
  periodSeconds: 10

readinessProbe:
  httpGet:
    path: /ready
    port: 8080
  initialDelaySeconds: 10
  periodSeconds: 5
```

### 4. Utiliser des labels cohérents

```yaml
labels:
  app: webapp              # Nom de l'application
  component: frontend      # Partie de l'application
  version: v1.2.3         # Version
  environment: production  # Environnement
  team: platform          # Équipe responsable
  managed-by: kubectl     # Outil de déploiement
```

### 5. Sécuriser les conteneurs

```yaml
securityContext:
  runAsNonRoot: true
  runAsUser: 1000
  fsGroup: 2000
  capabilities:
    drop:
    - ALL
  readOnlyRootFilesystem: true
  allowPrivilegeEscalation: false
```

### 6. Documenter avec des annotations

```yaml
annotations:
  description: "Service web principal de l'application"
  documentation: "https://docs.internal/webapp"
  prometheus.io/scrape: "true"
  prometheus.io/port: "9090"
```

## Erreurs courantes à éviter

### 1. Problèmes d'indentation

```yaml
# ERREUR - indentation incohérente
spec:
  replicas: 3
   selector:  # 3 espaces au lieu de 2
    matchLabels:
      app: webapp

# CORRECT
spec:
  replicas: 3
  selector:
    matchLabels:
      app: webapp
```

### 2. Types de données incorrects

```yaml
# ERREUR - nombre comme chaîne
replicas: "3"  # Devrait être: replicas: 3

# ERREUR - booléen comme chaîne
enabled: "true"  # Devrait être: enabled: true

# ERREUR - port comme nombre décimal
port: 80.0  # Devrait être: port: 80
```

### 3. Labels non correspondants

```yaml
# ERREUR - Le selector ne correspond pas aux labels du template
spec:
  selector:
    matchLabels:
      app: webapp
  template:
    metadata:
      labels:
        app: web-application  # Ne correspond pas!
```

### 4. Noms invalides

```yaml
# ERREUR - caractères non autorisés
name: Web_App  # Underscore non autorisé
name: WebApp   # Majuscules non autorisées
name: web.app  # Point non autorisé (sauf FQDN)

# CORRECT
name: web-app
name: webapp
name: web-app-v2
```

## Outils utiles

### kubectl et manipulation YAML

```bash
# Générer un manifeste de base
kubectl create deployment webapp --image=nginx --dry-run=client -o yaml > deployment.yaml

# Extraire la configuration d'une ressource existante
kubectl get deployment webapp -o yaml > webapp-backup.yaml

# Éditer directement
kubectl edit deployment webapp

# Appliquer avec remplacement
kubectl replace -f deployment.yaml

# Patcher une ressource
kubectl patch deployment webapp -p '{"spec":{"replicas":5}}'
```

### Conversion et formatage

```bash
# JSON vers YAML
kubectl get deployment webapp -o json | yq eval -P - > deployment.yaml

# YAML vers JSON
yq eval -o=json deployment.yaml > deployment.json

# Formater/valider YAML
yq eval deployment.yaml > deployment-formatted.yaml
```

## Conclusion

Les manifestes YAML sont votre principal moyen de communication avec Kubernetes. Maîtriser leur écriture est essentiel pour :
- Déployer des applications de manière reproductible
- Maintenir une configuration claire et documentée
- Faciliter la collaboration en équipe
- Automatiser vos déploiements

Points clés à retenir :
1. **Structure standard** : apiVersion, kind, metadata, spec
2. **Indentation cohérente** : Toujours 2 espaces, jamais de tabs
3. **Labels cohérents** : Pour organiser et sélectionner
4. **Ressources définies** : Toujours spécifier requests/limits
5. **Health checks** : Liveness et readiness probes
6. **Versions fixes** : Pas de "latest" en production

Dans la prochaine section (7.3), nous mettrons en pratique ces connaissances en déployant une application web complète, étape par étape, en utilisant tous les types de manifestes que nous venons d'étudier.

⏭️
