🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 6.5 Middleware et annotations

## Introduction aux annotations et middleware

Les annotations sont comme des "post-its" que vous collez sur vos ressources Ingress pour personnaliser leur comportement. Elles permettent d'ajouter des fonctionnalités middleware sans modifier la configuration globale du controller.

### Qu'est-ce qu'un middleware ?

Un middleware est un composant logiciel qui s'intercale entre la requête entrante et votre application. Il peut :
- Modifier les requêtes (ajouter des headers, réécrire les URLs)
- Vérifier l'authentification
- Limiter le trafic
- Compresser les réponses
- Gérer les redirections
- Ajouter de la sécurité

### Analogie simple

Imaginez un hôtel de luxe :
- **Portier** (Rate limiting) : Contrôle le flux d'entrée
- **Réceptionniste** (Authentification) : Vérifie l'identité
- **Concierge** (Rewrite) : Redirige vers le bon service
- **Service de sécurité** (WAF) : Protège contre les menaces
- **Standard téléphonique** (Load balancing) : Distribue les appels

Les annotations configurent ces "services" pour votre Ingress.

## Anatomie des annotations

### Structure de base

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: exemple-ingress
  annotations:
    # Format : nginx.ingress.kubernetes.io/nom-annotation: "valeur"
    nginx.ingress.kubernetes.io/rewrite-target: "/"
    nginx.ingress.kubernetes.io/ssl-redirect: "true"
    nginx.ingress.kubernetes.io/proxy-body-size: "50m"
```

### Portée des annotations

Les annotations s'appliquent à toute la ressource Ingress :
- ✅ Tous les hosts définis dans cette Ingress
- ✅ Tous les paths définis dans cette Ingress
- ❌ Pas aux autres ressources Ingress

Pour des comportements différents, créez des Ingress séparées.

## Catégories principales d'annotations

### 1. Redirections et rewrites

#### Redirection HTTPS

```yaml
# ingress-ssl-redirect.yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: force-https
  annotations:
    # Force toujours HTTPS
    nginx.ingress.kubernetes.io/ssl-redirect: "true"

    # Ou redirection conditionnelle
    nginx.ingress.kubernetes.io/force-ssl-redirect: "true"
spec:
  tls:
  - hosts:
    - secure.monlab.local
    secretName: tls-secret
  rules:
  - host: secure.monlab.local
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

#### Redirections permanentes

```yaml
# ingress-redirects.yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: redirections
  annotations:
    # Redirection 301 permanente
    nginx.ingress.kubernetes.io/permanent-redirect: "https://nouveau-site.com"

    # Ou redirection temporaire 302
    nginx.ingress.kubernetes.io/temporal-redirect: "https://maintenance.com"

    # Redirection avec code personnalisé
    nginx.ingress.kubernetes.io/permanent-redirect-code: "308"
```

#### Réécriture d'URL avancée

```yaml
# ingress-rewrite-advanced.yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: rewrite-complex
  annotations:
    # Capture avec regex
    nginx.ingress.kubernetes.io/rewrite-target: "/$2"

    # Préserver les query strings
    nginx.ingress.kubernetes.io/app-root: "/app"

    # Ajouter/enlever trailing slash
    nginx.ingress.kubernetes.io/configuration-snippet: |
      rewrite ^(/app)$ $1/ permanent;
spec:
  rules:
  - host: monlab.local
    http:
      paths:
      - path: /old-api(/|$)(.*)
        pathType: Prefix
        backend:
          service:
            name: new-api
            port:
              number: 8080
```

### 2. Authentification et autorisation

#### Authentification basique

```yaml
# ingress-basic-auth.yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: protected-app
  annotations:
    # Type d'authentification
    nginx.ingress.kubernetes.io/auth-type: basic

    # Secret contenant les credentials
    nginx.ingress.kubernetes.io/auth-secret: basic-auth-secret

    # Message du prompt
    nginx.ingress.kubernetes.io/auth-realm: "Zone protégée - Authentification requise"
spec:
  rules:
  - host: private.monlab.local
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: private-app
            port:
              number: 80
---
# Créer le secret pour l'authentification
# htpasswd -c auth admin
# kubectl create secret generic basic-auth-secret --from-file=auth
```

#### OAuth2 Proxy

```yaml
# ingress-oauth2.yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: oauth-protected
  annotations:
    # URL de l'OAuth2 proxy
    nginx.ingress.kubernetes.io/auth-url: "https://auth.monlab.local/oauth2/auth"
    nginx.ingress.kubernetes.io/auth-signin: "https://auth.monlab.local/oauth2/start?rd=$escaped_request_uri"

    # Headers à transmettre
    nginx.ingress.kubernetes.io/auth-response-headers: "X-Auth-Request-User, X-Auth-Request-Email"
spec:
  rules:
  - host: app.monlab.local
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: protected-app
            port:
              number: 80
```

#### Whitelist IP

```yaml
# ingress-ip-whitelist.yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: ip-restricted
  annotations:
    # Autoriser seulement certaines IPs/réseaux
    nginx.ingress.kubernetes.io/whitelist-source-range: "192.168.1.0/24,10.0.0.0/8,203.0.113.42/32"

    # Message personnalisé pour les IPs refusées
    nginx.ingress.kubernetes.io/custom-http-errors: "403"
    nginx.ingress.kubernetes.io/default-backend: error-backend
spec:
  rules:
  - host: internal.monlab.local
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: internal-app
            port:
              number: 80
```

### 3. Rate limiting et protection

#### Limitation du taux de requêtes

```yaml
# ingress-rate-limit.yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: rate-limited
  annotations:
    # Requêtes par seconde
    nginx.ingress.kubernetes.io/limit-rps: "10"

    # Requêtes par minute
    nginx.ingress.kubernetes.io/limit-rpm: "100"

    # Connexions simultanées
    nginx.ingress.kubernetes.io/limit-connections: "5"

    # Burst (rafale autorisée)
    nginx.ingress.kubernetes.io/limit-burst-multiplier: "2"

    # Whitelist pour bypass
    nginx.ingress.kubernetes.io/limit-whitelist: "192.168.1.0/24"
spec:
  rules:
  - host: api.monlab.local
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: api-service
            port:
              number: 8080
```

#### Protection DDoS

```yaml
# ingress-ddos-protection.yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: ddos-protected
  annotations:
    # Limite globale de bande passante
    nginx.ingress.kubernetes.io/limit-rate: "100k"

    # Après X bytes, appliquer la limite
    nginx.ingress.kubernetes.io/limit-rate-after: "500k"

    # Zone de limite personnalisée
    nginx.ingress.kubernetes.io/server-snippet: |
      limit_req_zone $binary_remote_addr zone=mylimit:10m rate=10r/s;
      limit_req zone=mylimit burst=20 nodelay;
      limit_req_status 429;
spec:
  rules:
  - host: public.monlab.local
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: public-app
            port:
              number: 80
```

### 4. Headers et CORS

#### Gestion des headers

```yaml
# ingress-headers.yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: custom-headers
  annotations:
    # Ajouter des headers aux requêtes
    nginx.ingress.kubernetes.io/configuration-snippet: |
      proxy_set_header X-Custom-Header "CustomValue";
      proxy_set_header X-Forwarded-Host $host;
      proxy_set_header X-Original-URI $request_uri;

    # Ajouter des headers aux réponses
    nginx.ingress.kubernetes.io/add-headers: "default:custom-headers-configmap"

    # Headers de sécurité
    nginx.ingress.kubernetes.io/server-snippet: |
      add_header X-Frame-Options "SAMEORIGIN" always;
      add_header X-Content-Type-Options "nosniff" always;
      add_header X-XSS-Protection "1; mode=block" always;
      add_header Referrer-Policy "strict-origin-when-cross-origin" always;
spec:
  rules:
  - host: secure.monlab.local
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

#### Configuration CORS complète

```yaml
# ingress-cors.yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: cors-enabled
  annotations:
    # Activer CORS
    nginx.ingress.kubernetes.io/enable-cors: "true"

    # Origines autorisées
    nginx.ingress.kubernetes.io/cors-allow-origin: "https://app.exemple.com,https://autre.exemple.com"

    # Ou permettre toutes les origines (attention en production!)
    # nginx.ingress.kubernetes.io/cors-allow-origin: "*"

    # Méthodes autorisées
    nginx.ingress.kubernetes.io/cors-allow-methods: "GET, POST, PUT, DELETE, OPTIONS, HEAD"

    # Headers autorisés
    nginx.ingress.kubernetes.io/cors-allow-headers: "DNT,Keep-Alive,User-Agent,X-Requested-With,If-Modified-Since,Cache-Control,Content-Type,Range,Authorization,X-Custom-Header"

    # Headers exposés au client
    nginx.ingress.kubernetes.io/cors-expose-headers: "Content-Length,Content-Range,X-Custom-Response"

    # Autoriser les credentials
    nginx.ingress.kubernetes.io/cors-allow-credentials: "true"

    # Durée du cache preflight (en secondes)
    nginx.ingress.kubernetes.io/cors-max-age: "86400"
spec:
  rules:
  - host: api.monlab.local
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: api-service
            port:
              number: 8080
```

### 5. Proxy et timeouts

#### Configuration des timeouts

```yaml
# ingress-timeouts.yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: custom-timeouts
  annotations:
    # Timeout de connexion au backend (secondes)
    nginx.ingress.kubernetes.io/proxy-connect-timeout: "10"

    # Timeout d'envoi au backend
    nginx.ingress.kubernetes.io/proxy-send-timeout: "120"

    # Timeout de lecture du backend
    nginx.ingress.kubernetes.io/proxy-read-timeout: "120"

    # Timeout du body de la requête
    nginx.ingress.kubernetes.io/proxy-request-timeout: "120"

    # Keep-alive timeout
    nginx.ingress.kubernetes.io/upstream-keepalive-timeout: "60"

    # Timeout pour les WebSockets
    nginx.ingress.kubernetes.io/proxy-stream-timeout: "600"
spec:
  rules:
  - host: longrunning.monlab.local
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: slow-app
            port:
              number: 80
```

#### Taille des uploads

```yaml
# ingress-upload-size.yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: large-uploads
  annotations:
    # Taille maximale du body (0 = illimité)
    nginx.ingress.kubernetes.io/proxy-body-size: "100m"

    # Buffer size pour le body
    nginx.ingress.kubernetes.io/client-body-buffer-size: "1m"

    # Taille maximale du header
    nginx.ingress.kubernetes.io/large-client-header-buffers: "4 32k"
spec:
  rules:
  - host: upload.monlab.local
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: upload-service
            port:
              number: 80
```

### 6. WebSocket et streaming

#### Support WebSocket

```yaml
# ingress-websocket.yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: websocket-app
  annotations:
    # Activer le support WebSocket
    nginx.ingress.kubernetes.io/proxy-http-version: "1.1"

    # Headers nécessaires pour WebSocket
    nginx.ingress.kubernetes.io/configuration-snippet: |
      proxy_set_header Upgrade $http_upgrade;
      proxy_set_header Connection "upgrade";

    # Timeout pour connexions WebSocket (10 heures)
    nginx.ingress.kubernetes.io/proxy-read-timeout: "36000"
    nginx.ingress.kubernetes.io/proxy-send-timeout: "36000"
spec:
  rules:
  - host: ws.monlab.local
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: websocket-service
            port:
              number: 3000
```

#### Server-Sent Events (SSE)

```yaml
# ingress-sse.yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: sse-streaming
  annotations:
    # Désactiver la mise en buffer pour SSE
    nginx.ingress.kubernetes.io/proxy-buffering: "off"

    # Headers pour SSE
    nginx.ingress.kubernetes.io/configuration-snippet: |
      proxy_set_header Connection '';
      proxy_http_version 1.1;
      chunked_transfer_encoding off;
      proxy_cache off;

    # Long timeout pour streaming
    nginx.ingress.kubernetes.io/proxy-read-timeout: "86400"
spec:
  rules:
  - host: events.monlab.local
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: sse-service
            port:
              number: 8080
```

### 7. Caching et performance

#### Configuration du cache

```yaml
# ingress-caching.yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: cached-content
  annotations:
    # Activer la compression
    nginx.ingress.kubernetes.io/enable-gzip: "true"
    nginx.ingress.kubernetes.io/gzip-types: "application/json text/css application/javascript text/xml"
    nginx.ingress.kubernetes.io/gzip-level: "5"

    # Cache pour fichiers statiques
    nginx.ingress.kubernetes.io/configuration-snippet: |
      if ($request_uri ~* \.(js|css|gif|jpg|jpeg|png|svg|ico|woff|woff2)$) {
        add_header Cache-Control "public, max-age=31536000, immutable";
      }
      if ($request_uri ~* \.(html|json)$) {
        add_header Cache-Control "public, max-age=3600, must-revalidate";
      }

    # Activer le cache proxy
    nginx.ingress.kubernetes.io/proxy-cache: "on"
    nginx.ingress.kubernetes.io/proxy-cache-valid: "200 201 302 1h"
spec:
  rules:
  - host: static.monlab.local
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: static-service
            port:
              number: 80
```

### 8. Sécurité avancée

#### HSTS et sécurité SSL

```yaml
# ingress-hsts.yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: strict-security
  annotations:
    # HTTP Strict Transport Security
    nginx.ingress.kubernetes.io/hsts: "true"
    nginx.ingress.kubernetes.io/hsts-max-age: "31536000"
    nginx.ingress.kubernetes.io/hsts-include-subdomains: "true"
    nginx.ingress.kubernetes.io/hsts-preload: "true"

    # Protocoles SSL/TLS
    nginx.ingress.kubernetes.io/ssl-protocols: "TLSv1.2 TLSv1.3"
    nginx.ingress.kubernetes.io/ssl-ciphers: "HIGH:!aNULL:!MD5"
    nginx.ingress.kubernetes.io/ssl-prefer-server-ciphers: "true"

    # Session SSL
    nginx.ingress.kubernetes.io/ssl-session-cache: "true"
    nginx.ingress.kubernetes.io/ssl-session-cache-size: "10m"
    nginx.ingress.kubernetes.io/ssl-session-timeout: "10m"
spec:
  tls:
  - hosts:
    - secure.monlab.local
    secretName: secure-tls
  rules:
  - host: secure.monlab.local
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

#### ModSecurity WAF

```yaml
# ingress-waf.yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: waf-protected
  annotations:
    # Activer ModSecurity
    nginx.ingress.kubernetes.io/enable-modsecurity: "true"

    # Activer les règles OWASP Core Rule Set
    nginx.ingress.kubernetes.io/enable-owasp-modsecurity-crs: "true"

    # Mode de ModSecurity (DetectionOnly ou On)
    nginx.ingress.kubernetes.io/modsecurity-mode: "DetectionOnly"

    # Transaction ID pour les logs
    nginx.ingress.kubernetes.io/modsecurity-transaction-id: "$request_id"

    # Règles personnalisées
    nginx.ingress.kubernetes.io/modsecurity-snippet: |
      SecRuleEngine On
      SecRule ARGS:testparam "@contains test" "id:1234,deny,log,msg:'Test rule'"
spec:
  rules:
  - host: protected.monlab.local
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: protected-app
            port:
              number: 80
```

## Snippets personnalisés

### Server snippets

```yaml
# ingress-server-snippet.yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: server-customization
  annotations:
    # Code NGINX au niveau server {}
    nginx.ingress.kubernetes.io/server-snippet: |
      # Variable personnalisée
      set $my_var "value";

      # Location personnalisée
      location /health {
        access_log off;
        return 200 "OK\n";
        add_header Content-Type text/plain;
      }

      # Règle de sécurité
      if ($http_user_agent ~* (bot|crawler)) {
        return 403;
      }

      # Log conditionnel
      set $log_ua 1;
      if ($http_user_agent ~* (kube-probe|health)) {
        set $log_ua 0;
      }
      access_log /var/log/nginx/access.log combined if=$log_ua;
spec:
  rules:
  - host: custom.monlab.local
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: custom-app
            port:
              number: 80
```

### Configuration snippets

```yaml
# ingress-config-snippet.yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: location-customization
  annotations:
    # Code NGINX au niveau location {}
    nginx.ingress.kubernetes.io/configuration-snippet: |
      # Headers personnalisés
      more_set_headers "X-Backend-Server: $upstream_addr";
      more_set_headers "X-Request-ID: $request_id";

      # Règles de proxy
      proxy_next_upstream error timeout http_500 http_502;
      proxy_next_upstream_tries 3;

      # Variables personnalisées
      set $my_backend "backend1";
      if ($arg_version = "v2") {
        set $my_backend "backend2";
      }
spec:
  rules:
  - host: advanced.monlab.local
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: advanced-app
            port:
              number: 80
```

## Combinaisons d'annotations

### API sécurisée complète

```yaml
# ingress-secure-api.yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: production-api
  annotations:
    # SSL/TLS
    nginx.ingress.kubernetes.io/ssl-redirect: "true"
    nginx.ingress.kubernetes.io/hsts: "true"

    # Rate limiting
    nginx.ingress.kubernetes.io/limit-rps: "100"
    nginx.ingress.kubernetes.io/limit-connections: "10"

    # CORS
    nginx.ingress.kubernetes.io/enable-cors: "true"
    nginx.ingress.kubernetes.io/cors-allow-origin: "https://app.production.com"

    # Authentification
    nginx.ingress.kubernetes.io/auth-type: basic
    nginx.ingress.kubernetes.io/auth-secret: api-auth

    # Timeouts
    nginx.ingress.kubernetes.io/proxy-read-timeout: "30"

    # Sécurité
    nginx.ingress.kubernetes.io/enable-modsecurity: "true"

    # Headers de sécurité
    nginx.ingress.kubernetes.io/configuration-snippet: |
      more_set_headers "X-Frame-Options: DENY";
      more_set_headers "X-Content-Type-Options: nosniff";
      more_set_headers "Content-Security-Policy: default-src 'self'";
spec:
  tls:
  - hosts:
    - api.production.com
    secretName: api-tls
  rules:
  - host: api.production.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: production-api
            port:
              number: 8080
```

### Application web optimisée

```yaml
# ingress-optimized-webapp.yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: webapp-optimized
  annotations:
    # Compression
    nginx.ingress.kubernetes.io/enable-gzip: "true"
    nginx.ingress.kubernetes.io/gzip-level: "6"

    # Cache statique
    nginx.ingress.kubernetes.io/configuration-snippet: |
      location ~* \.(jpg|jpeg|png|gif|ico|css|js|woff2)$ {
        expires 30d;
        add_header Cache-Control "public, immutable";
      }

    # HTTP/2 Push
    nginx.ingress.kubernetes.io/server-snippet: |
      http2_push /css/main.css;
      http2_push /js/app.js;

    # Optimisations proxy
    nginx.ingress.kubernetes.io/proxy-buffering: "on"
    nginx.ingress.kubernetes.io/proxy-buffer-size: "8k"
    nginx.ingress.kubernetes.io/proxy-buffers: "8 8k"
spec:
  rules:
  - host: app.monlab.local
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: webapp
            port:
              number: 3000
```

## Debugging et test des annotations

### Vérification de la configuration

```bash
# Voir les annotations appliquées
microk8s kubectl describe ingress mon-ingress

# Vérifier la configuration NGINX générée
microk8s kubectl exec -n ingress [nginx-pod] -- nginx -T | grep -A10 "server_name monlab.local"

# Voir les logs pour débugger
microk8s kubectl logs -n ingress -l name=nginx-ingress-microk8s -f

# Tester avec curl
curl -v -H "Host: monlab.local" http://[CLUSTER-IP]
```

### Test des middleware

```bash
# Tester l'authentification basique
curl -u user:password http://protected.monlab.local

# Tester les headers CORS
curl -H "Origin: https://exemple.com" \
     -H "Access-Control-Request-Method: GET" \
     -H "Access-Control-Request-Headers: X-Custom-Header" \
     -X OPTIONS \
     http://api.monlab.local

# Tester le rate limiting
for i in {1..20}; do curl http://api.monlab.local/test; done

# Vérifier les headers de réponse
curl -I http://secure.monlab.local
```

## Exemple complet pour un lab

### Stack complète avec tous les middleware

```yaml
# ingress-complete-middleware.yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: lab-complete-middleware
  annotations:
    # === SÉCURITÉ ===
    nginx.ingress.kubernetes.io/ssl-redirect: "true"
    nginx.ingress.kubernetes.io/hsts: "true"
    nginx.ingress.kubernetes.io/hsts-max-age: "15768000"

    # === AUTHENTIFICATION ===
    nginx.ingress.kubernetes.io/auth-type: basic
    nginx.ingress.kubernetes.io/auth-secret: lab-auth
    nginx.ingress.kubernetes.io/auth-realm: "Lab Authentication"

    # === RATE LIMITING ===
    nginx.ingress.kubernetes.io/limit-rps: "50"
    nginx.ingress.kubernetes.io/limit-connections: "5"

    # === CORS ===
    nginx.ingress.kubernetes.io/enable-cors: "true"
    nginx.ingress.kubernetes.io/cors-allow-origin: "*"
    nginx.ingress.kubernetes.io/cors-allow-methods: "GET, POST, PUT, DELETE, OPTIONS"

    # === PROXY ===
    nginx.ingress.kubernetes.io/proxy-body-size: "50m"
    nginx.ingress.kubernetes.io/proxy-read-timeout: "300"

    # === PERFORMANCE ===
    nginx.ingress.kubernetes.io/enable-gzip: "true"

    # === HEADERS CUSTOM ===
    nginx.ingress.kubernetes.io/configuration-snippet: |
      more_set_headers "X-Environment: lab";
      more_set_headers "X-Frame-Options: SAMEORIGIN";
      more_set_headers "X-Content-Type-Options: nosniff";
      more_set_headers "X-XSS-Protection: 1; mode=block";

    # === MONITORING ===
    nginx.ingress.kubernetes.io/enable-access-log: "true"
    nginx.ingress.kubernetes.io/enable-rewrite-log: "true"
spec:
  tls:
  - hosts:
    - lab.local
    secretName: lab-tls
  rules:
  - host: lab.local
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: lab-app
            port:
              number: 8080
```

## Résumé et points clés

### Ce qu'il faut retenir

✅ **Les annotations personnalisent** le comportement par Ingress
✅ **Portée limitée** : Une annotation affecte toute l'Ingress
✅ **Préfixe important** : `nginx.ingress.kubernetes.io/`
✅ **Snippets pour cas spéciaux** : Server et configuration snippets
✅ **Combinaisons possibles** : Plusieurs annotations ensemble
✅ **Test systématique** : Toujours vérifier la configuration générée
✅ **Documentation cruciale** : Commentez vos annotations

### Catégories principales d'annotations

1. **Redirections** : SSL, rewrites, redirections permanentes
2. **Authentification** : Basic auth, OAuth2, IP whitelist
3. **Rate limiting** : Protection contre les abus
4. **Headers** : CORS, sécurité, personnalisation
5. **Proxy** : Timeouts, tailles, buffering
6. **WebSocket/SSE** : Support temps réel
7. **Cache** : Optimisation des performances
8. **Sécurité** : HSTS, WAF, SSL/TLS

### Bonnes pratiques

#### Organisation des annotations

```yaml
metadata:
  annotations:
    # === SECTION SÉCURITÉ ===
    nginx.ingress.kubernetes.io/ssl-redirect: "true"

    # === SECTION PERFORMANCE ===
    nginx.ingress.kubernetes.io/enable-gzip: "true"

    # === SECTION AUTHENTIFICATION ===
    nginx.ingress.kubernetes.io/auth-type: basic
```

#### Annotations par environnement

**Développement** :
- Pas de SSL obligatoire
- CORS permissif (`*`)
- Logs détaillés
- Timeouts généreux

**Staging** :
- SSL activé
- CORS restrictif
- Rate limiting modéré
- Monitoring activé

**Production** :
- HSTS strict
- WAF activé
- Rate limiting strict
- Cache optimisé

### Pièges courants à éviter

#### ❌ Mauvaises pratiques

```yaml
# MAUVAIS : Annotations contradictoires
annotations:
  nginx.ingress.kubernetes.io/ssl-redirect: "false"
  nginx.ingress.kubernetes.io/force-ssl-redirect: "true"

# MAUVAIS : Valeurs sans guillemets
annotations:
  nginx.ingress.kubernetes.io/limit-rps: 10  # Doit être "10"

# MAUVAIS : Préfixe incorrect
annotations:
  ingress.kubernetes.io/ssl-redirect: "true"  # Manque 'nginx.'
```

#### ✅ Bonnes pratiques

```yaml
# BON : Valeurs entre guillemets
annotations:
  nginx.ingress.kubernetes.io/limit-rps: "10"

# BON : Documentation inline
annotations:
  # Limite à 10 req/sec pour éviter la surcharge
  nginx.ingress.kubernetes.io/limit-rps: "10"

# BON : Groupement logique
annotations:
  # Sécurité
  nginx.ingress.kubernetes.io/ssl-redirect: "true"
  nginx.ingress.kubernetes.io/hsts: "true"
```

### Tableau de référence rapide

| Cas d'usage | Annotation clé | Valeur exemple |
|------------|---------------|----------------|
| Forcer HTTPS | `ssl-redirect` | `"true"` |
| Auth basique | `auth-type` | `basic` |
| Limite requêtes | `limit-rps` | `"100"` |
| CORS | `enable-cors` | `"true"` |
| Upload gros fichiers | `proxy-body-size` | `"100m"` |
| WebSocket | `proxy-http-version` | `"1.1"` |
| Cache statique | `configuration-snippet` | Cache headers |
| WAF | `enable-modsecurity` | `"true"` |

### Commandes de diagnostic

```bash
# Vérifier les annotations appliquées
microk8s kubectl get ingress mon-ingress -o yaml | grep annotations -A 20

# Tester une annotation spécifique
microk8s kubectl annotate ingress mon-ingress \
  nginx.ingress.kubernetes.io/limit-rps="5" --overwrite

# Supprimer une annotation
microk8s kubectl annotate ingress mon-ingress \
  nginx.ingress.kubernetes.io/limit-rps-

# Voir l'effet dans NGINX
microk8s kubectl exec -n ingress [pod] -- nginx -T | less
```

## Cas d'usage avancés pour un lab

### Multi-tenant avec isolation

```yaml
# ingress-tenant-a.yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: tenant-a
  namespace: tenant-a
  annotations:
    nginx.ingress.kubernetes.io/whitelist-source-range: "10.0.1.0/24"
    nginx.ingress.kubernetes.io/limit-rps: "100"
    nginx.ingress.kubernetes.io/auth-secret: tenant-a-auth
spec:
  rules:
  - host: tenant-a.lab.local
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: app-tenant-a
            port:
              number: 80
---
# ingress-tenant-b.yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: tenant-b
  namespace: tenant-b
  annotations:
    nginx.ingress.kubernetes.io/whitelist-source-range: "10.0.2.0/24"
    nginx.ingress.kubernetes.io/limit-rps: "50"
    nginx.ingress.kubernetes.io/auth-secret: tenant-b-auth
spec:
  rules:
  - host: tenant-b.lab.local
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: app-tenant-b
            port:
              number: 80
```

### A/B Testing avec annotations

```yaml
# ingress-canary.yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: production
spec:
  rules:
  - host: app.lab.local
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
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: canary
  annotations:
    # Activer le mode canary
    nginx.ingress.kubernetes.io/canary: "true"

    # 20% du trafic vers la version canary
    nginx.ingress.kubernetes.io/canary-weight: "20"

    # Ou par header
    # nginx.ingress.kubernetes.io/canary-by-header: "x-canary"

    # Ou par cookie
    # nginx.ingress.kubernetes.io/canary-by-cookie: "canary"
spec:
  rules:
  - host: app.lab.local
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

### Circuit breaker pattern

```yaml
# ingress-circuit-breaker.yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: resilient-app
  annotations:
    # Retry automatique
    nginx.ingress.kubernetes.io/proxy-next-upstream: "error timeout http_503"
    nginx.ingress.kubernetes.io/proxy-next-upstream-tries: "3"
    nginx.ingress.kubernetes.io/proxy-next-upstream-timeout: "10"

    # Timeout court pour fail-fast
    nginx.ingress.kubernetes.io/proxy-connect-timeout: "5"
    nginx.ingress.kubernetes.io/proxy-read-timeout: "10"

    # Custom error pages
    nginx.ingress.kubernetes.io/custom-http-errors: "503,504"
    nginx.ingress.kubernetes.io/default-backend: error-handler

    # Health checks
    nginx.ingress.kubernetes.io/server-snippet: |
      location /health {
        access_log off;
        return 200 "healthy\n";
      }
spec:
  rules:
  - host: resilient.lab.local
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: main-app
            port:
              number: 80
```

## Troubleshooting des annotations

### Problèmes fréquents et solutions

#### 1. Annotation non prise en compte

```bash
# Vérifier la syntaxe
microk8s kubectl get ingress mon-ingress -o yaml

# Vérifier les logs du controller
microk8s kubectl logs -n ingress [nginx-pod] | grep "error"

# Causes communes :
# - Guillemets manquants autour de la valeur
# - Préfixe incorrect (nginx.ingress.kubernetes.io/)
# - Annotation non supportée par la version
```

#### 2. Conflits entre annotations

```yaml
# Problème : Annotations contradictoires
annotations:
  nginx.ingress.kubernetes.io/rewrite-target: "/"
  nginx.ingress.kubernetes.io/app-root: "/app"

# Solution : Choisir une seule méthode
annotations:
  nginx.ingress.kubernetes.io/rewrite-target: "/"
```

#### 3. Performance dégradée

```yaml
# Problème : Trop de snippets complexes
annotations:
  nginx.ingress.kubernetes.io/server-snippet: |
    # 100 lignes de configuration...

# Solution : Utiliser un ConfigMap global ou
# diviser en plusieurs Ingress
```

#### 4. Sécurité compromise

```yaml
# Problème : Configuration trop permissive
annotations:
  nginx.ingress.kubernetes.io/cors-allow-origin: "*"
  nginx.ingress.kubernetes.io/enable-modsecurity: "false"

# Solution : Restreindre en production
annotations:
  nginx.ingress.kubernetes.io/cors-allow-origin: "https://app.exemple.com"
  nginx.ingress.kubernetes.io/enable-modsecurity: "true"
```

## Migration et évolution

### Passage de dev à production

```yaml
# ingress-dev.yaml (Développement)
metadata:
  annotations:
    nginx.ingress.kubernetes.io/ssl-redirect: "false"
    nginx.ingress.kubernetes.io/enable-cors: "true"
    nginx.ingress.kubernetes.io/cors-allow-origin: "*"

---
# ingress-prod.yaml (Production)
metadata:
  annotations:
    nginx.ingress.kubernetes.io/ssl-redirect: "true"
    nginx.ingress.kubernetes.io/hsts: "true"
    nginx.ingress.kubernetes.io/enable-cors: "true"
    nginx.ingress.kubernetes.io/cors-allow-origin: "https://app.production.com"
    nginx.ingress.kubernetes.io/enable-modsecurity: "true"
    nginx.ingress.kubernetes.io/limit-rps: "1000"
```

### Stratégie de test

1. **Environnement de test isolé** : Créer une Ingress de test
2. **Application progressive** : Ajouter une annotation à la fois
3. **Monitoring** : Surveiller les métriques après chaque changement
4. **Rollback rapide** : Garder l'ancienne configuration

## Prochaine étape

Vous maîtrisez maintenant les middleware et annotations pour personnaliser finement vos Ingress. La section suivante (6.6) se concentrera sur la gestion des redirections HTTPS et la sécurisation de vos connexions.

---

💡 **Conseil final** : Les annotations sont puissantes mais peuvent rapidement devenir complexes. Documentez toujours pourquoi vous utilisez une annotation spécifique et testez l'impact sur les performances. Dans un lab, expérimentez librement, mais en production, privilégiez la simplicité et la sécurité.

⏭️
