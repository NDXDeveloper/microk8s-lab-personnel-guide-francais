🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 6.2 Configuration NGINX Ingress Controller

## Introduction

Maintenant que vous comprenez les concepts d'Ingress, passons à la pratique ! Dans cette section, nous allons installer, configurer et personnaliser NGINX Ingress Controller dans votre cluster MicroK8s. NGINX est le choix par défaut de MicroK8s car il est robuste, performant et largement utilisé dans l'industrie.

## Installation dans MicroK8s

### La méthode ultra-simple

La beauté de MicroK8s réside dans sa simplicité. L'installation de NGINX Ingress Controller se fait en une seule commande :

```bash
# Activer l'addon Ingress
microk8s enable ingress

# Sortie attendue :
# Enabling Ingress
# ingress is enabled
```

C'est tout ! MicroK8s s'occupe de :
- Télécharger l'image Docker de NGINX Ingress Controller
- Créer le namespace `ingress`
- Déployer le controller avec la configuration optimale
- Exposer les ports 80 et 443 sur votre machine hôte
- Configurer les règles de routage internes

### Vérification de l'installation

Après l'activation, vérifions que tout fonctionne :

```bash
# 1. Vérifier que le pod du controller est en cours d'exécution
microk8s kubectl get pods -n ingress

# Résultat attendu :
# NAME                                      READY   STATUS    RESTARTS   AGE
# nginx-ingress-microk8s-controller-xxxxx  1/1     Running   0          2m

# 2. Vérifier les services exposés
microk8s kubectl get svc -n ingress

# Résultat attendu :
# NAME                                 TYPE        CLUSTER-IP       PORT(S)
# ingress                             ClusterIP   10.152.183.xxx   80/TCP,443/TCP

# 3. Vérifier le DaemonSet (qui gère le pod)
microk8s kubectl get daemonset -n ingress

# 4. Voir les détails complets du controller
microk8s kubectl describe pod -n ingress -l name=nginx-ingress-microk8s
```

### Comprendre ce qui a été installé

L'addon Ingress de MicroK8s déploie plusieurs composants :

```
Namespace: ingress/
│
├── DaemonSet: nginx-ingress-microk8s-controller
│   └── Pod: nginx-ingress-microk8s-controller-xxxxx
│       ├── Container: nginx-ingress-microk8s
│       ├── Ports: 80, 443 (HTTP/HTTPS)
│       └── Image: k8s.gcr.io/ingress-nginx/controller
│
├── ConfigMap: nginx-load-balancer-microk8s-conf
│   └── Configuration NGINX personnalisée
│
├── ConfigMap: nginx-ingress-tcp-microk8s-conf
│   └── Configuration pour services TCP
│
└── ConfigMap: nginx-ingress-udp-microk8s-conf
    └── Configuration pour services UDP
```

## Architecture du NGINX Ingress Controller

### Comment NGINX s'intègre dans MicroK8s

```
[Internet/Réseau Local]
         ↓
    [Machine Hôte]
    Ports 80/443
         ↓
[Pod NGINX Ingress Controller]
    ├── Surveille les ressources Ingress
    ├── Génère la config NGINX
    ├── Recharge NGINX si nécessaire
    └── Route le trafic vers les Services
         ↓
    [Services Backend]
         ↓
      [Pods]
```

### Le cycle de vie d'une configuration

1. **Création** : Vous créez une ressource Ingress
2. **Détection** : Le controller détecte le changement
3. **Génération** : Il génère une nouvelle configuration NGINX
4. **Validation** : Test de la configuration
5. **Rechargement** : NGINX recharge sans interruption
6. **Routage** : Le nouveau routage est actif

## Configuration de base

### Le ConfigMap principal

NGINX Ingress Controller utilise un ConfigMap pour sa configuration globale :

```bash
# Voir la configuration actuelle
microk8s kubectl get configmap -n ingress nginx-load-balancer-microk8s-conf -o yaml
```

### Modifier la configuration globale

Pour personnaliser le comportement global du controller :

```yaml
# nginx-config.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: nginx-load-balancer-microk8s-conf
  namespace: ingress
data:
  # Taille maximale du body des requêtes (upload de fichiers)
  proxy-body-size: "20m"

  # Timeout de lecture du proxy (en secondes)
  proxy-read-timeout: "600"

  # Timeout de connexion
  proxy-connect-timeout: "10"

  # Activer les vraies IPs des clients
  use-forwarded-headers: "true"

  # Format des logs personnalisé
  log-format-upstream: '$remote_addr - $remote_user [$time_local] "$request" $status $body_bytes_sent "$http_referer" "$http_user_agent"'

  # Niveau de log (debug, info, notice, warn, error, crit)
  error-log-level: "notice"

  # Nombre de worker processes
  worker-processes: "auto"

  # Connexions par worker
  worker-connections: "1024"
```

Appliquer les modifications :

```bash
# Appliquer le nouveau ConfigMap
microk8s kubectl apply -f nginx-config.yaml

# Le controller recharge automatiquement la configuration
# Vérifier les logs pour confirmer
microk8s kubectl logs -n ingress -l name=nginx-ingress-microk8s --tail=20
```

## Paramètres de configuration importants

### Performance et optimisation

```yaml
data:
  # === PERFORMANCE ===

  # Utiliser HTTP/2
  use-http2: "true"

  # Compression GZIP
  use-gzip: "true"
  gzip-level: "5"
  gzip-types: "application/json text/css application/javascript"

  # Keep-alive
  keep-alive: "75"
  keep-alive-requests: "100"

  # Buffer sizes
  client-body-buffer-size: "1m"
  client-header-buffer-size: "1k"
  large-client-header-buffers: "4 8k"

  # Caching DNS
  resolver-timeout: "5s"
```

### Sécurité

```yaml
data:
  # === SÉCURITÉ ===

  # Masquer la version de NGINX
  server-tokens: "false"

  # Headers de sécurité
  enable-modsecurity: "false"  # WAF (Web Application Firewall)
  enable-owasp-modsecurity-crs: "false"

  # SSL/TLS
  ssl-protocols: "TLSv1.2 TLSv1.3"
  ssl-ciphers: "HIGH:!aNULL:!MD5"
  ssl-prefer-server-ciphers: "true"

  # HSTS (HTTP Strict Transport Security)
  hsts: "true"
  hsts-max-age: "31536000"
  hsts-include-subdomains: "true"

  # Limites de rate
  limit-rate: "0"  # 0 = pas de limite
  limit-rate-after: "0"
```

### Monitoring et logs

```yaml
data:
  # === MONITORING ===

  # Métriques Prometheus
  enable-metrics: "true"

  # Access logs
  disable-access-log: "false"
  access-log-path: "/var/log/nginx/access.log"
  error-log-path: "/var/log/nginx/error.log"

  # Format de log JSON (utile pour l'analyse)
  log-format-escape-json: "true"
  log-format-upstream: '{"time": "$time_iso8601", "remote_addr": "$remote_addr", "request": "$request", "status": $status, "bytes": $body_bytes_sent, "duration": $request_time}'
```

## Configuration par annotations

### Qu'est-ce qu'une annotation ?

Les annotations permettent de configurer le comportement d'Ingress au cas par cas, sans modifier la configuration globale :

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: mon-app
  annotations:
    # Ces annotations configurent NGINX pour cette Ingress spécifiquement
    nginx.ingress.kubernetes.io/rewrite-target: /
    nginx.ingress.kubernetes.io/ssl-redirect: "true"
spec:
  rules:
  - host: app.exemple.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: mon-service
            port:
              number: 80
```

### Annotations courantes et utiles

#### Redirections et rewrites

```yaml
annotations:
  # Forcer HTTPS
  nginx.ingress.kubernetes.io/ssl-redirect: "true"

  # Redirection permanente
  nginx.ingress.kubernetes.io/permanent-redirect: "https://nouveau-site.com"

  # Réécriture d'URL
  nginx.ingress.kubernetes.io/rewrite-target: /$2
  # Avec path: /api(/|$)(.*)  → /api/users devient /users

  # Ajouter/supprimer trailing slash
  nginx.ingress.kubernetes.io/app-root: /app
```

#### Timeouts et limites

```yaml
annotations:
  # Timeouts (en secondes)
  nginx.ingress.kubernetes.io/proxy-connect-timeout: "600"
  nginx.ingress.kubernetes.io/proxy-send-timeout: "600"
  nginx.ingress.kubernetes.io/proxy-read-timeout: "600"

  # Taille maximale du body (upload)
  nginx.ingress.kubernetes.io/proxy-body-size: "50m"

  # Rate limiting
  nginx.ingress.kubernetes.io/limit-rps: "10"
  nginx.ingress.kubernetes.io/limit-connections: "20"
```

#### Authentification

```yaml
annotations:
  # Authentification basique
  nginx.ingress.kubernetes.io/auth-type: basic
  nginx.ingress.kubernetes.io/auth-secret: basic-auth
  nginx.ingress.kubernetes.io/auth-realm: "Authentification requise"

  # Whitelist IP
  nginx.ingress.kubernetes.io/whitelist-source-range: "192.168.1.0/24,10.0.0.0/8"
```

#### CORS (Cross-Origin Resource Sharing)

```yaml
annotations:
  # Activer CORS
  nginx.ingress.kubernetes.io/enable-cors: "true"

  # Configuration CORS détaillée
  nginx.ingress.kubernetes.io/cors-allow-origin: "*"
  nginx.ingress.kubernetes.io/cors-allow-methods: "GET, POST, PUT, DELETE, OPTIONS"
  nginx.ingress.kubernetes.io/cors-allow-headers: "DNT,X-CustomHeader,Keep-Alive,User-Agent,X-Requested-With,If-Modified-Since,Cache-Control,Content-Type,Authorization"
  nginx.ingress.kubernetes.io/cors-allow-credentials: "true"
```

#### WebSocket

```yaml
annotations:
  # Support WebSocket
  nginx.ingress.kubernetes.io/proxy-http-version: "1.1"
  nginx.ingress.kubernetes.io/configuration-snippet: |
    proxy_set_header Upgrade $http_upgrade;
    proxy_set_header Connection "upgrade";
```

## Personnalisation avancée

### Snippets de configuration

Les snippets permettent d'injecter du code NGINX personnalisé :

```yaml
annotations:
  # Server snippet (dans le bloc server)
  nginx.ingress.kubernetes.io/server-snippet: |
    location /health {
      return 200 "OK";
      add_header Content-Type text/plain;
    }

  # Configuration snippet (dans le bloc location)
  nginx.ingress.kubernetes.io/configuration-snippet: |
    proxy_set_header X-Custom-Header "valeur";
    add_header X-Response-Time $request_time;
```

### Configuration TCP/UDP

NGINX Ingress Controller peut aussi router du trafic TCP/UDP :

```yaml
# tcp-services-configmap.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: nginx-ingress-tcp-microk8s-conf
  namespace: ingress
data:
  # Port externe: "namespace/service:port"
  3306: "default/mysql:3306"
  5432: "default/postgres:5432"
```

```yaml
# udp-services-configmap.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: nginx-ingress-udp-microk8s-conf
  namespace: ingress
data:
  53: "kube-system/kube-dns:53"
```

## Monitoring du controller

### Métriques Prometheus

NGINX Ingress Controller expose des métriques pour Prometheus :

```bash
# Activer les métriques dans le ConfigMap
data:
  enable-metrics: "true"

# Les métriques sont disponibles sur le port 10254
microk8s kubectl get svc -n ingress nginx-ingress-microk8s-controller-metrics

# Tester les métriques
microk8s kubectl port-forward -n ingress svc/nginx-ingress-microk8s-controller-metrics 10254:10254

# Dans un autre terminal
curl http://localhost:10254/metrics
```

Métriques importantes à surveiller :
- `nginx_ingress_controller_requests` : Nombre de requêtes
- `nginx_ingress_controller_request_duration_seconds` : Latence
- `nginx_ingress_controller_response_size` : Taille des réponses
- `nginx_ingress_controller_success` : Taux de succès

### Logs et débogage

```bash
# Voir les logs du controller
microk8s kubectl logs -n ingress -l name=nginx-ingress-microk8s -f

# Logs avec plus de détails
microk8s kubectl logs -n ingress -l name=nginx-ingress-microk8s --tail=100

# Voir la configuration NGINX générée
microk8s kubectl exec -n ingress -it $(microk8s kubectl get pods -n ingress -l name=nginx-ingress-microk8s -o jsonpath='{.items[0].metadata.name}') -- cat /etc/nginx/nginx.conf
```

## Optimisation pour un lab personnel

### Configuration recommandée pour un lab

```yaml
# lab-nginx-config.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: nginx-load-balancer-microk8s-conf
  namespace: ingress
data:
  # Performance modérée pour économiser les ressources
  worker-processes: "2"
  worker-connections: "512"

  # Timeouts généreux pour le développement
  proxy-read-timeout: "3600"
  proxy-connect-timeout: "30"

  # Upload de fichiers volumineux
  proxy-body-size: "100m"

  # Logs détaillés pour le debug
  error-log-level: "info"

  # Compression activée
  use-gzip: "true"

  # HTTP/2 pour la performance
  use-http2: "true"

  # Headers de sécurité basiques
  server-tokens: "false"

  # Métriques pour monitoring
  enable-metrics: "true"
```

### Réglages pour différents scénarios

#### Lab de développement

```yaml
data:
  # Logs verbeux
  error-log-level: "debug"
  # Timeouts longs pour debug
  proxy-read-timeout: "3600"
  # CORS permissif
  cors-allow-origin: "*"
```

#### Lab de test/staging

```yaml
data:
  # Configuration proche de la production
  error-log-level: "warn"
  proxy-read-timeout: "60"
  # Sécurité renforcée
  ssl-protocols: "TLSv1.2 TLSv1.3"
  hsts: "true"
```

#### Services publics (blog, portfolio)

```yaml
data:
  # Performance optimisée
  use-gzip: "true"
  use-http2: "true"
  # Cache headers
  proxy-cache-bypass: "$http_upgrade"
  # Sécurité maximale
  enable-modsecurity: "true"
```

## Dépannage courant

### Le controller ne démarre pas

```bash
# Vérifier les logs
microk8s kubectl logs -n ingress -l name=nginx-ingress-microk8s --previous

# Causes fréquentes :
# - Ports 80/443 déjà utilisés
# - Mémoire insuffisante
# - Image Docker non téléchargée

# Solution : libérer les ports
sudo lsof -i :80
sudo lsof -i :443
```

### Les règles Ingress ne fonctionnent pas

```bash
# 1. Vérifier que l'Ingress est créée
microk8s kubectl get ingress -A

# 2. Décrire l'Ingress pour voir les erreurs
microk8s kubectl describe ingress mon-ingress

# 3. Vérifier la configuration NGINX générée
microk8s kubectl exec -n ingress -it [pod-name] -- nginx -T | grep "server_name"

# 4. Tester directement sur le controller
microk8s kubectl port-forward -n ingress [pod-name] 8080:80
curl -H "Host: mon-domaine.local" http://localhost:8080
```

### Performance dégradée

```bash
# Vérifier l'utilisation des ressources
microk8s kubectl top pod -n ingress

# Augmenter les limites si nécessaire
microk8s kubectl edit daemonset -n ingress nginx-ingress-microk8s-controller

# Modifier les limites de ressources
resources:
  limits:
    memory: "512Mi"
    cpu: "500m"
  requests:
    memory: "256Mi"
    cpu: "250m"
```

## Résumé des commandes essentielles

```bash
# Installation
microk8s enable ingress

# Statut
microk8s kubectl get all -n ingress

# Configuration
microk8s kubectl edit configmap -n ingress nginx-load-balancer-microk8s-conf

# Logs
microk8s kubectl logs -n ingress -l name=nginx-ingress-microk8s -f

# Restart du controller
microk8s kubectl rollout restart daemonset -n ingress nginx-ingress-microk8s-controller

# Désinstallation
microk8s disable ingress
```

## Points clés à retenir

✅ **Installation simple** : Une commande avec MicroK8s
✅ **Configuration flexible** : Global (ConfigMap) ou spécifique (annotations)
✅ **Monitoring intégré** : Métriques Prometheus disponibles
✅ **Personnalisation puissante** : Snippets pour cas spéciaux
✅ **Support TCP/UDP** : Au-delà du simple HTTP/HTTPS

## Prochaine étape

Maintenant que NGINX Ingress Controller est configuré et optimisé, vous êtes prêt à créer vos premières règles de routage. La section suivante (6.3) vous montrera comment router différents domaines vers différents services.

---

💡 **Conseil pro** : Gardez votre configuration simple au début. Commencez avec les paramètres par défaut, puis ajustez progressivement selon vos besoins. La sur-optimisation prématurée est souvent contre-productive dans un environnement de lab.

⏭️
