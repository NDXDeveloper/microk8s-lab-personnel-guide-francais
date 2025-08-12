🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 7.4 - Exposition via Service et Ingress

## Introduction

Votre application tourne dans des Pods, mais comment la rendre accessible ? C'est là qu'interviennent les Services et Ingress. Pensez à votre application comme à un restaurant :
- Les **Pods** sont les cuisines où la nourriture est préparée
- Le **Service** est le système de service en salle qui distribue les plats
- L'**Ingress** est la porte d'entrée avec l'hôte d'accueil qui oriente les clients

Dans cette section, nous allons explorer en détail comment exposer vos applications, depuis l'accès interne entre Pods jusqu'à l'exposition publique sur Internet avec HTTPS.

## Vue d'ensemble de l'exposition réseau

### Le problème fondamental

Les Pods dans Kubernetes ont plusieurs caractéristiques qui rendent l'exposition directe problématique :

1. **Éphémères** : Les Pods peuvent mourir et renaître à tout moment
2. **IPs dynamiques** : Chaque nouveau Pod obtient une nouvelle IP
3. **Multiples instances** : Vous avez souvent plusieurs replicas du même Pod
4. **Isolation** : Par défaut, les Pods ne sont pas accessibles depuis l'extérieur

### La solution Kubernetes

Kubernetes résout ces problèmes avec une architecture en couches :

```
Internet
    ↓
[Ingress Controller] - Couche 7 (HTTP/HTTPS)
    ↓
[Service] - Couche 4 (TCP/UDP)
    ↓
[Pods] - Vos applications
```

Chaque couche a un rôle spécifique :
- **Service** : Fournit une IP stable et du load balancing interne
- **Ingress** : Ajoute le routage HTTP/HTTPS et les fonctionnalités web

## Partie 1 : Les Services en détail

### Qu'est-ce qu'un Service ?

Un Service Kubernetes est un objet qui :
- Fournit une **IP virtuelle stable** (ClusterIP)
- Offre un **nom DNS** interne
- Fait du **load balancing** automatique
- **Découvre** automatiquement les Pods via les labels

### Types de Services

#### 1. ClusterIP (par défaut)

Le type le plus basique, accessible uniquement depuis l'intérieur du cluster.

```yaml
apiVersion: v1
kind: Service
metadata:
  name: backend-service
  namespace: my-app
  labels:
    app: backend
    tier: service
spec:
  type: ClusterIP  # Type par défaut, peut être omis
  selector:
    app: backend    # Sélectionne les Pods avec ce label
    tier: api
  ports:
  - name: http
    port: 80        # Port du Service
    targetPort: 8080 # Port du conteneur
    protocol: TCP
```

**Utilisation typique :**
- Communication entre microservices
- Bases de données internes
- Services de cache (Redis, Memcached)

**Accès au service :**
```bash
# Depuis un Pod dans le même namespace
curl http://backend-service

# Depuis un Pod dans un autre namespace
curl http://backend-service.my-app.svc.cluster.local

# Format DNS complet
<service-name>.<namespace>.svc.cluster.local
```

#### 2. NodePort

Expose le Service sur un port statique de chaque nœud (30000-32767).

```yaml
apiVersion: v1
kind: Service
metadata:
  name: web-nodeport
  namespace: my-app
spec:
  type: NodePort
  selector:
    app: web
  ports:
  - name: http
    port: 80         # Port du Service (interne)
    targetPort: 8080  # Port du conteneur
    nodePort: 30080   # Port sur le nœud (optionnel, auto-assigné si omis)
    protocol: TCP
```

**Flux du trafic :**
```
Client → Node IP:30080 → Service → Pod:8080
```

**Utilisation typique :**
- Développement local
- Clusters on-premise sans load balancer
- Accès d'urgence/debug

**Accès :**
```bash
# Depuis l'extérieur du cluster
curl http://<node-ip>:30080

# Fonctionne sur n'importe quel nœud du cluster
curl http://node1.example.com:30080
curl http://node2.example.com:30080
```

#### 3. LoadBalancer

Demande un load balancer externe (cloud provider ou MetalLB pour on-premise).

```yaml
apiVersion: v1
kind: Service
metadata:
  name: web-loadbalancer
  namespace: my-app
  annotations:
    # Annotations spécifiques au cloud provider
    service.beta.kubernetes.io/aws-load-balancer-type: "nlb"
spec:
  type: LoadBalancer
  selector:
    app: web
  ports:
  - name: http
    port: 80
    targetPort: 8080
    protocol: TCP
  - name: https
    port: 443
    targetPort: 8443
    protocol: TCP
  # Optionnel : restreindre les IPs sources
  loadBalancerSourceRanges:
  - 10.0.0.0/8
  - 192.168.1.0/24
```

**Configuration MetalLB pour MicroK8s :**
```bash
# Activer MetalLB
microk8s enable metallb:10.64.140.43-10.64.140.49

# Le Service recevra une IP de cette plage
```

**Utilisation typique :**
- Production sur cloud (AWS, GCP, Azure)
- Exposition publique simple
- Services nécessitant une IP dédiée

#### 4. ExternalName

Crée un alias DNS vers un service externe.

```yaml
apiVersion: v1
kind: Service
metadata:
  name: external-database
  namespace: my-app
spec:
  type: ExternalName
  externalName: database.example.com
  ports:
  - port: 5432
```

**Utilisation :**
```bash
# Les Pods peuvent utiliser
postgresql://external-database:5432/mydb

# Qui résout vers
postgresql://database.example.com:5432/mydb
```

**Cas d'usage :**
- Migration progressive vers Kubernetes
- Services externes (bases de données managées)
- APIs tierces

### Configuration avancée des Services

#### Session Affinity (Sticky Sessions)

Pour router toujours un client vers le même Pod :

```yaml
apiVersion: v1
kind: Service
metadata:
  name: stateful-app
spec:
  selector:
    app: stateful
  sessionAffinity: ClientIP  # ou None (défaut)
  sessionAffinityConfig:
    clientIP:
      timeoutSeconds: 10800  # 3 heures
  ports:
  - port: 80
    targetPort: 8080
```

**Quand l'utiliser :**
- Applications avec état de session
- WebSockets
- Uploads de fichiers volumineux

#### Multi-port Services

Un Service peut exposer plusieurs ports :

```yaml
apiVersion: v1
kind: Service
metadata:
  name: multi-port-service
spec:
  selector:
    app: complex-app
  ports:
  - name: http
    port: 80
    targetPort: 8080
    protocol: TCP
  - name: https
    port: 443
    targetPort: 8443
    protocol: TCP
  - name: grpc
    port: 9090
    targetPort: 9090
    protocol: TCP
  - name: metrics
    port: 9191
    targetPort: 9191
    protocol: TCP
```

#### Headless Services

Un Service sans ClusterIP pour accès direct aux Pods :

```yaml
apiVersion: v1
kind: Service
metadata:
  name: headless-service
spec:
  clusterIP: None  # Pas de VIP
  selector:
    app: stateful-app
  ports:
  - port: 9042
    targetPort: 9042
```

**DNS retourne toutes les IPs des Pods :**
```bash
nslookup headless-service
# Retourne :
# 10.1.0.5
# 10.1.0.6
# 10.1.0.7
```

**Utilisation :**
- StatefulSets (Cassandra, Elasticsearch)
- Service discovery personnalisé
- Applications nécessitant une connexion directe aux Pods

### Endpoints : Le lien Service-Pods

Les Endpoints sont automatiquement créés et gérés par Kubernetes :

```bash
# Voir les endpoints d'un Service
kubectl get endpoints web-service -n my-app

# Sortie exemple :
NAME          ENDPOINTS                                   AGE
web-service   10.1.0.5:8080,10.1.0.6:8080,10.1.0.7:8080   5m
```

**Endpoints manuels** pour services externes :

```yaml
# Service sans selector
apiVersion: v1
kind: Service
metadata:
  name: external-service
spec:
  ports:
  - port: 80
    targetPort: 80
---
# Endpoints manuels
apiVersion: v1
kind: Endpoints
metadata:
  name: external-service  # Même nom que le Service
subsets:
- addresses:
  - ip: 192.168.1.100
  - ip: 192.168.1.101
  ports:
  - port: 80
```

## Partie 2 : Ingress en détail

### Qu'est-ce qu'un Ingress ?

L'Ingress est une API qui gère l'accès HTTP/HTTPS externe vers les Services. C'est comme un reverse proxy intelligent qui :
- Route le trafic basé sur les domaines et chemins
- Termine le SSL/TLS
- Fournit des fonctionnalités HTTP avancées
- Centralise la configuration d'accès

### Architecture Ingress

```
                Internet
                    |
            [Load Balancer]
                    |
          [Ingress Controller]
          /         |         \
    [Service A] [Service B] [Service C]
        |           |           |
    [Pods A]    [Pods B]    [Pods C]
```

### Ingress Controller

L'Ingress est juste une configuration. Vous avez besoin d'un **Ingress Controller** pour l'implémenter.

**Options populaires :**
- **NGINX** : Le plus utilisé, robuste
- **Traefik** : Moderne, configuration dynamique
- **HAProxy** : Haute performance
- **Contour** : Basé sur Envoy
- **Kong** : API Gateway complet

**Installation NGINX Ingress sur MicroK8s :**
```bash
# Activer l'addon ingress (NGINX)
microk8s enable ingress

# Vérifier l'installation
kubectl get pods -n ingress
```

### Configuration Ingress de base

#### Routage par nom d'hôte

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: virtual-host-ingress
  namespace: my-app
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
spec:
  ingressClassName: nginx  # Classe d'Ingress à utiliser
  rules:
  # Premier domaine
  - host: app1.example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: app1-service
            port:
              number: 80

  # Deuxième domaine
  - host: app2.example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: app2-service
            port:
              number: 80

  # Domaine par défaut (sans host)
  - http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: default-service
            port:
              number: 80
```

#### Routage par chemin

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: path-based-ingress
  namespace: my-app
  annotations:
    nginx.ingress.kubernetes.io/use-regex: "true"
spec:
  ingressClassName: nginx
  rules:
  - host: api.example.com
    http:
      paths:
      # API v1
      - path: /v1
        pathType: Prefix
        backend:
          service:
            name: api-v1
            port:
              number: 8080

      # API v2
      - path: /v2
        pathType: Prefix
        backend:
          service:
            name: api-v2
            port:
              number: 8080

      # Frontend
      - path: /app
        pathType: Prefix
        backend:
          service:
            name: frontend
            port:
              number: 3000

      # Documentation
      - path: /docs
        pathType: Prefix
        backend:
          service:
            name: docs
            port:
              number: 80

      # Tout le reste vers le service par défaut
      - path: /
        pathType: Prefix
        backend:
          service:
            name: default-backend
            port:
              number: 80
```

#### Types de chemins

```yaml
# PathType: Exact - Correspondance exacte
- path: /api/v1/users
  pathType: Exact
  # Matche : /api/v1/users
  # Ne matche pas : /api/v1/users/

# PathType: Prefix - Préfixe
- path: /api
  pathType: Prefix
  # Matche : /api, /api/, /api/v1, /api/v1/users
  # Ne matche pas : /ap, /apis

# PathType: ImplementationSpecific - Dépend du controller
- path: /api/.*
  pathType: ImplementationSpecific
  # Comportement défini par l'Ingress Controller
```

### Configuration HTTPS/TLS

#### Avec certificats manuels

```yaml
# D'abord, créer le Secret TLS
apiVersion: v1
kind: Secret
metadata:
  name: tls-secret
  namespace: my-app
type: kubernetes.io/tls
data:
  tls.crt: |
    LS0tLS1CRUdJTi... # Certificat en base64
  tls.key: |
    LS0tLS1CRUdJTi... # Clé privée en base64
---
# Ou créer depuis des fichiers
# kubectl create secret tls tls-secret \
#   --cert=path/to/cert.crt \
#   --key=path/to/cert.key \
#   -n my-app
---
# Ingress avec TLS
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: tls-ingress
  namespace: my-app
  annotations:
    nginx.ingress.kubernetes.io/ssl-redirect: "true"
    nginx.ingress.kubernetes.io/force-ssl-redirect: "true"
spec:
  ingressClassName: nginx
  tls:
  - hosts:
    - secure.example.com
    - www.secure.example.com
    secretName: tls-secret
  rules:
  - host: secure.example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: secure-app
            port:
              number: 80
```

#### Avec Cert-Manager (recommandé)

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: letsencrypt-ingress
  namespace: my-app
  annotations:
    # Cert-Manager annotations
    cert-manager.io/cluster-issuer: "letsencrypt-prod"
    cert-manager.io/acme-challenge-type: http01

    # NGINX annotations
    nginx.ingress.kubernetes.io/ssl-redirect: "true"
spec:
  ingressClassName: nginx
  tls:
  - hosts:
    - app.example.com
    secretName: app-tls  # Cert-Manager créera ce Secret
  rules:
  - host: app.example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: app-service
            port:
              number: 80
```

### Annotations Ingress avancées

Les annotations permettent de configurer des comportements spécifiques :

#### Redirections et rewrites

```yaml
metadata:
  annotations:
    # Redirection HTTP vers HTTPS
    nginx.ingress.kubernetes.io/ssl-redirect: "true"

    # Redirection permanente
    nginx.ingress.kubernetes.io/permanent-redirect: "https://new-site.com"

    # Réécriture d'URL
    nginx.ingress.kubernetes.io/rewrite-target: /$2
    # Avec path: /api(/|$)(.*)
    # /api/v1/users → /v1/users

    # Ajout/suppression de www
    nginx.ingress.kubernetes.io/from-to-www-redirect: "true"
```

#### Limites et timeouts

```yaml
metadata:
  annotations:
    # Taille maximale du body
    nginx.ingress.kubernetes.io/proxy-body-size: "50m"

    # Timeouts
    nginx.ingress.kubernetes.io/proxy-connect-timeout: "600"
    nginx.ingress.kubernetes.io/proxy-send-timeout: "600"
    nginx.ingress.kubernetes.io/proxy-read-timeout: "600"

    # Rate limiting
    nginx.ingress.kubernetes.io/limit-rps: "10"
    nginx.ingress.kubernetes.io/limit-rpm: "300"
    nginx.ingress.kubernetes.io/limit-connections: "5"

    # Limite par IP
    nginx.ingress.kubernetes.io/limit-whitelist: "10.0.0.0/8,192.168.0.0/16"
```

#### Authentification

```yaml
metadata:
  annotations:
    # Basic auth
    nginx.ingress.kubernetes.io/auth-type: basic
    nginx.ingress.kubernetes.io/auth-secret: basic-auth
    nginx.ingress.kubernetes.io/auth-realm: "Authentication Required"

    # OAuth
    nginx.ingress.kubernetes.io/auth-url: "https://auth.example.com/oauth2/auth"
    nginx.ingress.kubernetes.io/auth-signin: "https://auth.example.com/oauth2/start"
```

#### CORS

```yaml
metadata:
  annotations:
    nginx.ingress.kubernetes.io/enable-cors: "true"
    nginx.ingress.kubernetes.io/cors-allow-origin: "*"
    nginx.ingress.kubernetes.io/cors-allow-methods: "GET, POST, PUT, DELETE, OPTIONS"
    nginx.ingress.kubernetes.io/cors-allow-headers: "DNT,X-CustomHeader,Keep-Alive,User-Agent"
    nginx.ingress.kubernetes.io/cors-expose-headers: "X-Custom-Header"
    nginx.ingress.kubernetes.io/cors-allow-credentials: "true"
    nginx.ingress.kubernetes.io/cors-max-age: "86400"
```

#### Headers personnalisés

```yaml
metadata:
  annotations:
    # Ajouter des headers de réponse
    nginx.ingress.kubernetes.io/configuration-snippet: |
      more_set_headers "X-Frame-Options: SAMEORIGIN";
      more_set_headers "X-Content-Type-Options: nosniff";
      more_set_headers "X-XSS-Protection: 1; mode=block";
      more_set_headers "Referrer-Policy: strict-origin-when-cross-origin";

    # Headers de requête vers le backend
    nginx.ingress.kubernetes.io/proxy-set-headers: "my-app/custom-headers"
```

### Patterns d'exposition avancés

#### 1. Canary Deployments

Déploiement progressif avec routage pondéré :

```yaml
# Ingress principal (90% du trafic)
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: production-ingress
  namespace: my-app
spec:
  rules:
  - host: app.example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: app-stable
            port:
              number: 80
---
# Ingress canary (10% du trafic)
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: canary-ingress
  namespace: my-app
  annotations:
    nginx.ingress.kubernetes.io/canary: "true"
    nginx.ingress.kubernetes.io/canary-weight: "10"
spec:
  rules:
  - host: app.example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: app-canary
            port:
              number: 80
```

#### 2. Blue-Green Deployment

Basculement entre deux versions :

```yaml
# Service pour Blue (v1)
apiVersion: v1
kind: Service
metadata:
  name: app-blue
spec:
  selector:
    app: myapp
    version: v1
  ports:
  - port: 80
---
# Service pour Green (v2)
apiVersion: v1
kind: Service
metadata:
  name: app-green
spec:
  selector:
    app: myapp
    version: v2
  ports:
  - port: 80
---
# Ingress pointant vers la version active
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: app-ingress
spec:
  rules:
  - host: app.example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: app-blue  # Changer vers app-green pour basculer
            port:
              number: 80
```

#### 3. A/B Testing

Routage basé sur les headers ou cookies :

```yaml
# Version A (par défaut)
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: app-version-a
spec:
  rules:
  - host: app.example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: app-v1
            port:
              number: 80
---
# Version B (utilisateurs avec header spécifique)
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: app-version-b
  annotations:
    nginx.ingress.kubernetes.io/canary: "true"
    nginx.ingress.kubernetes.io/canary-by-header: "x-version"
    nginx.ingress.kubernetes.io/canary-by-header-value: "beta"
spec:
  rules:
  - host: app.example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: app-v2
            port:
              number: 80
```

## Partie 3 : Flux complet d'exposition

### Exemple complet : Application e-commerce

Créons une exposition complète pour une application e-commerce avec :
- Frontend public
- API backend
- Admin panel
- Webhooks

```yaml
# 1. Services pour chaque composant
---
apiVersion: v1
kind: Service
metadata:
  name: frontend
  namespace: ecommerce
spec:
  selector:
    app: frontend
  ports:
  - port: 80
    targetPort: 3000
---
apiVersion: v1
kind: Service
metadata:
  name: api
  namespace: ecommerce
spec:
  selector:
    app: api
  ports:
  - port: 80
    targetPort: 8080
---
apiVersion: v1
kind: Service
metadata:
  name: admin
  namespace: ecommerce
spec:
  selector:
    app: admin
  ports:
  - port: 80
    targetPort: 4000
---
apiVersion: v1
kind: Service
metadata:
  name: webhooks
  namespace: ecommerce
spec:
  selector:
    app: webhooks
  ports:
  - port: 80
    targetPort: 9000
---
# 2. Ingress avec routage complexe
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: ecommerce-ingress
  namespace: ecommerce
  annotations:
    cert-manager.io/cluster-issuer: "letsencrypt-prod"
    nginx.ingress.kubernetes.io/ssl-redirect: "true"

    # Rate limiting global
    nginx.ingress.kubernetes.io/limit-rps: "100"

    # Configuration CORS pour l'API
    nginx.ingress.kubernetes.io/enable-cors: "true"
    nginx.ingress.kubernetes.io/cors-allow-origin: "https://shop.example.com"

    # Sécurité
    nginx.ingress.kubernetes.io/configuration-snippet: |
      more_set_headers "X-Frame-Options: DENY";
      more_set_headers "Content-Security-Policy: default-src 'self'";
      more_set_headers "X-Content-Type-Options: nosniff";
spec:
  ingressClassName: nginx
  tls:
  - hosts:
    - shop.example.com
    - api.example.com
    - admin.example.com
    secretName: ecommerce-tls
  rules:

  # Frontend public
  - host: shop.example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: frontend
            port:
              number: 80

  # API avec versioning
  - host: api.example.com
    http:
      paths:
      # API v1
      - path: /v1
        pathType: Prefix
        backend:
          service:
            name: api
            port:
              number: 80

      # Webhooks
      - path: /webhooks
        pathType: Prefix
        backend:
          service:
            name: webhooks
            port:
              number: 80

      # Documentation API
      - path: /docs
        pathType: Prefix
        backend:
          service:
            name: api
            port:
              number: 80

  # Admin avec authentification
  - host: admin.example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: admin
            port:
              number: 80
---
# 3. Ingress séparé pour l'admin avec auth
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: admin-auth-ingress
  namespace: ecommerce
  annotations:
    nginx.ingress.kubernetes.io/auth-type: basic
    nginx.ingress.kubernetes.io/auth-secret: admin-auth
    nginx.ingress.kubernetes.io/auth-realm: "Admin Authentication"

    # IP whitelist pour l'admin
    nginx.ingress.kubernetes.io/whitelist-source-range: "10.0.0.0/8,192.168.1.0/24"
spec:
  ingressClassName: nginx
  tls:
  - hosts:
    - admin.example.com
    secretName: ecommerce-tls
  rules:
  - host: admin.example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: admin
            port:
              number: 80
```

## Monitoring et debugging

### Commandes de diagnostic

```bash
# === Services ===
# Lister tous les Services
kubectl get svc -A

# Détails d'un Service
kubectl describe svc my-service -n my-app

# Voir les Endpoints (Pods ciblés)
kubectl get endpoints my-service -n my-app -o yaml

# Tester la résolution DNS
kubectl run test --rm -it --image=busybox -- nslookup my-service.my-app

# === Ingress ===
# Lister tous les Ingress
kubectl get ingress -A

# Détails d'un Ingress
kubectl describe ingress my-ingress -n my-app

# Voir la configuration générée
kubectl get ingress my-ingress -n my-app -o yaml

# Logs de l'Ingress Controller
kubectl logs -n ingress deployment/nginx-ingress-microk8s-controller -f
```

### Tests de connectivité

```bash
# Test depuis l'intérieur du cluster
kubectl run curl-test --rm -it --image=curlimages/curl -- sh
# Dans le pod:
curl http://my-service.my-app.svc.cluster.local
curl -H "Host: app.example.com" http://ingress-nginx-controller.ingress

# Port-forward pour test local
kubectl port-forward svc/my-service 8080:80 -n my-app
# Puis: curl http://localhost:8080

# Test Ingress avec hosts file
# Ajouter dans /etc/hosts:
# <node-ip> app.example.com
curl https://app.example.com
```

### Problèmes courants et solutions

#### Service ne trouve pas les Pods

**Symptôme :** Endpoints vides

```bash
kubectl get endpoints my-service -n my-app
# NAME         ENDPOINTS   AGE
# my-service   <none>      5m
```

**Solutions :**
1. Vérifier les labels des Pods
2. Vérifier le selector du Service
3. Vérifier que les Pods sont Running et Ready

```bash
# Comparer les labels
kubectl get pods -n my-app --show-labels
kubectl get svc my-service -n my-app -o yaml | grep -A5 selector
```

#### Ingress ne route pas le trafic

**Vérifications :**
```bash
# 1. Ingress Controller fonctionne
kubectl get pods -n ingress

# 2. Ingress a une adresse
kubectl get ingress -n my-app
# NAME         CLASS   HOSTS              ADDRESS     PORTS
# my-ingress   nginx   app.example.com    10.0.0.10   80,443

# 3. Service backend existe
kubectl get svc -n my-app

# 4. Certificats TLS (si HTTPS)
kubectl get secret my-tls -n my-app
```

#### Erreurs 502 Bad Gateway

**Causes possibles :**
- Pods pas prêts (readiness probe failed)
- Port incorrect dans le Service
- Service pointe vers des Pods inexistants
- Timeout trop court pour l'application

**Diagnostic :**
```bash
# Vérifier que les Pods sont Ready
kubectl get pods -n my-app

# Tester directement le Pod
kubectl port-forward pod/my-pod 8080:8080 -n my-app
curl http://localhost:8080

# Vérifier les logs du Pod
kubectl logs pod/my-pod -n my-app

# Augmenter les timeouts si nécessaire
kubectl annotate ingress my-ingress \
  nginx.ingress.kubernetes.io/proxy-read-timeout="300" \
  -n my-app
```

## Patterns de sécurité

### Exposition minimale

Principe : N'exposez que le strict nécessaire.

```yaml
# Service interne uniquement
apiVersion: v1
kind: Service
metadata:
  name: database
spec:
  type: ClusterIP  # Pas d'exposition externe
  selector:
    app: postgres
  ports:
  - port: 5432
---
# API exposée avec restrictions
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: api-restricted
  annotations:
    # Whitelist IP
    nginx.ingress.kubernetes.io/whitelist-source-range: "203.0.113.0/24"

    # Rate limiting strict
    nginx.ingress.kubernetes.io/limit-rps: "5"

    # Authentification requise
    nginx.ingress.kubernetes.io/auth-type: basic
    nginx.ingress.kubernetes.io/auth-secret: api-auth
spec:
  rules:
  - host: api-internal.example.com
    http:
      paths:
      - path: /admin
        pathType: Prefix
        backend:
          service:
            name: api-admin
            port:
              number: 80
```

### Network Policies

Contrôlez le trafic entre Services :

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: api-network-policy
  namespace: my-app
spec:
  podSelector:
    matchLabels:
      app: api
  policyTypes:
  - Ingress
  - Egress
  ingress:
  # Autorise uniquement depuis le frontend et l'ingress
  - from:
    - podSelector:
        matchLabels:
          app: frontend
    - namespaceSelector:
        matchLabels:
          name: ingress
    ports:
    - protocol: TCP
      port: 8080
  egress:
  # Autorise uniquement vers la base de données
  - to:
    - podSelector:
        matchLabels:
          app: database
    ports:
    - protocol: TCP
      port: 5432
  # DNS toujours autorisé
  - to:
    - namespaceSelector: {}
      podSelector:
        matchLabels:
          k8s-app: kube-dns
    ports:
    - protocol: UDP
      port: 53
```

### mTLS entre Services

Pour une sécurité maximale, utilisez le mTLS (mutual TLS) :

```yaml
# Avec Istio ou Linkerd (service mesh)
apiVersion: security.istio.io/v1beta1
kind: PeerAuthentication
metadata:
  name: default
  namespace: my-app
spec:
  mtls:
    mode: STRICT  # Force mTLS
```

## Optimisation des performances

### Caching avec Ingress

```yaml
metadata:
  annotations:
    # Cache statique
    nginx.ingress.kubernetes.io/configuration-snippet: |
      location ~* \.(jpg|jpeg|png|gif|ico|css|js)$ {
        expires 30d;
        add_header Cache-Control "public, immutable";
      }

    # Proxy cache
    nginx.ingress.kubernetes.io/proxy-buffering: "on"
    nginx.ingress.kubernetes.io/proxy-cache-bypass: "$http_pragma"
    nginx.ingress.kubernetes.io/proxy-cache-valid: "200 1h"
```

### Compression

```yaml
metadata:
  annotations:
    nginx.ingress.kubernetes.io/enable-gzip: "true"
    nginx.ingress.kubernetes.io/gzip-types: "application/json text/css application/javascript"
    nginx.ingress.kubernetes.io/gzip-min-length: "1000"
```

### Connection pooling

```yaml
metadata:
  annotations:
    # Keep-alive connections
    nginx.ingress.kubernetes.io/upstream-keepalive-connections: "100"
    nginx.ingress.kubernetes.io/upstream-keepalive-timeout: "60"
    nginx.ingress.kubernetes.io/upstream-keepalive-requests: "1000"
```

## Stratégies d'exposition pour différents environnements

### Développement local

```yaml
# Simple NodePort pour dev
apiVersion: v1
kind: Service
metadata:
  name: dev-app
spec:
  type: NodePort
  selector:
    app: my-app
  ports:
  - port: 80
    targetPort: 8080
    nodePort: 30080  # http://localhost:30080
```

### Staging

```yaml
# Ingress avec sous-domaine staging
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: staging-ingress
  annotations:
    # Certificat staging Let's Encrypt
    cert-manager.io/cluster-issuer: "letsencrypt-staging"

    # Basic auth pour protection
    nginx.ingress.kubernetes.io/auth-type: basic
    nginx.ingress.kubernetes.io/auth-secret: staging-auth
spec:
  rules:
  - host: staging.example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: app-staging
            port:
              number: 80
```

### Production

```yaml
# Production avec toutes les protections
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: production-ingress
  annotations:
    # Certificat production
    cert-manager.io/cluster-issuer: "letsencrypt-prod"

    # Sécurité maximale
    nginx.ingress.kubernetes.io/ssl-protocols: "TLSv1.2 TLSv1.3"
    nginx.ingress.kubernetes.io/ssl-ciphers: "HIGH:!aNULL:!MD5"

    # Headers de sécurité
    nginx.ingress.kubernetes.io/configuration-snippet: |
      more_set_headers "Strict-Transport-Security: max-age=31536000; includeSubDomains; preload";
      more_set_headers "X-Frame-Options: DENY";
      more_set_headers "X-Content-Type-Options: nosniff";
      more_set_headers "X-XSS-Protection: 1; mode=block";
      more_set_headers "Content-Security-Policy: default-src 'self'";
      more_set_headers "Referrer-Policy: strict-origin-when-cross-origin";

    # Rate limiting
    nginx.ingress.kubernetes.io/limit-rps: "100"
    nginx.ingress.kubernetes.io/limit-connections: "50"

    # WAF (Web Application Firewall)
    nginx.ingress.kubernetes.io/enable-modsecurity: "true"
    nginx.ingress.kubernetes.io/enable-owasp-core-rules: "true"
spec:
  rules:
  - host: www.example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: app-production
            port:
              number: 80
```

## Cas d'usage spéciaux

### WebSockets

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: websocket-ingress
  annotations:
    nginx.ingress.kubernetes.io/proxy-read-timeout: "3600"
    nginx.ingress.kubernetes.io/proxy-send-timeout: "3600"

    # Support WebSocket
    nginx.ingress.kubernetes.io/configuration-snippet: |
      proxy_set_header Upgrade $http_upgrade;
      proxy_set_header Connection "upgrade";
spec:
  rules:
  - host: ws.example.com
    http:
      paths:
      - path: /socket.io
        pathType: Prefix
        backend:
          service:
            name: websocket-server
            port:
              number: 3000
```

### gRPC

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: grpc-ingress
  annotations:
    nginx.ingress.kubernetes.io/backend-protocol: "GRPC"
    nginx.ingress.kubernetes.io/grpc-backend: "true"
spec:
  tls:
  - hosts:
    - grpc.example.com
    secretName: grpc-tls
  rules:
  - host: grpc.example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: grpc-service
            port:
              number: 50051
```

### Server-Sent Events (SSE)

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: sse-ingress
  annotations:
    nginx.ingress.kubernetes.io/proxy-buffering: "off"
    nginx.ingress.kubernetes.io/proxy-read-timeout: "86400"
    nginx.ingress.kubernetes.io/configuration-snippet: |
      proxy_set_header Connection '';
      proxy_http_version 1.1;
      chunked_transfer_encoding off;
      proxy_cache off;
spec:
  rules:
  - host: events.example.com
    http:
      paths:
      - path: /events
        pathType: Prefix
        backend:
          service:
            name: sse-server
            port:
              number: 8080
```

## Migration depuis des architectures traditionnelles

### De load balancer hardware vers Kubernetes

**Architecture traditionnelle :**
```
Internet → F5/HAProxy → Serveurs physiques
```

**Migration vers Kubernetes :**
```
Internet → Cloud LB → Ingress → Service → Pods
```

**Étapes de migration :**
1. Créer les Services ClusterIP pour vos applications
2. Configurer l'Ingress avec les mêmes règles que votre LB
3. Tester avec un sous-domaine de test
4. Basculer progressivement le trafic
5. Décommissionner l'ancien système

### De Apache/Nginx vers Ingress

**Configuration Apache VirtualHost :**
```apache
<VirtualHost *:80>
    ServerName www.example.com
    ProxyPass / http://backend:8080/
    ProxyPassReverse / http://backend:8080/
</VirtualHost>
```

**Équivalent Kubernetes :**
```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: example-ingress
spec:
  rules:
  - host: www.example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: backend
            port:
              number: 8080
```

## Métriques et observabilité

### Métriques de Service

```yaml
# ServiceMonitor pour Prometheus
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: app-metrics
spec:
  selector:
    matchLabels:
      app: my-app
  endpoints:
  - port: metrics
    interval: 30s
    path: /metrics
```

### Métriques d'Ingress

```bash
# Activer les métriques NGINX
kubectl annotate ingress my-ingress \
  nginx.ingress.kubernetes.io/enable-metrics="true" \
  -n my-app

# Accéder aux métriques
kubectl port-forward -n ingress service/nginx-ingress-controller-metrics 9113:9113
curl http://localhost:9113/metrics
```

### Dashboard de monitoring

Requêtes Prometheus utiles :
```promql
# Requêtes par seconde par Ingress
rate(nginx_ingress_controller_requests[5m])

# Latence P95
histogram_quantile(0.95,
  rate(nginx_ingress_controller_request_duration_seconds_bucket[5m])
)

# Erreurs 5xx
rate(nginx_ingress_controller_requests{status=~"5.."}[5m])

# Utilisation de la bande passante
rate(nginx_ingress_controller_response_size_sum[5m])
```

## Checklist de déploiement

### Pour les Services

- [ ] Type de Service approprié (ClusterIP/NodePort/LoadBalancer)
- [ ] Labels et selectors corrects
- [ ] Ports correctement mappés
- [ ] Session affinity si nécessaire
- [ ] Headless si besoin de découverte directe

### Pour l'Ingress

- [ ] Ingress Controller installé et fonctionnel
- [ ] Classe d'Ingress spécifiée
- [ ] Certificats TLS configurés
- [ ] Redirections HTTP → HTTPS
- [ ] Rate limiting configuré
- [ ] Headers de sécurité ajoutés
- [ ] CORS configuré si API
- [ ] Monitoring activé
- [ ] Backup/failover configuré

### Tests à effectuer

- [ ] Résolution DNS fonctionne
- [ ] Certificat SSL valide
- [ ] Load balancing entre Pods
- [ ] Health checks passent
- [ ] Performances acceptables
- [ ] Sécurité validée (scan SSL, headers)

## Conclusion

L'exposition via Service et Ingress est un aspect fondamental de Kubernetes qui permet de rendre vos applications accessibles de manière fiable et sécurisée.

**Points clés à retenir :**

1. **Services** : Fournissent une abstraction stable pour accéder aux Pods
   - ClusterIP pour l'interne
   - NodePort pour les tests
   - LoadBalancer pour la production cloud
   - ExternalName pour les services externes

2. **Ingress** : Gère l'accès HTTP/HTTPS avec des fonctionnalités avancées
   - Routage par domaine et chemin
   - Terminaison SSL/TLS
   - Rate limiting et sécurité
   - Support WebSocket, gRPC, SSE

3. **Sécurité** : Toujours implémenter les bonnes pratiques
   - HTTPS partout
   - Headers de sécurité
   - Rate limiting
   - Network policies

4. **Performance** : Optimiser pour votre cas d'usage
   - Caching approprié
   - Compression
   - Connection pooling
   - Métriques et monitoring

5. **Évolution** : Commencer simple, évoluer progressivement
   - Dev : NodePort simple
   - Staging : Ingress basique
   - Production : Full featured avec sécurité

Dans la prochaine section (7.5), nous verrons comment gérer la configuration de vos applications avec ConfigMaps, permettant de séparer la configuration du code pour plus de flexibilité.

⏭️
