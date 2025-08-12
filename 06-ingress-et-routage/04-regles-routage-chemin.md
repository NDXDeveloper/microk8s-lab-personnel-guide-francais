🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 6.4 Règles de routage par chemin

## Introduction au routage par chemin

Le routage par chemin (path-based routing) permet de diriger le trafic vers différents services en fonction de l'URL demandée. Au lieu d'utiliser différents domaines, vous utilisez différents chemins sur le même domaine.

### Analogie simple

Imaginez un grand magasin avec différents rayons :
- Entrée principale → Hall d'accueil
- `/alimentation` → Rayon nourriture
- `/vetements` → Rayon vêtements
- `/electronique` → Rayon high-tech

Le routage par chemin fonctionne de la même manière :
- `monsite.com/` → Page d'accueil
- `monsite.com/api` → Service API
- `monsite.com/blog` → Application blog
- `monsite.com/admin` → Interface d'administration

## Comprendre les chemins URL

### Anatomie d'une URL

```
https://monsite.com/blog/articles/kubernetes?page=2#section1
        \_________/  \__________________________/\_______/\______/
             |                    |                  |        |
          Domaine             Chemin            Query    Fragment
                                 |
                          C'est cette partie
                         qu'Ingress examine
```

### Types de correspondance de chemin

Kubernetes Ingress supporte trois types de correspondance :

#### 1. Prefix (Préfixe) - Le plus courant

```yaml
pathType: Prefix
path: /api
```
- ✅ Correspond à : `/api`, `/api/`, `/api/users`, `/api/v1/products`
- ❌ Ne correspond pas à : `/apidocs`, `/myapi`

#### 2. Exact (Exact)

```yaml
pathType: Exact
path: /api
```
- ✅ Correspond à : `/api`
- ❌ Ne correspond pas à : `/api/`, `/api/users`, `/API`

#### 3. ImplementationSpecific (Spécifique à l'implémentation)

```yaml
pathType: ImplementationSpecific
path: /api/*
```
Dépend du controller utilisé (NGINX supporte les regex).

## Configuration de base du routage par chemin

### Exemple simple : Plusieurs services sur un domaine

```yaml
# ingress-path-routing.yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: path-based-routing
spec:
  rules:
  - host: monsite.local
    http:
      paths:
      # Page d'accueil
      - path: /
        pathType: Prefix
        backend:
          service:
            name: frontend-service
            port:
              number: 80

      # API
      - path: /api
        pathType: Prefix
        backend:
          service:
            name: api-service
            port:
              number: 3000

      # Blog
      - path: /blog
        pathType: Prefix
        backend:
          service:
            name: blog-service
            port:
              number: 80

      # Admin
      - path: /admin
        pathType: Prefix
        backend:
          service:
            name: admin-service
            port:
              number: 8080
```

### Ordre de priorité des chemins

L'ordre de priorité est important ! NGINX traite les chemins du plus spécifique au plus général :

```yaml
# ingress-path-priority.yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: path-priority
spec:
  rules:
  - host: monsite.local
    http:
      paths:
      # Plus spécifique - Priorité haute
      - path: /api/v2/users
        pathType: Prefix
        backend:
          service:
            name: users-v2-service
            port:
              number: 8080

      # Moyennement spécifique
      - path: /api/v2
        pathType: Prefix
        backend:
          service:
            name: api-v2-service
            port:
              number: 8080

      # Moins spécifique
      - path: /api
        pathType: Prefix
        backend:
          service:
            name: api-v1-service
            port:
              number: 8080

      # Catch-all - Priorité basse
      - path: /
        pathType: Prefix
        backend:
          service:
            name: default-service
            port:
              number: 80
```

**Résultat** :
- `/api/v2/users/123` → `users-v2-service`
- `/api/v2/products` → `api-v2-service`
- `/api/v1/anything` → `api-v1-service`
- `/home` → `default-service`

## Réécriture de chemins (Path Rewriting)

### Problème : Décalage de chemin

Souvent, vos applications ne s'attendent pas à recevoir le préfixe du chemin :

```
Requête : GET /blog/articles
          ↓
Ingress route vers blog-service
          ↓
Blog reçoit : GET /blog/articles
          ↓
❌ Blog cherche /blog/articles mais n'a que /articles
```

### Solution : Rewrite Target

```yaml
# ingress-rewrite.yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: rewrite-example
  annotations:
    # Supprime le préfixe /blog
    nginx.ingress.kubernetes.io/rewrite-target: /
spec:
  rules:
  - host: monsite.local
    http:
      paths:
      - path: /blog
        pathType: Prefix
        backend:
          service:
            name: blog-service
            port:
              number: 80
```

**Résultat** :
- Requête : `GET /blog/articles`
- Service reçoit : `GET /articles`

### Réécriture avancée avec capture

```yaml
# ingress-rewrite-advanced.yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: advanced-rewrite
  annotations:
    # Capture et réutilisation avec regex
    nginx.ingress.kubernetes.io/rewrite-target: /$2
spec:
  rules:
  - host: monsite.local
    http:
      paths:
      # Regex : capture tout après /api/
      - path: /api(/|$)(.*)
        pathType: Prefix
        backend:
          service:
            name: api-service
            port:
              number: 3000
```

**Exemples de transformation** :
- `/api/users` → `/users`
- `/api/v1/products` → `/v1/products`
- `/api/` → `/`
- `/api` → `/`

## Architectures de microservices

### Pattern API Gateway

Utiliser Ingress comme API Gateway pour vos microservices :

```yaml
# ingress-api-gateway.yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: api-gateway
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /$2
spec:
  rules:
  - host: api.monsite.local
    http:
      paths:
      # Service Authentification
      - path: /auth(/|$)(.*)
        pathType: Prefix
        backend:
          service:
            name: auth-service
            port:
              number: 8080

      # Service Utilisateurs
      - path: /users(/|$)(.*)
        pathType: Prefix
        backend:
          service:
            name: users-service
            port:
              number: 8080

      # Service Produits
      - path: /products(/|$)(.*)
        pathType: Prefix
        backend:
          service:
            name: products-service
            port:
              number: 8080

      # Service Commandes
      - path: /orders(/|$)(.*)
        pathType: Prefix
        backend:
          service:
            name: orders-service
            port:
              number: 8080

      # Service Paiements
      - path: /payments(/|$)(.*)
        pathType: Prefix
        backend:
          service:
            name: payments-service
            port:
              number: 8080
```

### Pattern Backend for Frontend (BFF)

Différents backends pour différents clients :

```yaml
# ingress-bff.yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: bff-routing
spec:
  rules:
  - host: monapp.local
    http:
      paths:
      # BFF pour application web
      - path: /api/web
        pathType: Prefix
        backend:
          service:
            name: bff-web
            port:
              number: 8080

      # BFF pour application mobile
      - path: /api/mobile
        pathType: Prefix
        backend:
          service:
            name: bff-mobile
            port:
              number: 8080

      # BFF pour partenaires externes
      - path: /api/partner
        pathType: Prefix
        backend:
          service:
            name: bff-partner
            port:
              number: 8080

      # GraphQL endpoint
      - path: /graphql
        pathType: Exact
        backend:
          service:
            name: graphql-gateway
            port:
              number: 4000
```

## Gestion des ressources statiques

### Servir des fichiers statiques

```yaml
# ingress-static-files.yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: static-content
  annotations:
    # Cache pour les ressources statiques
    nginx.ingress.kubernetes.io/configuration-snippet: |
      location ~* \.(jpg|jpeg|png|gif|ico|css|js|woff|woff2)$ {
        expires 30d;
        add_header Cache-Control "public, immutable";
      }
spec:
  rules:
  - host: monsite.local
    http:
      paths:
      # Fichiers statiques
      - path: /static
        pathType: Prefix
        backend:
          service:
            name: static-server
            port:
              number: 80

      # Images
      - path: /images
        pathType: Prefix
        backend:
          service:
            name: image-server
            port:
              number: 80

      # Application dynamique
      - path: /
        pathType: Prefix
        backend:
          service:
            name: app-server
            port:
              number: 3000
```

### CDN et optimisation

```yaml
# ingress-cdn-optimization.yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: cdn-optimized
  annotations:
    # Compression
    nginx.ingress.kubernetes.io/server-snippet: |
      gzip on;
      gzip_types text/plain text/css application/json application/javascript;
      gzip_min_length 1000;

    # Headers de cache
    nginx.ingress.kubernetes.io/configuration-snippet: |
      if ($request_uri ~* \.(js|css|png|jpg|jpeg|gif|ico|svg)$) {
        add_header Cache-Control "public, max-age=31536000, immutable";
      }
spec:
  rules:
  - host: assets.monsite.local
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: cdn-service
            port:
              number: 80
```

## Versioning d'API

### Versioning par chemin

```yaml
# ingress-api-versioning.yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: api-versioning
spec:
  rules:
  - host: api.monsite.local
    http:
      paths:
      # Version 1 - Ancienne API (maintenance)
      - path: /v1
        pathType: Prefix
        backend:
          service:
            name: api-v1
            port:
              number: 8080

      # Version 2 - API stable actuelle
      - path: /v2
        pathType: Prefix
        backend:
          service:
            name: api-v2
            port:
              number: 8080

      # Version 3 - Beta/Preview
      - path: /v3-beta
        pathType: Prefix
        backend:
          service:
            name: api-v3-beta
            port:
              number: 8080

      # Redirection de la racine vers la version stable
      - path: /
        pathType: Exact
        backend:
          service:
            name: api-v2
            port:
              number: 8080
```

### Dépréciation progressive

```yaml
# ingress-api-deprecation.yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: api-deprecation
  annotations:
    # Ajouter des headers de dépréciation pour v1
    nginx.ingress.kubernetes.io/configuration-snippet: |
      if ($request_uri ~* ^/v1) {
        add_header X-API-Deprecation-Warning "Version 1 is deprecated. Please migrate to v2" always;
        add_header X-API-Deprecation-Date "2025-12-31" always;
        add_header X-API-Deprecation-Info "https://api.monsite.local/migration-guide" always;
      }
spec:
  rules:
  - host: api.monsite.local
    http:
      paths:
      - path: /v1
        pathType: Prefix
        backend:
          service:
            name: api-v1-deprecated
            port:
              number: 8080

      - path: /v2
        pathType: Prefix
        backend:
          service:
            name: api-v2-stable
            port:
              number: 8080
```

## Cas spéciaux et edge cases

### Gestion du trailing slash

Le trailing slash (`/`) peut causer des problèmes :

```yaml
# ingress-trailing-slash.yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: trailing-slash-handling
  annotations:
    # Forcer l'ajout du trailing slash
    nginx.ingress.kubernetes.io/configuration-snippet: |
      rewrite ^(/app)$ $1/ permanent;
spec:
  rules:
  - host: monsite.local
    http:
      paths:
      # Accepte /app et /app/
      - path: /app
        pathType: Prefix
        backend:
          service:
            name: app-service
            port:
              number: 80
```

### Chemins avec caractères spéciaux

```yaml
# ingress-special-chars.yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: special-chars
  annotations:
    # Utiliser ImplementationSpecific pour les regex
    nginx.ingress.kubernetes.io/use-regex: "true"
spec:
  rules:
  - host: monsite.local
    http:
      paths:
      # Accepte les IDs numériques
      - path: /users/[0-9]+
        pathType: ImplementationSpecific
        backend:
          service:
            name: users-service
            port:
              number: 8080

      # Accepte les UUIDs
      - path: /items/[a-f0-9]{8}-[a-f0-9]{4}-[a-f0-9]{4}-[a-f0-9]{4}-[a-f0-9]{12}
        pathType: ImplementationSpecific
        backend:
          service:
            name: items-service
            port:
              number: 8080
```

### Chemins sensibles à la casse

```yaml
# ingress-case-sensitive.yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: case-handling
  annotations:
    # Rendre insensible à la casse
    nginx.ingress.kubernetes.io/configuration-snippet: |
      location ~* ^/api {
        proxy_pass http://upstream_balancer;
      }
spec:
  rules:
  - host: monsite.local
    http:
      paths:
      # /API, /Api, /api tous routés ici
      - path: /api
        pathType: Prefix
        backend:
          service:
            name: api-service
            port:
              number: 8080
```

## Combinaison avec le routage par hôte

### Architecture hybride

```yaml
# ingress-hybrid-routing.yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: hybrid-routing
spec:
  rules:
  # Site principal avec chemins
  - host: monsite.local
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: main-site
            port:
              number: 80
      - path: /api
        pathType: Prefix
        backend:
          service:
            name: api-service
            port:
              number: 3000
      - path: /blog
        pathType: Prefix
        backend:
          service:
            name: blog-service
            port:
              number: 80

  # API dédiée avec versioning
  - host: api.monsite.local
    http:
      paths:
      - path: /v1
        pathType: Prefix
        backend:
          service:
            name: api-v1
            port:
              number: 8080
      - path: /v2
        pathType: Prefix
        backend:
          service:
            name: api-v2
            port:
              number: 8080

  # Admin sur sous-domaine
  - host: admin.monsite.local
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: admin-panel
            port:
              number: 8080
```

## Patterns et anti-patterns

### ✅ Bonnes pratiques

```yaml
# BIEN : Chemins clairs et organisés
paths:
- path: /api/v1     # Versioning explicite
- path: /auth       # Fonction claire
- path: /static     # Type de contenu évident
- path: /healthz    # Convention standard
```

### ❌ À éviter

```yaml
# MAL : Chemins ambigus ou conflictuels
paths:
- path: /a          # Trop court, pas clair
- path: /api-v1     # Utilisez / plutôt que -
- path: /API        # Évitez les majuscules
- path: //double    # Double slash problématique
```

## Sécurité et contrôle d'accès

### Protection de chemins sensibles

```yaml
# ingress-secure-paths.yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: secure-paths
  annotations:
    # Authentification pour /admin
    nginx.ingress.kubernetes.io/auth-type: basic
    nginx.ingress.kubernetes.io/auth-secret: admin-auth
    nginx.ingress.kubernetes.io/auth-realm: "Admin Area"
spec:
  rules:
  - host: monsite.local
    http:
      paths:
      # Public
      - path: /
        pathType: Prefix
        backend:
          service:
            name: public-site
            port:
              number: 80

      # Protégé
      - path: /admin
        pathType: Prefix
        backend:
          service:
            name: admin-service
            port:
              number: 8080
```

### Restriction par IP pour certains chemins

```yaml
# ingress-ip-restriction.yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: ip-restricted-paths
spec:
  rules:
  - host: monsite.local
    http:
      paths:
      - path: /public
        pathType: Prefix
        backend:
          service:
            name: public-service
            port:
              number: 80
---
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: private-paths
  annotations:
    # Seulement réseau local
    nginx.ingress.kubernetes.io/whitelist-source-range: "192.168.1.0/24,10.0.0.0/8"
spec:
  rules:
  - host: monsite.local
    http:
      paths:
      - path: /internal
        pathType: Prefix
        backend:
          service:
            name: internal-service
            port:
              number: 8080
```

## Monitoring et debug

### Logs par chemin

```yaml
# ingress-path-logging.yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: logged-paths
  annotations:
    # Log détaillé pour debug
    nginx.ingress.kubernetes.io/configuration-snippet: |
      access_log /var/log/nginx/api-access.log combined if=$is_api_request;
      set $is_api_request 0;
      if ($request_uri ~* ^/api) {
        set $is_api_request 1;
      }
spec:
  rules:
  - host: monsite.local
    http:
      paths:
      - path: /api
        pathType: Prefix
        backend:
          service:
            name: api-service
            port:
              number: 3000
```

### Test des chemins

```bash
# Tester différents chemins
curl http://monsite.local/
curl http://monsite.local/api
curl http://monsite.local/api/users
curl http://monsite.local/blog/posts

# Tester avec headers
curl -H "Accept: application/json" http://monsite.local/api/data

# Voir le routage réel
microk8s kubectl describe ingress path-based-routing

# Vérifier la configuration NGINX
microk8s kubectl exec -n ingress [nginx-pod] -- cat /etc/nginx/nginx.conf | grep location
```

## Exemple complet pour un lab

### Application complète avec tous les patterns

```yaml
# ingress-complete-lab.yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: lab-complete
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /$2
    nginx.ingress.kubernetes.io/ssl-redirect: "true"
    cert-manager.io/cluster-issuer: "letsencrypt-staging"
spec:
  tls:
  - hosts:
    - lab.local
    secretName: lab-tls

  rules:
  - host: lab.local
    http:
      paths:
      # Frontend React/Vue/Angular
      - path: /
        pathType: Exact
        backend:
          service:
            name: frontend
            port:
              number: 3000

      # Assets statiques
      - path: /static
        pathType: Prefix
        backend:
          service:
            name: static-files
            port:
              number: 80

      # API REST versionnée
      - path: /api/v1(/|$)(.*)
        pathType: Prefix
        backend:
          service:
            name: api-v1
            port:
              number: 8080

      - path: /api/v2(/|$)(.*)
        pathType: Prefix
        backend:
          service:
            name: api-v2
            port:
              number: 8080

      # GraphQL endpoint
      - path: /graphql
        pathType: Exact
        backend:
          service:
            name: graphql
            port:
              number: 4000

      # WebSocket
      - path: /ws
        pathType: Prefix
        backend:
          service:
            name: websocket
            port:
              number: 3001

      # Monitoring
      - path: /metrics
        pathType: Exact
        backend:
          service:
            name: prometheus-metrics
            port:
              number: 9090

      # Health checks
      - path: /health
        pathType: Exact
        backend:
          service:
            name: health-service
            port:
              number: 8080

      # Documentation
      - path: /docs
        pathType: Prefix
        backend:
          service:
            name: swagger-ui
            port:
              number: 8080

      # Admin (protégé)
      - path: /admin(/|$)(.*)
        pathType: Prefix
        backend:
          service:
            name: admin-panel
            port:
              number: 8080
```

## Résumé et points clés

### Ce qu'il faut retenir

✅ **PathType compte** : Prefix vs Exact vs ImplementationSpecific
✅ **L'ordre est important** : Du plus spécifique au plus général
✅ **Rewrite souvent nécessaire** : Les apps ne s'attendent pas au préfixe
✅ **Regex possible** : Pour des patterns complexes
✅ **Combiner avec host routing** : Maximum de flexibilité
✅ **Sécurité par chemin** : Différentes règles pour différents endpoints
✅ **Monitoring granulaire** : Métriques et logs par chemin

### Avantages du routage par chemin

1. **Un seul domaine** : Simplifie la gestion DNS
2. **Organisation logique** : `/api`, `/admin`, `/docs`
3. **Économie de ressources** : Un seul certificat SSL
4. **Architecture RESTful** : Suit les conventions web
5. **Facilité de CORS** : Même origine pour tout

### Cas d'usage idéaux

- **APIs versionnées** : `/v1`, `/v2`, `/v3`
- **Microservices** : `/users`, `/products`, `/orders`
- **Sites multilingues** : `/en`, `/fr`, `/es`
- **Environnements** : `/dev`, `/staging`, `/prod`
- **Documentation** : `/docs`, `/api-docs`, `/swagger`

## Prochaine étape

Vous maîtrisez maintenant le routage par chemin. La section suivante (6.5) explorera les middleware et annotations pour personnaliser encore plus finement le comportement de vos Ingress.

---

💡 **Conseil pratique** : Commencez avec des chemins simples et évitez les regex complexes. Utilisez `Prefix` dans 90% des cas, c'est le plus simple et le plus prévisible. Testez toujours vos rewrites avec `curl` avant de les déployer en production.

⏭️
