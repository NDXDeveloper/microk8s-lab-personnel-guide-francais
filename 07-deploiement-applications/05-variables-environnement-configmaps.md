🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 7.5 - Variables d'environnement et ConfigMaps

## Introduction

Imaginez que vous développez une application qui doit fonctionner dans différents environnements : développement, test, production. Chaque environnement a ses propres bases de données, URLs d'API, niveaux de log... Comment gérer ces différences sans modifier votre code ou reconstruire vos images Docker ? C'est exactement le problème que résolvent les variables d'environnement et les ConfigMaps.

Dans cette section, nous allons explorer comment externaliser la configuration de vos applications pour les rendre flexibles, portables et faciles à maintenir.

## Pourquoi externaliser la configuration ?

### Le problème de la configuration codée en dur

**Mauvaise pratique :**
```python
# app.py - Configuration en dur dans le code
DATABASE_URL = "postgresql://prod_user:SuperSecret123@db.prod.example.com:5432/myapp"
API_KEY = "sk_live_abc123def456"
DEBUG_MODE = False
CACHE_TTL = 3600
```

**Problèmes de cette approche :**
- 🔒 **Sécurité** : Les secrets sont dans le code source
- 🔄 **Flexibilité** : Besoin de recompiler pour changer la config
- 🚀 **Déploiement** : Images différentes par environnement
- 👥 **Collaboration** : Conflits Git sur les fichiers de config
- 🔑 **Gestion des secrets** : Mots de passe visibles dans le repository

### La solution : Configuration externalisée

```python
# app.py - Configuration externalisée
import os

DATABASE_URL = os.getenv('DATABASE_URL')
API_KEY = os.getenv('API_KEY')
DEBUG_MODE = os.getenv('DEBUG_MODE', 'false').lower() == 'true'
CACHE_TTL = int(os.getenv('CACHE_TTL', '3600'))
```

**Avantages :**
- ✅ Une seule image pour tous les environnements
- ✅ Configuration modifiable sans rebuild
- ✅ Secrets séparés du code
- ✅ Gestion centralisée
- ✅ Principe des 12-Factor Apps respecté

## Partie 1 : Variables d'environnement natives

### Définition directe dans le Pod

La façon la plus simple de définir des variables d'environnement :

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: app-with-env
spec:
  containers:
  - name: app
    image: myapp:v1.0.0
    env:
    # Variable simple
    - name: APP_NAME
      value: "My Application"

    # Variable numérique
    - name: PORT
      value: "8080"

    # Variable booléenne
    - name: DEBUG_MODE
      value: "false"

    # Variable complexe (JSON)
    - name: FEATURES
      value: '{"newUI": true, "betaFeatures": false, "maxUpload": 10485760}'
```

### Variables d'environnement dynamiques

Kubernetes peut injecter des informations dynamiques :

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: dynamic-env-app
spec:
  replicas: 3
  selector:
    matchLabels:
      app: myapp
  template:
    metadata:
      labels:
        app: myapp
    spec:
      containers:
      - name: app
        image: myapp:v1.0.0
        env:
        # Métadonnées du Pod
        - name: POD_NAME
          valueFrom:
            fieldRef:
              fieldPath: metadata.name

        - name: POD_NAMESPACE
          valueFrom:
            fieldRef:
              fieldPath: metadata.namespace

        - name: POD_IP
          valueFrom:
            fieldRef:
              fieldPath: status.podIP

        - name: NODE_NAME
          valueFrom:
            fieldRef:
              fieldPath: spec.nodeName

        - name: SERVICE_ACCOUNT
          valueFrom:
            fieldRef:
              fieldPath: spec.serviceAccountName

        # Ressources du conteneur
        - name: MEMORY_LIMIT
          valueFrom:
            resourceFieldRef:
              containerName: app
              resource: limits.memory

        - name: CPU_REQUEST
          valueFrom:
            resourceFieldRef:
              containerName: app
              resource: requests.cpu
              divisor: "1m"  # Convertir en millicores
```

### Variables d'environnement dépendantes

Vous pouvez référencer d'autres variables :

```yaml
env:
- name: PROTOCOL
  value: "https"
- name: DOMAIN
  value: "api.example.com"
- name: PORT
  value: "443"
- name: BASE_URL
  value: "$(PROTOCOL)://$(DOMAIN):$(PORT)"
# Résultat: BASE_URL="https://api.example.com:443"

- name: DATABASE_HOST
  value: "db.example.com"
- name: DATABASE_PORT
  value: "5432"
- name: DATABASE_NAME
  value: "myapp"
- name: DATABASE_URL
  value: "postgresql://user:pass@$(DATABASE_HOST):$(DATABASE_PORT)/$(DATABASE_NAME)"
```

## Partie 2 : ConfigMaps - La solution Kubernetes

### Qu'est-ce qu'un ConfigMap ?

Un ConfigMap est un objet Kubernetes qui stocke des données de configuration non-sensibles sous forme de paires clé-valeur. Pensez-y comme un fichier de configuration centralisé et versionné.

**Caractéristiques :**
- 📝 Stocke du texte ou des données binaires
- 🔄 Modifiable sans redéployer les Pods
- 📦 Peut contenir des fichiers entiers
- 🏷️ Versionnable et traçable
- 🚫 **NON sécurisé** (utiliser Secrets pour les données sensibles)

### Création de ConfigMaps

#### Méthode 1 : Déclarative (YAML)

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config
  namespace: my-app
  labels:
    app: myapp
    environment: production
data:
  # Propriétés simples (clé: valeur)
  database.host: "db.production.svc.cluster.local"
  database.port: "5432"
  database.name: "production_db"

  log.level: "info"
  cache.ttl: "3600"
  feature.flags: "new-ui,dark-mode,analytics"

  # Fichier de configuration complet
  application.properties: |
    # Configuration Java/Spring
    server.port=8080
    server.context-path=/api

    # Database
    spring.datasource.url=jdbc:postgresql://db.production:5432/myapp
    spring.datasource.driver-class-name=org.postgresql.Driver
    spring.jpa.hibernate.ddl-auto=validate
    spring.jpa.show-sql=false

    # Cache
    spring.cache.type=redis
    spring.redis.host=redis.production
    spring.redis.port=6379
    spring.redis.timeout=2000

    # Monitoring
    management.endpoints.web.exposure.include=health,info,metrics
    management.metrics.export.prometheus.enabled=true

  # Configuration JSON
  features.json: |
    {
      "features": {
        "newUI": {
          "enabled": true,
          "rolloutPercentage": 100
        },
        "darkMode": {
          "enabled": true,
          "default": false
        },
        "advancedSearch": {
          "enabled": false,
          "betaUsers": ["user1", "user2"]
        }
      },
      "maintenance": {
        "enabled": false,
        "message": "System under maintenance",
        "expectedEnd": "2024-01-01T00:00:00Z"
      }
    }

  # Script shell
  startup.sh: |
    #!/bin/bash
    echo "Starting application..."

    # Vérifier la connexion à la base de données
    until pg_isready -h $DATABASE_HOST -p $DATABASE_PORT; do
      echo "Waiting for database..."
      sleep 2
    done

    # Migrations
    echo "Running database migrations..."
    flyway migrate

    # Démarrer l'application
    echo "Starting server..."
    exec java -jar /app/myapp.jar
```

#### Méthode 2 : Impérative (kubectl)

```bash
# Depuis des valeurs littérales
kubectl create configmap app-config \
  --from-literal=database.host=localhost \
  --from-literal=database.port=5432 \
  --from-literal=log.level=debug \
  -n my-app

# Depuis un fichier
kubectl create configmap app-config \
  --from-file=application.properties \
  --from-file=features.json \
  -n my-app

# Depuis un répertoire entier
kubectl create configmap app-config \
  --from-file=./config/ \
  -n my-app

# Depuis un fichier env
kubectl create configmap app-config \
  --from-env-file=production.env \
  -n my-app
```

#### Méthode 3 : Depuis des fichiers avec générateurs

```yaml
# kustomization.yaml
configMapGenerator:
- name: app-config
  files:
  - application.properties
  - config/database.yaml
  - config/features.json
  literals:
  - ENVIRONMENT=production
  - VERSION=1.2.3
```

### Utilisation des ConfigMaps dans les Pods

#### 1. Comme variables d'environnement individuelles

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: app-deployment
spec:
  replicas: 3
  selector:
    matchLabels:
      app: myapp
  template:
    metadata:
      labels:
        app: myapp
    spec:
      containers:
      - name: app
        image: myapp:v1.0.0
        env:
        # Variable unique depuis ConfigMap
        - name: DATABASE_HOST
          valueFrom:
            configMapKeyRef:
              name: app-config
              key: database.host

        - name: DATABASE_PORT
          valueFrom:
            configMapKeyRef:
              name: app-config
              key: database.port

        - name: LOG_LEVEL
          valueFrom:
            configMapKeyRef:
              name: app-config
              key: log.level
              optional: true  # Ne pas échouer si la clé n'existe pas

        # Combinaison avec d'autres variables
        - name: DATABASE_URL
          value: "postgresql://$(DATABASE_HOST):$(DATABASE_PORT)/mydb"
```

#### 2. Toutes les clés comme variables d'environnement

```yaml
spec:
  containers:
  - name: app
    image: myapp:v1.0.0
    envFrom:
    # Charger tout le ConfigMap
    - configMapRef:
        name: app-config

    # Avec un préfixe (optionnel)
    - configMapRef:
        name: database-config
        prefix: DB_  # DB_HOST, DB_PORT, etc.

    # Combiner plusieurs sources
    - configMapRef:
        name: app-config
    - configMapRef:
        name: feature-flags
    - secretRef:
        name: app-secrets
```

#### 3. Comme fichiers montés

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: app-with-config-files
spec:
  selector:
    matchLabels:
      app: myapp
  template:
    metadata:
      labels:
        app: myapp
    spec:
      containers:
      - name: app
        image: myapp:v1.0.0
        volumeMounts:
        # Monter tout le ConfigMap
        - name: config-volume
          mountPath: /etc/config
          readOnly: true

        # Monter des clés spécifiques
        - name: app-properties
          mountPath: /app/config/application.properties
          subPath: application.properties

        - name: features-config
          mountPath: /app/config/features.json
          subPath: features.json

        # Monter avec un mode spécifique
        - name: scripts
          mountPath: /scripts
          defaultMode: 0755  # Exécutable

      volumes:
      # Volume depuis ConfigMap complet
      - name: config-volume
        configMap:
          name: app-config

      # Volume avec sélection de clés
      - name: app-properties
        configMap:
          name: app-config
          items:
          - key: application.properties
            path: application.properties

      - name: features-config
        configMap:
          name: app-config
          items:
          - key: features.json
            path: features.json
            mode: 0644  # Permissions spécifiques

      # Volume pour scripts exécutables
      - name: scripts
        configMap:
          name: app-config
          defaultMode: 0755
          items:
          - key: startup.sh
            path: startup.sh
```

### ConfigMaps immutables

Pour améliorer les performances et la stabilité :

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config-v2
data:
  config.yaml: |
    version: 2.0.0
immutable: true  # Ne peut plus être modifié
```

**Avantages des ConfigMaps immutables :**
- 🚀 Performance : Pas de watch API nécessaire
- 🔒 Sécurité : Protection contre les modifications accidentelles
- 📦 Versionning : Force l'utilisation de nouvelles versions

## Partie 3 : Patterns et bonnes pratiques

### Pattern 1 : Configuration par environnement

Structure organisée pour multiple environnements :

```yaml
# configmap-base.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config-base
data:
  app.name: "MyApplication"
  app.version: "1.0.0"
  default.timeout: "30"
---
# configmap-dev.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config
  namespace: development
data:
  environment: "development"
  database.host: "localhost"
  database.port: "5432"
  log.level: "debug"
  debug.enabled: "true"
---
# configmap-staging.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config
  namespace: staging
data:
  environment: "staging"
  database.host: "db-staging.example.com"
  database.port: "5432"
  log.level: "info"
  debug.enabled: "false"
---
# configmap-prod.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config
  namespace: production
data:
  environment: "production"
  database.host: "db-prod.example.com"
  database.port: "5432"
  log.level: "warn"
  debug.enabled: "false"
  monitoring.enabled: "true"
  backup.enabled: "true"
```

### Pattern 2 : Configuration hiérarchique

Combiner plusieurs ConfigMaps pour une configuration en couches :

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: layered-config-app
spec:
  template:
    spec:
      containers:
      - name: app
        image: myapp:v1.0.0
        envFrom:
        # Ordre important : du plus général au plus spécifique
        # Les derniers écrasent les premiers

        # 1. Configuration globale
        - configMapRef:
            name: global-config
            optional: true

        # 2. Configuration de l'environnement
        - configMapRef:
            name: environment-config

        # 3. Configuration de l'application
        - configMapRef:
            name: app-config

        # 4. Configuration spécifique au déploiement
        - configMapRef:
            name: deployment-config
            optional: true

        # 5. Secrets (toujours en dernier)
        - secretRef:
            name: app-secrets
```

### Pattern 3 : Hot reload de configuration

Certaines applications peuvent recharger leur configuration sans redémarrage :

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: dynamic-config
  annotations:
    reloader.stakater.com/match: "true"  # Pour l'outil Reloader
data:
  feature-flags.json: |
    {
      "features": {
        "newCheckout": true,
        "darkMode": false
      }
    }
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: app-with-hot-reload
  annotations:
    reloader.stakater.com/auto: "true"  # Redémarre si ConfigMap change
spec:
  template:
    spec:
      containers:
      - name: app
        image: myapp:v1.0.0
        volumeMounts:
        - name: config
          mountPath: /config
        env:
        - name: CONFIG_PATH
          value: "/config/feature-flags.json"
        - name: RELOAD_CONFIG
          value: "true"  # L'app surveille les changements
      volumes:
      - name: config
        configMap:
          name: dynamic-config
```

### Pattern 4 : Configuration avec validation

Valider la configuration avant utilisation :

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: validated-config
  annotations:
    config-validator.io/schema: "v1.2.0"
data:
  config.yaml: |
    # YAML avec schéma strict
    version: 1.2.0
    database:
      host: db.example.com
      port: 5432
      pool:
        min: 10
        max: 100
    cache:
      ttl: 3600
      provider: redis
---
# InitContainer pour validation
apiVersion: apps/v1
kind: Deployment
metadata:
  name: app-with-validation
spec:
  template:
    spec:
      initContainers:
      # Valider la configuration avant de démarrer
      - name: config-validator
        image: config-validator:latest
        command: ['sh', '-c']
        args:
        - |
          echo "Validating configuration..."
          validate-config /config/config.yaml
          if [ $? -ne 0 ]; then
            echo "Configuration validation failed!"
            exit 1
          fi
          echo "Configuration valid."
        volumeMounts:
        - name: config
          mountPath: /config

      containers:
      - name: app
        image: myapp:v1.0.0
        volumeMounts:
        - name: config
          mountPath: /app/config

      volumes:
      - name: config
        configMap:
          name: validated-config
```

## Partie 4 : Cas d'usage avancés

### Configuration multi-formats

Gérer différents formats de configuration :

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: multi-format-config
data:
  # YAML
  app-config.yaml: |
    server:
      port: 8080
      host: 0.0.0.0
    database:
      connections: 10

  # JSON
  features.json: |
    {"features": ["api", "ui", "analytics"]}

  # Properties (Java)
  application.properties: |
    spring.application.name=myapp
    server.port=8080

  # TOML
  config.toml: |
    [server]
    port = 8080
    host = "0.0.0.0"

    [database]
    url = "postgresql://localhost/myapp"

  # INI
  settings.ini: |
    [database]
    host=localhost
    port=5432

    [cache]
    provider=redis
    ttl=3600

  # XML
  web.xml: |
    <?xml version="1.0" encoding="UTF-8"?>
    <web-app>
      <context-param>
        <param-name>contextPath</param-name>
        <param-value>/api</param-value>
      </context-param>
    </web-app>

  # Dotenv
  .env: |
    NODE_ENV=production
    PORT=3000
    DATABASE_URL=postgresql://localhost/myapp
```

### Configuration avec templates

Utiliser des templates pour générer la configuration :

```yaml
# ConfigMap avec template
apiVersion: v1
kind: ConfigMap
metadata:
  name: template-config
data:
  nginx-template.conf: |
    server {
        listen ${PORT};
        server_name ${SERVER_NAME};

        location / {
            proxy_pass http://${BACKEND_HOST}:${BACKEND_PORT};
            proxy_set_header Host $host;
            proxy_set_header X-Real-IP $remote_addr;
        }

        location /health {
            return 200 "healthy\n";
        }
    }
---
# Deployment avec substitution
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-with-template
spec:
  template:
    spec:
      initContainers:
      # Générer la config depuis le template
      - name: config-generator
        image: busybox
        env:
        - name: PORT
          value: "80"
        - name: SERVER_NAME
          value: "example.com"
        - name: BACKEND_HOST
          value: "backend-service"
        - name: BACKEND_PORT
          value: "8080"
        command: ['sh', '-c']
        args:
        - |
          # Substituer les variables
          cat /templates/nginx-template.conf | envsubst > /config/nginx.conf
          echo "Generated configuration:"
          cat /config/nginx.conf
        volumeMounts:
        - name: template
          mountPath: /templates
        - name: config
          mountPath: /config

      containers:
      - name: nginx
        image: nginx:alpine
        volumeMounts:
        - name: config
          mountPath: /etc/nginx/conf.d

      volumes:
      - name: template
        configMap:
          name: template-config
      - name: config
        emptyDir: {}
```

### Configuration avec secrets partiels

Combiner ConfigMaps et Secrets de manière sécurisée :

```yaml
# ConfigMap avec placeholders
apiVersion: v1
kind: ConfigMap
metadata:
  name: database-config-template
data:
  database.conf: |
    [database]
    host = db.production.example.com
    port = 5432
    database = myapp_prod
    # Les credentials viendront des secrets
    username = ${DB_USERNAME}
    password = ${DB_PASSWORD}

    [connection_pool]
    min_connections = 10
    max_connections = 100
    timeout = 30
---
# Secret avec les données sensibles
apiVersion: v1
kind: Secret
metadata:
  name: database-credentials
type: Opaque
stringData:
  username: produser
  password: SuperSecretPassword123!
---
# Deployment combinant les deux
apiVersion: apps/v1
kind: Deployment
metadata:
  name: app-with-mixed-config
spec:
  template:
    spec:
      containers:
      - name: app
        image: myapp:v1.0.0
        env:
        # Depuis le Secret
        - name: DB_USERNAME
          valueFrom:
            secretKeyRef:
              name: database-credentials
              key: username
        - name: DB_PASSWORD
          valueFrom:
            secretKeyRef:
              name: database-credentials
              key: password

        # Depuis le ConfigMap
        - name: DB_HOST
          valueFrom:
            configMapKeyRef:
              name: database-config-template
              key: database.host

        # Construction de l'URL complète
        - name: DATABASE_URL
          value: "postgresql://$(DB_USERNAME):$(DB_PASSWORD)@$(DB_HOST):5432/myapp_prod"
```

## Gestion du cycle de vie

### Mise à jour de ConfigMaps

```bash
# Voir le ConfigMap actuel
kubectl get configmap app-config -n my-app -o yaml

# Éditer directement
kubectl edit configmap app-config -n my-app

# Remplacer depuis un fichier
kubectl replace -f updated-configmap.yaml

# Patcher une valeur spécifique
kubectl patch configmap app-config -n my-app \
  --type merge -p '{"data":{"log.level":"debug"}}'

# Ajouter une nouvelle clé
kubectl patch configmap app-config -n my-app \
  --type merge -p '{"data":{"new.property":"value"}}'
```

### Propagation des changements

Les changements de ConfigMap ne sont pas appliqués immédiatement aux Pods :

```yaml
# Option 1 : Redémarrage manuel
kubectl rollout restart deployment/myapp -n my-app

# Option 2 : Avec un hash dans les annotations
apiVersion: apps/v1
kind: Deployment
metadata:
  name: auto-restart-app
spec:
  template:
    metadata:
      annotations:
        configmap.reloader.stakater.com/reload: "app-config"
        # Ou utiliser un hash manuel
        config-hash: "{{ .Values.configHash }}"  # Avec Helm
    spec:
      containers:
      - name: app
        image: myapp:v1.0.0
```

### Versionning des ConfigMaps

```yaml
# Stratégie 1 : Noms versionnés
---
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config-v1
data:
  version: "1.0"
  feature: "old"
---
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config-v2
data:
  version: "2.0"
  feature: "new"
---
# Deployment référençant une version spécifique
apiVersion: apps/v1
kind: Deployment
metadata:
  name: versioned-app
spec:
  template:
    spec:
      containers:
      - name: app
        envFrom:
        - configMapRef:
            name: app-config-v2  # Version spécifique
```

## Debugging et troubleshooting

### Vérifier les ConfigMaps

```bash
# Lister les ConfigMaps
kubectl get configmaps -n my-app

# Voir le contenu
kubectl describe configmap app-config -n my-app

# Exporter en YAML
kubectl get configmap app-config -n my-app -o yaml

# Voir une clé spécifique
kubectl get configmap app-config -n my-app -o jsonpath='{.data.database\.host}'

# Décoder si encodé
kubectl get configmap app-config -n my-app -o jsonpath='{.data.config}' | base64 -d
```

### Vérifier l'injection dans les Pods

```bash
# Variables d'environnement dans un Pod
kubectl exec -n my-app deployment/myapp -- env | grep -E "^DB_|^APP_"

# Fichiers montés
kubectl exec -n my-app deployment/myapp -- ls -la /etc/config/
kubectl exec -n my-app deployment/myapp -- cat /etc/config/application.properties

# Décrire le Pod pour voir les montages
kubectl describe pod <pod-name> -n my-app | grep -A 10 "Mounts:"
```

### Problèmes courants

#### ConfigMap introuvable

```bash
# Erreur : configmap "app-config" not found
# Vérifier le namespace
kubectl get configmap -A | grep app-config

# Créer si manquant
kubectl create configmap app-config --from-literal=key=value -n my-app
```

#### Variables non injectées

```yaml
# Vérifier l'orthographe des clés
env:
- name: DATABASE_HOST
  valueFrom:
    configMapKeyRef:
      name: app-config
      key: database.host  # La clé doit exister exactement
      optional: false     # Échouera si la clé n'existe pas
```

#### Montage de fichiers vides

```yaml
# Problème : fichier monté vide ou manquant
volumeMounts:
- name: config
  mountPath: /etc/config
  subPath: application.properties  # Vérifier que la clé existe

volumes:
- name: config
  configMap:
    name: app-config
    items:
    - key: application.properties  # Doit correspondre exactement
      path: application.properties
```

## Bonnes pratiques

### 1. Organisation et nommage

```yaml
# Conventions de nommage
metadata:
  name: <app>-<type>-<env>
  # Exemples:
  # webapp-config-prod
  # api-settings-dev
  # frontend-features-staging

  labels:
    app: webapp
    component: configuration
    environment: production
    version: v1.2.3
```

### 2. Séparation des responsabilités

```yaml
# ConfigMap pour configuration non-sensible
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config
data:
  server.port: "8080"
  cache.ttl: "3600"
  feature.flags: "new-ui,dark-mode"

---
# Secret pour données sensibles
apiVersion: v1
kind: Secret
metadata:
  name: app-secrets
stringData:
  database.password: "SecretPassword123"
  api.key: "sk_live_abc123"
  jwt.secret: "super-secret-jwt-key"
```

### 3. Documentation

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: documented-config
  annotations:
    description: "Configuration principale de l'application"
    documentation: "https://wiki.internal/app-config"
    last-updated: "2024-01-15"
    owner: "platform-team"
data:
  # Format: TYPE|DEFAULT|DESCRIPTION
  # STRING|localhost|Hôte de la base de données
  database.host: "db.production.local"

  # NUMBER|5432|Port de la base de données
  database.port: "5432"

  # BOOLEAN|false|Active le mode debug
  debug.enabled: "false"

  # LIST|[]|Features activées
  features: "api,ui,analytics"
```

### 4. Validation et tests

```bash
# Script de validation
#!/bin/bash
# validate-config.sh

CONFIG_FILE="configmap.yaml"

# Extraire et valider chaque propriété
DATABASE_HOST=$(kubectl get configmap app-config -n my-app -o jsonpath='{.data.database\.host}')
if [ -z "$DATABASE_HOST" ]; then
  echo "ERROR: database.host is not defined"
  exit 1
fi

# Valider le format
DATABASE_PORT=$(kubectl get configmap app-config -n my-app -o jsonpath='{.data.database\.port}')
if ! [[ "$DATABASE_PORT" =~ ^[0-9]+$ ]]; then
  echo "ERROR: database.port must be a number"
  exit 1
fi

echo "Configuration validation passed"
```

### 5. Taille et limites

**Limites des ConfigMaps :**
- Taille maximale : **1 MB** (1048576 bytes)
- Nombre de ConfigMaps : Illimité (dans les limites du quota)
- Mise à jour : Propagation en ~1 minute

```yaml
# Pour les grandes configurations, diviser en plusieurs ConfigMaps
---
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config-core
data:
  # Configuration essentielle < 1MB
  core.settings: |
    ...
---
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config-features
data:
  # Configuration des features < 1MB
  features.json: |
    ...
---
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config-ui
data:
  # Configuration UI < 1MB
  ui.settings: |
    ...
```

## Exemples concrets par type d'application

### Application Node.js

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: nodejs-config
data:
  # Variables d'environnement Node.js standard
  NODE_ENV: "production"
  PORT: "3000"

  # Configuration personnalisée
  API_URL: "https://api.example.com"
  REDIS_URL: "redis://redis-service:6379"
  SESSION_SECRET: "changeme-in-production"

  # Configuration PM2
  ecosystem.config.js: |
    module.exports = {
      apps: [{
        name: 'api',
        script: './dist/index.js',
        instances: 'max',
        exec_mode: 'cluster',
        env: {
          NODE_ENV: 'production'
        },
        error_file: '/var/log/pm2/error.log',
        out_file: '/var/log/pm2/out.log',
        log_date_format: 'YYYY-MM-DD HH:mm:ss Z'
      }]
    }
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nodejs-app
spec:
  template:
    spec:
      containers:
      - name: app
        image: node:16-alpine
        command: ["node"]
        args: ["server.js"]
        envFrom:
        - configMapRef:
            name: nodejs-config
        env:
        # Override ou ajouter des variables spécifiques
        - name: DATABASE_URL
          valueFrom:
            secretKeyRef:
              name: database-secret
              key: url
        volumeMounts:
        - name: pm2-config
          mountPath: /app/ecosystem.config.js
          subPath: ecosystem.config.js
      volumes:
      - name: pm2-config
        configMap:
          name: nodejs-config
```

### Application Python/Django

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: django-config
data:
  # Django settings
  DJANGO_SETTINGS_MODULE: "myproject.settings.production"
  DJANGO_DEBUG: "False"
  DJANGO_ALLOWED_HOSTS: "*.example.com,example.com"

  # Database (URL sera dans Secret)
  DATABASE_ENGINE: "django.db.backends.postgresql"
  DATABASE_NAME: "django_db"
  DATABASE_HOST: "postgres-service"
  DATABASE_PORT: "5432"

  # Cache
  CACHE_BACKEND: "django.core.cache.backends.redis.RedisCache"
  CACHE_LOCATION: "redis://redis-service:6379/1"

  # Static files
  STATIC_URL: "/static/"
  STATIC_ROOT: "/app/staticfiles"
  MEDIA_URL: "/media/"
  MEDIA_ROOT: "/app/media"

  # Gunicorn config
  gunicorn.conf.py: |
    import multiprocessing

    bind = "0.0.0.0:8000"
    workers = multiprocessing.cpu_count() * 2 + 1
    worker_class = "sync"
    worker_connections = 1000
    max_requests = 1000
    max_requests_jitter = 50
    timeout = 30
    keepalive = 2

    accesslog = "-"
    errorlog = "-"
    loglevel = "info"

    preload_app = True

  # Celery config
  celery.py: |
    import os
    from celery import Celery

    os.environ.setdefault('DJANGO_SETTINGS_MODULE', 'myproject.settings')

    app = Celery('myproject')
    app.config_from_object('django.conf:settings', namespace='CELERY')
    app.autodiscover_tasks()

    # Celery Configuration
    app.conf.update(
        broker_url='redis://redis-service:6379/0',
        result_backend='redis://redis-service:6379/0',
        task_serializer='json',
        accept_content=['json'],
        result_serializer='json',
        timezone='UTC',
        enable_utc=True,
    )
```

### Application Java/Spring Boot

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: spring-config
data:
  # Spring profiles
  SPRING_PROFILES_ACTIVE: "production,kubernetes"

  # JVM options
  JAVA_OPTS: "-Xms512m -Xmx2048m -XX:+UseG1GC"

  # Application properties
  application.yaml: |
    spring:
      application:
        name: my-spring-app

      profiles:
        active: production

      datasource:
        url: jdbc:postgresql://postgres-service:5432/springdb
        driver-class-name: org.postgresql.Driver
        hikari:
          minimum-idle: 10
          maximum-pool-size: 50
          idle-timeout: 600000
          max-lifetime: 1800000

      jpa:
        hibernate:
          ddl-auto: validate
        properties:
          hibernate:
            dialect: org.hibernate.dialect.PostgreSQLDialect
            format_sql: false
            show_sql: false

      cache:
        type: redis
        redis:
          time-to-live: 3600000
          cache-null-values: false

      redis:
        host: redis-service
        port: 6379
        timeout: 2000ms
        lettuce:
          pool:
            max-active: 8
            max-idle: 8
            min-idle: 0

    server:
      port: 8080
      servlet:
        context-path: /api
      compression:
        enabled: true
        mime-types: application/json,application/xml,text/html,text/xml,text/plain

    management:
      endpoints:
        web:
          exposure:
            include: health,info,metrics,prometheus
      metrics:
        export:
          prometheus:
            enabled: true

    logging:
      level:
        root: WARN
        com.mycompany: INFO
      pattern:
        console: "%d{yyyy-MM-dd HH:mm:ss} - %msg%n"

  # Logback configuration
  logback-spring.xml: |
    <?xml version="1.0" encoding="UTF-8"?>
    <configuration>
        <include resource="org/springframework/boot/logging/logback/defaults.xml"/>

        <appender name="CONSOLE" class="ch.qos.logback.core.ConsoleAppender">
            <encoder>
                <pattern>${CONSOLE_LOG_PATTERN}</pattern>
                <charset>utf8</charset>
            </encoder>
        </appender>

        <root level="INFO">
            <appender-ref ref="CONSOLE"/>
        </root>

        <logger name="com.mycompany" level="DEBUG"/>
    </configuration>
```

### Application Go

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: go-config
data:
  # Variables d'environnement
  APP_ENV: "production"
  SERVER_PORT: "8080"
  SERVER_HOST: "0.0.0.0"

  LOG_LEVEL: "info"
  LOG_FORMAT: "json"

  # Timeouts
  READ_TIMEOUT: "30s"
  WRITE_TIMEOUT: "30s"
  IDLE_TIMEOUT: "120s"

  # Configuration YAML
  config.yaml: |
    app:
      name: "go-microservice"
      version: "1.0.0"
      environment: "production"

    server:
      host: "0.0.0.0"
      port: 8080
      read_timeout: 30s
      write_timeout: 30s
      idle_timeout: 120s
      max_header_bytes: 1048576

    database:
      driver: "postgres"
      host: "postgres-service"
      port: 5432
      name: "godb"
      max_open_connections: 100
      max_idle_connections: 10
      connection_max_lifetime: 1h

    redis:
      addr: "redis-service:6379"
      password: ""
      db: 0
      pool_size: 10
      min_idle_conns: 5
      max_retries: 3
      read_timeout: 3s
      write_timeout: 3s

    metrics:
      enabled: true
      path: "/metrics"
      port: 9090

    tracing:
      enabled: true
      service_name: "go-microservice"
      jaeger_endpoint: "http://jaeger-collector:14268/api/traces"
      sampling_rate: 0.1

  # Configuration TOML alternative
  config.toml: |
    [app]
    name = "go-microservice"
    version = "1.0.0"

    [server]
    host = "0.0.0.0"
    port = 8080

    [database]
    driver = "postgres"
    dsn = "host=postgres-service port=5432 dbname=godb sslmode=disable"

    [cache]
    provider = "redis"
    addr = "redis-service:6379"
```

## Intégration avec d'autres outils

### Helm Values et ConfigMaps

```yaml
# values.yaml
config:
  database:
    host: postgres-service
    port: 5432
    name: mydb
  cache:
    provider: redis
    ttl: 3600
  features:
    - api
    - ui
    - analytics

# templates/configmap.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: {{ .Release.Name }}-config
data:
  database.host: {{ .Values.config.database.host }}
  database.port: {{ .Values.config.database.port | quote }}
  database.name: {{ .Values.config.database.name }}
  cache.provider: {{ .Values.config.cache.provider }}
  cache.ttl: {{ .Values.config.cache.ttl | quote }}
  features: {{ .Values.config.features | join "," }}
```

### Kustomize et ConfigMaps

```yaml
# kustomization.yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

configMapGenerator:
- name: app-config
  literals:
  - APP_NAME=myapp
  - ENVIRONMENT=production
  files:
  - application.properties
  - configs/database.yaml
  options:
    labels:
      app: myapp
    annotations:
      generated-by: kustomize

- name: app-config-env
  envs:
  - production.env

# Remplacement de variables
replacements:
- source:
    kind: ConfigMap
    name: app-config
    fieldPath: data.database\.host
  targets:
  - select:
      kind: Deployment
      name: app
    fieldPaths:
    - spec.template.spec.containers.[name=app].env.[name=DB_HOST].value
```

### ArgoCD et ConfigMaps

```yaml
# Application ArgoCD avec ConfigMap
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: app-with-config
spec:
  source:
    repoURL: https://github.com/myorg/myapp
    path: k8s
    targetRevision: main
    plugin:
      name: argocd-vault-plugin
      env:
      - name: AVP_TYPE
        value: gcpsecretmanager
  destination:
    server: https://kubernetes.default.svc
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
    syncOptions:
    - RespectIgnoreDifferences=true
  # Ignorer les changements automatiques des ConfigMaps
  ignoreDifferences:
  - group: ""
    kind: ConfigMap
    jsonPointers:
    - /data/last-updated
```

## Monitoring et alerting

### Surveiller les ConfigMaps

```yaml
# PrometheusRule pour alertes
apiVersion: monitoring.coreos.com/v1
kind: PrometheusRule
metadata:
  name: configmap-alerts
spec:
  groups:
  - name: configmap.rules
    interval: 30s
    rules:
    - alert: ConfigMapNotFound
      expr: |
        kube_configmap_info{configmap="app-config"} == 0
      for: 5m
      labels:
        severity: critical
      annotations:
        summary: "ConfigMap {{ $labels.configmap }} not found"
        description: "ConfigMap {{ $labels.configmap }} in namespace {{ $labels.namespace }} is missing"

    - alert: ConfigMapTooLarge
      expr: |
        kube_configmap_size_bytes > 900000
      for: 5m
      labels:
        severity: warning
      annotations:
        summary: "ConfigMap approaching size limit"
        description: "ConfigMap {{ $labels.configmap }} is {{ $value | humanizeBytes }}, approaching 1MB limit"
```

## Migration depuis d'autres systèmes

### Depuis des fichiers .env

```bash
# Script de migration
#!/bin/bash

# Convertir .env en ConfigMap
ENV_FILE="production.env"
CONFIGMAP_NAME="app-config"
NAMESPACE="production"

# Créer le ConfigMap depuis le fichier .env
kubectl create configmap $CONFIGMAP_NAME \
  --from-env-file=$ENV_FILE \
  --namespace=$NAMESPACE \
  --dry-run=client -o yaml > configmap.yaml

# Ajouter des métadonnées
cat >> configmap.yaml <<EOF
  labels:
    app: myapp
    environment: production
  annotations:
    migrated-from: "$ENV_FILE"
    migration-date: "$(date -I)"
EOF

# Appliquer
kubectl apply -f configmap.yaml
```

### Depuis Consul/etcd

```go
// migrate-from-consul.go
package main

import (
    "fmt"
    consulapi "github.com/hashicorp/consul/api"
    "k8s.io/client-go/kubernetes"
    v1 "k8s.io/api/core/v1"
)

func migrateFromConsul() {
    // Connexion Consul
    consul, _ := consulapi.NewClient(consulapi.DefaultConfig())
    kv := consul.KV()

    // Récupérer les configs
    pairs, _, _ := kv.List("config/", nil)

    // Créer ConfigMap
    configMap := &v1.ConfigMap{
        ObjectMeta: metav1.ObjectMeta{
            Name: "app-config",
            Namespace: "default",
        },
        Data: make(map[string]string),
    }

    for _, pair := range pairs {
        configMap.Data[pair.Key] = string(pair.Value)
    }

    // Créer dans Kubernetes
    clientset.CoreV1().ConfigMaps("default").Create(configMap)
}
```

## Sécurité et conformité

### Audit des ConfigMaps

```yaml
# Audit Policy
apiVersion: audit.k8s.io/v1
kind: Policy
rules:
- level: RequestResponse
  omitStages:
  - RequestReceived
  resources:
  - group: ""
    resources: ["configmaps"]
  namespaces: ["production", "staging"]
```

### Encryption at rest

```yaml
# EncryptionConfiguration
apiVersion: apiserver.config.k8s.io/v1
kind: EncryptionConfiguration
resources:
- resources:
  - configmaps
  providers:
  - aescbc:
      keys:
      - name: key1
        secret: <base64-encoded-secret>
  - identity: {}
```

## Conclusion

Les variables d'environnement et ConfigMaps sont des outils essentiels pour créer des applications cloud-native flexibles et maintenables.

**Points clés à retenir :**

1. **Externalisation** : Toujours séparer configuration et code
   - Facilite les déploiements multi-environnements
   - Permet les changements sans rebuild

2. **ConfigMaps** : Pour la configuration non-sensible
   - Multiples formats supportés
   - Montage comme fichiers ou variables
   - Limite de 1MB par ConfigMap

3. **Patterns** : Utiliser les bonnes pratiques
   - Configuration hiérarchique
   - Versionning des ConfigMaps
   - Validation avant utilisation

4. **Intégration** : ConfigMaps fonctionnent avec tout
   - Helm pour le templating
   - Kustomize pour les overlays
   - ArgoCD pour GitOps

5. **Monitoring** : Surveiller vos configurations
   - Alertes sur ConfigMaps manquants
   - Audit des changements
   - Validation continue

Dans la prochaine section (7.6), nous verrons comment gérer les données sensibles de manière sécurisée avec les Secrets Kubernetes, complétant ainsi notre stratégie de gestion de configuration.

⏭️
