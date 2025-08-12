🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 7.3 - Déploiement d'une application web simple

## Introduction

Maintenant que nous comprenons l'anatomie d'un déploiement Kubernetes et savons écrire des manifestes YAML, il est temps de mettre ces connaissances en pratique. Nous allons déployer une application web simple mais complète, étape par étape, en expliquant chaque décision et chaque ligne de configuration.

Notre objectif est de créer un déploiement réaliste qui illustre tous les concepts fondamentaux : depuis le simple Pod jusqu'à l'exposition publique avec HTTPS, en passant par la configuration et la persistance des données.

## L'application que nous allons déployer

Pour cet exemple, nous allons déployer une application de blog simple avec les caractéristiques suivantes :

- **Frontend** : Interface web servie par Nginx
- **Configuration** : Personnalisation via ConfigMap
- **Secrets** : Clés API stockées de manière sécurisée
- **Stockage** : Volume persistant pour les uploads
- **Exposition** : Accessible via HTTPS avec un nom de domaine
- **Monitoring** : Health checks pour la disponibilité

Cette application est représentative de nombreux cas d'usage réels tout en restant simple à comprendre.

## Architecture de notre déploiement

```
┌─────────────────────────────────────────────────────────────┐
│                         Internet                            │
└────────────────────┬────────────────────────────────────────┘
                     │ HTTPS (webapp.monlab.local)
                     ▼
            ┌──────────────────┐
            │     Ingress      │
            │   (NGINX + SSL)  │
            └────────┬─────────┘
                     │ HTTP
                     ▼
            ┌──────────────────┐
            │     Service      │
            │   (ClusterIP)    │
            └────────┬─────────┘
                     │ Load Balancing
        ┌────────────┼────────────┐
        ▼            ▼            ▼
   ┌─────────┐  ┌─────────┐  ┌─────────┐
   │  Pod 1  │  │  Pod 2  │  │  Pod 3  │
   │  Nginx  │  │  Nginx  │  │  Nginx  │
   └────┬────┘  └────┬────┘  └────┬────┘
        │            │            │
        ├────────────┼────────────┤
        │            │            │
   ConfigMap    Secret      PersistentVolume
```

## Étape 1 : Préparation de l'environnement

### Créer un namespace dédié

Un namespace isole nos ressources et facilite la gestion. C'est comme avoir un dossier dédié pour notre projet.

```yaml
# 01-namespace.yaml
apiVersion: v1
kind: Namespace
metadata:
  name: webapp-demo
  labels:
    name: webapp-demo
    environment: demo
  annotations:
    description: "Namespace pour l'application web de démonstration"
```

**Pourquoi un namespace ?**
- **Isolation** : Sépare les ressources de différentes applications
- **Organisation** : Regroupe tous les composants liés
- **Quotas** : Permet de limiter les ressources (CPU, mémoire)
- **Sécurité** : Facilite l'application de politiques RBAC
- **Nettoyage** : Suppression facile de tout avec `kubectl delete namespace webapp-demo`

### Vérifications préalables

Avant de commencer, assurons-nous que notre cluster est prêt :

```bash
# Vérifier que MicroK8s fonctionne
microk8s status

# Vérifier les addons nécessaires
# Devraient être enabled : dns, storage, ingress

# Vérifier que kubectl est configuré
kubectl cluster-info

# Lister les namespaces existants
kubectl get namespaces
```

## Étape 2 : Configuration avec ConfigMap

Commençons par externaliser la configuration de notre application. Cela permet de modifier les paramètres sans reconstruire l'image Docker.

```yaml
# 02-configmap.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: webapp-config
  namespace: webapp-demo
  labels:
    app: webapp
    component: configuration
data:
  # Variables de configuration simples
  app_name: "Mon Blog Personnel"
  app_version: "1.0.0"
  environment: "demo"
  debug_mode: "false"
  max_upload_size: "10485760"  # 10MB en bytes

  # Configuration Nginx personnalisée
  nginx.conf: |
    server {
        listen 80;
        server_name webapp.monlab.local;
        root /usr/share/nginx/html;
        index index.html;

        # Taille max des uploads
        client_max_body_size 10M;

        # Compression gzip
        gzip on;
        gzip_types text/plain text/css application/json application/javascript;

        # Headers de sécurité
        add_header X-Frame-Options "SAMEORIGIN" always;
        add_header X-Content-Type-Options "nosniff" always;
        add_header X-XSS-Protection "1; mode=block" always;

        # Cache pour les assets statiques
        location ~* \.(jpg|jpeg|png|gif|ico|css|js)$ {
            expires 30d;
            add_header Cache-Control "public, immutable";
        }

        # Page d'erreur personnalisée
        error_page 404 /404.html;
        error_page 500 502 503 504 /50x.html;

        # Health check endpoint
        location /health {
            access_log off;
            return 200 "healthy\n";
            add_header Content-Type text/plain;
        }
    }

  # Page d'accueil HTML personnalisée
  index.html: |
    <!DOCTYPE html>
    <html lang="fr">
    <head>
        <meta charset="UTF-8">
        <meta name="viewport" content="width=device-width, initial-scale=1.0">
        <title>Mon Blog Personnel</title>
        <style>
            body {
                font-family: -apple-system, BlinkMacSystemFont, 'Segoe UI', Roboto, sans-serif;
                max-width: 800px;
                margin: 0 auto;
                padding: 2rem;
                background: linear-gradient(135deg, #667eea 0%, #764ba2 100%);
                min-height: 100vh;
            }
            .container {
                background: white;
                border-radius: 10px;
                padding: 2rem;
                box-shadow: 0 20px 40px rgba(0,0,0,0.1);
            }
            h1 {
                color: #333;
                border-bottom: 3px solid #667eea;
                padding-bottom: 1rem;
            }
            .info {
                background: #f7f7f7;
                padding: 1rem;
                border-radius: 5px;
                margin: 1rem 0;
            }
            .status {
                display: inline-block;
                padding: 0.25rem 0.5rem;
                background: #10b981;
                color: white;
                border-radius: 3px;
                font-size: 0.875rem;
            }
        </style>
    </head>
    <body>
        <div class="container">
            <h1>🚀 Mon Blog Personnel</h1>
            <p>Bienvenue sur mon blog déployé avec Kubernetes !</p>

            <div class="info">
                <h2>Informations du déploiement</h2>
                <p>📦 <strong>Version :</strong> 1.0.0</p>
                <p>🌍 <strong>Environnement :</strong> Demo</p>
                <p>🖥️ <strong>Plateforme :</strong> MicroK8s</p>
                <p>📊 <strong>Status :</strong> <span class="status">En ligne</span></p>
            </div>

            <div class="info">
                <h2>Fonctionnalités</h2>
                <ul>
                    <li>✅ Déploiement avec Kubernetes</li>
                    <li>✅ Configuration externalisée (ConfigMap)</li>
                    <li>✅ HTTPS avec certificat SSL</li>
                    <li>✅ Load balancing automatique</li>
                    <li>✅ Health checks intégrés</li>
                    <li>✅ Stockage persistant</li>
                </ul>
            </div>

            <div class="info">
                <h2>Architecture</h2>
                <p>Cette application illustre un déploiement Kubernetes complet avec :</p>
                <ul>
                    <li>3 replicas pour la haute disponibilité</li>
                    <li>Service de type ClusterIP pour l'exposition interne</li>
                    <li>Ingress NGINX pour le routage externe</li>
                    <li>ConfigMap pour la configuration</li>
                    <li>Secret pour les données sensibles</li>
                    <li>PersistentVolume pour le stockage</li>
                </ul>
            </div>

            <footer style="text-align: center; margin-top: 2rem; color: #666;">
                <p>Déployé avec ❤️ sur MicroK8s | <a href="/health">Health Check</a></p>
            </footer>
        </div>
    </body>
    </html>
```

**Points importants du ConfigMap :**
- **Séparation configuration/code** : La config est externe à l'image Docker
- **Multiple formats** : Variables simples et fichiers complets
- **Réutilisabilité** : Même config pour tous les Pods
- **Mise à jour facile** : Modifiable sans rebuild

## Étape 3 : Gestion des secrets

Les données sensibles ne doivent jamais être dans les ConfigMaps. Utilisons un Secret pour stocker nos clés API et mots de passe.

```yaml
# 03-secret.yaml
apiVersion: v1
kind: Secret
metadata:
  name: webapp-secret
  namespace: webapp-demo
  labels:
    app: webapp
    component: security
type: Opaque
stringData:  # Kubernetes encodera automatiquement en base64
  api_key: "sk_demo_a1b2c3d4e5f6g7h8i9j0"
  admin_password: "SuperSecretPassword123!"
  jwt_secret: "my-super-secret-jwt-key-for-tokens"
  database_url: "postgresql://user:pass@db.local:5432/webapp"

  # Certificat SSL auto-signé pour tests (optionnel)
  # En production, Cert-Manager gérera cela
  tls.crt: |
    -----BEGIN CERTIFICATE-----
    # Contenu du certificat...
    -----END CERTIFICATE-----

  tls.key: |
    -----BEGIN PRIVATE KEY-----
    # Contenu de la clé privée...
    -----END PRIVATE KEY-----
```

**Bonnes pratiques pour les Secrets :**
- **Ne jamais commiter** les vrais secrets dans Git
- **Utiliser stringData** pour la lisibilité (Kubernetes encode)
- **Rotation régulière** des secrets en production
- **RBAC strict** pour limiter l'accès
- **Chiffrement au repos** activé sur le cluster

## Étape 4 : Stockage persistant

Notre application a besoin de stocker des fichiers uploadés. Créons un volume persistant.

```yaml
# 04-persistentvolume.yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: webapp-storage
  namespace: webapp-demo
  labels:
    app: webapp
    component: storage
spec:
  accessModes:
    - ReadWriteMany  # RWX car plusieurs Pods y accèdent
  resources:
    requests:
      storage: 5Gi
  storageClassName: microk8s-hostpath  # Classe de stockage MicroK8s

  # Optionnel : sélecteur pour un PV spécifique
  # selector:
  #   matchLabels:
  #     app: webapp
```

**Types d'accès au stockage :**
- **ReadWriteOnce (RWO)** : Un seul nœud peut monter en lecture/écriture
- **ReadOnlyMany (ROX)** : Plusieurs nœuds peuvent monter en lecture seule
- **ReadWriteMany (RWX)** : Plusieurs nœuds peuvent monter en lecture/écriture

**Note pour MicroK8s :**
Le storage class `microk8s-hostpath` supporte RWX car tout est sur le même nœud. En production multi-nœuds, vous auriez besoin de NFS, Ceph, ou un storage cloud.

## Étape 5 : Le Deployment principal

Maintenant, créons le cœur de notre application : le Deployment qui gère nos Pods.

```yaml
# 05-deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: webapp
  namespace: webapp-demo
  labels:
    app: webapp
    version: v1.0.0
  annotations:
    description: "Deployment principal de l'application web"
spec:
  replicas: 3  # Nombre de Pods pour la haute disponibilité

  # Stratégie de mise à jour
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1        # 1 Pod supplémentaire pendant la mise à jour
      maxUnavailable: 1  # 1 Pod peut être indisponible

  # Historique des révisions
  revisionHistoryLimit: 5

  # Sélecteur pour identifier les Pods
  selector:
    matchLabels:
      app: webapp
      tier: frontend

  # Template du Pod
  template:
    metadata:
      labels:
        app: webapp
        tier: frontend
        version: v1.0.0
      annotations:
        prometheus.io/scrape: "true"  # Pour le monitoring
        prometheus.io/port: "9113"

    spec:
      # Affinité pour répartir les Pods sur différents nœuds (si multi-nœuds)
      affinity:
        podAntiAffinity:
          preferredDuringSchedulingIgnoredDuringExecution:
          - weight: 100
            podAffinityTerm:
              labelSelector:
                matchExpressions:
                - key: app
                  operator: In
                  values:
                  - webapp
              topologyKey: kubernetes.io/hostname

      # Conteneur principal
      containers:
      - name: nginx
        image: nginx:1.21-alpine  # Image légère Alpine Linux
        imagePullPolicy: IfNotPresent

        # Ports exposés
        ports:
        - name: http
          containerPort: 80
          protocol: TCP

        # Variables d'environnement depuis ConfigMap et Secret
        env:
        - name: APP_NAME
          valueFrom:
            configMapKeyRef:
              name: webapp-config
              key: app_name
        - name: ENVIRONMENT
          valueFrom:
            configMapKeyRef:
              name: webapp-config
              key: environment
        - name: API_KEY
          valueFrom:
            secretKeyRef:
              name: webapp-secret
              key: api_key

        # Ou charger toutes les variables d'un coup
        envFrom:
        - configMapRef:
            name: webapp-config
            prefix: CONFIG_  # Préfixe optionnel
        - secretRef:
            name: webapp-secret
            prefix: SECRET_

        # Montage des volumes
        volumeMounts:
        # Configuration Nginx personnalisée
        - name: nginx-config
          mountPath: /etc/nginx/conf.d/default.conf
          subPath: nginx.conf
          readOnly: true

        # Page HTML personnalisée
        - name: webapp-content
          mountPath: /usr/share/nginx/html/index.html
          subPath: index.html
          readOnly: true

        # Stockage persistant pour uploads
        - name: uploads
          mountPath: /usr/share/nginx/html/uploads

        # Répertoire temporaire en mémoire (performance)
        - name: cache
          mountPath: /var/cache/nginx

        - name: run
          mountPath: /var/run

        # Ressources
        resources:
          requests:
            memory: "64Mi"
            cpu: "50m"      # 0.05 CPU
          limits:
            memory: "128Mi"
            cpu: "200m"     # 0.2 CPU

        # Health checks
        livenessProbe:
          httpGet:
            path: /health
            port: http
            httpHeaders:
            - name: X-Health-Check
              value: liveness
          initialDelaySeconds: 10
          periodSeconds: 10
          timeoutSeconds: 5
          successThreshold: 1
          failureThreshold: 3

        readinessProbe:
          httpGet:
            path: /health
            port: http
            httpHeaders:
            - name: X-Health-Check
              value: readiness
          initialDelaySeconds: 5
          periodSeconds: 5
          timeoutSeconds: 3
          successThreshold: 1
          failureThreshold: 3

        # Probe de démarrage pour applications lentes
        startupProbe:
          httpGet:
            path: /health
            port: http
          initialDelaySeconds: 0
          periodSeconds: 10
          timeoutSeconds: 3
          successThreshold: 1
          failureThreshold: 30  # 30 * 10 = 5 minutes max pour démarrer

        # Sécurité
        securityContext:
          runAsNonRoot: true
          runAsUser: 101  # User nginx dans Alpine
          allowPrivilegeEscalation: false
          readOnlyRootFilesystem: true
          capabilities:
            drop:
            - ALL
            add:
            - NET_BIND_SERVICE  # Pour écouter sur le port 80

      # Conteneur sidecar pour export de métriques (optionnel)
      - name: nginx-exporter
        image: nginx/nginx-prometheus-exporter:0.10.0
        args:
        - -nginx.scrape-uri=http://localhost/nginx_status
        ports:
        - name: metrics
          containerPort: 9113
        resources:
          requests:
            memory: "16Mi"
            cpu: "10m"
          limits:
            memory: "32Mi"
            cpu: "50m"

      # Volumes
      volumes:
      # ConfigMap monté comme fichiers
      - name: nginx-config
        configMap:
          name: webapp-config
          items:
          - key: nginx.conf
            path: nginx.conf

      - name: webapp-content
        configMap:
          name: webapp-config
          items:
          - key: index.html
            path: index.html

      # Volume persistant
      - name: uploads
        persistentVolumeClaim:
          claimName: webapp-storage

      # Volumes temporaires en mémoire
      - name: cache
        emptyDir:
          medium: Memory
          sizeLimit: 100Mi

      - name: run
        emptyDir:
          medium: Memory
          sizeLimit: 10Mi

      # Configuration DNS personnalisée (optionnel)
      dnsPolicy: ClusterFirst
      dnsConfig:
        options:
        - name: ndots
          value: "2"

      # Redémarrage automatique
      restartPolicy: Always

      # Temps d'arrêt gracieux
      terminationGracePeriodSeconds: 30
```

**Points clés du Deployment :**

1. **Haute disponibilité** : 3 replicas pour la redondance
2. **Rolling updates** : Mise à jour sans interruption
3. **Anti-affinité** : Répartition sur différents nœuds
4. **Health checks complets** : Liveness, readiness, et startup
5. **Sécurité renforcée** : ReadOnly filesystem, non-root user
6. **Monitoring** : Sidecar pour export Prometheus
7. **Volumes mixtes** : ConfigMap, Secret, PVC, emptyDir

## Étape 6 : Exposition avec Service

Créons un Service pour exposer nos Pods de manière stable.

```yaml
# 06-service.yaml
apiVersion: v1
kind: Service
metadata:
  name: webapp
  namespace: webapp-demo
  labels:
    app: webapp
    tier: frontend
  annotations:
    description: "Service principal pour l'application web"
spec:
  type: ClusterIP  # Exposition interne uniquement

  # Sélecteur pour trouver les Pods
  selector:
    app: webapp
    tier: frontend

  # Ports exposés
  ports:
  - name: http
    port: 80        # Port du Service
    targetPort: http # Port du conteneur (nommé)
    protocol: TCP

  - name: metrics
    port: 9113
    targetPort: metrics
    protocol: TCP

  # Session affinity (optionnel)
  # sessionAffinity: ClientIP  # Stick sessions
  # sessionAffinityConfig:
  #   clientIP:
  #     timeoutSeconds: 10800  # 3 heures
```

**Types de Service expliqués :**
- **ClusterIP** : IP interne uniquement (notre choix)
- **NodePort** : Expose sur chaque nœud (port 30000-32767)
- **LoadBalancer** : Demande un LB externe (cloud)
- **ExternalName** : Alias DNS vers service externe

## Étape 7 : Ingress pour l'accès externe

Configurons l'Ingress pour rendre notre application accessible depuis Internet.

```yaml
# 07-ingress.yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: webapp
  namespace: webapp-demo
  labels:
    app: webapp
  annotations:
    # Classe d'Ingress Controller
    kubernetes.io/ingress.class: nginx

    # Cert-Manager pour SSL automatique
    cert-manager.io/cluster-issuer: "letsencrypt-staging"  # ou letsencrypt-prod

    # Configuration NGINX
    nginx.ingress.kubernetes.io/rewrite-target: /
    nginx.ingress.kubernetes.io/ssl-redirect: "true"
    nginx.ingress.kubernetes.io/force-ssl-redirect: "true"

    # Limite de taille pour uploads
    nginx.ingress.kubernetes.io/proxy-body-size: "10m"

    # Timeout personnalisés
    nginx.ingress.kubernetes.io/proxy-connect-timeout: "30"
    nginx.ingress.kubernetes.io/proxy-send-timeout: "30"
    nginx.ingress.kubernetes.io/proxy-read-timeout: "30"

    # Rate limiting
    nginx.ingress.kubernetes.io/limit-rps: "10"
    nginx.ingress.kubernetes.io/limit-connections: "20"

    # Headers de sécurité supplémentaires
    nginx.ingress.kubernetes.io/configuration-snippet: |
      more_set_headers "X-Frame-Options: DENY";
      more_set_headers "X-Content-Type-Options: nosniff";
      more_set_headers "Referrer-Policy: strict-origin-when-cross-origin";
spec:
  # Classe d'Ingress (pour Kubernetes 1.18+)
  ingressClassName: nginx

  # Configuration TLS/SSL
  tls:
  - hosts:
    - webapp.monlab.local
    - www.webapp.monlab.local
    secretName: webapp-tls  # Cert-Manager créera ce secret

  # Règles de routage
  rules:
  # Domaine principal
  - host: webapp.monlab.local
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: webapp
            port:
              number: 80

  # Redirection www
  - host: www.webapp.monlab.local
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: webapp
            port:
              number: 80

  # Catch-all pour IP directe (optionnel)
  - http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: webapp
            port:
              number: 80
```

## Étape 8 : Déploiement complet

### Ordre de déploiement

L'ordre est important pour éviter les erreurs de dépendances :

```bash
# 1. Créer le namespace
kubectl apply -f 01-namespace.yaml

# 2. Créer la configuration et les secrets
kubectl apply -f 02-configmap.yaml
kubectl apply -f 03-secret.yaml

# 3. Créer le stockage
kubectl apply -f 04-persistentvolume.yaml

# 4. Déployer l'application
kubectl apply -f 05-deployment.yaml

# 5. Exposer avec le Service
kubectl apply -f 06-service.yaml

# 6. Configurer l'accès externe
kubectl apply -f 07-ingress.yaml
```

### Ou tout d'un coup avec Kustomize

Créez un fichier `kustomization.yaml` :

```yaml
# kustomization.yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

namespace: webapp-demo

resources:
  - 01-namespace.yaml
  - 02-configmap.yaml
  - 03-secret.yaml
  - 04-persistentvolume.yaml
  - 05-deployment.yaml
  - 06-service.yaml
  - 07-ingress.yaml

# Labels communs à toutes les ressources
commonLabels:
  app: webapp
  managed-by: kustomize

# Annotations communes
commonAnnotations:
  version: "1.0.0"
  team: "devops"
```

Puis déployez avec :
```bash
kubectl apply -k .
```

## Vérification du déploiement

### Commandes de vérification essentielles

```bash
# Vue d'ensemble du namespace
kubectl get all -n webapp-demo

# Détails du Deployment
kubectl describe deployment webapp -n webapp-demo

# Status des Pods
kubectl get pods -n webapp-demo -o wide

# Logs d'un Pod
kubectl logs -n webapp-demo deployment/webapp

# Logs en temps réel
kubectl logs -f -n webapp-demo deployment/webapp

# Vérifier le Service
kubectl get service webapp -n webapp-demo
kubectl describe service webapp -n webapp-demo

# Vérifier l'Ingress
kubectl get ingress -n webapp-demo
kubectl describe ingress webapp -n webapp-demo

# Vérifier le stockage
kubectl get pvc -n webapp-demo
kubectl describe pvc webapp-storage -n webapp-demo

# ConfigMap et Secret
kubectl get configmap,secret -n webapp-demo
```

### Tests de connectivité

```bash
# Test interne au cluster
kubectl run test-pod --rm -i --tty --image=alpine --namespace=webapp-demo -- sh
# Dans le pod:
wget -O- http://webapp/health
exit

# Port-forward pour test local
kubectl port-forward -n webapp-demo service/webapp 8080:80
# Puis ouvrir http://localhost:8080

# Test via Ingress (après configuration DNS)
curl -k https://webapp.monlab.local
```

### Configuration DNS locale

Pour accéder via le nom de domaine, ajoutez dans `/etc/hosts` :
```bash
# IP de votre nœud MicroK8s
192.168.1.100  webapp.monlab.local www.webapp.monlab.local
```

Ou configurez votre DNS local pour résoudre `*.monlab.local` vers votre cluster.

## Mise à jour de l'application

### Mise à jour de la configuration

```bash
# Modifier le ConfigMap
kubectl edit configmap webapp-config -n webapp-demo

# Redémarrer les Pods pour prendre en compte les changements
kubectl rollout restart deployment webapp -n webapp-demo

# Suivre le rollout
kubectl rollout status deployment webapp -n webapp-demo
```

### Mise à jour de l'image

```bash
# Changer l'image
kubectl set image deployment/webapp nginx=nginx:1.22-alpine -n webapp-demo

# Ou éditer le Deployment
kubectl edit deployment webapp -n webapp-demo

# Vérifier l'historique
kubectl rollout history deployment webapp -n webapp-demo

# Rollback si nécessaire
kubectl rollout undo deployment webapp -n webapp-demo
```

### Scaling

```bash
# Augmenter le nombre de replicas
kubectl scale deployment webapp --replicas=5 -n webapp-demo

# Ou avec autoscaling
kubectl autoscale deployment webapp --min=2 --max=10 --cpu-percent=80 -n webapp-demo
```

## Monitoring et debugging

### Métriques et performances

```bash
# Utilisation des ressources
kubectl top pods -n webapp-demo
kubectl top nodes

# Events du namespace
kubectl get events -n webapp-demo --sort-by='.lastTimestamp'

# Décrire un Pod problématique
kubectl describe pod <pod-name> -n webapp-demo
```

### Debugging avancé

```bash
# Accès shell dans un conteneur
kubectl exec -it <pod-name> -n webapp-demo -- /bin/sh

# Copier des fichiers depuis/vers un Pod
kubectl cp -n webapp-demo <pod-name>:/usr/share/nginx/html/index.html ./index-backup.html
kubectl cp -n webapp-demo ./new-index.html <pod-name>:/usr/share/nginx/html/index.html

# Debug avec un conteneur temporaire
kubectl debug <pod-name> -n webapp-demo -it --image=busybox
```

### Dashboard Kubernetes

Si vous avez activé le dashboard MicroK8s :
```bash
# Activer le dashboard
microk8s enable dashboard

# Obtenir le token
token=$(microk8s kubectl -n kube-system get secret | grep default-token | cut -d " " -f1)
microk8s kubectl -n kube-system describe secret $token

# Proxy
microk8s kubectl proxy

# Accéder à http://localhost:8001/api/v1/namespaces/kube-system/services/https:kubernetes-dashboard:/proxy/
```

## Nettoyage

Pour supprimer complètement l'application :

```bash
# Méthode 1 : Supprimer le namespace (supprime tout)
kubectl delete namespace webapp-demo

# Méthode 2 : Supprimer ressource par ressource
kubectl delete -f 07-ingress.yaml
kubectl delete -f 06-service.yaml
kubectl delete -f 05-deployment.yaml
kubectl delete -f 04-persistentvolume.yaml
kubectl delete -f 03-secret.yaml
kubectl delete -f 02-configmap.yaml
kubectl delete -f 01-namespace.yaml

# Méthode 3 : Si vous avez utilisé Kustomize
kubectl delete -k .

# Vérifier que tout est supprimé
kubectl get all -n webapp-demo
```

## Optimisations et bonnes pratiques

### 1. Optimisation des images Docker

Pour une application de production, créez votre propre image optimisée :

```dockerfile
# Dockerfile optimisé
FROM nginx:1.21-alpine AS production

# Installer les outils nécessaires
RUN apk add --no-cache curl

# Copier la configuration
COPY nginx.conf /etc/nginx/conf.d/default.conf

# Copier le contenu statique
COPY ./static /usr/share/nginx/html

# Health check intégré
HEALTHCHECK --interval=30s --timeout=3s --start-period=5s --retries=3 \
  CMD curl -f http://localhost/health || exit 1

# User non-root
USER nginx

EXPOSE 80

CMD ["nginx", "-g", "daemon off;"]
```

### 2. Configuration multi-environnements

Structurez vos manifestes pour plusieurs environnements :

```
webapp/
├── base/
│   ├── deployment.yaml
│   ├── service.yaml
│   ├── configmap.yaml
│   └── kustomization.yaml
├── overlays/
│   ├── development/
│   │   ├── kustomization.yaml
│   │   ├── deployment-patch.yaml
│   │   └── configmap-patch.yaml
│   ├── staging/
│   │   ├── kustomization.yaml
│   │   ├── deployment-patch.yaml
│   │   └── ingress.yaml
│   └── production/
│       ├── kustomization.yaml
│       ├── deployment-patch.yaml
│       ├── resources-patch.yaml
│       └── ingress.yaml
```

Exemple de patch pour production :
```yaml
# overlays/production/deployment-patch.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: webapp
spec:
  replicas: 10  # Plus de replicas en production
  template:
    spec:
      containers:
      - name: nginx
        resources:
          requests:
            memory: "256Mi"
            cpu: "500m"
          limits:
            memory: "512Mi"
            cpu: "1000m"
```

### 3. Gestion des secrets avec Sealed Secrets

Pour versionner les secrets de manière sécurisée :

```bash
# Installer Sealed Secrets
kubectl apply -f https://github.com/bitnami-labs/sealed-secrets/releases/download/v0.18.0/controller.yaml

# Créer un secret scellé
echo -n "mypassword" | kubectl create secret generic webapp-secret \
  --dry-run=client \
  --from-file=password=/dev/stdin \
  -o yaml | kubeseal -o yaml > sealed-secret.yaml
```

### 4. Monitoring avec Prometheus

Configurez le scraping Prometheus :

```yaml
# servicemonitor.yaml (si vous utilisez Prometheus Operator)
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: webapp
  namespace: webapp-demo
spec:
  selector:
    matchLabels:
      app: webapp
  endpoints:
  - port: metrics
    interval: 30s
    path: /metrics
```

### 5. Backup automatique

Script de backup pour vos configurations :

```bash
#!/bin/bash
# backup-webapp.sh

BACKUP_DIR="./backups/$(date +%Y%m%d_%H%M%S)"
NAMESPACE="webapp-demo"

mkdir -p $BACKUP_DIR

# Exporter toutes les ressources
for resource in deployment service configmap secret ingress pvc; do
  kubectl get $resource -n $NAMESPACE -o yaml > $BACKUP_DIR/${resource}.yaml
done

# Créer une archive
tar -czf backups/webapp-backup-$(date +%Y%m%d_%H%M%S).tar.gz $BACKUP_DIR

echo "Backup créé dans $BACKUP_DIR"
```

## Troubleshooting courant

### Problème : Les Pods ne démarrent pas

**Symptômes :** Pods en état `Pending`, `CrashLoopBackOff`, ou `ImagePullBackOff`

**Diagnostic :**
```bash
# Vérifier les events
kubectl describe pod <pod-name> -n webapp-demo

# Vérifier les logs
kubectl logs <pod-name> -n webapp-demo --previous

# Vérifier les ressources disponibles
kubectl describe nodes
kubectl top nodes
```

**Solutions communes :**
- `ImagePullBackOff` : Vérifier le nom de l'image et les credentials
- `CrashLoopBackOff` : Vérifier les logs, health checks trop stricts
- `Pending` : Pas assez de ressources, PVC non bound

### Problème : Service non accessible

**Diagnostic :**
```bash
# Vérifier les endpoints
kubectl get endpoints webapp -n webapp-demo

# Test depuis un Pod
kubectl run test --rm -it --image=busybox --namespace=webapp-demo -- wget -O- http://webapp/health

# Vérifier les labels
kubectl get pods -n webapp-demo --show-labels
kubectl get service webapp -n webapp-demo -o yaml | grep selector -A 2
```

**Solutions :**
- Vérifier que les labels des Pods correspondent au selector du Service
- Vérifier que les Pods sont en état `Running` et `Ready`

### Problème : Ingress ne fonctionne pas

**Diagnostic :**
```bash
# Vérifier l'Ingress Controller
kubectl get pods -n ingress

# Vérifier la configuration Ingress
kubectl describe ingress webapp -n webapp-demo

# Logs de l'Ingress Controller
kubectl logs -n ingress deployment/nginx-ingress-microk8s-controller
```

**Solutions :**
- Vérifier que l'addon ingress est activé : `microk8s enable ingress`
- Vérifier la résolution DNS
- Vérifier les annotations de l'Ingress

### Problème : Stockage persistant non accessible

**Diagnostic :**
```bash
# Vérifier le PVC
kubectl describe pvc webapp-storage -n webapp-demo

# Vérifier le PV
kubectl get pv
kubectl describe pv <pv-name>
```

**Solutions :**
- Vérifier que l'addon storage est activé : `microk8s enable storage`
- Vérifier les accessModes correspondent
- Vérifier l'espace disque disponible

## Évolutions possibles

### 1. Ajouter une base de données

Exemple avec PostgreSQL :
```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: postgres
  namespace: webapp-demo
spec:
  serviceName: postgres
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
        image: postgres:14-alpine
        env:
        - name: POSTGRES_DB
          value: webapp
        - name: POSTGRES_PASSWORD
          valueFrom:
            secretKeyRef:
              name: webapp-secret
              key: database_password
        volumeMounts:
        - name: postgres-storage
          mountPath: /var/lib/postgresql/data
  volumeClaimTemplates:
  - metadata:
      name: postgres-storage
    spec:
      accessModes: ["ReadWriteOnce"]
      resources:
        requests:
          storage: 10Gi
```

### 2. Ajouter du caching avec Redis

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: redis
  namespace: webapp-demo
spec:
  replicas: 1
  selector:
    matchLabels:
      app: redis
  template:
    metadata:
      labels:
        app: redis
    spec:
      containers:
      - name: redis
        image: redis:7-alpine
        ports:
        - containerPort: 6379
        resources:
          requests:
            memory: "128Mi"
            cpu: "100m"
          limits:
            memory: "256Mi"
            cpu: "500m"
---
apiVersion: v1
kind: Service
metadata:
  name: redis
  namespace: webapp-demo
spec:
  selector:
    app: redis
  ports:
  - port: 6379
    targetPort: 6379
```

### 3. Ajouter une API backend

Créez un second Deployment pour votre API :
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: api
  namespace: webapp-demo
spec:
  replicas: 3
  selector:
    matchLabels:
      app: api
  template:
    metadata:
      labels:
        app: api
    spec:
      containers:
      - name: api
        image: myregistry/api:v1.0.0
        ports:
        - containerPort: 3000
        env:
        - name: DATABASE_URL
          valueFrom:
            secretKeyRef:
              name: webapp-secret
              key: database_url
        - name: REDIS_URL
          value: "redis://redis:6379"
```

Puis modifiez l'Ingress pour router `/api` vers ce service :
```yaml
spec:
  rules:
  - host: webapp.monlab.local
    http:
      paths:
      - path: /api
        pathType: Prefix
        backend:
          service:
            name: api
            port:
              number: 3000
      - path: /
        pathType: Prefix
        backend:
          service:
            name: webapp
            port:
              number: 80
```

### 4. Ajouter l'autoscaling

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: webapp-hpa
  namespace: webapp-demo
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: webapp
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
  behavior:
    scaleDown:
      stabilizationWindowSeconds: 300
      policies:
      - type: Percent
        value: 50
        periodSeconds: 60
    scaleUp:
      stabilizationWindowSeconds: 0
      policies:
      - type: Percent
        value: 100
        periodSeconds: 60
```

## Points clés à retenir

1. **Déploiement progressif** : Commencez simple, ajoutez des fonctionnalités progressivement

2. **Configuration externalisée** : Utilisez ConfigMaps et Secrets, jamais de hardcoding

3. **Health checks essentiels** : Toujours définir liveness et readiness probes

4. **Ressources définies** : Spécifiez toujours requests et limits

5. **Labels cohérents** : Utilisez un système de labels uniforme

6. **Monitoring intégré** : Préparez vos applications pour l'observabilité

7. **Sécurité par défaut** : ReadOnly filesystem, non-root, capabilities minimales

8. **Documentation** : Commentez vos manifestes, utilisez les annotations

9. **Versionning** : Utilisez Git pour tous vos manifestes

10. **Tests** : Testez les déploiements, les rollbacks, les montées en charge

## Conclusion

Vous avez maintenant déployé une application web complète sur MicroK8s avec :
- ✅ Configuration externalisée (ConfigMap)
- ✅ Secrets sécurisés
- ✅ Stockage persistant
- ✅ Haute disponibilité (3 replicas)
- ✅ Load balancing automatique
- ✅ Exposition HTTPS avec Ingress
- ✅ Health checks et monitoring
- ✅ Sécurité renforcée

Cette base solide peut être étendue avec des bases de données, du caching, des APIs, et bien plus. L'important est de comprendre comment chaque composant s'articule et contribue à créer une application robuste et scalable.

Dans les prochaines sections, nous explorerons comment exposer cette application via Service et Ingress (7.4), gérer la configuration avec ConfigMaps (7.5), sécuriser les données sensibles avec Secrets (7.6), et implémenter le stockage persistant (7.7).

N'oubliez pas : dans votre lab personnel, n'hésitez pas à expérimenter, casser, et reconstruire. C'est en pratiquant que vous maîtriserez vraiment Kubernetes !

⏭️
