🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 5.5 Wildcard certificates

## Qu'est-ce qu'un certificat wildcard ?

Un certificat wildcard est un type spécial de certificat SSL/TLS qui peut sécuriser un domaine principal et tous ses sous-domaines de premier niveau en utilisant un seul certificat. Le terme "wildcard" (joker) fait référence au caractère astérisque (*) utilisé dans le nom du domaine pour représenter n'importe quel sous-domaine.

### Analogie simple

Imaginez que vous organisez un événement dans un grand bâtiment avec plusieurs salles. Au lieu de créer un badge d'accès spécifique pour chaque salle (salle A, salle B, salle C), vous créez un badge "accès universel" qui ouvre toutes les salles du bâtiment. Le certificat wildcard fonctionne de la même manière : un seul certificat pour accéder à tous les sous-domaines.

### Format d'un certificat wildcard

Un certificat wildcard couvre un domaine en utilisant la notation suivante :
- **Certificat pour** : `*.example.com`
- **Couvre** : `api.example.com`, `web.example.com`, `admin.example.com`, `blog.example.com`, etc.
- **Ne couvre PAS** : `example.com` (domaine racine), `sub.api.example.com` (sous-domaine de deuxième niveau)

## Avantages des certificats wildcard

### Simplification de la gestion

**Un seul certificat à maintenir**
- Plus besoin de créer un certificat par service
- Renouvellement centralisé et simplifié
- Moins de risques d'oubli d'expiration

**Déploiement rapide de nouveaux services**
- Nouveaux sous-domaines automatiquement couverts
- Pas de délai d'attente pour l'émission de certificats
- Développement et déploiement accélérés

**Réduction de la complexité opérationnelle**
- Moins de secrets Kubernetes à gérer
- Configuration Ingress simplifiée
- Monitoring centralisé

### Avantages économiques et techniques

**Optimisation des quotas**
- Un seul certificat Let's Encrypt au lieu de multiples
- Respect plus facile des limites de débit
- Moins de sollicitation des autorités de certification

**Performance améliorée**
- Moins de négociations TLS différentes
- Cache des certificats optimisé côté client
- Réduction de la latence de connexion

**Flexibilité architecturale**
- Ajout de microservices sans impact SSL
- Environnements dynamiques (dev, test, staging)
- Scaling horizontal simplifié

## Inconvénients et limitations

### Limitations de couverture

**Profondeur limitée**
- `*.example.com` ne couvre que les sous-domaines de premier niveau
- `api.v1.example.com` nécessiterait `*.v1.example.com`
- Pas de solution unique pour une hiérarchie complexe

**Domaine racine non inclus**
- `*.example.com` ne couvre pas `example.com`
- Nécessité d'inclure explicitement le domaine racine
- Configuration : `["*.example.com", "example.com"]`

### Considérations de sécurité

**Portée étendue en cas de compromission**
- Si la clé privée est compromise, tous les sous-domaines sont affectés
- Surface d'attaque potentiellement plus large
- Révocation impacte tous les services

**Validation moins granulaire**
- Impossible de révoquer l'accès à un seul sous-domaine
- Gestion des droits moins fine
- Audit de sécurité plus complexe

### Contraintes techniques

**Disponibilité limitée selon les CA**
- Tous les émetteurs ne proposent pas de wildcard gratuits
- Let's Encrypt nécessite obligatoirement DNS-01
- Processus de validation plus complexe

## Obtention avec Let's Encrypt

### Prérequis obligatoires

**Validation DNS-01 uniquement**
Let's Encrypt n'autorise les certificats wildcard qu'avec la validation DNS-01. La validation HTTP-01 ne fonctionne pas car :
- Impossible de créer des enregistrements HTTP pour tous les sous-domaines possibles
- Sécurité : éviter la validation par défaut de domaines non contrôlés

**Accès API DNS requis**
Vous devez disposer d'un accès programmatique à la gestion DNS de votre domaine via :
- Un fournisseur DNS supporté par cert-manager
- Des clés API avec permissions de modification des enregistrements
- Une configuration réseau permettant l'accès aux APIs DNS

### Configuration ClusterIssuer pour wildcard

Exemple avec Cloudflare (l'un des providers les plus populaires) :

```yaml
# Secret contenant le token API Cloudflare
apiVersion: v1
kind: Secret
metadata:
  name: cloudflare-api-token-secret
  namespace: cert-manager
type: Opaque
stringData:
  api-token: "votre-token-cloudflare-avec-permissions-dns"
---
# ClusterIssuer configuré pour les certificats wildcard
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: letsencrypt-wildcard
spec:
  acme:
    server: https://acme-v02.api.letsencrypt.org/directory
    email: votre-email@example.com
    privateKeySecretRef:
      name: letsencrypt-wildcard-private-key
    solvers:
    - dns01:
        cloudflare:
          email: compte-cloudflare@example.com
          apiTokenSecretRef:
            name: cloudflare-api-token-secret
            key: api-token
      # Optionnel : limiter aux domaines spécifiques
      selector:
        dnsZones:
        - "example.com"
```

### Demande de certificat wildcard

```yaml
# Certificat wildcard avec domaine racine inclus
apiVersion: cert-manager.io/v1
kind: Certificate
metadata:
  name: wildcard-example-com
  namespace: default
spec:
  secretName: wildcard-example-com-tls
  issuerRef:
    name: letsencrypt-wildcard
    kind: ClusterIssuer
    group: cert-manager.io

  # Inclusion explicite du domaine racine ET du wildcard
  dnsNames:
  - "*.example.com"    # Tous les sous-domaines
  - "example.com"      # Domaine racine

  # Configuration de renouvellement
  duration: 2160h      # 90 jours (maximum Let's Encrypt)
  renewBefore: 720h    # Renouveler 30 jours avant expiration

  # Paramètres de clé privée
  privateKey:
    algorithm: RSA
    size: 2048
```

## Providers DNS supportés

### Providers majeurs

**Cloudflare**
- Très populaire et fiable
- API simple et bien documentée
- Support excellent de cert-manager
- Gratuit pour la plupart des usages

**Route53 (AWS)**
- Intégration native avec l'écosystème AWS
- Performance et fiabilité excellentes
- Coût modéré pour les zones DNS

**Google Cloud DNS**
- Intégration avec Google Cloud Platform
- Scalabilité et performance élevées
- Facturation à l'usage

**Azure DNS**
- Intégration Microsoft Azure
- Fiabilité enterprise
- Tarification compétitive

### Configuration selon le provider

**Exemple Route53 (AWS)**
```yaml
# ClusterIssuer pour Route53
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: letsencrypt-route53
spec:
  acme:
    server: https://acme-v02.api.letsencrypt.org/directory
    email: votre-email@example.com
    privateKeySecretRef:
      name: letsencrypt-route53-key
    solvers:
    - dns01:
        route53:
          region: eu-west-1
          accessKeyID: AKIAIOSFODNN7EXAMPLE
          secretAccessKeySecretRef:
            name: route53-secret
            key: secret-access-key
```

**Exemple Google Cloud DNS**
```yaml
# ClusterIssuer pour Google Cloud DNS
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: letsencrypt-clouddns
spec:
  acme:
    server: https://acme-v02.api.letsencrypt.org/directory
    email: votre-email@example.com
    privateKeySecretRef:
      name: letsencrypt-clouddns-key
    solvers:
    - dns01:
        cloudDNS:
          project: mon-projet-gcp
          serviceAccountSecretRef:
            name: clouddns-service-account
            key: service-account.json
```

## Utilisation avec Ingress

### Ingress simple avec certificat wildcard

```yaml
# Ingress utilisant le certificat wildcard
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: wildcard-services-ingress
  annotations:
    nginx.ingress.kubernetes.io/ssl-redirect: "true"
    nginx.ingress.kubernetes.io/use-regex: "true"
spec:
  ingressClassName: public

  # Utilisation du certificat wildcard existant
  tls:
  - hosts:
    - "*.example.com"
    - "example.com"
    secretName: wildcard-example-com-tls  # Référence au secret créé

  rules:
  # API principal
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

  # Interface web
  - host: web.example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: web-service
            port:
              number: 80

  # Administration
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

### Ingress avec routing avancé

```yaml
# Ingress avec routing plus complexe
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: advanced-wildcard-ingress
  annotations:
    nginx.ingress.kubernetes.io/ssl-redirect: "true"
    nginx.ingress.kubernetes.io/rewrite-target: /$2
spec:
  ingressClassName: public
  tls:
  - hosts:
    - "*.example.com"
    secretName: wildcard-example-com-tls

  rules:
  # Sous-domaine API avec versions
  - host: api.example.com
    http:
      paths:
      - path: /v1(/|$)(.*)
        pathType: Prefix
        backend:
          service:
            name: api-v1-service
            port:
              number: 80
      - path: /v2(/|$)(.*)
        pathType: Prefix
        backend:
          service:
            name: api-v2-service
            port:
              number: 80

  # Monitoring avec authentification
  - host: grafana.example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: grafana-service
            port:
              number: 3000
```

## Stratégies de déploiement

### Approche centralisée

**Un seul certificat par domaine**
- Création du certificat wildcard dans un namespace dédié
- Partage du secret vers les namespaces nécessaires
- Gestion centralisée du renouvellement

```yaml
# Certificat dans namespace cert-manager
apiVersion: cert-manager.io/v1
kind: Certificate
metadata:
  name: company-wildcard
  namespace: cert-manager
spec:
  secretName: company-wildcard-tls
  issuerRef:
    name: letsencrypt-wildcard
    kind: ClusterIssuer
  dnsNames:
  - "*.company.com"
  - "company.com"
```

**Réplication du secret**
```bash
# Script pour répliquer le secret dans d'autres namespaces
kubectl get secret company-wildcard-tls -n cert-manager -o yaml | \
  sed 's/namespace: cert-manager/namespace: production/' | \
  kubectl apply -f -
```

### Approche distribuée

**Certificats par environnement**
- Certificats wildcard séparés pour dev/test/prod
- Isolation des environnements
- Gestion des droits granulaire

```yaml
# Certificat pour développement
apiVersion: cert-manager.io/v1
kind: Certificate
metadata:
  name: wildcard-dev
  namespace: development
spec:
  secretName: wildcard-dev-tls
  issuerRef:
    name: letsencrypt-wildcard
    kind: ClusterIssuer
  dnsNames:
  - "*.dev.company.com"
  - "dev.company.com"
---
# Certificat pour production
apiVersion: cert-manager.io/v1
kind: Certificate
metadata:
  name: wildcard-prod
  namespace: production
spec:
  secretName: wildcard-prod-tls
  issuerRef:
    name: letsencrypt-wildcard
    kind: ClusterIssuer
  dnsNames:
  - "*.company.com"
  - "company.com"
```

## Gestion multi-niveau

### Certificats wildcard hiérarchiques

Pour des architectures complexes avec plusieurs niveaux de sous-domaines :

```yaml
# Certificat pour sous-domaines de premier niveau
apiVersion: cert-manager.io/v1
kind: Certificate
metadata:
  name: wildcard-level1
spec:
  secretName: wildcard-level1-tls
  issuerRef:
    name: letsencrypt-wildcard
    kind: ClusterIssuer
  dnsNames:
  - "*.example.com"
  - "example.com"
---
# Certificat pour API versionnée
apiVersion: cert-manager.io/v1
kind: Certificate
metadata:
  name: wildcard-api-versions
spec:
  secretName: wildcard-api-tls
  issuerRef:
    name: letsencrypt-wildcard
    kind: ClusterIssuer
  dnsNames:
  - "*.api.example.com"   # v1.api.example.com, v2.api.example.com
  - "api.example.com"
---
# Certificat pour environnements
apiVersion: cert-manager.io/v1
kind: Certificate
metadata:
  name: wildcard-environments
spec:
  secretName: wildcard-env-tls
  issuerRef:
    name: letsencrypt-wildcard
    kind: ClusterIssuer
  dnsNames:
  - "*.dev.example.com"   # app.dev.example.com, api.dev.example.com
  - "*.staging.example.com"
  - "dev.example.com"
  - "staging.example.com"
```

## Monitoring et maintenance

### Surveillance des certificats wildcard

```bash
# Vérifier l'état des certificats wildcard
kubectl get certificates -o custom-columns=\
"NAME:.metadata.name,READY:.status.conditions[0].status,DNS:.spec.dnsNames,EXPIRES:.status.notAfter"

# Détails spécifiques d'un certificat wildcard
kubectl describe certificate wildcard-example-com

# Vérifier la couverture des domaines
openssl x509 -in <(kubectl get secret wildcard-example-com-tls -o jsonpath='{.data.tls\.crt}' | base64 -d) \
  -text -noout | grep -A 5 "Subject Alternative Name"
```

### Validation de la couverture

```bash
# Tester différents sous-domaines
for subdomain in api web admin grafana prometheus; do
  echo "Testing $subdomain.example.com..."
  curl -I https://$subdomain.example.com/ 2>/dev/null | grep "HTTP"
done

# Vérifier les détails du certificat depuis l'extérieur
echo | openssl s_client -servername api.example.com -connect api.example.com:443 2>/dev/null | \
  openssl x509 -noout -text | grep -E "(Subject:|DNS:)"
```

### Automatisation du monitoring

```yaml
# ServiceMonitor pour Prometheus (si utilisé)
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: wildcard-cert-monitor
spec:
  selector:
    matchLabels:
      app: cert-manager
  endpoints:
  - port: http-metrics
    interval: 30s
    path: /metrics
```

## Troubleshooting spécifique

### Problèmes courants avec les wildcards

**Challenge DNS-01 échoue**
- Vérifier les permissions API du provider DNS
- Contrôler la propagation DNS (`dig TXT _acme-challenge.example.com`)
- Vérifier les logs cert-manager pour les erreurs API

**Certificat ne couvre pas le domaine attendu**
- Vérifier l'inclusion explicite du domaine racine
- Contrôler la syntaxe du wildcard (`*.example.com` et non `example.com/*`)
- Vérifier les caractères spéciaux ou internationaux

**Performance de validation DNS lente**
- Choisir un provider DNS avec TTL bas
- Configurer des serveurs DNS rapides
- Optimiser la configuration cert-manager pour les timeouts

### Commandes de diagnostic

```bash
# Vérifier les challenges DNS en cours
kubectl get challenges --all-namespaces

# Logs détaillés cert-manager
kubectl logs -n cert-manager deployment/cert-manager -f | grep -i dns

# Tester la propagation DNS manuellement
dig @8.8.8.8 TXT _acme-challenge.example.com

# Vérifier les quotas Let's Encrypt
curl -s "https://crt.sh/?q=%.example.com&output=json" | jq length
```

## Bonnes pratiques

### Sécurité

**Limitation des permissions API**
- Créer des tokens DNS avec scope minimal (zone spécifique)
- Rotation régulière des clés API
- Audit des accès aux providers DNS

**Sauvegarde des certificats**
- Backup automatique des secrets contenant les wildcards
- Procédure de restauration documentée
- Tests de restauration réguliers

### Performance

**Optimisation des renouvellements**
- Renouvellement anticipé (30 jours avant expiration)
- Horaires de renouvellement en dehors des pics
- Monitoring proactif des échecs

**Cache et réutilisation**
- Partage des certificats wildcard entre services similaires
- Éviter la création de certificats redondants
- Centralisation de la gestion

## Migration et évolution

### Passage de certificats individuels vers wildcard

```bash
# Script de migration
#!/bin/bash
DOMAIN="example.com"
WILDCARD_SECRET="wildcard-${DOMAIN}-tls"

# Identifier les certificats individuels existants
kubectl get certificates -o json | jq -r \
  ".items[] | select(.spec.dnsNames[] | contains(\".$DOMAIN\")) | .metadata.name"

# Mettre à jour les Ingress pour utiliser le wildcard
kubectl get ingress -o yaml | \
  sed "s/secretName: .*-tls/secretName: $WILDCARD_SECRET/g" | \
  kubectl apply -f -
```

### Évolution vers une architecture multi-domaines

```yaml
# Configuration pour multiple domaines d'entreprise
apiVersion: cert-manager.io/v1
kind: Certificate
metadata:
  name: multi-company-wildcard
spec:
  secretName: multi-company-tls
  issuerRef:
    name: letsencrypt-wildcard
    kind: ClusterIssuer
  dnsNames:
  # Domaine principal
  - "*.company.com"
  - "company.com"
  # Domaine international
  - "*.company.fr"
  - "company.fr"
  # Domaine produit
  - "*.product.com"
  - "product.com"
```

## Préparation pour les sections suivantes

Maintenant que vous maîtrisez les certificats wildcard, vous êtes prêt pour :

- Implémenter des stratégies de renouvellement automatique robustes
- Diagnostiquer et résoudre les problèmes complexes de certificats
- Optimiser les performances et la sécurité de votre infrastructure SSL/TLS

Les certificats wildcard constituent un pilier essentiel pour une gestion efficace et scalable des certificats SSL/TLS dans votre lab MicroK8s, particulièrement quand vous hébergez de nombreux services avec des sous-domaines multiples.

⏭️
