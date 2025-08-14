🔝 Retour au [Sommaire](/SOMMAIRE.md)

# Annexe B - Templates de manifestes YAML

## Introduction aux Manifestes YAML

Les manifestes YAML sont le langage de configuration de Kubernetes. Ils décrivent l'état désiré de vos applications et ressources dans le cluster. Cette annexe fournit une collection de templates prêts à l'emploi, commentés et adaptables pour vos besoins.

### Qu'est-ce qu'un Manifeste YAML ?

Un manifeste YAML est un fichier texte qui décrit une ou plusieurs ressources Kubernetes. Il suit une structure précise avec quatre éléments essentiels :

- **apiVersion** : La version de l'API Kubernetes utilisée
- **kind** : Le type de ressource (Pod, Service, Deployment, etc.)
- **metadata** : Les métadonnées (nom, labels, annotations)
- **spec** : La spécification détaillée de la ressource

### Pourquoi Utiliser des Templates ?

Les templates vous permettent de :
- **Gagner du temps** en évitant de réécrire les mêmes structures
- **Éviter les erreurs** de syntaxe et de configuration
- **Apprendre** les bonnes pratiques par l'exemple
- **Standardiser** vos déploiements
- **Documenter** votre infrastructure as code

## Templates de Base

### 1. Pod Simple

Le Pod est l'unité de base dans Kubernetes. Voici un template minimal :

```yaml
# pod-simple.yaml
# Un Pod basique avec un seul conteneur
apiVersion: v1
kind: Pod
metadata:
  name: mon-pod-simple
  namespace: default  # Namespace où créer le pod
  labels:
    app: mon-app
    environment: dev
spec:
  containers:
  - name: mon-conteneur
    image: nginx:1.21  # Image Docker à utiliser
    ports:
    - containerPort: 80  # Port exposé par le conteneur
    resources:
      # Limites de ressources pour éviter la surconsommation
      requests:
        memory: "64Mi"
        cpu: "250m"
      limits:
        memory: "128Mi"
        cpu: "500m"
```

### 2. Deployment Complet

Un Deployment gère plusieurs répliques de votre application avec stratégie de mise à jour :

```yaml
# deployment-complet.yaml
# Deployment avec toutes les options essentielles
apiVersion: apps/v1
kind: Deployment
metadata:
  name: mon-app-deployment
  namespace: apps  # Créer ce namespace au préalable
  labels:
    app: mon-app
    version: v1.0.0
  annotations:
    description: "Application web principale"
spec:
  replicas: 3  # Nombre de pods à maintenir
  strategy:
    type: RollingUpdate  # Mise à jour progressive
    rollingUpdate:
      maxSurge: 1        # Max pods supplémentaires pendant update
      maxUnavailable: 1  # Max pods indisponibles pendant update
  selector:
    matchLabels:
      app: mon-app
  template:
    metadata:
      labels:
        app: mon-app
        version: v1.0.0
    spec:
      containers:
      - name: app-container
        image: mon-registry/mon-app:1.0.0
        imagePullPolicy: IfNotPresent  # Always, Never, IfNotPresent
        ports:
        - containerPort: 8080
          name: http
          protocol: TCP

        # Variables d'environnement
        env:
        - name: NODE_ENV
          value: "production"
        - name: PORT
          value: "8080"
        - name: DATABASE_URL
          valueFrom:
            secretKeyRef:
              name: app-secrets
              key: database-url

        # Sondes de santé
        livenessProbe:
          httpGet:
            path: /health
            port: 8080
          initialDelaySeconds: 30  # Attendre avant première vérification
          periodSeconds: 10        # Vérifier toutes les 10s
          timeoutSeconds: 5
          failureThreshold: 3       # Redémarrer après 3 échecs

        readinessProbe:
          httpGet:
            path: /ready
            port: 8080
          initialDelaySeconds: 10
          periodSeconds: 5
          timeoutSeconds: 3
          successThreshold: 1
          failureThreshold: 3

        # Ressources
        resources:
          requests:
            memory: "128Mi"
            cpu: "100m"
          limits:
            memory: "256Mi"
            cpu: "500m"

        # Montage de volumes
        volumeMounts:
        - name: config-volume
          mountPath: /app/config
        - name: data-volume
          mountPath: /app/data

      # Définition des volumes
      volumes:
      - name: config-volume
        configMap:
          name: app-config
      - name: data-volume
        persistentVolumeClaim:
          claimName: app-data-pvc
```

### 3. Service ClusterIP

Expose votre application à l'intérieur du cluster :

```yaml
# service-clusterip.yaml
# Service accessible uniquement depuis l'intérieur du cluster
apiVersion: v1
kind: Service
metadata:
  name: mon-app-service
  namespace: apps
  labels:
    app: mon-app
spec:
  type: ClusterIP  # Type par défaut, accessible en interne seulement
  selector:
    app: mon-app  # Sélectionne les pods avec ce label
  ports:
  - port: 80        # Port du service
    targetPort: 8080  # Port du conteneur
    protocol: TCP
    name: http
  sessionAffinity: None  # ou "ClientIP" pour sticky sessions
```

### 4. Service NodePort

Expose votre application sur un port de chaque node :

```yaml
# service-nodeport.yaml
# Service accessible depuis l'extérieur via port du node
apiVersion: v1
kind: Service
metadata:
  name: mon-app-nodeport
  namespace: apps
spec:
  type: NodePort
  selector:
    app: mon-app
  ports:
  - port: 80
    targetPort: 8080
    nodePort: 30080  # Port sur le node (30000-32767)
    protocol: TCP
    name: http
```

### 5. Service LoadBalancer

Pour exposition externe avec MetalLB :

```yaml
# service-loadbalancer.yaml
# Service avec LoadBalancer (nécessite MetalLB sur MicroK8s)
apiVersion: v1
kind: Service
metadata:
  name: mon-app-lb
  namespace: apps
  annotations:
    # Annotations spécifiques MetalLB (optionnel)
    metallb.universe.tf/address-pool: production-pool
spec:
  type: LoadBalancer
  selector:
    app: mon-app
  ports:
  - port: 80
    targetPort: 8080
    protocol: TCP
    name: http
  - port: 443
    targetPort: 8443
    protocol: TCP
    name: https
  # IP fixe optionnelle (doit être dans la plage MetalLB)
  # loadBalancerIP: 192.168.1.200
```

## Templates d'Ingress

### 6. Ingress NGINX Basic

Route le trafic HTTP/HTTPS vers vos services :

```yaml
# ingress-basic.yaml
# Ingress simple pour un seul service
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: mon-app-ingress
  namespace: apps
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
    # Redirection HTTP vers HTTPS
    nginx.ingress.kubernetes.io/ssl-redirect: "true"
spec:
  ingressClassName: nginx  # ou "public" selon votre configuration
  rules:
  - host: mon-app.example.com  # Votre domaine
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: mon-app-service
            port:
              number: 80
```

### 7. Ingress Multi-Path

Route différents chemins vers différents services :

```yaml
# ingress-multi-path.yaml
# Ingress avec multiple chemins et services
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: multi-app-ingress
  namespace: apps
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /$2
    nginx.ingress.kubernetes.io/use-regex: "true"
spec:
  ingressClassName: nginx
  rules:
  - host: apps.example.com
    http:
      paths:
      # Application principale sur /
      - path: /
        pathType: Prefix
        backend:
          service:
            name: main-app-service
            port:
              number: 80

      # API sur /api
      - path: /api(/|$)(.*)
        pathType: Prefix
        backend:
          service:
            name: api-service
            port:
              number: 3000

      # Interface admin sur /admin
      - path: /admin(/|$)(.*)
        pathType: Prefix
        backend:
          service:
            name: admin-service
            port:
              number: 8080
```

### 8. Ingress avec TLS/SSL

Configuration avec certificat SSL :

```yaml
# ingress-tls.yaml
# Ingress avec certificat TLS (HTTPS)
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: secure-app-ingress
  namespace: apps
  annotations:
    # Cert-Manager pour certificats Let's Encrypt automatiques
    cert-manager.io/cluster-issuer: "letsencrypt-prod"
    nginx.ingress.kubernetes.io/ssl-redirect: "true"
    nginx.ingress.kubernetes.io/force-ssl-redirect: "true"
spec:
  ingressClassName: nginx
  tls:
  - hosts:
    - secure-app.example.com
    secretName: secure-app-tls  # Secret contenant le certificat
  rules:
  - host: secure-app.example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: secure-app-service
            port:
              number: 80
```

## Templates de Stockage

### 9. PersistentVolumeClaim

Demande de stockage persistant :

```yaml
# pvc-standard.yaml
# PersistentVolumeClaim pour stockage persistant
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: app-data-pvc
  namespace: apps
  labels:
    app: mon-app
spec:
  accessModes:
    - ReadWriteOnce  # RWO: un seul pod en écriture
    # - ReadOnlyMany   # ROX: plusieurs pods en lecture seule
    # - ReadWriteMany  # RWX: plusieurs pods en écriture (rare)
  resources:
    requests:
      storage: 5Gi  # Taille du volume
  storageClassName: microk8s-hostpath  # Classe de stockage MicroK8s
  # volumeMode: Filesystem  # ou "Block" pour stockage bloc
```

### 10. StatefulSet avec Stockage

Pour applications avec état (bases de données, etc.) :

```yaml
# statefulset-with-storage.yaml
# StatefulSet pour applications avec état persistant
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: postgres-db
  namespace: apps
spec:
  serviceName: postgres-service
  replicas: 1
  selector:
    matchLabels:
      app: postgres
  template:
    metadata:
      labels:
        app: postgres
    spec:
      containers:
      - name: postgres
        image: postgres:14
        ports:
        - containerPort: 5432
          name: postgres
        env:
        - name: POSTGRES_DB
          value: "myappdb"
        - name: POSTGRES_USER
          value: "appuser"
        - name: POSTGRES_PASSWORD
          valueFrom:
            secretKeyRef:
              name: postgres-secret
              key: password
        - name: PGDATA
          value: /var/lib/postgresql/data/pgdata
        volumeMounts:
        - name: postgres-storage
          mountPath: /var/lib/postgresql/data
        resources:
          requests:
            memory: "256Mi"
            cpu: "250m"
          limits:
            memory: "512Mi"
            cpu: "500m"
  # Template de PVC pour chaque réplique
  volumeClaimTemplates:
  - metadata:
      name: postgres-storage
    spec:
      accessModes: ["ReadWriteOnce"]
      storageClassName: microk8s-hostpath
      resources:
        requests:
          storage: 10Gi
```

## Templates de Configuration

### 11. ConfigMap

Pour stocker la configuration non-sensible :

```yaml
# configmap-app.yaml
# ConfigMap pour configuration d'application
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config
  namespace: apps
data:
  # Valeurs simples
  app.name: "Mon Application"
  app.version: "1.0.0"
  log.level: "info"

  # Fichier de configuration complet
  application.yaml: |
    server:
      port: 8080
      host: 0.0.0.0
    database:
      host: postgres-service
      port: 5432
      name: myappdb
    cache:
      enabled: true
      ttl: 3600
    features:
      - name: feature1
        enabled: true
      - name: feature2
        enabled: false

  # Script d'initialisation
  init.sh: |
    #!/bin/bash
    echo "Initialisation de l'application..."
    # Créer les répertoires nécessaires
    mkdir -p /app/data /app/logs
    # Définir les permissions
    chmod 755 /app/data /app/logs
    echo "Initialisation terminée"
```

### 12. Secret

Pour stocker des données sensibles :

```yaml
# secret-app.yaml
# Secret pour données sensibles (mots de passe, tokens, etc.)
apiVersion: v1
kind: Secret
metadata:
  name: app-secrets
  namespace: apps
type: Opaque  # Type générique
data:
  # Les valeurs doivent être encodées en base64
  # echo -n "monmotdepasse" | base64
  database-password: bW9ubW90ZGVwYXNzZQ==
  api-key: YWJjZGVmZ2hpams=
  jwt-secret: c3VwZXJzZWNyZXQ=
---
# Alternative avec stringData (auto-encodé)
apiVersion: v1
kind: Secret
metadata:
  name: app-secrets-v2
  namespace: apps
type: Opaque
stringData:
  database-url: "postgresql://user:pass@postgres:5432/mydb"
  redis-url: "redis://redis-service:6379"
  smtp-password: "smtp-secret-password"
```

### 13. Secret TLS

Pour certificats SSL/TLS :

```yaml
# secret-tls.yaml
# Secret pour certificat TLS
apiVersion: v1
kind: Secret
metadata:
  name: app-tls-secret
  namespace: apps
type: kubernetes.io/tls
data:
  # Générer avec : cat cert.crt | base64 -w 0
  tls.crt: LS0tLS1CRUdJTi... # Certificat encodé base64
  tls.key: LS0tLS1CRUdJTi... # Clé privée encodée base64
```

## Templates de Jobs et CronJobs

### 14. Job One-Time

Pour tâches ponctuelles :

```yaml
# job-backup.yaml
# Job pour exécuter une tâche unique
apiVersion: batch/v1
kind: Job
metadata:
  name: database-backup
  namespace: apps
spec:
  completions: 1  # Nombre d'exécutions réussies souhaitées
  parallelism: 1  # Nombre de pods en parallèle
  backoffLimit: 3  # Nombre de tentatives en cas d'échec
  ttlSecondsAfterFinished: 3600  # Nettoyer après 1h
  template:
    metadata:
      labels:
        job: database-backup
    spec:
      restartPolicy: Never  # ou OnFailure
      containers:
      - name: backup
        image: postgres:14
        command:
        - /bin/bash
        - -c
        - |
          echo "Démarrage du backup..."
          PGPASSWORD=$POSTGRES_PASSWORD pg_dump \
            -h postgres-service \
            -U $POSTGRES_USER \
            -d $POSTGRES_DB \
            > /backup/db-$(date +%Y%m%d-%H%M%S).sql
          echo "Backup terminé"
        env:
        - name: POSTGRES_USER
          value: "appuser"
        - name: POSTGRES_DB
          value: "myappdb"
        - name: POSTGRES_PASSWORD
          valueFrom:
            secretKeyRef:
              name: postgres-secret
              key: password
        volumeMounts:
        - name: backup-volume
          mountPath: /backup
      volumes:
      - name: backup-volume
        persistentVolumeClaim:
          claimName: backup-pvc
```

### 15. CronJob Récurrent

Pour tâches planifiées :

```yaml
# cronjob-cleanup.yaml
# CronJob pour tâches récurrentes
apiVersion: batch/v1
kind: CronJob
metadata:
  name: log-cleanup
  namespace: apps
spec:
  schedule: "0 2 * * *"  # Tous les jours à 2h du matin
  # Format: minute heure jour mois jour-semaine
  # Exemples:
  # "*/5 * * * *"    - Toutes les 5 minutes
  # "0 */6 * * *"    - Toutes les 6 heures
  # "0 0 * * 0"      - Tous les dimanches à minuit
  # "0 0 1 * *"      - Premier jour du mois à minuit

  jobTemplate:
    spec:
      template:
        spec:
          restartPolicy: OnFailure
          containers:
          - name: cleanup
            image: busybox:1.35
            command:
            - /bin/sh
            - -c
            - |
              echo "Nettoyage des logs de plus de 7 jours..."
              find /logs -type f -name "*.log" -mtime +7 -delete
              echo "Nettoyage terminé"
            volumeMounts:
            - name: logs-volume
              mountPath: /logs
          volumes:
          - name: logs-volume
            persistentVolumeClaim:
              claimName: logs-pvc
  successfulJobsHistoryLimit: 3  # Garder 3 derniers jobs réussis
  failedJobsHistoryLimit: 1      # Garder 1 dernier job échoué
  concurrencyPolicy: Forbid       # Forbid, Allow, Replace
```

## Templates RBAC (Sécurité)

### 16. ServiceAccount et Role

Configuration des permissions :

```yaml
# rbac-app.yaml
# Configuration RBAC pour une application
---
# ServiceAccount pour l'application
apiVersion: v1
kind: ServiceAccount
metadata:
  name: app-service-account
  namespace: apps
---
# Role avec permissions spécifiques
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: app-role
  namespace: apps
rules:
# Permissions sur les ConfigMaps
- apiGroups: [""]
  resources: ["configmaps"]
  verbs: ["get", "list", "watch"]
# Permissions sur les Secrets (lecture seule)
- apiGroups: [""]
  resources: ["secrets"]
  verbs: ["get"]
# Permissions sur les Pods (pour monitoring)
- apiGroups: [""]
  resources: ["pods"]
  verbs: ["get", "list"]
---
# RoleBinding pour lier le Role au ServiceAccount
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: app-rolebinding
  namespace: apps
subjects:
- kind: ServiceAccount
  name: app-service-account
  namespace: apps
roleRef:
  kind: Role
  name: app-role
  apiGroup: rbac.authorization.k8s.io
```

### 17. NetworkPolicy

Isolation réseau entre pods :

```yaml
# networkpolicy-app.yaml
# NetworkPolicy pour contrôler le trafic réseau
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: app-network-policy
  namespace: apps
spec:
  podSelector:
    matchLabels:
      app: mon-app
  policyTypes:
  - Ingress
  - Egress
  ingress:
  # Autoriser depuis l'Ingress Controller
  - from:
    - namespaceSelector:
        matchLabels:
          name: ingress
    ports:
    - protocol: TCP
      port: 8080
  # Autoriser depuis les pods du même namespace
  - from:
    - podSelector:
        matchLabels:
          tier: frontend
    ports:
    - protocol: TCP
      port: 8080
  egress:
  # Autoriser vers la base de données
  - to:
    - podSelector:
        matchLabels:
          app: postgres
    ports:
    - protocol: TCP
      port: 5432
  # Autoriser DNS
  - to:
    - namespaceSelector:
        matchLabels:
          name: kube-system
    - podSelector:
        matchLabels:
          k8s-app: kube-dns
    ports:
    - protocol: UDP
      port: 53
  # Autoriser Internet (APIs externes)
  - to:
    - ipBlock:
        cidr: 0.0.0.0/0
        except:
        - 169.254.169.254/32  # Metadata service AWS
        - 10.0.0.0/8          # Réseau interne
        - 192.168.0.0/16
```

## Templates de Monitoring

### 18. ServiceMonitor pour Prometheus

Exposition des métriques :

```yaml
# servicemonitor-app.yaml
# ServiceMonitor pour scraping Prometheus
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: app-metrics
  namespace: apps
  labels:
    app: mon-app
    prometheus: kube-prometheus
spec:
  selector:
    matchLabels:
      app: mon-app
  endpoints:
  - port: metrics  # Port nommé dans le Service
    interval: 30s
    path: /metrics
    scheme: http
    # Relabeling optionnel
    relabelings:
    - sourceLabels: [__meta_kubernetes_pod_name]
      targetLabel: pod
    - sourceLabels: [__meta_kubernetes_pod_node_name]
      targetLabel: node
```

### 19. HorizontalPodAutoscaler

Mise à l'échelle automatique :

```yaml
# hpa-app.yaml
# HorizontalPodAutoscaler pour scaling automatique
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: app-hpa
  namespace: apps
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: mon-app-deployment
  minReplicas: 2
  maxReplicas: 10
  metrics:
  # Scaling basé sur CPU
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 70
  # Scaling basé sur mémoire
  - type: Resource
    resource:
      name: memory
      target:
        type: Utilization
        averageUtilization: 80
  # Scaling basé sur métrique custom (nécessite metrics-server)
  # - type: Pods
  #   pods:
  #     metric:
  #       name: requests_per_second
  #     target:
  #       type: AverageValue
  #       averageValue: "1000"
  behavior:
    scaleDown:
      stabilizationWindowSeconds: 300  # Attendre 5min avant scale down
      policies:
      - type: Percent
        value: 50  # Réduire max 50% des pods
        periodSeconds: 60
    scaleUp:
      stabilizationWindowSeconds: 60  # Attendre 1min avant scale up
      policies:
      - type: Percent
        value: 100  # Doubler max les pods
        periodSeconds: 60
```

## Templates Avancés

### 20. DaemonSet pour Agents

Déployer sur chaque node :

```yaml
# daemonset-monitoring.yaml
# DaemonSet pour agent de monitoring sur chaque node
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: node-exporter
  namespace: monitoring
spec:
  selector:
    matchLabels:
      app: node-exporter
  template:
    metadata:
      labels:
        app: node-exporter
    spec:
      hostNetwork: true  # Utiliser le réseau de l'hôte
      hostPID: true      # Accès aux processus de l'hôte
      containers:
      - name: node-exporter
        image: prom/node-exporter:latest
        ports:
        - containerPort: 9100
          hostPort: 9100
        resources:
          requests:
            memory: "30Mi"
            cpu: "100m"
          limits:
            memory: "50Mi"
            cpu: "200m"
        volumeMounts:
        - name: proc
          mountPath: /host/proc
          readOnly: true
        - name: sys
          mountPath: /host/sys
          readOnly: true
      volumes:
      - name: proc
        hostPath:
          path: /proc
      - name: sys
        hostPath:
          path: /sys
      tolerations:
      # S'exécuter même sur les nodes avec taints
      - effect: NoSchedule
        operator: Exists
```

### 21. Init Containers

Conteneurs d'initialisation :

```yaml
# deployment-with-init.yaml
# Deployment avec init containers
apiVersion: apps/v1
kind: Deployment
metadata:
  name: app-with-init
  namespace: apps
spec:
  replicas: 1
  selector:
    matchLabels:
      app: app-with-init
  template:
    metadata:
      labels:
        app: app-with-init
    spec:
      initContainers:
      # Attendre que la base de données soit prête
      - name: wait-for-db
        image: busybox:1.35
        command:
        - sh
        - -c
        - |
          echo "Attente de la base de données..."
          until nc -z postgres-service 5432; do
            echo "Base de données non prête, attente..."
            sleep 2
          done
          echo "Base de données prête!"

      # Effectuer les migrations
      - name: db-migration
        image: mon-app:latest
        command: ["./migrate.sh"]
        env:
        - name: DATABASE_URL
          valueFrom:
            secretKeyRef:
              name: app-secrets
              key: database-url

      # Télécharger des assets
      - name: download-assets
        image: busybox:1.35
        command:
        - sh
        - -c
        - |
          echo "Téléchargement des assets..."
          wget -O /shared/assets.tar.gz https://example.com/assets.tar.gz
          cd /shared && tar -xzf assets.tar.gz
          echo "Assets téléchargés"
        volumeMounts:
        - name: shared-data
          mountPath: /shared

      # Conteneur principal
      containers:
      - name: main-app
        image: mon-app:latest
        ports:
        - containerPort: 8080
        volumeMounts:
        - name: shared-data
          mountPath: /app/assets

      volumes:
      - name: shared-data
        emptyDir: {}
```

## Guide d'Utilisation des Templates

### Comment Utiliser ces Templates

#### 1. Copier et Adapter

```bash
# Copier le template
cp deployment-complet.yaml mon-app-deployment.yaml

# Éditer avec vos valeurs
nano mon-app-deployment.yaml

# Remplacer :
# - mon-app → nom de votre application
# - namespace → votre namespace
# - image → votre image Docker
# - ports → vos ports
# - variables d'environnement → vos configs
```

#### 2. Valider la Syntaxe

```bash
# Valider sans appliquer
kubectl apply --dry-run=client -f mon-app-deployment.yaml

# Voir ce qui sera créé
kubectl apply --dry-run=server -f mon-app-deployment.yaml
```

#### 3. Appliquer le Manifeste

```bash
# Créer/Mettre à jour les ressources
kubectl apply -f mon-app-deployment.yaml

# Vérifier la création
kubectl get all -n apps
```

#### 4. Déboguer si Nécessaire

```bash
# Voir les événements
kubectl describe deployment mon-app-deployment -n apps

# Voir les logs
kubectl logs -l app=mon-app -n apps

# Éditer en direct (déconseillé en production)
kubectl edit deployment mon-app-deployment -n apps
```

### Conventions et Bonnes Pratiques

#### Nommage

- **Lowercase** : Utiliser des minuscules et tirets
- **Descriptif** : `app-name-resource-type`
- **Cohérent** : Même préfixe pour ressources liées

```yaml
# Bon
name: blog-api-deployment
name: blog-api-service
name: blog-api-ingress

# Mauvais
name: MyApp
name: app_deployment
name: deploy1
```

#### Labels Recommandés

```yaml
metadata:
  labels:
    app.kubernetes.io/name: mon-app
    app.kubernetes.io/version: "1.0.0"
    app.kubernetes.io/component: backend
    app.kubernetes.io/part-of: mon-système
    app.kubernetes.io/managed-by: kubectl
    environment: production
    tier: backend
```

#### Structure des Fichiers

```bash
# Organisation recommandée
k8s/
├── base/
│   ├── namespace.yaml
│   ├── configmap.yaml
│   ├── secret.yaml
│   └── rbac.yaml
├── apps/
│   ├── app1/
│   │   ├── deployment.yaml
│   │   ├── service.yaml
│   │   └── ingress.yaml
│   └── app2/
│       ├── statefulset.yaml
│       ├── service.yaml
│       └── pvc.yaml
├── monitoring/
│   ├── prometheus/
│   └── grafana/
└── scripts/
    ├── deploy-all.sh
    └── cleanup.sh
```

### Ordre de Déploiement

L'ordre est important pour éviter les erreurs :

1. **Namespace** - Créer l'espace de travail
2. **ConfigMaps et Secrets** - Configuration nécessaire aux apps
3. **PersistentVolumeClaims** - Stockage requis
4. **Services** - Pour la découverte de services
5. **Deployments/StatefulSets** - Les applications
6. **Ingress** - Exposition externe

```bash
# Script de déploiement ordonné
#!/bin/bash
kubectl apply -f namespace.yaml
kubectl apply -f configmap.yaml
kubectl apply -f secret.yaml
kubectl apply -f pvc.yaml
kubectl apply -f service.yaml
kubectl apply -f deployment.yaml
kubectl apply -f ingress.yaml
```

### Gestion des Ressources

#### Limites Recommandées par Type d'Application

```yaml
# Application légère (site statique, proxy)
resources:
  requests:
    memory: "64Mi"
    cpu: "50m"
  limits:
    memory: "128Mi"
    cpu: "100m"

# Application web standard
resources:
  requests:
    memory: "256Mi"
    cpu: "100m"
  limits:
    memory: "512Mi"
    cpu: "500m"

# Base de données
resources:
  requests:
    memory: "512Mi"
    cpu: "250m"
  limits:
    memory: "1Gi"
    cpu: "1000m"

# Application Java/JVM
resources:
  requests:
    memory: "1Gi"
    cpu: "500m"
  limits:
    memory: "2Gi"
    cpu: "2000m"
```

## Templates Composites

### 22. Stack Application Complète

Exemple d'application complète avec tous les composants :

```yaml
# stack-complete.yaml
# Stack complète : Namespace + Config + Deployment + Service + Ingress
---
# 1. Namespace
apiVersion: v1
kind: Namespace
metadata:
  name: myapp-prod
  labels:
    environment: production
---
# 2. ConfigMap
apiVersion: v1
kind: ConfigMap
metadata:
  name: myapp-config
  namespace: myapp-prod
data:
  NODE_ENV: "production"
  API_URL: "https://api.example.com"
  CACHE_TTL: "3600"
---
# 3. Secret
apiVersion: v1
kind: Secret
metadata:
  name: myapp-secret
  namespace: myapp-prod
type: Opaque
stringData:
  database-url: "postgresql://user:pass@postgres:5432/myapp"
  jwt-secret: "super-secret-jwt-key"
  api-key: "external-api-key-12345"
---
# 4. PVC pour données persistantes
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: myapp-data
  namespace: myapp-prod
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 5Gi
  storageClassName: microk8s-hostpath
---
# 5. Deployment
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp
  namespace: myapp-prod
  labels:
    app: myapp
    version: v1.0.0
spec:
  replicas: 3
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 0
  selector:
    matchLabels:
      app: myapp
  template:
    metadata:
      labels:
        app: myapp
        version: v1.0.0
    spec:
      containers:
      - name: myapp
        image: myregistry/myapp:1.0.0
        imagePullPolicy: IfNotPresent
        ports:
        - containerPort: 3000
          name: http
        envFrom:
        - configMapRef:
            name: myapp-config
        env:
        - name: DATABASE_URL
          valueFrom:
            secretKeyRef:
              name: myapp-secret
              key: database-url
        - name: JWT_SECRET
          valueFrom:
            secretKeyRef:
              name: myapp-secret
              key: jwt-secret
        livenessProbe:
          httpGet:
            path: /health
            port: 3000
          initialDelaySeconds: 30
          periodSeconds: 10
        readinessProbe:
          httpGet:
            path: /ready
            port: 3000
          initialDelaySeconds: 10
          periodSeconds: 5
        resources:
          requests:
            memory: "256Mi"
            cpu: "100m"
          limits:
            memory: "512Mi"
            cpu: "500m"
        volumeMounts:
        - name: data
          mountPath: /app/data
      volumes:
      - name: data
        persistentVolumeClaim:
          claimName: myapp-data
---
# 6. Service
apiVersion: v1
kind: Service
metadata:
  name: myapp-service
  namespace: myapp-prod
  labels:
    app: myapp
spec:
  type: ClusterIP
  selector:
    app: myapp
  ports:
  - port: 80
    targetPort: 3000
    protocol: TCP
    name: http
---
# 7. Ingress
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: myapp-ingress
  namespace: myapp-prod
  annotations:
    cert-manager.io/cluster-issuer: "letsencrypt-prod"
    nginx.ingress.kubernetes.io/ssl-redirect: "true"
spec:
  ingressClassName: nginx
  tls:
  - hosts:
    - myapp.example.com
    secretName: myapp-tls
  rules:
  - host: myapp.example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: myapp-service
            port:
              number: 80
---
# 8. HPA pour autoscaling
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: myapp-hpa
  namespace: myapp-prod
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: myapp
  minReplicas: 2
  maxReplicas: 10
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 70
  - type: Resource
    resource:
      name: memory
      target:
        type: Utilization
        averageUtilization: 80
```

### 23. Stack Base de Données

Configuration complète pour PostgreSQL :

```yaml
# stack-postgres.yaml
# Stack PostgreSQL complète avec backup
---
# Secret pour mot de passe
apiVersion: v1
kind: Secret
metadata:
  name: postgres-secret
  namespace: databases
type: Opaque
stringData:
  postgres-password: "ChangeMeSecurePassword123!"
  replication-password: "ReplicationPassword456!"
---
# ConfigMap pour configuration
apiVersion: v1
kind: ConfigMap
metadata:
  name: postgres-config
  namespace: databases
data:
  POSTGRES_DB: "production"
  POSTGRES_USER: "dbadmin"
  PGDATA: "/var/lib/postgresql/data/pgdata"
  # Configuration PostgreSQL
  postgresql.conf: |
    max_connections = 100
    shared_buffers = 256MB
    effective_cache_size = 1GB
    maintenance_work_mem = 64MB
    checkpoint_completion_target = 0.9
    wal_buffers = 16MB
    default_statistics_target = 100
    random_page_cost = 1.1
    effective_io_concurrency = 200
    work_mem = 4MB
    min_wal_size = 1GB
    max_wal_size = 4GB
---
# Service Headless pour StatefulSet
apiVersion: v1
kind: Service
metadata:
  name: postgres-headless
  namespace: databases
spec:
  clusterIP: None
  selector:
    app: postgres
  ports:
  - port: 5432
    targetPort: 5432
---
# Service pour accès lecture/écriture
apiVersion: v1
kind: Service
metadata:
  name: postgres
  namespace: databases
spec:
  type: ClusterIP
  selector:
    app: postgres
    role: master
  ports:
  - port: 5432
    targetPort: 5432
---
# StatefulSet PostgreSQL
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: postgres
  namespace: databases
spec:
  serviceName: postgres-headless
  replicas: 1
  selector:
    matchLabels:
      app: postgres
  template:
    metadata:
      labels:
        app: postgres
        role: master
    spec:
      containers:
      - name: postgres
        image: postgres:14-alpine
        ports:
        - containerPort: 5432
          name: postgres
        env:
        - name: POSTGRES_DB
          valueFrom:
            configMapKeyRef:
              name: postgres-config
              key: POSTGRES_DB
        - name: POSTGRES_USER
          valueFrom:
            configMapKeyRef:
              name: postgres-config
              key: POSTGRES_USER
        - name: POSTGRES_PASSWORD
          valueFrom:
            secretKeyRef:
              name: postgres-secret
              key: postgres-password
        - name: PGDATA
          valueFrom:
            configMapKeyRef:
              name: postgres-config
              key: PGDATA
        volumeMounts:
        - name: postgres-data
          mountPath: /var/lib/postgresql/data
        - name: postgres-config-volume
          mountPath: /etc/postgresql/postgresql.conf
          subPath: postgresql.conf
        resources:
          requests:
            memory: "512Mi"
            cpu: "250m"
          limits:
            memory: "1Gi"
            cpu: "1000m"
        livenessProbe:
          exec:
            command:
            - pg_isready
            - -U
            - $(POSTGRES_USER)
          initialDelaySeconds: 30
          periodSeconds: 10
        readinessProbe:
          exec:
            command:
            - pg_isready
            - -U
            - $(POSTGRES_USER)
          initialDelaySeconds: 5
          periodSeconds: 5
      volumes:
      - name: postgres-config-volume
        configMap:
          name: postgres-config
  volumeClaimTemplates:
  - metadata:
      name: postgres-data
    spec:
      accessModes: ["ReadWriteOnce"]
      storageClassName: microk8s-hostpath
      resources:
        requests:
          storage: 20Gi
---
# CronJob pour backup quotidien
apiVersion: batch/v1
kind: CronJob
metadata:
  name: postgres-backup
  namespace: databases
spec:
  schedule: "0 2 * * *"  # 2h du matin tous les jours
  jobTemplate:
    spec:
      template:
        spec:
          restartPolicy: OnFailure
          containers:
          - name: postgres-backup
            image: postgres:14-alpine
            command:
            - /bin/sh
            - -c
            - |
              DATE=$(date +%Y%m%d_%H%M%S)
              echo "Démarrage backup PostgreSQL - $DATE"
              PGPASSWORD=$POSTGRES_PASSWORD pg_dump \
                -h postgres \
                -U $POSTGRES_USER \
                -d $POSTGRES_DB \
                --verbose \
                --format=custom \
                --file=/backup/postgres-$DATE.dump

              # Nettoyer les backups de plus de 7 jours
              find /backup -name "postgres-*.dump" -mtime +7 -delete

              echo "Backup terminé - postgres-$DATE.dump"
            env:
            - name: POSTGRES_DB
              valueFrom:
                configMapKeyRef:
                  name: postgres-config
                  key: POSTGRES_DB
            - name: POSTGRES_USER
              valueFrom:
                configMapKeyRef:
                  name: postgres-config
                  key: POSTGRES_USER
            - name: POSTGRES_PASSWORD
              valueFrom:
                secretKeyRef:
                  name: postgres-secret
                  key: postgres-password
            volumeMounts:
            - name: backup-volume
              mountPath: /backup
          volumes:
          - name: backup-volume
            persistentVolumeClaim:
              claimName: postgres-backup-pvc
---
# PVC pour les backups
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: postgres-backup-pvc
  namespace: databases
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 50Gi
  storageClassName: microk8s-hostpath
```

## Commandes Utiles pour les Templates

### Commandes de Base

```bash
# Appliquer un template
kubectl apply -f template.yaml

# Appliquer tous les fichiers d'un répertoire
kubectl apply -f ./k8s/

# Appliquer avec substitution de variables
envsubst < template.yaml | kubectl apply -f -

# Supprimer les ressources d'un template
kubectl delete -f template.yaml

# Voir ce qui sera créé sans l'appliquer
kubectl apply --dry-run=client -f template.yaml -o yaml

# Générer un template depuis une ressource existante
kubectl get deployment mon-app -o yaml > mon-app-export.yaml
```

### Commandes de Débogage

```bash
# Voir les événements récents
kubectl get events --sort-by='.lastTimestamp' -n apps

# Décrire une ressource pour voir les détails
kubectl describe deployment mon-app -n apps

# Voir les logs d'un pod
kubectl logs -f deployment/mon-app -n apps

# Exécuter une commande dans un pod
kubectl exec -it deployment/mon-app -n apps -- /bin/bash

# Port-forward pour tester localement
kubectl port-forward service/mon-app-service 8080:80 -n apps

# Voir l'utilisation des ressources
kubectl top pods -n apps
kubectl top nodes
```

### Scripts Helper

#### Script de Validation des Templates

```bash
#!/bin/bash
# validate-templates.sh
# Valide tous les templates YAML

for file in k8s/**/*.yaml; do
    echo "Validation de $file..."
    if kubectl apply --dry-run=client -f "$file" > /dev/null 2>&1; then
        echo "✓ $file est valide"
    else
        echo "✗ $file contient des erreurs"
        kubectl apply --dry-run=client -f "$file"
    fi
done
```

#### Script de Déploiement avec Rollback

```bash
#!/bin/bash
# deploy-with-rollback.sh
# Déploie avec possibilité de rollback

NAMESPACE="apps"
DEPLOYMENT="mon-app"

# Sauvegarder l'état actuel
kubectl get deployment $DEPLOYMENT -n $NAMESPACE -o yaml > rollback-$DEPLOYMENT.yaml

# Appliquer le nouveau déploiement
if kubectl apply -f deployment-new.yaml; then
    echo "Déploiement réussi"

    # Attendre que le déploiement soit prêt
    kubectl rollout status deployment/$DEPLOYMENT -n $NAMESPACE

    # Vérifier la santé
    if kubectl get pods -l app=$DEPLOYMENT -n $NAMESPACE | grep -q "0/"; then
        echo "Erreur détectée, rollback..."
        kubectl apply -f rollback-$DEPLOYMENT.yaml
        kubectl rollout status deployment/$DEPLOYMENT -n $NAMESPACE
    else
        echo "Déploiement validé"
        rm rollback-$DEPLOYMENT.yaml
    fi
else
    echo "Erreur de déploiement"
    exit 1
fi
```

## Dépannage des Templates Courants

### Problèmes Fréquents et Solutions

#### Image Pull Errors

```yaml
# Problème : ErrImagePull ou ImagePullBackOff
# Solutions :

# 1. Vérifier le nom de l'image
image: nginx:1.21  # ✓ Correct
# image: ngix:1.21  # ✗ Typo

# 2. Pour images privées, ajouter imagePullSecrets
spec:
  imagePullSecrets:
  - name: registry-secret
  containers:
  - name: app
    image: private-registry.com/app:latest

# 3. Créer le secret pour registry privé
kubectl create secret docker-registry registry-secret \
  --docker-server=private-registry.com \
  --docker-username=user \
  --docker-password=pass \
  --docker-email=email@example.com
```

#### CrashLoopBackOff

```yaml
# Problème : Le conteneur redémarre constamment
# Solutions :

# 1. Vérifier les logs
kubectl logs pod-name -n namespace --previous

# 2. Augmenter les délais de probe
livenessProbe:
  initialDelaySeconds: 60  # Plus de temps pour démarrer
  periodSeconds: 20
  timeoutSeconds: 10
  failureThreshold: 5

# 3. Vérifier les variables d'environnement requises
env:
- name: REQUIRED_VAR
  value: "must-be-set"
```

#### Pending Pods

```yaml
# Problème : Pods restent en Pending
# Solutions :

# 1. Vérifier les ressources disponibles
kubectl describe nodes
kubectl top nodes

# 2. Réduire les demandes de ressources
resources:
  requests:
    memory: "64Mi"   # Réduire si nécessaire
    cpu: "50m"       # Réduire si nécessaire

# 3. Vérifier le PVC
kubectl get pvc -n namespace
kubectl describe pvc pvc-name -n namespace
```

## Conclusion

Cette collection de templates constitue une base solide pour déployer vos applications sur MicroK8s. Les points clés à retenir :

1. **Commencez simple** : Utilisez les templates de base et complexifiez progressivement
2. **Testez toujours** : Utilisez `--dry-run` avant d'appliquer en production
3. **Versionnez vos manifestes** : Gardez une trace des changements avec Git
4. **Documentez vos modifications** : Ajoutez des commentaires pour expliquer les choix
5. **Respectez les conventions** : Nommage cohérent et labels standardisés

Ces templates sont des points de départ que vous devez adapter à vos besoins spécifiques. N'hésitez pas à les modifier, les combiner et les enrichir selon votre expérience et vos requirements.

---

*Note : Les templates fournis utilisent des valeurs par défaut sécurisées mais génériques. Assurez-vous de les personnaliser avec vos propres valeurs, notamment pour les mots de passe, les noms de domaine et les configurations spécifiques à votre environnement.*

⏭️
