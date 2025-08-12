🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 7.1 - Anatomie d'un déploiement Kubernetes

## Introduction

Un déploiement Kubernetes est comme un orchestre symphonique : plusieurs composants travaillent ensemble harmonieusement pour faire fonctionner votre application. Comprendre comment ces pièces s'assemblent est essentiel pour maîtriser Kubernetes. Dans cette section, nous allons disséquer un déploiement typique pour comprendre chaque composant et son rôle.

## Vue d'ensemble de l'architecture

Quand vous déployez une application sur Kubernetes, vous ne créez pas simplement un conteneur qui tourne. Vous mettez en place tout un écosystème de ressources qui collaborent :

```
Internet → Ingress → Service → Deployment → ReplicaSet → Pod(s) → Container(s)
                        ↓
                  ConfigMap/Secret
                        ↓
                 PersistentVolume
```

Chaque flèche représente une relation ou un flux de communication. Voyons maintenant chaque composant en détail.

## Les composants fondamentaux

### 1. Le Pod : l'unité de base

Le **Pod** est l'unité atomique de Kubernetes, la plus petite chose que vous pouvez déployer. Pensez à un Pod comme une "machine virtuelle logique" qui contient :

- **Un ou plusieurs conteneurs** (généralement un seul)
- **Un réseau partagé** (même adresse IP pour tous les conteneurs du Pod)
- **Un stockage partagé** (volumes accessibles par tous les conteneurs)
- **Une identité unique** dans le cluster

**Caractéristiques importantes d'un Pod :**
- Il est éphémère par nature (peut disparaître à tout moment)
- Il a une seule adresse IP
- Les conteneurs à l'intérieur peuvent communiquer via `localhost`
- Il vit sur un seul nœud (ne peut pas être réparti sur plusieurs machines)

**Exemple de définition d'un Pod simple :**
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: mon-application
  labels:
    app: webapp
spec:
  containers:
  - name: app-container
    image: nginx:latest
    ports:
    - containerPort: 80
```

**Pourquoi on ne déploie jamais directement des Pods :**
En pratique, vous ne créez presque jamais de Pods directement. Pourquoi ? Parce qu'ils sont mortels. Si un Pod crash ou si le nœud sur lequel il tourne a un problème, le Pod disparaît et ne revient pas. C'est là qu'intervient le Deployment.

### 2. Le Deployment : le gestionnaire intelligent

Le **Deployment** est votre chef d'orchestre. Il s'assure que votre application tourne toujours avec le bon nombre d'instances (replicas) et gère les mises à jour.

**Responsabilités du Deployment :**
- Maintenir le nombre désiré de Pods en fonctionnement
- Remplacer les Pods défaillants automatiquement
- Gérer les montées de version (rolling updates)
- Permettre les rollbacks en cas de problème
- Distribuer les Pods sur différents nœuds pour la résilience

**Structure d'un Deployment :**
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: webapp-deployment
  labels:
    app: webapp
spec:
  replicas: 3  # Nombre de Pods désirés
  selector:
    matchLabels:
      app: webapp  # Sélectionne les Pods avec ce label
  template:
    metadata:
      labels:
        app: webapp  # Labels appliqués aux Pods créés
    spec:
      containers:
      - name: webapp
        image: myapp:v1.0
        ports:
        - containerPort: 8080
        resources:
          requests:
            memory: "128Mi"
            cpu: "250m"
          limits:
            memory: "256Mi"
            cpu: "500m"
```

**Le ReplicaSet : le bras droit du Deployment**

Quand vous créez un Deployment, il crée automatiquement un **ReplicaSet**. Le ReplicaSet est responsable de maintenir le nombre exact de Pods spécifié. Si vous demandez 3 replicas et qu'un Pod meurt, le ReplicaSet en crée immédiatement un nouveau.

Vous n'interagissez généralement pas directement avec les ReplicaSets, mais il est important de savoir qu'ils existent pour comprendre la hiérarchie :
- Deployment → gère → ReplicaSet → gère → Pods

### 3. Le Service : l'annuaire téléphonique

Les Pods sont éphémères et leurs IPs changent constamment. Comment faire pour accéder à votre application de manière stable ? C'est le rôle du **Service**.

**Le Service offre :**
- Une adresse IP stable (ClusterIP) qui ne change jamais
- Un nom DNS interne (ex: `webapp-service.default.svc.cluster.local`)
- Du load balancing automatique entre les Pods
- La découverte de service (service discovery)

**Types de Services :**

**ClusterIP (par défaut)** : Accessible uniquement depuis l'intérieur du cluster
```yaml
apiVersion: v1
kind: Service
metadata:
  name: webapp-service
spec:
  type: ClusterIP
  selector:
    app: webapp  # Sélectionne les Pods avec ce label
  ports:
  - port: 80        # Port du Service
    targetPort: 8080 # Port du conteneur
    protocol: TCP
```

**NodePort** : Expose le service sur un port de chaque nœud (30000-32767)
```yaml
spec:
  type: NodePort
  ports:
  - port: 80
    targetPort: 8080
    nodePort: 30080  # Accessible via <IP-du-nœud>:30080
```

**LoadBalancer** : Demande un load balancer externe (cloud provider ou MetalLB)
```yaml
spec:
  type: LoadBalancer
  ports:
  - port: 80
    targetPort: 8080
```

### 4. L'Ingress : la porte d'entrée

L'**Ingress** est votre réceptionniste. Il reçoit le trafic HTTP/HTTPS externe et le dirige vers les bons Services basé sur des règles (nom de domaine, chemin URL).

**Fonctionnalités de l'Ingress :**
- Routage basé sur le nom d'hôte (`app1.example.com` → Service A)
- Routage basé sur le chemin (`/api` → Service API, `/web` → Service Web)
- Terminaison SSL/TLS (HTTPS)
- Load balancing
- Hôtes virtuels

**Exemple d'Ingress :**
```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: webapp-ingress
  annotations:
    cert-manager.io/cluster-issuer: "letsencrypt"
    nginx.ingress.kubernetes.io/rewrite-target: /
spec:
  tls:
  - hosts:
    - webapp.monlab.local
    secretName: webapp-tls
  rules:
  - host: webapp.monlab.local
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

### 5. ConfigMap : la configuration externalisée

Un **ConfigMap** stocke des données de configuration non sensibles que vos applications peuvent utiliser.

**Utilités du ConfigMap :**
- Fichiers de configuration
- Variables d'environnement
- Arguments de ligne de commande
- URLs d'API, paramètres d'application

**Création d'un ConfigMap :**
```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: webapp-config
data:
  # Propriétés simples
  database_url: "postgres://db.monlab.local:5432/myapp"
  log_level: "info"

  # Fichier de configuration complet
  app.properties: |
    server.port=8080
    app.name=MonApplication
    cache.enabled=true
```

**Utilisation dans un Pod :**
```yaml
spec:
  containers:
  - name: webapp
    image: myapp:v1.0
    env:
    - name: DATABASE_URL
      valueFrom:
        configMapKeyRef:
          name: webapp-config
          key: database_url
    volumeMounts:
    - name: config-volume
      mountPath: /etc/config
  volumes:
  - name: config-volume
    configMap:
      name: webapp-config
```

### 6. Secret : les données sensibles

Un **Secret** est similaire à un ConfigMap mais conçu pour stocker des données sensibles (mots de passe, tokens, clés SSH).

**Caractéristiques des Secrets :**
- Données encodées en base64 (pas chiffrées !)
- Peuvent être montés en mémoire (tmpfs) dans les Pods
- Accès contrôlé par RBAC
- Ne sont pas loggés par défaut

**Création d'un Secret :**
```yaml
apiVersion: v1
kind: Secret
metadata:
  name: webapp-secret
type: Opaque
data:
  # Les valeurs doivent être en base64
  username: YWRtaW4=  # "admin" en base64
  password: cEBzc3cwcmQ=  # "p@ssw0rd" en base64
```

**Ou plus simplement avec stringData :**
```yaml
apiVersion: v1
kind: Secret
metadata:
  name: webapp-secret
type: Opaque
stringData:
  username: admin
  password: p@ssw0rd
```

### 7. PersistentVolume et PersistentVolumeClaim

Pour les applications qui ont besoin de stocker des données de manière permanente, Kubernetes offre les volumes persistants.

**PersistentVolume (PV)** : Représente un espace de stockage physique
**PersistentVolumeClaim (PVC)** : Une demande de stockage par un Pod

**Workflow du stockage persistant :**
1. L'admin crée un PV (ou il est créé dynamiquement)
2. L'utilisateur crée un PVC pour demander du stockage
3. Kubernetes lie automatiquement le PVC à un PV compatible
4. Le Pod monte le PVC comme un volume

**Exemple de PVC :**
```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: webapp-storage
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 5Gi
  storageClassName: microk8s-hostpath  # Classe de stockage MicroK8s
```

**Utilisation dans un Pod :**
```yaml
spec:
  containers:
  - name: webapp
    image: myapp:v1.0
    volumeMounts:
    - name: data-volume
      mountPath: /var/data
  volumes:
  - name: data-volume
    persistentVolumeClaim:
      claimName: webapp-storage
```

## Cycle de vie d'un déploiement complet

Voyons maintenant comment tous ces composants s'assemblent dans un déploiement réel :

### Étape 1 : Création du Deployment
```bash
kubectl apply -f deployment.yaml
```
- Kubernetes crée le Deployment
- Le Deployment crée un ReplicaSet
- Le ReplicaSet crée les Pods selon le nombre de replicas

### Étape 2 : Les Pods démarrent
- Kubernetes trouve des nœuds avec suffisamment de ressources
- Les images Docker sont téléchargées (pull)
- Les conteneurs démarrent
- Les health checks (probes) vérifient que l'application est prête

### Étape 3 : Exposition via Service
```bash
kubectl apply -f service.yaml
```
- Le Service trouve les Pods via les labels
- Une IP stable (ClusterIP) est assignée
- Le kube-proxy configure les règles de routage

### Étape 4 : Accès externe via Ingress
```bash
kubectl apply -f ingress.yaml
```
- L'Ingress Controller détecte la nouvelle règle
- Les certificats SSL sont configurés (si spécifiés)
- Le routage externe est établi

### Étape 5 : Configuration et secrets
```bash
kubectl apply -f configmap.yaml
kubectl apply -f secret.yaml
```
- Les ConfigMaps et Secrets sont créés
- Les Pods sont redémarrés si nécessaire pour prendre en compte les changements

## Les labels et selectors : le système nerveux

Les **labels** sont des paires clé-valeur attachées aux objets Kubernetes. Ils sont cruciaux car ils permettent aux différents composants de se trouver.

**Exemple de flux avec labels :**
```yaml
# Deployment
metadata:
  labels:
    app: webapp
    version: v1
    environment: lab

# Service (trouve les Pods via selector)
spec:
  selector:
    app: webapp

# Ingress (trouve le Service par son nom)
backend:
  service:
    name: webapp-service
```

**Bonnes pratiques pour les labels :**
- `app` : nom de l'application
- `version` : version de l'application
- `environment` : dev, test, staging, prod
- `component` : frontend, backend, database
- `managed-by` : outil de déploiement (kubectl, helm, etc.)

## Health checks : la surveillance de santé

Kubernetes surveille constamment la santé de vos Pods via des **probes** :

### Liveness Probe
Vérifie si le conteneur est vivant. S'il échoue, Kubernetes redémarre le conteneur.

```yaml
livenessProbe:
  httpGet:
    path: /health
    port: 8080
  initialDelaySeconds: 30
  periodSeconds: 10
```

### Readiness Probe
Vérifie si le conteneur est prêt à recevoir du trafic. S'il échoue, le Pod est retiré du Service.

```yaml
readinessProbe:
  httpGet:
    path: /ready
    port: 8080
  initialDelaySeconds: 10
  periodSeconds: 5
```

### Startup Probe
Pour les applications avec un démarrage lent, évite que les autres probes échouent pendant le démarrage.

```yaml
startupProbe:
  httpGet:
    path: /started
    port: 8080
  failureThreshold: 30
  periodSeconds: 10
```

## Stratégies de déploiement

Le Deployment offre plusieurs stratégies pour mettre à jour vos applications :

### RollingUpdate (par défaut)
Remplace progressivement les anciens Pods par les nouveaux.

```yaml
spec:
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1        # Nombre max de Pods en plus
      maxUnavailable: 1  # Nombre max de Pods indisponibles
```

**Processus :**
1. Crée un nouveau Pod avec la nouvelle version
2. Attend qu'il soit Ready
3. Supprime un ancien Pod
4. Répète jusqu'à ce que tous soient mis à jour

### Recreate
Supprime tous les anciens Pods avant de créer les nouveaux (downtime).

```yaml
spec:
  strategy:
    type: Recreate
```

**Utilisation :** Pour les applications qui ne peuvent pas avoir plusieurs versions en même temps.

## Gestion des ressources

Définir les ressources est crucial pour la stabilité du cluster :

### Requests
Les ressources minimum garanties pour le Pod.

### Limits
Les ressources maximum que le Pod peut utiliser.

```yaml
resources:
  requests:
    memory: "128Mi"  # 128 mégaoctets
    cpu: "250m"      # 0.25 CPU
  limits:
    memory: "512Mi"
    cpu: "1000m"     # 1 CPU
```

**Unités de mesure :**
- CPU : en millicores (1000m = 1 CPU)
- Mémoire : en bytes (Ki, Mi, Gi)

## Namespaces : l'organisation logique

Les **Namespaces** permettent d'organiser et d'isoler les ressources :

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: mon-projet
```

**Namespaces par défaut :**
- `default` : namespace par défaut
- `kube-system` : composants système de Kubernetes
- `kube-public` : ressources publiques
- `kube-node-lease` : infos de santé des nœuds

**Utilisation :**
```bash
# Créer des ressources dans un namespace
kubectl apply -f deployment.yaml -n mon-projet

# Changer le namespace par défaut
kubectl config set-context --current --namespace=mon-projet
```

## Architecture complète d'un déploiement

Voici comment tous ces éléments s'assemblent dans une application réelle :

```
┌──────────────────────────────────────────────────────┐
│                      Cluster Kubernetes              │
│                                                      │
│  ┌────────────────────────────────────────────────┐  │
│  │                Namespace: mon-app              │  │
│  │                                                │  │
│  │  ┌─────────────┐  ┌─────────────┐              │  │
│  │  │ ConfigMap   │  │   Secret    │              │  │
│  │  └──────┬──────┘  └──────┬──────┘              │  │
│  │         │                 │                    │  │
│  │  ┌──────▼─────────────────▼──────┐             │  │
│  │  │         Deployment             │            │  │
│  │  │     replicas: 3                │            │  │
│  │  └──────────────┬─────────────────┘            │  │
│  │                 │                              │  │
│  │         ┌───────▼────────┐                     │  │
│  │         │   ReplicaSet   │                     │  │
│  │         └───────┬────────┘                     │  │
│  │                 │                              │  │
│  │     ┌───────────┼───────────┐                  │  │
│  │     ▼           ▼           ▼                  │  │
│  │  ┌──────┐   ┌──────┐   ┌──────┐                │  │
│  │  │ Pod1 │   │ Pod2 │   │ Pod3 │                │  │
│  │  └──┬───┘   └──┬───┘   └──┬───┘                │  │
│  │     │          │          │                    │  │
│  │     └──────────┼──────────┘                    │  │
│  │                ▼                               │  │
│  │         ┌─────────────┐                        │  │
│  │         │   Service   │◄────────┐              │  │
│  │         │ ClusterIP   │         │              │  │
│  │         └─────────────┘         │              │  │
│  │                                 │              │  │
│  │                          ┌──────┴──────┐       │  │
│  │                          │   Ingress   │       │  │
│  │                          └──────▲──────┘       │  │
│  └─────────────────────────────────┼──────────────┘  │
│                                    │                 │
└────────────────────────────────────┼─────────────────┘
                                     │
                                 Internet
```

## Points clés à retenir

1. **Hiérarchie des objets** : Deployment → ReplicaSet → Pod → Container

2. **Séparation des responsabilités** :
   - Deployment : gestion du cycle de vie
   - Service : réseau stable
   - Ingress : accès externe
   - ConfigMap/Secret : configuration
   - PV/PVC : stockage

3. **Les labels sont essentiels** : Ils permettent aux composants de se trouver

4. **Tout est déclaratif** : Vous décrivez l'état désiré, Kubernetes s'occupe du reste

5. **La résilience est intégrée** : Auto-healing, rolling updates, load balancing

6. **L'isolation par namespace** : Organisez vos applications logiquement

## Conclusion

Comprendre l'anatomie d'un déploiement Kubernetes est fondamental pour utiliser efficacement votre cluster MicroK8s. Chaque composant a un rôle spécifique, et ensemble ils créent un système robuste et scalable pour vos applications.

Dans la prochaine section (7.2), nous verrons comment écrire des manifestes YAML efficaces pour définir tous ces composants. Puis nous mettrons en pratique ces concepts en déployant une vraie application (7.3).

N'oubliez pas : dans votre lab personnel, vous avez la liberté d'expérimenter. Créez, cassez, recréez. C'est en manipulant ces objets que vous développerez une compréhension intuitive de Kubernetes.

⏭️
