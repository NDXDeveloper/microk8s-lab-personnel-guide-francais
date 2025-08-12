🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 6.3 Règles de routage par nom d'hôte

## Introduction au routage par nom d'hôte

Le routage par nom d'hôte (host-based routing) est l'une des fonctionnalités les plus puissantes d'Ingress. Il permet d'utiliser différents noms de domaine pour accéder à différentes applications, tout en utilisant la même adresse IP et les mêmes ports (80/443).

### Analogie simple

Imaginez un grand immeuble de bureaux avec une seule adresse postale. Quand le courrier arrive, il est trié selon le nom de l'entreprise destinataire :
- Courrier pour "Entreprise A" → 3ème étage
- Courrier pour "Entreprise B" → 5ème étage
- Courrier pour "Entreprise C" → 7ème étage

Le routage par nom d'hôte fonctionne exactement pareil :
- Requête vers `blog.monlab.local` → Service Blog
- Requête vers `api.monlab.local` → Service API
- Requête vers `shop.monlab.local` → Service Boutique

## Comment fonctionne le routage par nom d'hôte

### Le processus de routage

```
1. L'utilisateur tape : blog.monlab.local
                           ↓
2. DNS résout : blog.monlab.local → 192.168.1.100
                           ↓
3. Navigateur envoie la requête HTTP avec header :
   Host: blog.monlab.local
                           ↓
4. NGINX Ingress Controller reçoit la requête
                           ↓
5. NGINX examine le header "Host"
                           ↓
6. NGINX trouve la règle correspondante
                           ↓
7. NGINX route vers le service Blog
                           ↓
8. Le service Blog répond
```

### L'importance du header HTTP "Host"

Le header "Host" est crucial pour le routage par nom d'hôte :

```http
GET / HTTP/1.1
Host: blog.monlab.local    ← C'est ce header qui détermine le routage
User-Agent: Mozilla/5.0
Accept: text/html
```

Sans ce header, NGINX ne saurait pas vers quelle application router la requête.

## Configuration DNS préalable

### Option 1 : Fichier hosts (pour tests locaux)

La méthode la plus simple pour un lab personnel :

```bash
# Linux/Mac : /etc/hosts
# Windows : C:\Windows\System32\drivers\etc\hosts

# Ajouter ces lignes (remplacer l'IP par celle de votre machine)
192.168.1.100 blog.monlab.local
192.168.1.100 api.monlab.local
192.168.1.100 shop.monlab.local
192.168.1.100 admin.monlab.local
```

### Option 2 : DNS local (dnsmasq, Pi-hole)

Pour une solution plus flexible :

```bash
# Configuration dnsmasq
# /etc/dnsmasq.conf
address=/monlab.local/192.168.1.100

# Cela résout tous les *.monlab.local vers 192.168.1.100
```

### Option 3 : Domaine réel avec DNS public

Pour exposer vos services sur Internet :

```dns
# Enregistrements DNS chez votre registrar
blog.mondomaine.com    A    IP-PUBLIQUE
api.mondomaine.com     A    IP-PUBLIQUE
shop.mondomaine.com    A    IP-PUBLIQUE
```

## Création de règles de routage simples

### Exemple basique : Un domaine, un service

```yaml
# ingress-blog.yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: blog-ingress
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
spec:
  rules:
  - host: blog.monlab.local    # Le nom de domaine
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: blog-service   # Le service cible
            port:
              number: 80
```

### Exemple multiple : Plusieurs domaines dans une Ingress

```yaml
# ingress-multi-hosts.yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: multi-host-ingress
spec:
  rules:
  # Premier domaine
  - host: blog.monlab.local
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: blog-service
            port:
              number: 80

  # Deuxième domaine
  - host: api.monlab.local
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: api-service
            port:
              number: 3000

  # Troisième domaine
  - host: shop.monlab.local
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: shop-service
            port:
              number: 8080
```

## Gestion des sous-domaines

### Sous-domaines pour environnements

Une approche courante est d'utiliser les sous-domaines pour séparer les environnements :

```yaml
# ingress-environments.yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: app-environments
spec:
  rules:
  # Développement
  - host: dev.monapp.local
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: app-dev
            port:
              number: 80

  # Staging/Test
  - host: staging.monapp.local
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: app-staging
            port:
              number: 80

  # Production
  - host: prod.monapp.local
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: app-prod
            port:
              number: 80
```

### Sous-domaines pour microservices

Organisation par fonction métier :

```yaml
# ingress-microservices.yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: microservices-ingress
spec:
  rules:
  # Service authentification
  - host: auth.monlab.local
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: auth-service
            port:
              number: 8080

  # Service utilisateurs
  - host: users.monlab.local
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: users-service
            port:
              number: 8080

  # Service commandes
  - host: orders.monlab.local
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: orders-service
            port:
              number: 8080

  # Service paiements
  - host: payments.monlab.local
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: payments-service
            port:
              number: 8080
```

## Wildcards et domaines par défaut

### Wildcard DNS et Ingress

Pour gérer dynamiquement de nombreux sous-domaines :

```yaml
# ingress-wildcard.yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: wildcard-ingress
spec:
  rules:
  # Wildcard : capture tous les sous-domaines
  - host: "*.apps.monlab.local"
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: dynamic-app-router
            port:
              number: 80
```

**Note importante** : Le wildcard dans Ingress nécessite :
1. Un certificat SSL wildcard si vous utilisez HTTPS
2. Une configuration DNS wildcard correspondante
3. Une logique dans votre application pour gérer les différents sous-domaines

### Domaine par défaut (catch-all)

Pour capturer tout le trafic qui ne correspond à aucune règle :

```yaml
# ingress-default.yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: default-backend
spec:
  # Pas de host spécifié = capture tout
  defaultBackend:
    service:
      name: default-service
      port:
        number: 80
---
# Ou avec une règle sans host
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: catch-all
spec:
  rules:
  - http:  # Pas de host défini
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: error-page-service
            port:
              number: 80
```

## HTTPS et certificats SSL/TLS

### Configuration HTTPS basique

```yaml
# ingress-https.yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: secure-ingress
  annotations:
    # Force la redirection HTTP vers HTTPS
    nginx.ingress.kubernetes.io/ssl-redirect: "true"
spec:
  tls:
  - hosts:
    - blog.monlab.local
    secretName: blog-tls-secret
  rules:
  - host: blog.monlab.local
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: blog-service
            port:
              number: 80
```

### Certificats multiples pour plusieurs domaines

```yaml
# ingress-multi-tls.yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: multi-domain-secure
spec:
  tls:
  # Certificat pour le blog
  - hosts:
    - blog.monlab.local
    secretName: blog-tls

  # Certificat pour l'API
  - hosts:
    - api.monlab.local
    secretName: api-tls

  # Certificat wildcard pour tous les sous-domaines
  - hosts:
    - "*.apps.monlab.local"
    secretName: wildcard-tls

  rules:
  - host: blog.monlab.local
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: blog-service
            port:
              number: 80

  - host: api.monlab.local
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: api-service
            port:
              number: 3000
```

## Patterns et bonnes pratiques

### Organisation par namespace

Séparer les applications par namespace pour une meilleure organisation :

```yaml
# ingress-namespace-blog.yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: blog-ingress
  namespace: blog  # Namespace dédié
spec:
  rules:
  - host: blog.monlab.local
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: blog-service  # Service dans le même namespace
            port:
              number: 80
---
# ingress-namespace-api.yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: api-ingress
  namespace: api  # Namespace différent
spec:
  rules:
  - host: api.monlab.local
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: api-service  # Service dans le namespace api
            port:
              number: 3000
```

### Annotations spécifiques par domaine

Personnaliser le comportement pour chaque domaine :

```yaml
# ingress-custom-behavior.yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: blog-custom
  annotations:
    # Spécifique au blog : uploads volumineux
    nginx.ingress.kubernetes.io/proxy-body-size: "100m"
    nginx.ingress.kubernetes.io/proxy-read-timeout: "600"
spec:
  rules:
  - host: blog.monlab.local
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: blog-service
            port:
              number: 80
---
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: api-custom
  annotations:
    # Spécifique à l'API : rate limiting
    nginx.ingress.kubernetes.io/limit-rps: "100"
    nginx.ingress.kubernetes.io/limit-connections: "10"
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
              number: 3000
```

## Cas d'usage avancés

### Routage basé sur les headers HTTP

Au-delà du simple nom d'hôte, router selon d'autres headers :

```yaml
# ingress-header-routing.yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: header-based-routing
  annotations:
    nginx.ingress.kubernetes.io/server-snippet: |
      # Router selon le User-Agent
      if ($http_user_agent ~* "Mobile") {
        return 301 https://mobile.monlab.local$request_uri;
      }
spec:
  rules:
  - host: monlab.local
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: desktop-service
            port:
              number: 80

  - host: mobile.monlab.local
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: mobile-service
            port:
              number: 80
```

### Routage géographique ou par langue

```yaml
# ingress-geo-routing.yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: geo-routing
spec:
  rules:
  # Version française
  - host: fr.monsite.local
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: site-fr
            port:
              number: 80

  # Version anglaise
  - host: en.monsite.local
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: site-en
            port:
              number: 80

  # Version espagnole
  - host: es.monsite.local
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: site-es
            port:
              number: 80
```

### A/B Testing avec plusieurs domaines

```yaml
# ingress-ab-testing.yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: version-a
spec:
  rules:
  - host: monapp.local
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: app-version-a
            port:
              number: 80
---
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: version-b
spec:
  rules:
  - host: beta.monapp.local
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: app-version-b
            port:
              number: 80
```

## Dépannage du routage par nom d'hôte

### Problèmes DNS courants

```bash
# Vérifier la résolution DNS
nslookup blog.monlab.local
dig blog.monlab.local
host blog.monlab.local

# Tester avec curl en spécifiant le header Host
curl -H "Host: blog.monlab.local" http://192.168.1.100

# Vérifier depuis l'intérieur du cluster
microk8s kubectl run test-pod --image=busybox -it --rm -- sh
# Dans le pod :
nslookup blog.monlab.local
wget -O- --header="Host: blog.monlab.local" http://ingress.ingress.svc.cluster.local
```

### Vérification des règles Ingress

```bash
# Lister toutes les Ingress
microk8s kubectl get ingress -A

# Détails d'une Ingress spécifique
microk8s kubectl describe ingress blog-ingress

# Vérifier la configuration NGINX générée
microk8s kubectl exec -n ingress [nginx-pod] -- cat /etc/nginx/nginx.conf | grep server_name

# Voir les backends configurés
microk8s kubectl get ingress blog-ingress -o jsonpath='{.spec.rules[*].host}'
```

### Problèmes de routage

```bash
# Logs du controller pour voir les requêtes
microk8s kubectl logs -n ingress -l name=nginx-ingress-microk8s -f | grep "blog.monlab.local"

# Tester le service directement (sans Ingress)
microk8s kubectl port-forward svc/blog-service 8080:80
curl http://localhost:8080

# Vérifier que le service a des endpoints
microk8s kubectl get endpoints blog-service
```

## Monitoring et observabilité

### Métriques par nom d'hôte

NGINX Ingress Controller expose des métriques par host :

```prometheus
# Requêtes par host
nginx_ingress_controller_requests{host="blog.monlab.local"}

# Latence par host
nginx_ingress_controller_request_duration_seconds{host="api.monlab.local"}

# Erreurs par host
nginx_ingress_controller_requests{host="shop.monlab.local",status=~"5.."}
```

### Logs structurés

Configurer les logs pour faciliter l'analyse :

```yaml
# ConfigMap pour logs JSON
data:
  log-format-escape-json: "true"
  log-format-upstream: '{"time": "$time_iso8601", "host": "$host", "uri": "$request_uri", "status": $status, "latency": $request_time, "client": "$remote_addr"}'
```

## Exemples complets pour un lab

### Stack de développement complète

```yaml
# ingress-dev-stack.yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: dev-stack
  annotations:
    nginx.ingress.kubernetes.io/ssl-redirect: "false"  # Pas de SSL en dev
spec:
  rules:
  # Frontend
  - host: app.dev.local
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: frontend
            port:
              number: 3000

  # Backend API
  - host: api.dev.local
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: backend
            port:
              number: 8080

  # Base de données admin
  - host: db.dev.local
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: adminer
            port:
              number: 8080

  # Documentation
  - host: docs.dev.local
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: swagger-ui
            port:
              number: 8080

  # Monitoring
  - host: metrics.dev.local
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: grafana
            port:
              number: 3000
```

### Services personnels

```yaml
# ingress-personal-services.yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: personal-services
  annotations:
    cert-manager.io/cluster-issuer: "letsencrypt-prod"
spec:
  tls:
  - hosts:
    - "*.home.mondomaine.com"
    secretName: wildcard-home-tls

  rules:
  # Nextcloud personnel
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

  # Serveur média Jellyfin
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

  # Gestionnaire de mots de passe
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
```

## Résumé et points clés

### Ce qu'il faut retenir

✅ **Le header Host** est la clé du routage par nom d'hôte
✅ **DNS doit être configuré** pour pointer vers votre cluster
✅ **Plusieurs domaines** peuvent pointer vers la même IP
✅ **Une Ingress peut gérer** plusieurs domaines
✅ **Les certificats TLS** peuvent être spécifiques ou wildcard
✅ **Les namespaces** permettent une meilleure organisation
✅ **Les annotations** personnalisent le comportement par domaine

### Avantages du routage par nom d'hôte

1. **URLs mémorables** : `blog.local` vs `192.168.1.100:31234`
2. **Organisation claire** : Un domaine par application
3. **Isolation** : Chaque domaine peut avoir ses propres règles
4. **Scalabilité** : Ajouter des domaines sans changer l'infrastructure
5. **Professionnalisme** : URLs propres pour les utilisateurs

### Limitations à connaître

- **DNS obligatoire** : Nécessite une résolution DNS fonctionnelle
- **Port unique** : Tous les domaines partagent les ports 80/443
- **Certificats** : Un certificat par domaine (sauf wildcard)
- **Performances** : Toutes les requêtes passent par le même controller

## Prochaine étape

Vous maîtrisez maintenant le routage par nom d'hôte. La section suivante (6.4) vous montrera comment combiner cela avec le routage par chemin pour créer des architectures encore plus sophistiquées.

---

💡 **Astuce pratique** : Commencez simple avec un domaine par application, puis évoluez vers des configurations plus complexes. Utilisez des noms de domaine descriptifs qui reflètent clairement la fonction de chaque service.

⏭️
