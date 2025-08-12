🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 6.7 Exemples pratiques d'Ingress

## Introduction

Cette section rassemble des exemples concrets et complets d'Ingress pour des cas d'usage réels. Chaque exemple est documenté, testé et prêt à être adapté à vos besoins. Ces configurations couvrent les scénarios les plus courants dans un lab personnel ou un environnement de développement.

## 1. Blog WordPress avec HTTPS

### Cas d'usage
Héberger un blog WordPress accessible publiquement avec certificat SSL automatique.

```yaml
# wordpress-ingress.yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: wordpress-ingress
  namespace: wordpress
  annotations:
    # Redirection HTTPS obligatoire
    nginx.ingress.kubernetes.io/ssl-redirect: "true"

    # HSTS pour sécurité
    nginx.ingress.kubernetes.io/hsts: "true"
    nginx.ingress.kubernetes.io/hsts-max-age: "31536000"

    # Taille des uploads (pour les médias)
    nginx.ingress.kubernetes.io/proxy-body-size: "64m"

    # Timeout pour les opérations longues
    nginx.ingress.kubernetes.io/proxy-read-timeout: "600"
    nginx.ingress.kubernetes.io/proxy-send-timeout: "600"

    # Certificat Let's Encrypt automatique
    cert-manager.io/cluster-issuer: "letsencrypt-prod"

    # Headers de sécurité
    nginx.ingress.kubernetes.io/configuration-snippet: |
      more_set_headers "X-Frame-Options: SAMEORIGIN";
      more_set_headers "X-Content-Type-Options: nosniff";
      more_set_headers "X-XSS-Protection: 1; mode=block";
      more_set_headers "Referrer-Policy: strict-origin-when-cross-origin";
spec:
  tls:
  - hosts:
    - blog.monsite.com
    - www.blog.monsite.com
    secretName: wordpress-tls

  rules:
  # Version sans www
  - host: blog.monsite.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: wordpress
            port:
              number: 80

  # Version avec www
  - host: www.blog.monsite.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: wordpress
            port:
              number: 80
---
# Service WordPress
apiVersion: v1
kind: Service
metadata:
  name: wordpress
  namespace: wordpress
spec:
  selector:
    app: wordpress
  ports:
  - port: 80
    targetPort: 80
  type: ClusterIP
```

### Points clés
- Upload de 64MB pour les images/vidéos
- Timeout généreux pour l'admin
- Headers de sécurité standards
- Support www et non-www

## 2. API REST avec versioning

### Cas d'usage
API REST avec plusieurs versions, rate limiting et CORS configuré.

```yaml
# api-versioned-ingress.yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: api-ingress
  namespace: api
  annotations:
    # HTTPS obligatoire pour l'API
    nginx.ingress.kubernetes.io/ssl-redirect: "true"

    # Rate limiting pour protection
    nginx.ingress.kubernetes.io/limit-rps: "100"
    nginx.ingress.kubernetes.io/limit-connections: "10"
    nginx.ingress.kubernetes.io/limit-burst-multiplier: "2"

    # CORS pour applications frontend
    nginx.ingress.kubernetes.io/enable-cors: "true"
    nginx.ingress.kubernetes.io/cors-allow-origin: "https://app.monsite.com, https://mobile.monsite.com"
    nginx.ingress.kubernetes.io/cors-allow-methods: "GET, POST, PUT, DELETE, OPTIONS, PATCH"
    nginx.ingress.kubernetes.io/cors-allow-headers: "DNT,Keep-Alive,User-Agent,X-Requested-With,If-Modified-Since,Cache-Control,Content-Type,Range,Authorization,X-API-Key"
    nginx.ingress.kubernetes.io/cors-expose-headers: "Content-Length,Content-Range,X-Total-Count,X-Page-Count"
    nginx.ingress.kubernetes.io/cors-allow-credentials: "true"
    nginx.ingress.kubernetes.io/cors-max-age: "86400"

    # Timeouts API
    nginx.ingress.kubernetes.io/proxy-connect-timeout: "5"
    nginx.ingress.kubernetes.io/proxy-read-timeout: "60"

    # Compression
    nginx.ingress.kubernetes.io/enable-gzip: "true"
    nginx.ingress.kubernetes.io/gzip-types: "application/json application/xml text/plain"

    # Métriques
    nginx.ingress.kubernetes.io/enable-metrics: "true"

    # Rewrite pour supprimer /api du path
    nginx.ingress.kubernetes.io/rewrite-target: /$2
spec:
  tls:
  - hosts:
    - api.monsite.com
    secretName: api-tls

  rules:
  - host: api.monsite.com
    http:
      paths:
      # API v1 (maintenance uniquement)
      - path: /v1(/|$)(.*)
        pathType: Prefix
        backend:
          service:
            name: api-v1
            port:
              number: 8080

      # API v2 (stable)
      - path: /v2(/|$)(.*)
        pathType: Prefix
        backend:
          service:
            name: api-v2
            port:
              number: 8080

      # API v3 (beta)
      - path: /v3-beta(/|$)(.*)
        pathType: Prefix
        backend:
          service:
            name: api-v3-beta
            port:
              number: 8080

      # Documentation Swagger
      - path: /docs
        pathType: Prefix
        backend:
          service:
            name: api-docs
            port:
              number: 8080

      # Health check (pas de rewrite)
      - path: /health
        pathType: Exact
        backend:
          service:
            name: api-v2
            port:
              number: 8080
---
# Ingress pour les webhooks (sans rate limiting)
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: webhook-ingress
  namespace: api
  annotations:
    # Pas de redirection HTTPS (certains webhooks n'aiment pas)
    nginx.ingress.kubernetes.io/ssl-redirect: "false"

    # Timeout plus long pour les webhooks
    nginx.ingress.kubernetes.io/proxy-read-timeout: "300"

    # Pas de rate limiting
spec:
  rules:
  - host: webhooks.monsite.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: webhook-processor
            port:
              number: 8080
```

### Points clés
- Versioning d'API clair (v1, v2, v3-beta)
- Rate limiting pour protection
- CORS configuré pour plusieurs origines
- Webhooks sur domaine séparé sans restrictions

## 3. Application SPA (Single Page Application)

### Cas d'usage
Application React/Vue/Angular avec routage côté client et API backend.

```yaml
# spa-ingress.yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: spa-ingress
  namespace: frontend
  annotations:
    # HTTPS obligatoire
    nginx.ingress.kubernetes.io/ssl-redirect: "true"

    # Support du routage côté client (SPA)
    nginx.ingress.kubernetes.io/configuration-snippet: |
      # Toujours retourner index.html pour les routes non trouvées
      try_files $uri $uri/ /index.html;

    # Cache agressif pour les assets statiques
    nginx.ingress.kubernetes.io/server-snippet: |
      location ~* \.(js|css|png|jpg|jpeg|gif|ico|svg|woff|woff2|ttf|eot)$ {
        expires 1y;
        add_header Cache-Control "public, immutable";
      }
      location ~* \.(html|json)$ {
        expires 1h;
        add_header Cache-Control "public, must-revalidate";
      }

    # Compression
    nginx.ingress.kubernetes.io/enable-gzip: "true"
    nginx.ingress.kubernetes.io/gzip-level: "6"

    # CSP Header
    nginx.ingress.kubernetes.io/add-headers: |
      Content-Security-Policy: default-src 'self'; script-src 'self' 'unsafe-inline' https://cdn.jsdelivr.net; style-src 'self' 'unsafe-inline' https://fonts.googleapis.com; font-src 'self' https://fonts.gstatic.com; img-src 'self' data: https:; connect-src 'self' https://api.monsite.com;
spec:
  tls:
  - hosts:
    - app.monsite.com
    secretName: spa-tls

  rules:
  - host: app.monsite.com
    http:
      paths:
      # Frontend SPA
      - path: /
        pathType: Prefix
        backend:
          service:
            name: frontend
            port:
              number: 80

      # Proxy vers l'API backend
      - path: /api
        pathType: Prefix
        backend:
          service:
            name: backend-api
            port:
              number: 8080

      # WebSocket pour temps réel
      - path: /ws
        pathType: Prefix
        backend:
          service:
            name: websocket-server
            port:
              number: 3000
---
# Configuration WebSocket séparée
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: websocket-ingress
  namespace: frontend
  annotations:
    nginx.ingress.kubernetes.io/proxy-http-version: "1.1"
    nginx.ingress.kubernetes.io/configuration-snippet: |
      proxy_set_header Upgrade $http_upgrade;
      proxy_set_header Connection "upgrade";
    nginx.ingress.kubernetes.io/proxy-read-timeout: "3600"
    nginx.ingress.kubernetes.io/proxy-send-timeout: "3600"
spec:
  tls:
  - hosts:
    - app.monsite.com
    secretName: spa-tls

  rules:
  - host: app.monsite.com
    http:
      paths:
      - path: /ws
        pathType: Prefix
        backend:
          service:
            name: websocket-server
            port:
              number: 3000
```

### Points clés
- Support du routage SPA (toujours retourner index.html)
- Cache agressif pour les assets (1 an)
- CSP headers pour sécurité
- WebSocket configuré séparément

## 4. Stack de microservices

### Cas d'usage
Architecture microservices complète avec gateway, services et monitoring.

```yaml
# microservices-ingress.yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: microservices-gateway
  namespace: microservices
  annotations:
    # Gateway principal avec auth
    nginx.ingress.kubernetes.io/ssl-redirect: "true"

    # Authentification OAuth2
    nginx.ingress.kubernetes.io/auth-url: "https://auth.monsite.com/oauth2/auth"
    nginx.ingress.kubernetes.io/auth-signin: "https://auth.monsite.com/oauth2/start?rd=$escaped_request_uri"
    nginx.ingress.kubernetes.io/auth-response-headers: "X-User,X-Email,X-Groups"

    # Rate limiting global
    nginx.ingress.kubernetes.io/limit-rps: "200"

    # Tracing distribué
    nginx.ingress.kubernetes.io/enable-opentracing: "true"
    nginx.ingress.kubernetes.io/opentracing-trust-incoming-span: "true"

    # Circuit breaker
    nginx.ingress.kubernetes.io/proxy-next-upstream: "error timeout http_503"
    nginx.ingress.kubernetes.io/proxy-next-upstream-tries: "3"

    # Rewrite paths
    nginx.ingress.kubernetes.io/rewrite-target: /$2
spec:
  tls:
  - hosts:
    - services.monsite.com
    secretName: microservices-tls

  rules:
  - host: services.monsite.com
    http:
      paths:
      # Service utilisateurs
      - path: /users(/|$)(.*)
        pathType: Prefix
        backend:
          service:
            name: users-service
            port:
              number: 8080

      # Service produits
      - path: /products(/|$)(.*)
        pathType: Prefix
        backend:
          service:
            name: products-service
            port:
              number: 8080

      # Service commandes
      - path: /orders(/|$)(.*)
        pathType: Prefix
        backend:
          service:
            name: orders-service
            port:
              number: 8080

      # Service paiements (sécurité renforcée)
      - path: /payments(/|$)(.*)
        pathType: Prefix
        backend:
          service:
            name: payments-service
            port:
              number: 8080

      # Service notifications
      - path: /notifications(/|$)(.*)
        pathType: Prefix
        backend:
          service:
            name: notifications-service
            port:
              number: 8080

      # GraphQL Gateway
      - path: /graphql
        pathType: Exact
        backend:
          service:
            name: graphql-gateway
            port:
              number: 4000
---
# Ingress pour le service d'authentification
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: auth-ingress
  namespace: microservices
  annotations:
    nginx.ingress.kubernetes.io/ssl-redirect: "true"
    # Pas d'auth pour le service d'auth lui-même
spec:
  tls:
  - hosts:
    - auth.monsite.com
    secretName: auth-tls

  rules:
  - host: auth.monsite.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: oauth2-proxy
            port:
              number: 4180
---
# Ingress pour le monitoring (accès restreint)
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: monitoring-ingress
  namespace: monitoring
  annotations:
    nginx.ingress.kubernetes.io/ssl-redirect: "true"
    nginx.ingress.kubernetes.io/whitelist-source-range: "10.0.0.0/8,192.168.1.0/24"
    nginx.ingress.kubernetes.io/auth-type: basic
    nginx.ingress.kubernetes.io/auth-secret: monitoring-auth
spec:
  tls:
  - hosts:
    - monitoring.monsite.com
    secretName: monitoring-tls

  rules:
  - host: monitoring.monsite.com
    http:
      paths:
      # Grafana
      - path: /
        pathType: Prefix
        backend:
          service:
            name: grafana
            port:
              number: 3000

      # Prometheus
      - path: /prometheus
        pathType: Prefix
        backend:
          service:
            name: prometheus
            port:
              number: 9090

      # Jaeger
      - path: /tracing
        pathType: Prefix
        backend:
          service:
            name: jaeger-query
            port:
              number: 16686
```

### Points clés
- OAuth2 pour authentification centralisée
- Tracing distribué activé
- Circuit breaker pour résilience
- Monitoring sur domaine séparé avec IP whitelist

## 5. Environnement de développement complet

### Cas d'usage
Stack de développement avec tous les outils nécessaires.

```yaml
# dev-environment-ingress.yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: dev-tools-ingress
  namespace: dev-tools
  annotations:
    # Pas de HTTPS en dev local
    nginx.ingress.kubernetes.io/ssl-redirect: "false"

    # Timeouts généreux pour le développement
    nginx.ingress.kubernetes.io/proxy-read-timeout: "3600"
    nginx.ingress.kubernetes.io/proxy-send-timeout: "3600"

    # Gros uploads pour les artifacts
    nginx.ingress.kubernetes.io/proxy-body-size: "500m"

    # CORS permissif en dev
    nginx.ingress.kubernetes.io/enable-cors: "true"
    nginx.ingress.kubernetes.io/cors-allow-origin: "*"
    nginx.ingress.kubernetes.io/cors-allow-methods: "*"
    nginx.ingress.kubernetes.io/cors-allow-headers: "*"
spec:
  rules:
  - host: dev.local
    http:
      paths:
      # GitLab
      - path: /gitlab
        pathType: Prefix
        backend:
          service:
            name: gitlab
            port:
              number: 80

      # Jenkins
      - path: /jenkins
        pathType: Prefix
        backend:
          service:
            name: jenkins
            port:
              number: 8080

      # SonarQube
      - path: /sonar
        pathType: Prefix
        backend:
          service:
            name: sonarqube
            port:
              number: 9000

      # Nexus Repository
      - path: /nexus
        pathType: Prefix
        backend:
          service:
            name: nexus
            port:
              number: 8081

      # Portainer
      - path: /portainer
        pathType: Prefix
        backend:
          service:
            name: portainer
            port:
              number: 9000

      # pgAdmin
      - path: /pgadmin
        pathType: Prefix
        backend:
          service:
            name: pgadmin
            port:
              number: 80

      # Redis Commander
      - path: /redis
        pathType: Prefix
        backend:
          service:
            name: redis-commander
            port:
              number: 8081

      # Adminer (DB admin)
      - path: /adminer
        pathType: Prefix
        backend:
          service:
            name: adminer
            port:
              number: 8080

      # Documentation
      - path: /docs
        pathType: Prefix
        backend:
          service:
            name: mkdocs
            port:
              number: 8000

      # MailCatcher (emails de dev)
      - path: /mail
        pathType: Prefix
        backend:
          service:
            name: mailcatcher
            port:
              number: 1080
```

### Points clés
- Pas de SSL en dev local
- Timeouts très longs (1h)
- CORS complètement ouvert
- Tous les outils sur un seul domaine

## 6. Services personnels (Homelab)

### Cas d'usage
Services personnels pour usage domestique.

```yaml
# homelab-ingress.yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: homelab-services
  namespace: homelab
  annotations:
    nginx.ingress.kubernetes.io/ssl-redirect: "true"
    cert-manager.io/cluster-issuer: "letsencrypt-prod"

    # Authentification basique pour tout
    nginx.ingress.kubernetes.io/auth-type: basic
    nginx.ingress.kubernetes.io/auth-secret: homelab-auth
    nginx.ingress.kubernetes.io/auth-realm: "Homelab Login"

    # IP whitelist (réseau local uniquement)
    nginx.ingress.kubernetes.io/whitelist-source-range: "192.168.1.0/24,10.0.0.0/8"
spec:
  tls:
  - hosts:
    - "*.home.mondomaine.com"
    secretName: homelab-wildcard-tls

  rules:
  # Nextcloud
  - host: cloud.home.mondomaine.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: nextcloud
            port:
              number: 80

  # Jellyfin/Plex
  - host: media.home.mondomaine.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: jellyfin
            port:
              number: 8096

  # Home Assistant
  - host: ha.home.mondomaine.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: home-assistant
            port:
              number: 8123

  # Bitwarden/Vaultwarden
  - host: vault.home.mondomaine.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: vaultwarden
            port:
              number: 80

  # Pi-hole
  - host: dns.home.mondomaine.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: pihole
            port:
              number: 80

  # Paperless-ng
  - host: docs.home.mondomaine.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: paperless
            port:
              number: 8000

  # PhotoPrism
  - host: photos.home.mondomaine.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: photoprism
            port:
              number: 2342

  # Bookstack
  - host: wiki.home.mondomaine.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: bookstack
            port:
              number: 80
---
# WebSocket pour Home Assistant
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: homeassistant-websocket
  namespace: homelab
  annotations:
    nginx.ingress.kubernetes.io/proxy-http-version: "1.1"
    nginx.ingress.kubernetes.io/configuration-snippet: |
      proxy_set_header Upgrade $http_upgrade;
      proxy_set_header Connection "upgrade";
spec:
  tls:
  - hosts:
    - ha.home.mondomaine.com
    secretName: homelab-wildcard-tls

  rules:
  - host: ha.home.mondomaine.com
    http:
      paths:
      - path: /api/websocket
        pathType: Exact
        backend:
          service:
            name: home-assistant
            port:
              number: 8123
```

### Points clés
- Certificat wildcard pour tous les sous-domaines
- Authentification basique globale
- Restriction IP au réseau local
- WebSocket séparé pour Home Assistant

## 7. E-commerce avec haute disponibilité

### Cas d'usage
Site e-commerce avec cache, CDN et optimisations.

```yaml
# ecommerce-ingress.yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: shop-ingress
  namespace: ecommerce
  annotations:
    # Sécurité maximale
    nginx.ingress.kubernetes.io/ssl-redirect: "true"
    nginx.ingress.kubernetes.io/hsts: "true"
    nginx.ingress.kubernetes.io/hsts-max-age: "63072000"
    nginx.ingress.kubernetes.io/hsts-include-subdomains: "true"
    nginx.ingress.kubernetes.io/hsts-preload: "true"

    # WAF activé
    nginx.ingress.kubernetes.io/enable-modsecurity: "true"
    nginx.ingress.kubernetes.io/enable-owasp-modsecurity-crs: "true"

    # Rate limiting anti-DDoS
    nginx.ingress.kubernetes.io/limit-rps: "50"
    nginx.ingress.kubernetes.io/limit-connections: "20"

    # Session affinity pour panier
    nginx.ingress.kubernetes.io/affinity: "cookie"
    nginx.ingress.kubernetes.io/affinity-mode: "persistent"
    nginx.ingress.kubernetes.io/session-cookie-name: "shop-session"
    nginx.ingress.kubernetes.io/session-cookie-expires: "86400"

    # Cache pour performances
    nginx.ingress.kubernetes.io/proxy-buffering: "on"
    nginx.ingress.kubernetes.io/server-snippet: |
      # Cache pour images produits
      location ~* ^/media/catalog/.*\.(jpg|jpeg|png|gif|webp)$ {
        expires 30d;
        add_header Cache-Control "public, immutable";
        add_header X-Cache-Status $upstream_cache_status;
      }

      # Cache pour CSS/JS
      location ~* \.(css|js)$ {
        expires 7d;
        add_header Cache-Control "public, must-revalidate";
      }

      # Pas de cache pour le panier et checkout
      location ~* ^/(cart|checkout|customer) {
        add_header Cache-Control "no-store, no-cache, must-revalidate";
      }

    # Compression maximale
    nginx.ingress.kubernetes.io/enable-gzip: "true"
    nginx.ingress.kubernetes.io/gzip-level: "9"

    # Headers de sécurité
    nginx.ingress.kubernetes.io/configuration-snippet: |
      more_set_headers "X-Frame-Options: DENY";
      more_set_headers "X-Content-Type-Options: nosniff";
      more_set_headers "Referrer-Policy: strict-origin-when-cross-origin";
      more_set_headers "Permissions-Policy: geolocation=(), microphone=(), camera=()";
spec:
  tls:
  - hosts:
    - shop.monsite.com
    - www.shop.monsite.com
    secretName: shop-tls

  rules:
  - host: shop.monsite.com
    http:
      paths:
      # Frontend boutique
      - path: /
        pathType: Prefix
        backend:
          service:
            name: shop-frontend
            port:
              number: 80

      # API backend
      - path: /api
        pathType: Prefix
        backend:
          service:
            name: shop-api
            port:
              number: 8080

      # Admin (IP restreinte)
      - path: /admin
        pathType: Prefix
        backend:
          service:
            name: shop-admin
            port:
              number: 8080

  # Redirection www
  - host: www.shop.monsite.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: shop-frontend
            port:
              number: 80
---
# Admin avec restrictions supplémentaires
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: shop-admin-secure
  namespace: ecommerce
  annotations:
    nginx.ingress.kubernetes.io/whitelist-source-range: "203.0.113.0/24"
    nginx.ingress.kubernetes.io/auth-type: basic
    nginx.ingress.kubernetes.io/auth-secret: admin-auth
spec:
  tls:
  - hosts:
    - admin.shop.monsite.com
    secretName: shop-admin-tls

  rules:
  - host: admin.shop.monsite.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: shop-admin
            port:
              number: 8080
```

### Points clés
- WAF activé pour sécurité
- Session affinity pour le panier
- Cache agressif pour les médias
- Admin sur domaine séparé avec IP whitelist

## 8. Plateforme d'apprentissage

### Cas d'usage
LMS (Learning Management System) avec vidéos et quiz.

```yaml
# lms-ingress.yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: lms-ingress
  namespace: education
  annotations:
    nginx.ingress.kubernetes.io/ssl-redirect: "true"

    # Gros uploads pour les vidéos de cours
    nginx.ingress.kubernetes.io/proxy-body-size: "2048m"

    # Timeout long pour upload vidéos
    nginx.ingress.kubernetes.io/proxy-read-timeout: "1800"
    nginx.ingress.kubernetes.io/proxy-send-timeout: "1800"

    # Streaming vidéo optimisé
    nginx.ingress.kubernetes.io/proxy-buffering: "off"
    nginx.ingress.kubernetes.io/configuration-snippet: |
      # Support du streaming vidéo
      proxy_set_header Range $http_range;
      proxy_set_header If-Range $http_if_range;

      # MP4 streaming
      location ~* \.mp4$ {
        mp4;
        mp4_buffer_size 1m;
        mp4_max_buffer_size 5m;
      }

    # Session longue pour les cours
    nginx.ingress.kubernetes.io/affinity: "cookie"
    nginx.ingress.kubernetes.io/session-cookie-expires: "14400"  # 4 heures

    # Protection contre le téléchargement
    nginx.ingress.kubernetes.io/server-snippet: |
      # Empêcher le téléchargement direct des vidéos
      location ~* \.(mp4|webm|mkv)$ {
        valid_referers none blocked *.monsite.com;
        if ($invalid_referer) {
          return 403;
        }
        add_header X-Frame-Options "SAMEORIGIN";
      }
spec:
  tls:
  - hosts:
    - learn.monsite.com
    secretName: lms-tls

  rules:
  - host: learn.monsite.com
    http:
      paths:
      # Application principale
      - path: /
        pathType: Prefix
        backend:
          service:
            name: lms-frontend
            port:
              number: 3000

      # API pour les quiz et progression
      - path: /api
        pathType: Prefix
        backend:
          service:
            name: lms-api
            port:
              number: 8080

      # Streaming vidéo
      - path: /media
        pathType: Prefix
        backend:
          service:
            name: video-streaming
            port:
              number: 8888

      # Live streaming (webinaires)
      - path: /live
        pathType: Prefix
        backend:
          service:
            name: rtmp-server
            port:
              number: 1935
```

### Points clés
- Upload de 2GB pour les vidéos
- Streaming optimisé (mp4 module)
- Protection anti-téléchargement
- Sessions longues pour les cours

## 9. CI/CD Pipeline complète

### Cas d'usage
Pipeline CI/CD avec GitLab, registre Docker et ArgoCD.

```yaml
# cicd-ingress.yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: gitlab-ingress
  namespace: gitlab
  annotations:
    nginx.ingress.kubernetes.io/ssl-redirect: "true"
    nginx.ingress.kubernetes.io/proxy-body-size: "0"  # Pas de limite
    nginx.ingress.kubernetes.io/proxy-read-timeout: "600"
    nginx.ingress.kubernetes.io/proxy-buffering: "off"

    # GitLab spécifique
    nginx.ingress.kubernetes.io/configuration-snippet: |
      proxy_set_header X-Forwarded-Proto https;
      proxy_set_header X-Forwarded-Ssl on;
spec:
  tls:
  - hosts:
    - git.monsite.com
    - registry.monsite.com
    secretName: gitlab-tls

  rules:
  # GitLab principal
  - host: git.monsite.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: gitlab
            port:
              number: 80

  # Docker Registry
  - host: registry.monsite.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: gitlab-registry
            port:
              number: 5000
---
# ArgoCD avec auth
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: argocd-ingress
  namespace: argocd
  annotations:
    nginx.ingress.kubernetes.io/ssl-redirect: "true"
    nginx.ingress.kubernetes.io/backend-protocol: "HTTPS"
    nginx.ingress.kubernetes.io/force-ssl-redirect: "true"

    # ArgoCD utilise gRPC
    nginx.ingress.kubernetes.io/grpc-backend: "true"
spec:
  tls:
  - hosts:
    - argocd.monsite.com
    secretName: argocd-tls

  rules:
  - host: argocd.monsite.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: argocd-server
            port:
              number: 443
---
# Jenkins avec contexte
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: jenkins-ingress
  namespace: jenkins
  annotations:
    nginx.ingress.kubernetes.io/ssl-redirect: "true"
    nginx.ingress.kubernetes.io/proxy-body-size: "50m"

    # Jenkins context path
    nginx.ingress.kubernetes.io/configuration-snippet: |
      more_set_headers "X-Forwarded-Prefix: /jenkins";
spec:
  tls:
  - hosts:
    - ci.monsite.com
    secretName: jenkins-tls

  rules:
  - host: ci.monsite.com
    http:
      paths:
      - path: /jenkins
        pathType: Prefix
        backend:
          service:
            name: jenkins
            port:
              number: 8080
---
# SonarQube pour analyse de code
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: sonarqube-ingress
  namespace: quality
  annotations:
    nginx.ingress.kubernetes.io/ssl-redirect: "true"
    nginx.ingress.kubernetes.io/proxy-body-size: "10m"
spec:
  tls:
  - hosts:
    - quality.monsite.com
    secretName: sonar-tls

  rules:
  - host: quality.monsite.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: sonarqube
            port:
              number: 9000
```

### Points clés
- GitLab avec registre Docker intégré
- ArgoCD avec support gRPC
- Jenkins avec context path
- Pas de limite pour GitLab (gros repos)

## 10. Gaming et serveurs de jeux

### Cas d'usage
Serveurs de jeux avec panel d'administration.

```yaml
# gaming-ingress.yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: gaming-panel
  namespace: gaming
  annotations:
    nginx.ingress.kubernetes.io/ssl-redirect: "true"

    # WebSocket pour consoles temps réel
    nginx.ingress.kubernetes.io/proxy-http-version: "1.1"
    nginx.ingress.kubernetes.io/configuration-snippet: |
      proxy_set_header Upgrade $http_upgrade;
      proxy_set_header Connection "upgrade";

    # Sessions longues pour les admins
    nginx.ingress.kubernetes.io/proxy-read-timeout: "86400"
spec:
  tls:
  - hosts:
    - gaming.monsite.com
    secretName: gaming-tls

  rules:
  - host: gaming.monsite.com
    http:
      paths:
      # Pterodactyl Panel
      - path: /
        pathType: Prefix
        backend:
          service:
            name: pterodactyl
            port:
              number: 80

      # Stats serveur
      - path: /stats
        pathType: Prefix
        backend:
          service:
            name: game-stats
            port:
              number: 3000
---
# TCP/UDP pour les serveurs de jeux (ConfigMap)
apiVersion: v1
kind: ConfigMap
metadata:
  name: nginx-ingress-tcp-microk8s-conf
  namespace: ingress
data:
  # Minecraft
  25565: "gaming/minecraft:25565"
  # Terraria
  7777: "gaming/terraria:7777"
  # CS:GO
  27015: "gaming/csgo:27015"
---
apiVersion: v1
kind: ConfigMap
metadata:
  name: nginx-ingress-udp-microk8s-conf
  namespace: ingress
data:
  # Minecraft Bedrock
  19132: "gaming/minecraft-bedrock:19132"
  # Source games
  27015: "gaming/source-games:27015"
```

### Points clés
- Panel web pour administration
- WebSocket pour consoles temps réel
- TCP/UDP pour connexions directes aux jeux
- Sessions très longues (24h)

## Template générique réutilisable

### Base pour tout nouveau projet

```yaml
# template-ingress.yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: ${APP_NAME}-ingress
  namespace: ${NAMESPACE}
  annotations:
    # === SÉCURITÉ DE BASE ===
    nginx.ingress.kubernetes.io/ssl-redirect: "${SSL_REDIRECT:-true}"

    # === CERTIFICAT ===
    cert-manager.io/cluster-issuer: "${CERT_ISSUER:-letsencrypt-prod}"

    # === PERFORMANCE ===
    nginx.ingress.kubernetes.io/enable-gzip: "true"
    nginx.ingress.kubernetes.io/gzip-level: "5"

    # === TIMEOUTS ===
    nginx.ingress.kubernetes.io/proxy-connect-timeout: "${CONNECT_TIMEOUT:-10}"
    nginx.ingress.kubernetes.io/proxy-read-timeout: "${READ_TIMEOUT:-60}"
    nginx.ingress.kubernetes.io/proxy-send-timeout: "${SEND_TIMEOUT:-60}"

    # === UPLOAD SIZE ===
    nginx.ingress.kubernetes.io/proxy-body-size: "${MAX_BODY_SIZE:-1m}"

    # === CORS (si nécessaire) ===
    # nginx.ingress.kubernetes.io/enable-cors: "${ENABLE_CORS:-false}"
    # nginx.ingress.kubernetes.io/cors-allow-origin: "${CORS_ORIGINS:-*}"

    # === RATE LIMITING (si nécessaire) ===
    # nginx.ingress.kubernetes.io/limit-rps: "${RATE_LIMIT:-0}"

    # === AUTH (si nécessaire) ===
    # nginx.ingress.kubernetes.io/auth-type: basic
    # nginx.ingress.kubernetes.io/auth-secret: ${AUTH_SECRET}
spec:
  tls:
  - hosts:
    - ${HOSTNAME}
    secretName: ${TLS_SECRET:-$APP_NAME-tls}

  rules:
  - host: ${HOSTNAME}
    http:
      paths:
      - path: ${PATH:-/}
        pathType: ${PATH_TYPE:-Prefix}
        backend:
          service:
            name: ${SERVICE_NAME:-$APP_NAME}
            port:
              number: ${SERVICE_PORT:-80}
```

### Utilisation du template

```bash
# Créer un fichier de variables
cat > myapp.env <<EOF
APP_NAME=myapp
NAMESPACE=production
HOSTNAME=myapp.monsite.com
SERVICE_NAME=myapp-service
SERVICE_PORT=8080
MAX_BODY_SIZE=10m
READ_TIMEOUT=120
ENABLE_CORS=true
CORS_ORIGINS=https://app.monsite.com
EOF

# Générer l'Ingress avec envsubst
envsubst < template-ingress.yaml > myapp-ingress.yaml

# Appliquer
microk8s kubectl apply -f myapp-ingress.yaml
```

## Debugging et validation

### Script de test complet

```bash
#!/bin/bash
# test-ingress.sh

INGRESS_NAME=$1
NAMESPACE=${2:-default}

echo "=== Test Ingress: $INGRESS_NAME ==="

# 1. Vérifier que l'Ingress existe
echo "1. Checking Ingress..."
microk8s kubectl get ingress $INGRESS_NAME -n $NAMESPACE

# 2. Obtenir les hosts
echo -e "\n2. Extracting hosts..."
HOSTS=$(microk8s kubectl get ingress $INGRESS_NAME -n $NAMESPACE -o jsonpath='{.spec.rules[*].host}')
echo "Hosts: $HOSTS"

# 3. Vérifier les backends
echo -e "\n3. Checking backends..."
microk8s kubectl describe ingress $INGRESS_NAME -n $NAMESPACE | grep -A5 "Rules:"

# 4. Tester chaque host
for HOST in $HOSTS; do
    echo -e "\n4. Testing $HOST..."

    # Test HTTP redirect
    echo "  - HTTP redirect:"
    curl -I -s http://$HOST | head -n 1

    # Test HTTPS
    echo "  - HTTPS:"
    curl -I -s -k https://$HOST | head -n 1

    # Test avec header Host
    echo "  - With Host header:"
    CLUSTER_IP=$(microk8s kubectl get svc -n ingress nginx-ingress-microk8s-controller -o jsonpath='{.status.loadBalancer.ingress[0].ip}')
    curl -I -s -H "Host: $HOST" http://$CLUSTER_IP | head -n 1
done

# 5. Vérifier les certificats
echo -e "\n5. Checking certificates..."
for HOST in $HOSTS; do
    echo "  - Certificate for $HOST:"
    echo | openssl s_client -connect $HOST:443 -servername $HOST 2>/dev/null | openssl x509 -noout -subject -dates 2>/dev/null || echo "    No certificate found"
done

# 6. Vérifier les annotations
echo -e "\n6. Active annotations:"
microk8s kubectl get ingress $INGRESS_NAME -n $NAMESPACE -o jsonpath='{.metadata.annotations}' | jq '.' 2>/dev/null || microk8s kubectl get ingress $INGRESS_NAME -n $NAMESPACE -o jsonpath='{.metadata.annotations}'

echo -e "\n=== Test completed ==="
```

### Utilisation du script

```bash
# Rendre le script exécutable
chmod +x test-ingress.sh

# Tester un Ingress
./test-ingress.sh wordpress-ingress wordpress
./test-ingress.sh api-ingress api
```

## Résumé et bonnes pratiques

### Checklist pour chaque Ingress

- [ ] **Sécurité**
  - [ ] SSL/TLS configuré
  - [ ] Redirection HTTPS activée
  - [ ] Headers de sécurité ajoutés
  - [ ] WAF activé si nécessaire

- [ ] **Performance**
  - [ ] Compression activée
  - [ ] Cache configuré pour les assets
  - [ ] Timeouts appropriés
  - [ ] Body size adapté

- [ ] **Résilience**
  - [ ] Rate limiting configuré
  - [ ] Session affinity si nécessaire
  - [ ] Health checks exclus des redirections

- [ ] **Monitoring**
  - [ ] Métriques activées
  - [ ] Logs configurés
  - [ ] Alertes sur expiration certificats

### Patterns à retenir

1. **Séparer les préoccupations** : Une Ingress par type de service
2. **Utiliser des namespaces** : Organisation logique
3. **Certificats wildcard** : Pour les sous-domaines multiples
4. **Templates réutilisables** : Pour standardiser
5. **Documentation inline** : Commentaires dans les YAML

### Anti-patterns à éviter

❌ Une seule Ingress gigantesque pour tout
❌ Pas de limits sur les uploads
❌ CORS avec `*` en production
❌ Pas de HTTPS en production
❌ Timeouts trop courts pour les opérations longues

## Prochaines étapes

Félicitations ! Vous avez maintenant une collection complète d'exemples d'Ingress pour tous les cas d'usage courants. Ces exemples peuvent être :

1. **Adaptés** à vos besoins spécifiques
2. **Combinés** pour des architectures complexes
3. **Automatisés** avec des templates et CI/CD
4. **Monitorés** avec Prometheus et Grafana

Pour aller plus loin, explorez :
- Service Mesh (Istio, Linkerd) pour du routage avancé
- Gateway API (successeur d'Ingress)
- Operators personnalisés pour automatisation

---

💡 **Conseil final** : Commencez simple avec un exemple basique, testez minutieusement, puis ajoutez progressivement des fonctionnalités. La complexité doit être justifiée par de vrais besoins, pas par la technique pour la technique.

⏭️
