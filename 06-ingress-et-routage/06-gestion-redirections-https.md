🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 6.6 Gestion des redirections HTTPS

## Introduction à HTTPS et aux redirections

HTTPS (HTTP Secure) est la version sécurisée du protocole HTTP. Il chiffre les communications entre le navigateur et votre serveur, protégeant ainsi les données sensibles comme les mots de passe, les informations personnelles et les données de paiement.

### Pourquoi HTTPS est essentiel

1. **Sécurité** : Chiffrement des données en transit
2. **Confiance** : Les navigateurs affichent un cadenas vert
3. **SEO** : Google favorise les sites HTTPS
4. **Fonctionnalités modernes** : HTTP/2, Service Workers, géolocalisation
5. **Conformité** : RGPD et autres réglementations exigent le chiffrement

### Le problème des connexions mixtes

Sans redirection HTTPS appropriée, vous risquez :
- Des utilisateurs accédant en HTTP non sécurisé
- Des avertissements "Mixed Content" dans les navigateurs
- Des failles de sécurité potentielles
- Une mauvaise expérience utilisateur

## Comment fonctionnent les redirections HTTPS

### Le flux de redirection

```
1. Utilisateur tape : http://monsite.com
                           ↓
2. Navigateur envoie requête HTTP (port 80)
                           ↓
3. NGINX Ingress reçoit la requête
                           ↓
4. NGINX envoie une redirection 301/308
   Location: https://monsite.com
                           ↓
5. Navigateur suit la redirection
                           ↓
6. Nouvelle requête en HTTPS (port 443)
                           ↓
7. Connexion sécurisée établie
```

### Types de redirections HTTP

| Code | Type | Utilisation | Méthode préservée |
|------|------|-------------|-------------------|
| 301 | Moved Permanently | Redirection permanente classique | Non (POST→GET) |
| 302 | Found | Redirection temporaire | Non (POST→GET) |
| 307 | Temporary Redirect | Redirection temporaire moderne | Oui |
| 308 | Permanent Redirect | Redirection permanente moderne | Oui |

Pour HTTPS, on utilise généralement **301** (permanent) ou **308** (permanent avec préservation de la méthode).

## Configuration de base des redirections HTTPS

### Redirection simple HTTP vers HTTPS

```yaml
# ingress-https-redirect.yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: redirect-to-https
  annotations:
    # Active la redirection automatique vers HTTPS
    nginx.ingress.kubernetes.io/ssl-redirect: "true"
spec:
  # Configuration TLS obligatoire
  tls:
  - hosts:
    - monsite.local
    secretName: monsite-tls-secret
  rules:
  - host: monsite.local
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

### Forcer HTTPS même sans TLS configuré

```yaml
# ingress-force-https.yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: force-https
  annotations:
    # Force HTTPS même si TLS n'est pas configuré dans l'Ingress
    nginx.ingress.kubernetes.io/force-ssl-redirect: "true"
spec:
  rules:
  - host: monsite.local
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

**Note** : `force-ssl-redirect` est utile quand le TLS est géré en amont (CDN, Load Balancer).

### Désactiver la redirection pour certains paths

```yaml
# ingress-selective-redirect.yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: api-no-redirect
  annotations:
    # Pas de redirection pour l'API (webhooks, etc.)
    nginx.ingress.kubernetes.io/ssl-redirect: "false"
spec:
  rules:
  - host: api.monsite.local
    http:
      paths:
      - path: /webhook
        pathType: Prefix
        backend:
          service:
            name: webhook-service
            port:
              number: 8080
---
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: app-with-redirect
  annotations:
    # Redirection pour l'application web
    nginx.ingress.kubernetes.io/ssl-redirect: "true"
spec:
  tls:
  - hosts:
    - app.monsite.local
    secretName: app-tls
  rules:
  - host: app.monsite.local
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: webapp-service
            port:
              number: 3000
```

## Certificats SSL/TLS pour HTTPS

### Création manuelle d'un certificat auto-signé

```bash
# Générer une clé privée
openssl genrsa -out tls.key 2048

# Générer un certificat auto-signé
openssl req -new -x509 -key tls.key -out tls.crt -days 365 \
  -subj "/CN=monsite.local/O=MonLab"

# Créer un Secret Kubernetes
microk8s kubectl create secret tls monsite-tls \
  --cert=tls.crt \
  --key=tls.key
```

### Utilisation du certificat dans Ingress

```yaml
# ingress-with-tls.yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: secure-ingress
  annotations:
    nginx.ingress.kubernetes.io/ssl-redirect: "true"
spec:
  tls:
  - hosts:
    - monsite.local
    - www.monsite.local
    secretName: monsite-tls  # Le Secret créé précédemment
  rules:
  - host: monsite.local
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: mon-service
            port:
              number: 80
  - host: www.monsite.local
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

### Certificats Let's Encrypt avec cert-manager

```yaml
# ingress-letsencrypt.yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: letsencrypt-ingress
  annotations:
    # Redirection HTTPS
    nginx.ingress.kubernetes.io/ssl-redirect: "true"

    # Cert-manager génère automatiquement le certificat
    cert-manager.io/cluster-issuer: "letsencrypt-prod"
spec:
  tls:
  - hosts:
    - monsite.com
    - www.monsite.com
    secretName: monsite-letsencrypt-tls  # Sera créé automatiquement
  rules:
  - host: monsite.com
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

## HSTS (HTTP Strict Transport Security)

### Qu'est-ce que HSTS ?

HSTS force les navigateurs à toujours utiliser HTTPS pour votre domaine, même si l'utilisateur tape `http://`. C'est une protection supplémentaire contre les attaques de type "man-in-the-middle".

### Configuration HSTS complète

```yaml
# ingress-hsts.yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: hsts-enabled
  annotations:
    # Redirection HTTPS
    nginx.ingress.kubernetes.io/ssl-redirect: "true"

    # Activer HSTS
    nginx.ingress.kubernetes.io/hsts: "true"

    # Durée en secondes (1 an)
    nginx.ingress.kubernetes.io/hsts-max-age: "31536000"

    # Inclure les sous-domaines
    nginx.ingress.kubernetes.io/hsts-include-subdomains: "true"

    # Autoriser le preloading (liste des navigateurs)
    nginx.ingress.kubernetes.io/hsts-preload: "true"
spec:
  tls:
  - hosts:
    - secure.monsite.local
    secretName: secure-tls
  rules:
  - host: secure.monsite.local
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: secure-service
            port:
              number: 80
```

### Progression recommandée pour HSTS

```yaml
# Phase 1 : Test (courte durée)
annotations:
  nginx.ingress.kubernetes.io/hsts: "true"
  nginx.ingress.kubernetes.io/hsts-max-age: "300"  # 5 minutes

# Phase 2 : Validation
annotations:
  nginx.ingress.kubernetes.io/hsts: "true"
  nginx.ingress.kubernetes.io/hsts-max-age: "86400"  # 1 jour

# Phase 3 : Production
annotations:
  nginx.ingress.kubernetes.io/hsts: "true"
  nginx.ingress.kubernetes.io/hsts-max-age: "31536000"  # 1 an
  nginx.ingress.kubernetes.io/hsts-include-subdomains: "true"
```

## Redirections avancées

### Redirection www vers non-www (ou inversement)

```yaml
# ingress-www-redirect.yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: www-to-apex
  annotations:
    nginx.ingress.kubernetes.io/ssl-redirect: "true"
    # Redirection permanente de www vers apex
    nginx.ingress.kubernetes.io/permanent-redirect: "https://monsite.com$request_uri"
spec:
  tls:
  - hosts:
    - www.monsite.com
    secretName: www-tls
  rules:
  - host: www.monsite.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: dummy-service  # Service factice, jamais atteint
            port:
              number: 80
---
# Ingress principal (sans www)
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: main-site
  annotations:
    nginx.ingress.kubernetes.io/ssl-redirect: "true"
spec:
  tls:
  - hosts:
    - monsite.com
    secretName: main-tls
  rules:
  - host: monsite.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: main-service
            port:
              number: 80
```

### Redirections conditionnelles

```yaml
# ingress-conditional-redirect.yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: conditional-redirects
  annotations:
    nginx.ingress.kubernetes.io/ssl-redirect: "true"

    # Redirections basées sur des conditions
    nginx.ingress.kubernetes.io/server-snippet: |
      # Redirection mobile
      if ($http_user_agent ~* "Mobile|Android|iPhone") {
        return 301 https://m.monsite.com$request_uri;
      }

      # Redirection maintenance
      set $maintenance 0;
      if (-f /tmp/maintenance.flag) {
        set $maintenance 1;
      }
      if ($maintenance = 1) {
        return 302 https://maintenance.monsite.com;
      }

      # Redirection ancienne URL
      location = /old-page {
        return 301 https://monsite.com/new-page;
      }
spec:
  tls:
  - hosts:
    - monsite.com
    secretName: site-tls
  rules:
  - host: monsite.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: main-service
            port:
              number: 80
```

### Préservation des query parameters

```yaml
# ingress-preserve-params.yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: preserve-params
  annotations:
    nginx.ingress.kubernetes.io/ssl-redirect: "true"

    # Préserver les paramètres lors de la redirection
    nginx.ingress.kubernetes.io/configuration-snippet: |
      if ($scheme = "http") {
        return 301 https://$host$request_uri;
      }

    # Pour des redirections de paths spécifiques
    nginx.ingress.kubernetes.io/server-snippet: |
      location ~* ^/old-api/(.*) {
        return 301 https://$host/new-api/$1$is_args$args;
      }
spec:
  tls:
  - hosts:
    - api.monsite.com
    secretName: api-tls
  rules:
  - host: api.monsite.com
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

## Gestion des ports non-standard

### HTTPS sur port personnalisé

```yaml
# ingress-custom-port.yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: custom-port-redirect
  annotations:
    # Redirection vers port personnalisé
    nginx.ingress.kubernetes.io/server-snippet: |
      if ($scheme = "http") {
        return 301 https://$host:8443$request_uri;
      }
spec:
  tls:
  - hosts:
    - custom.monsite.local
    secretName: custom-tls
  rules:
  - host: custom.monsite.local
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: custom-service
            port:
              number: 80
```

## Backends mixtes HTTP/HTTPS

### Quand le backend utilise aussi HTTPS

```yaml
# ingress-https-backend.yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: https-backend
  annotations:
    # Redirection côté client
    nginx.ingress.kubernetes.io/ssl-redirect: "true"

    # Le backend utilise HTTPS
    nginx.ingress.kubernetes.io/backend-protocol: "HTTPS"

    # Vérifier le certificat du backend (optionnel)
    nginx.ingress.kubernetes.io/proxy-ssl-verify: "false"
spec:
  tls:
  - hosts:
    - secure-backend.monsite.local
    secretName: frontend-tls
  rules:
  - host: secure-backend.monsite.local
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: https-backend-service
            port:
              number: 443  # Port HTTPS du backend
```

## Monitoring et debugging

### Vérification des redirections

```bash
# Test de redirection HTTP vers HTTPS
curl -I http://monsite.local
# Doit retourner : HTTP/1.1 301 Moved Permanently
# Location: https://monsite.local

# Test avec curl verbose
curl -v -L http://monsite.local
# -L suit les redirections
# -v affiche les détails

# Test avec wget
wget --server-response http://monsite.local

# Vérifier les headers HSTS
curl -I https://monsite.local | grep -i strict
# Doit afficher : Strict-Transport-Security: max-age=31536000
```

### Logs des redirections

```yaml
# ingress-redirect-logging.yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: logged-redirects
  annotations:
    nginx.ingress.kubernetes.io/ssl-redirect: "true"

    # Logger les redirections
    nginx.ingress.kubernetes.io/configuration-snippet: |
      access_log /var/log/nginx/redirects.log combined if=$redirect_logged;
      set $redirect_logged 0;
      if ($scheme = "http") {
        set $redirect_logged 1;
      }
```

### Métriques Prometheus

```yaml
# Les redirections sont visibles dans les métriques
nginx_ingress_controller_requests{status="301"}  # Redirections permanentes
nginx_ingress_controller_requests{status="308"}  # Redirections permanentes (méthode préservée)
nginx_ingress_controller_ssl_expire_time_seconds # Expiration des certificats
```

## Cas spéciaux et edge cases

### Health checks et redirections

```yaml
# ingress-health-no-redirect.yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: health-check-special
  annotations:
    # Pas de redirection pour les health checks
    nginx.ingress.kubernetes.io/server-snippet: |
      location /health {
        # Pas de redirection HTTPS pour ce path
        if ($scheme = "https") {
          access_log off;
          return 200 "OK";
        }
        access_log off;
        return 200 "OK";
      }

    # Redirection pour tout le reste
    nginx.ingress.kubernetes.io/ssl-redirect: "true"
spec:
  tls:
  - hosts:
    - app.monsite.local
    secretName: app-tls
  rules:
  - host: app.monsite.local
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

### WebSockets et HTTPS

```yaml
# ingress-websocket-https.yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: websocket-secure
  annotations:
    # Redirection HTTPS
    nginx.ingress.kubernetes.io/ssl-redirect: "true"

    # Support WebSocket sur HTTPS
    nginx.ingress.kubernetes.io/proxy-http-version: "1.1"
    nginx.ingress.kubernetes.io/configuration-snippet: |
      proxy_set_header Upgrade $http_upgrade;
      proxy_set_header Connection $connection_upgrade;
spec:
  tls:
  - hosts:
    - ws.monsite.local
    secretName: ws-tls
  rules:
  - host: ws.monsite.local
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

## Stratégies pour différents environnements

### Développement local

```yaml
# ingress-dev-local.yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: dev-local
  annotations:
    # Pas de redirection en dev local
    nginx.ingress.kubernetes.io/ssl-redirect: "false"
spec:
  rules:
  - host: dev.local
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: dev-service
            port:
              number: 3000
```

### Staging

```yaml
# ingress-staging.yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: staging
  annotations:
    # Redirection mais pas HSTS (pour les tests)
    nginx.ingress.kubernetes.io/ssl-redirect: "true"
    nginx.ingress.kubernetes.io/hsts: "false"
spec:
  tls:
  - hosts:
    - staging.monsite.com
    secretName: staging-tls
  rules:
  - host: staging.monsite.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: staging-service
            port:
              number: 80
```

### Production

```yaml
# ingress-production.yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: production
  annotations:
    # Configuration complète de sécurité
    nginx.ingress.kubernetes.io/ssl-redirect: "true"
    nginx.ingress.kubernetes.io/hsts: "true"
    nginx.ingress.kubernetes.io/hsts-max-age: "31536000"
    nginx.ingress.kubernetes.io/hsts-include-subdomains: "true"
    nginx.ingress.kubernetes.io/hsts-preload: "true"

    # Protocoles SSL stricts
    nginx.ingress.kubernetes.io/ssl-protocols: "TLSv1.2 TLSv1.3"
    nginx.ingress.kubernetes.io/ssl-ciphers: "HIGH:!aNULL:!MD5"
spec:
  tls:
  - hosts:
    - monsite.com
    - www.monsite.com
    secretName: production-tls
  rules:
  - host: monsite.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: production-service
            port:
              number: 80
```

## Exemple complet pour un lab

### Configuration complète avec toutes les redirections

```yaml
# ingress-complete-https.yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: lab-https-complete
  annotations:
    # === REDIRECTIONS DE BASE ===
    nginx.ingress.kubernetes.io/ssl-redirect: "true"

    # === HSTS ===
    nginx.ingress.kubernetes.io/hsts: "true"
    nginx.ingress.kubernetes.io/hsts-max-age: "15768000"  # 6 mois
    nginx.ingress.kubernetes.io/hsts-include-subdomains: "true"

    # === SÉCURITÉ SSL ===
    nginx.ingress.kubernetes.io/ssl-protocols: "TLSv1.2 TLSv1.3"
    nginx.ingress.kubernetes.io/ssl-prefer-server-ciphers: "true"

    # === REDIRECTIONS PERSONNALISÉES ===
    nginx.ingress.kubernetes.io/server-snippet: |
      # Redirection www vers apex
      if ($host = "www.lab.local") {
        return 301 https://lab.local$request_uri;
      }

      # Exceptions pour les health checks
      location = /health {
        if ($scheme = "http") {
          return 200 "OK";
        }
      }

      # Redirections d'anciennes URLs
      location = /old-api {
        return 301 https://lab.local/api/v2;
      }

    # === CERTIFICATS ===
    cert-manager.io/cluster-issuer: "letsencrypt-staging"
spec:
  tls:
  - hosts:
    - lab.local
    - www.lab.local
    - "*.lab.local"
    secretName: lab-wildcard-tls

  rules:
  # Site principal
  - host: lab.local
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: main-app
            port:
              number: 80

  # Redirection www
  - host: www.lab.local
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: main-app
            port:
              number: 80

  # API (sous-domaine)
  - host: api.lab.local
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

## Résumé et points clés

### Ce qu'il faut retenir

✅ **HTTPS est indispensable** pour la sécurité et la confiance
✅ **Redirection 301** est le standard pour HTTP→HTTPS
✅ **HSTS** ajoute une couche de sécurité supplémentaire
✅ **Certificats requis** : Auto-signés en dev, Let's Encrypt en prod
✅ **Exceptions possibles** : Health checks, webhooks
✅ **Test systématique** : Vérifiez avec curl
✅ **Progression graduelle** : Dev → Staging → Production

### Checklist de sécurisation

- [ ] Redirection HTTP→HTTPS activée
- [ ] Certificat SSL/TLS valide installé
- [ ] HSTS configuré (production)
- [ ] Protocoles TLS modernes uniquement
- [ ] www vers non-www (ou inverse) géré
- [ ] Health checks accessibles en HTTP si nécessaire
- [ ] Monitoring des expirations de certificats
- [ ] Tests de redirection validés

### Commandes utiles

```bash
# Vérifier la redirection
curl -I http://monsite.local

# Vérifier HSTS
curl -I https://monsite.local | grep -i strict

# Vérifier le certificat
openssl s_client -connect monsite.local:443 -servername monsite.local

# Tester la chaîne complète
curl -vL http://monsite.local
```

## Prochaine étape

Vous maîtrisez maintenant la gestion des redirections HTTPS et la sécurisation de vos connexions. La section suivante (6.7) présentera des exemples pratiques complets d'Ingress pour différents cas d'usage.

---

💡 **Conseil de sécurité** : En production, activez toujours HTTPS avec HSTS. Commencez avec une durée HSTS courte pour tester, puis augmentez progressivement. N'oubliez pas de surveiller l'expiration de vos certificats !

⏭️
