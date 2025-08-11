🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 5.3 Certificats Let's Encrypt automatiques

## Qu'est-ce que Let's Encrypt ?

Let's Encrypt est une autorité de certification gratuite, automatisée et ouverte. Lancée en 2016 par l'Internet Security Research Group (ISRG), elle a révolutionné l'accès aux certificats SSL/TLS en les rendant gratuits et facilement automatisables.

### Philosophie et objectifs

**Démocratiser HTTPS**
Let's Encrypt a été créé avec la mission de sécuriser l'ensemble du web en rendant les certificats SSL/TLS gratuits et accessibles à tous, des particuliers aux grandes entreprises.

**Automatisation complète**
Contrairement aux autorités de certification traditionnelles qui nécessitent des processus manuels, Let's Encrypt est conçu pour être entièrement automatisé via le protocole ACME (Automatic Certificate Management Environment).

**Transparence**
Tous les certificats émis sont publiquement loggés dans les Certificate Transparency logs, garantissant une transparence totale sur les certificats en circulation.

### Avantages pour un lab personnel

**Coût zéro**
Aucun frais d'émission, de renouvellement ou de maintenance. Idéal pour un environnement de lab où le budget peut être limité.

**Reconnus universellement**
Les certificats Let's Encrypt sont acceptés par tous les navigateurs modernes et systèmes d'exploitation, contrairement aux certificats auto-signés.

**Automation native**
Conçus dès le départ pour l'automatisation, ils s'intègrent parfaitement avec cert-manager et Kubernetes.

**Fiabilité**
Infrastructure robuste supportée par de nombreux sponsors majeurs (Mozilla, Cisco, Facebook, etc.) avec un uptime excellent.

## Protocole ACME

### Principe de fonctionnement

Le protocole ACME (Automatic Certificate Management Environment) est le cœur de Let's Encrypt. Il permet de prouver automatiquement que vous contrôlez un domaine avant d'émettre un certificat.

**Étapes du processus ACME**
1. **Inscription** : Création d'un compte ACME avec une paire de clés
2. **Demande** : Soumission d'une demande de certificat pour un ou plusieurs domaines
3. **Challenge** : Let's Encrypt propose un défi pour prouver le contrôle du domaine
4. **Validation** : Vous résolves le défi, Let's Encrypt vérifie
5. **Émission** : Si la validation réussit, le certificat est émis
6. **Récupération** : Téléchargement et installation du certificat

### Types de défis (Challenges)

**HTTP-01 Challenge**
Le plus commun et le plus simple à implémenter :
- Let's Encrypt demande la création d'un fichier temporaire
- Le fichier doit être accessible via `http://votre-domaine.com/.well-known/acme-challenge/TOKEN`
- Cert-manager gère automatiquement cette création via l'Ingress Controller

**DNS-01 Challenge**
Plus flexible mais plus complexe :
- Let's Encrypt demande la création d'un enregistrement DNS TXT
- L'enregistrement doit être créé à `_acme-challenge.votre-domaine.com`
- Nécessite l'accès API à votre fournisseur DNS
- Seule méthode pour obtenir des certificats wildcard

**TLS-ALPN-01 Challenge**
Moins utilisé dans le contexte Kubernetes :
- Utilise le protocole TLS avec l'extension ALPN
- Fonctionne sur le port 443
- Principalement pour les cas où HTTP-01 n'est pas possible

## Configuration pour HTTP-01

### Prérequis

Avant de configurer les certificats Let's Encrypt avec HTTP-01, assurez-vous que :

**DNS configuré**
- Votre domaine pointe vers l'IP publique de votre cluster MicroK8s
- Les enregistrements A ou AAAA sont propagés
- Test possible avec `nslookup votre-domaine.com`

**Port 80 accessible**
- Le port 80 doit être ouvert et routé vers votre Ingress Controller
- Vérification avec `telnet votre-domaine.com 80`
- Configuration du routeur/firewall si nécessaire

**Ingress Controller opérationnel**
- L'addon ingress de MicroK8s doit être activé
- NGINX Ingress Controller en fonctionnement
- Test avec un service simple

### ClusterIssuer pour HTTP-01

Configuration d'un ClusterIssuer utilisant la validation HTTP-01 :

```yaml
# clusterissuer-letsencrypt-http01.yaml
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: letsencrypt-prod
spec:
  acme:
    # Serveur ACME de production Let's Encrypt
    server: https://acme-v02.api.letsencrypt.org/directory

    # Email pour les notifications (obligatoire)
    email: votre-email@example.com

    # Secret qui stockera la clé privée du compte ACME
    privateKeySecretRef:
      name: letsencrypt-prod-account-key

    # Configuration du solver HTTP-01
    solvers:
    - http01:
        ingress:
          # Classe d'ingress à utiliser
          class: public

          # Optionnel : pods selector pour NGINX
          podTemplate:
            spec:
              nodeSelector:
                "kubernetes.io/os": linux
```

### Demande de certificat via Ingress

La méthode la plus simple consiste à utiliser les annotations directement sur une ressource Ingress :

```yaml
# ingress-avec-certificat.yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: mon-app-ingress
  annotations:
    # Spécifier l'issuer à utiliser
    cert-manager.io/cluster-issuer: "letsencrypt-prod"

    # Optionnel : classe d'ingress si multiple controllers
    kubernetes.io/ingress.class: "public"

    # Redirection HTTPS automatique
    nginx.ingress.kubernetes.io/ssl-redirect: "true"
spec:
  tls:
  - hosts:
    - mon-app.example.com
    # Nom du secret qui contiendra le certificat
    secretName: mon-app-tls-secret

  rules:
  - host: mon-app.example.com
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

### Demande de certificat explicite

Pour plus de contrôle, vous pouvez créer une ressource Certificate explicite :

```yaml
# certificate-explicite.yaml
apiVersion: cert-manager.io/v1
kind: Certificate
metadata:
  name: mon-app-certificate
  namespace: default
spec:
  # Nom du secret qui contiendra le certificat
  secretName: mon-app-tls-secret

  # Référence vers l'issuer
  issuerRef:
    name: letsencrypt-prod
    kind: ClusterIssuer
    group: cert-manager.io

  # Domaines couverts par le certificat
  dnsNames:
  - mon-app.example.com
  - www.mon-app.example.com

  # Optionnel : algorithme de clé
  privateKey:
    algorithm: RSA
    size: 2048
```

## Configuration pour DNS-01

### Avantages du DNS-01

**Certificats wildcard**
Seule méthode pour obtenir des certificats wildcard (*.example.com) avec Let's Encrypt.

**Pas d'exposition HTTP requise**
Idéal pour les services internes ou quand le port 80 n'est pas accessible.

**Validation de multiples domaines**
Peut valider de nombreux domaines simultanément sans limitation de ports.

### Prérequis DNS-01

**API DNS supportée**
Cert-manager supporte de nombreux fournisseurs DNS :
- Cloudflare (le plus populaire)
- Route53 (AWS)
- Google Cloud DNS
- Azure DNS
- OVH
- Et bien d'autres

**Clés API appropriées**
- Accès API avec permissions de modification des enregistrements DNS
- Tokens avec scope limité aux zones DNS nécessaires
- Stockage sécurisé des credentials

### Exemple avec Cloudflare

Configuration d'un ClusterIssuer utilisant Cloudflare pour la validation DNS-01 :

```yaml
# secret-cloudflare-api.yaml
apiVersion: v1
kind: Secret
metadata:
  name: cloudflare-api-token-secret
  namespace: cert-manager
type: Opaque
stringData:
  api-token: "votre-token-cloudflare-ici"
---
# clusterissuer-letsencrypt-dns01.yaml
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: letsencrypt-dns01
spec:
  acme:
    server: https://acme-v02.api.letsencrypt.org/directory
    email: votre-email@example.com
    privateKeySecretRef:
      name: letsencrypt-dns01-account-key

    solvers:
    - dns01:
        cloudflare:
          email: votre-email@cloudflare.com
          apiTokenSecretRef:
            name: cloudflare-api-token-secret
            key: api-token

      # Optionnel : sélecteur pour domaines spécifiques
      selector:
        dnsZones:
        - "example.com"
```

### Certificat wildcard

Avec DNS-01, vous pouvez obtenir des certificats wildcard :

```yaml
# certificate-wildcard.yaml
apiVersion: cert-manager.io/v1
kind: Certificate
metadata:
  name: wildcard-certificate
  namespace: default
spec:
  secretName: wildcard-tls-secret
  issuerRef:
    name: letsencrypt-dns01
    kind: ClusterIssuer
  dnsNames:
  - "*.example.com"
  - "example.com"  # Souvent utile d'inclure le domaine racine
```

## Gestion des environnements

### Staging vs Production

**Toujours commencer par staging**
Let's Encrypt propose un environnement de staging avec des limites de débit beaucoup plus souples :

```yaml
# clusterissuer-staging.yaml
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: letsencrypt-staging
spec:
  acme:
    # URL de staging (différente de production)
    server: https://acme-staging-v02.api.letsencrypt.org/directory
    email: votre-email@example.com
    privateKeySecretRef:
      name: letsencrypt-staging-account-key
    solvers:
    - http01:
        ingress:
          class: public
```

**Différences entre staging et production**

| Aspect | Staging | Production |
|--------|---------|------------|
| Limites de débit | Très souples | Strictes (50/semaine) |
| Confiance navigateur | Non reconnu | Reconnu universellement |
| Logs publics | Non | Oui (Certificate Transparency) |
| Usage recommandé | Tests et développement | Services réels |

### Migration staging vers production

Une fois vos tests concluants en staging :

1. **Créer l'issuer de production** avec la bonne URL ACME
2. **Modifier les annotations** des Ingress pour pointer vers le nouvel issuer
3. **Supprimer les anciens secrets** pour forcer la régénération
4. **Vérifier** que les nouveaux certificats sont reconnus par les navigateurs

## Limites et quotas

### Limites de débit Let's Encrypt

**Certificats par domaine enregistré**
- 50 certificats par domaine enregistré par semaine
- Un domaine enregistré = example.com (pas api.example.com)
- Compte les renouvellements ET les nouveaux certificats

**Nouveaux ordres**
- 300 nouveaux ordres par compte par 3 heures
- Un ordre peut couvrir jusqu'à 100 domaines

**Échecs de validation**
- 5 échecs de validation par compte par heure
- Important pour éviter les boucles infinies en cas de problème

**Certificats dupliqués**
- 5 certificats identiques par semaine
- Identique = mêmes domaines exactement

### Stratégies pour respecter les limites

**Utiliser des certificats multi-domaines**
Au lieu de créer un certificat par service, groupez plusieurs domaines :

```yaml
dnsNames:
- api.example.com
- web.example.com
- admin.example.com
```

**Privilégier les certificats wildcard**
Un seul certificat *.example.com couvre tous vos sous-domaines.

**Éviter les suppressions inutiles**
Ne supprimez pas les secrets contenant les certificats sauf nécessité absolue.

**Monitorer les échecs**
Surveillez les logs cert-manager pour détecter les problèmes rapidement.

## Monitoring et troubleshooting

### Commandes de diagnostic

```bash
# Vérifier l'état des certificats
microk8s kubectl get certificates --all-namespaces

# Détails d'un certificat spécifique
microk8s kubectl describe certificate mon-certificat

# Voir les demandes de certificat
microk8s kubectl get certificaterequests --all-namespaces

# Voir les challenges en cours
microk8s kubectl get challenges --all-namespaces

# Logs cert-manager
microk8s kubectl logs -n cert-manager deployment/cert-manager -f
```

### États des certificats

**Ready: True**
Le certificat est valide et utilisable.

**Ready: False**
Problème avec le certificat. Vérifiez les événements et les challenges.

**Renewal en cours**
Cert-manager renouvelle automatiquement le certificat (30 jours avant expiration).

### Événements courants

**Challenge Failed**
- Vérifiez la connectivité HTTP ou DNS
- Vérifiez les configurations de firewall
- Contrôlez les logs de l'Ingress Controller

**Rate Limit Exceeded**
- Vous avez atteint les limites Let's Encrypt
- Attendez ou utilisez l'environnement staging

**Invalid Email**
- L'email fourni dans l'issuer n'est pas valide
- Let's Encrypt l'utilise pour les notifications importantes

## Intégration avec les services

### Utilisation des certificats

Une fois émis, les certificats sont stockés dans des secrets Kubernetes et peuvent être utilisés par :

**Ingress Controllers**
Utilisation automatique via la section `tls` de l'Ingress.

**Services directs**
Montage du secret comme volume dans les pods qui en ont besoin.

**Applications custom**
Lecture directe du secret pour les applications qui gèrent TLS elles-mêmes.

### Exemple d'utilisation avancée

```yaml
# Ingress avec multiples certificats
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: multi-domain-ingress
  annotations:
    cert-manager.io/cluster-issuer: "letsencrypt-prod"
    nginx.ingress.kubernetes.io/ssl-redirect: "true"
spec:
  tls:
  - hosts:
    - api.example.com
    secretName: api-tls-secret
  - hosts:
    - admin.example.com
    secretName: admin-tls-secret

  rules:
  - host: api.example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: api-service
            port:
              number: 80
  - host: admin.example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: admin-service
            port:
              number: 80
```

## Préparation pour les sections suivantes

Maintenant que vous maîtrisez les certificats Let's Encrypt automatiques, vous êtes prêt pour :

- Gérer des certificats auto-signés pour le développement local
- Implémenter des certificats wildcard pour simplifier la gestion
- Mettre en place des mécanismes de renouvellement robustes
- Diagnostiquer et résoudre les problèmes courants

Les certificats Let's Encrypt automatiques constituent la base d'une infrastructure SSL/TLS moderne et sécurisée pour votre lab MicroK8s.

⏭️
