🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 6.1 Concepts d'Ingress

## Qu'est-ce qu'Ingress ?

Imaginez que vous gérez un immeuble avec plusieurs entreprises. Chaque entreprise a besoin que ses clients puissent la trouver facilement. Plutôt que d'avoir une entrée séparée pour chaque entreprise, vous avez **un hall d'entrée principal** avec un réceptionniste qui oriente les visiteurs vers le bon étage selon l'entreprise qu'ils cherchent.

**Ingress fonctionne exactement de cette manière** dans Kubernetes : c'est le "réceptionniste" de votre cluster qui dirige le trafic externe vers les bonnes applications internes.

### Définition technique

**Ingress** est une ressource Kubernetes qui définit des règles pour router le trafic HTTP et HTTPS externe vers des services à l'intérieur du cluster. C'est une couche d'abstraction qui permet d'exposer plusieurs services à travers un point d'entrée unique.

```yaml
# Exemple simplifié d'une ressource Ingress
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: exemple-ingress
spec:
  rules:
  - host: monapp.exemple.com
    http:
      paths:
      - path: /
        backend:
          service:
            name: mon-service
            port:
              number: 80
```

## Pourquoi Ingress existe-t-il ?

### Le problème sans Ingress

Sans Ingress, vous avez principalement trois options pour exposer vos applications :

1. **NodePort** : Chaque service obtient un port aléatoire (30000-32767)
   - ❌ Difficile à mémoriser : `http://monserveur:31487`
   - ❌ Limité en nombre de services
   - ❌ Pas professionnel pour les utilisateurs

2. **LoadBalancer** : Chaque service obtient une IP externe
   - ❌ Coûteux (une IP par service dans le cloud)
   - ❌ Complexe à gérer avec plusieurs IPs
   - ❌ Pas toujours disponible (bare-metal)

3. **Port-Forward** : Redirection manuelle des ports
   - ❌ Uniquement pour le développement
   - ❌ Pas permanent
   - ❌ Accès local uniquement

### La solution Ingress

Ingress résout ces problèmes en offrant :

✅ **Un point d'entrée unique** : Ports standards 80 (HTTP) et 443 (HTTPS)
✅ **Routage par nom de domaine** : `blog.monsite.com`, `api.monsite.com`
✅ **Routage par chemin** : `/blog`, `/api`, `/admin`
✅ **Gestion centralisée SSL/TLS** : Un seul endroit pour tous les certificats
✅ **Fonctionnalités avancées** : Redirections, authentification, rate limiting

## Architecture d'Ingress

### Les trois composants clés

```
┌─────────────────────────────────────────────┐
│           1. RESSOURCE INGRESS              │
│         (Règles de routage YAML)            │
└─────────────────────────────────────────────┘
                      ↓
┌─────────────────────────────────────────────┐
│         2. INGRESS CONTROLLER               │
│    (Programme qui implémente les règles)    │
│         (Ex: NGINX, Traefik, HAProxy)       │
└─────────────────────────────────────────────┘
                      ↓
┌─────────────────────────────────────────────┐
│            3. SERVICES BACKEND              │
│        (Vos applications dans le cluster)   │
└─────────────────────────────────────────────┘
```

### 1. La Ressource Ingress

C'est un objet Kubernetes (comme un Pod ou un Service) qui contient les règles de routage. Elle ne fait rien par elle-même, c'est juste une "recette" qui décrit comment router le trafic.

**Analogie** : C'est comme un plan d'architecte - il décrit ce qui doit être construit, mais ne construit rien lui-même.

### 2. L'Ingress Controller

C'est le programme qui lit les ressources Ingress et implémente réellement le routage. Dans MicroK8s, c'est généralement **NGINX Ingress Controller**.

**Analogie** : C'est l'entrepreneur qui lit le plan et construit réellement la maison.

### 3. Les Services Backend

Ce sont vos applications qui tournent dans le cluster et qui recevront le trafic routé par l'Ingress Controller.

## Comment fonctionne le routage ?

### Flux de trafic complet

Suivons le parcours d'une requête HTTP :

```
1. Utilisateur tape : https://blog.monlab.local
                            ↓
2. Résolution DNS : blog.monlab.local → 192.168.1.100
                            ↓
3. Requête arrive sur la machine hôte (port 443)
                            ↓
4. Ingress Controller reçoit la requête
                            ↓
5. Controller vérifie les règles Ingress
   "Quelle règle correspond à blog.monlab.local ?"
                            ↓
6. Trouve la règle : blog.monlab.local → service-blog
                            ↓
7. Transmet la requête au Service
                            ↓
8. Service route vers un Pod du blog
                            ↓
9. Pod génère la réponse
                            ↓
10. Réponse retourne par le même chemin
```

### Types de routage

#### 1. Routage par nom d'hôte (Host-based)

Différents domaines vers différents services :

```yaml
spec:
  rules:
  - host: blog.exemple.com
    http:
      paths:
      - backend:
          service:
            name: service-blog
  - host: shop.exemple.com
    http:
      paths:
      - backend:
          service:
            name: service-boutique
```

**Résultat** :
- `blog.exemple.com` → Service Blog
- `shop.exemple.com` → Service Boutique

#### 2. Routage par chemin (Path-based)

Différents chemins vers différents services :

```yaml
spec:
  rules:
  - host: monsite.com
    http:
      paths:
      - path: /blog
        backend:
          service:
            name: service-blog
      - path: /shop
        backend:
          service:
            name: service-boutique
      - path: /api
        backend:
          service:
            name: service-api
```

**Résultat** :
- `monsite.com/blog` → Service Blog
- `monsite.com/shop` → Service Boutique
- `monsite.com/api` → Service API

#### 3. Routage mixte

Combinaison des deux approches pour une flexibilité maximale :

```yaml
spec:
  rules:
  - host: app.exemple.com
    http:
      paths:
      - path: /v1
        backend:
          service:
            name: app-v1
      - path: /v2
        backend:
          service:
            name: app-v2
```

## Ingress Controller : Le moteur du système

### Qu'est-ce qu'un Ingress Controller ?

L'Ingress Controller est un pod qui tourne dans votre cluster et qui :
1. **Surveille** les ressources Ingress
2. **Configure** un reverse proxy (NGINX, Traefik, etc.)
3. **Route** le trafic selon les règles
4. **Gère** les certificats SSL/TLS

### NGINX Ingress Controller (dans MicroK8s)

MicroK8s utilise par défaut NGINX Ingress Controller, qui est :
- **Stable et mature** : Utilisé en production par des millions de sites
- **Performant** : Capable de gérer des milliers de requêtes/seconde
- **Flexible** : Support de nombreuses annotations pour personnalisation
- **Bien documenté** : Énorme communauté et ressources

### Où vit l'Ingress Controller ?

```bash
# L'Ingress Controller est un pod dans le namespace 'ingress'
microk8s kubectl get pods -n ingress

# Résultat typique :
NAME                                      READY   STATUS
nginx-ingress-microk8s-controller-xxxxx  1/1     Running
```

## Concepts avancés simplifiés

### Annotations

Les annotations permettent de personnaliser le comportement de l'Ingress :

```yaml
metadata:
  annotations:
    # Forcer la redirection vers HTTPS
    nginx.ingress.kubernetes.io/ssl-redirect: "true"

    # Limiter la taille des uploads
    nginx.ingress.kubernetes.io/proxy-body-size: "10m"

    # Timeout personnalisé
    nginx.ingress.kubernetes.io/proxy-read-timeout: "600"
```

**Analogie** : Les annotations sont comme des post-its sur votre plan - des instructions spéciales pour l'entrepreneur.

### Classes d'Ingress

Depuis Kubernetes 1.18, vous pouvez avoir plusieurs Ingress Controllers avec des classes différentes :

```yaml
spec:
  ingressClassName: nginx  # Utiliser NGINX
  # ou
  ingressClassName: traefik  # Utiliser Traefik
```

**Utilité** : Permet d'avoir différents controllers pour différents usages (interne/externe, dev/prod).

### TLS/SSL

Ingress peut terminer le SSL/TLS pour vous :

```yaml
spec:
  tls:
  - hosts:
    - monsite.com
    secretName: monsite-tls-secret  # Secret contenant le certificat
```

**Avantage** : Vos pods n'ont pas besoin de gérer SSL, tout est centralisé.

## Limitations et considérations

### Ce qu'Ingress peut faire

✅ Router HTTP/HTTPS (ports 80/443)
✅ Terminer SSL/TLS
✅ Load balancing basique
✅ Redirections et rewrites d'URL
✅ Authentification basique
✅ Rate limiting

### Ce qu'Ingress ne peut PAS faire

❌ Router TCP/UDP arbitraire (sauf configuration spéciale)
❌ Remplacer un firewall complet
❌ Gérer des protocoles non-HTTP (FTP, SSH, etc.)
❌ Load balancing géographique avancé

Pour ces cas, vous aurez besoin d'autres solutions comme :
- **MetalLB** pour du LoadBalancing L4
- **Service Mesh** (Istio, Linkerd) pour du routage avancé
- **NodePort** pour des protocoles non-HTTP simples

## Ingress vs Alternatives

### Ingress vs NodePort

| Aspect | Ingress | NodePort |
|--------|---------|----------|
| Port | 80/443 standards | 30000-32767 aléatoires |
| Nombre de services | Illimité | Limité (~2000) |
| Nom de domaine | ✅ Supporté | ❌ IP:Port uniquement |
| SSL/TLS | ✅ Centralisé | ⚠️ Par application |
| Complexité | Moyenne | Faible |

### Ingress vs LoadBalancer

| Aspect | Ingress | LoadBalancer |
|--------|---------|--------------|
| Coût (cloud) | 1 seul LB | 1 LB par service |
| Routage L7 | ✅ HTTP/HTTPS | ❌ L4 uniquement |
| Bare-metal | ✅ Fonctionne | ⚠️ Nécessite MetalLB |
| Configuration | Centralisée | Distribuée |

## Cas d'usage concrets dans votre lab

### 1. Hébergement de plusieurs sites web

```
blog.monlab.local    → Pod WordPress
wiki.monlab.local    → Pod MediaWiki
forum.monlab.local   → Pod Discourse
```

Un seul point d'entrée, plusieurs applications.

### 2. API versionnée

```
api.monlab.local/v1  → Ancienne version de l'API
api.monlab.local/v2  → Nouvelle version de l'API
```

Permet de maintenir plusieurs versions simultanément.

### 3. Environnements multiples

```
dev.app.local        → Environnement de développement
staging.app.local    → Environnement de staging
prod.app.local       → Environnement de production
```

Isolation claire entre environnements.

### 4. Microservices

```
monapp.local/        → Frontend React
monapp.local/api     → Backend API
monapp.local/auth    → Service d'authentification
monapp.local/static  → Fichiers statiques
```

Architecture microservices avec routage intelligent.

## Résumé des concepts clés

🎯 **Ingress** = Règles de routage HTTP/HTTPS
🎯 **Ingress Controller** = Programme qui implémente ces règles
🎯 **Host-based routing** = Routage par nom de domaine
🎯 **Path-based routing** = Routage par chemin URL
🎯 **Annotations** = Personnalisation du comportement
🎯 **TLS termination** = Gestion SSL centralisée

## Prochaines étapes

Maintenant que vous comprenez les concepts fondamentaux d'Ingress, vous êtes prêt à :

1. **Installer et configurer** NGINX Ingress Controller (section 6.2)
2. **Créer vos premières règles** de routage
3. **Implémenter HTTPS** avec des certificats
4. **Explorer les fonctionnalités avancées** via les annotations

---

💡 **Point clé à retenir** : Ingress n'est pas magique - c'est simplement un système intelligent de règles qui dit "si quelqu'un demande X, envoie-le vers Y". L'Ingress Controller fait le travail réel de routage, et vos Services/Pods fournissent le contenu final.

⏭️
