🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 5.4 Certificats auto-signés pour le développement

## Qu'est-ce qu'un certificat auto-signé ?

Un certificat auto-signé est un certificat SSL/TLS qui n'a pas été émis par une autorité de certification reconnue, mais qui a été généré et signé par la même entité qui l'utilise. En d'autres termes, au lieu de demander à une tierce partie de confiance (comme Let's Encrypt) de garantir votre identité, vous créez votre propre "carte d'identité" numérique.

### Analogie simple

Imaginez que vous organisiez une fête privée chez vous. Pour Let's Encrypt, c'est comme demander à la mairie de vous délivrer une autorisation officielle. Pour un certificat auto-signé, c'est comme écrire vous-même une invitation sur papier libre. Dans les deux cas, vos invités peuvent entrer, mais l'invitation officielle sera reconnue par tout le monde, tandis que votre invitation maison ne sera acceptée que par ceux qui vous font confiance.

## Pourquoi utiliser des certificats auto-signés ?

### Avantages pour le développement

**Rapidité de mise en place**
- Création instantanée, pas d'attente de validation
- Aucune configuration DNS nécessaire
- Pas de dépendance externe

**Contrôle total**
- Durée de validité personnalisable
- Domaines internes ou fictifs possibles (dev.local, test.internal)
- Aucune limite de débit ou quota

**Environnement hors ligne**
- Fonctionne sans connexion Internet
- Idéal pour les environnements isolés
- Développement en local sans contraintes

**Coût nul**
- Aucun frais, même pour des besoins complexes
- Pas de compte à créer
- Génération illimitée

### Inconvénients à connaître

**Avertissements navigateur**
- Messages d'erreur "Connexion non sécurisée"
- Nécessité de cliquer "Avancé" puis "Continuer"
- Interface utilisateur dégradée

**Pas de chaîne de confiance**
- Aucune autorité reconnue ne garantit l'identité
- Applications tierces peuvent rejeter la connexion
- APIs externes souvent incompatibles

**Gestion manuelle**
- Pas de renouvellement automatique par défaut
- Distribution manuelle si plusieurs développeurs
- Configuration client parfois nécessaire

## Cas d'usage appropriés

### Environnements de développement

**Développement local**
- Tests d'intégration HTTPS sur votre machine
- Simulation d'un environnement de production
- Debug d'applications nécessitant SSL/TLS

**Environnements de test isolés**
- CI/CD dans des environnements fermés
- Tests automatisés sans accès Internet
- Validation de configurations SSL

**Services internes**
- APIs internes entre microservices
- Dashboards d'administration
- Outils de monitoring internes

### Situations à éviter

**Production accessible publiquement**
- Sites web visitables par des utilisateurs finaux
- APIs publiques utilisées par des tiers
- Services critiques nécessitant la confiance

**Intégration avec des services externes**
- Webhooks vers des services tiers
- Authentification OAuth avec des providers externes
- APIs de paiement ou bancaires

## Configuration avec Cert-Manager

### ClusterIssuer auto-signé basique

La configuration la plus simple pour générer des certificats auto-signés :

```yaml
# clusterissuer-selfsigned.yaml
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: selfsigned-issuer
spec:
  selfSigned: {}
```

Cette configuration minimaliste permet à cert-manager de générer des certificats auto-signés pour n'importe quelle demande.

### ClusterIssuer avec CA personnalisée

Pour une approche plus professionnelle, créez votre propre autorité de certification :

```yaml
# ca-certificate.yaml
apiVersion: cert-manager.io/v1
kind: Certificate
metadata:
  name: my-selfsigned-ca
  namespace: cert-manager
spec:
  isCA: true
  commonName: "My Lab CA"
  secretName: my-selfsigned-ca-secret
  privateKey:
    algorithm: ECDSA
    size: 256
  duration: 8760h # 1 an
  renewBefore: 720h # 30 jours
  subject:
    organizationUnits:
    - "Lab Personnel"
    organizations:
    - "Mon Lab"
    countries:
    - "FR"
  issuerRef:
    name: selfsigned-issuer
    kind: ClusterIssuer
    group: cert-manager.io
---
# clusterissuer-ca.yaml
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: my-ca-issuer
spec:
  ca:
    secretName: my-selfsigned-ca-secret
```

### Avantages de la CA personnalisée

**Chaîne de confiance cohérente**
- Tous vos certificats signés par la même CA
- Possibilité d'installer la CA racine sur vos machines
- Elimination des avertissements après installation

**Gestion centralisée**
- Un seul certificat racine à distribuer
- Révocation possible via la CA
- Audit et traçabilité améliorés

**Professionnalisme**
- Approche similaire aux environnements d'entreprise
- Compétences transférables
- Meilleure compréhension de PKI

## Types de certificats auto-signés

### Certificat simple domaine

Pour un service spécifique avec un seul domaine :

```yaml
# certificate-simple.yaml
apiVersion: cert-manager.io/v1
kind: Certificate
metadata:
  name: simple-dev-cert
  namespace: default
spec:
  secretName: simple-dev-tls
  duration: 2160h # 90 jours
  renewBefore: 360h # 15 jours
  isCA: false
  privateKey:
    algorithm: RSA
    size: 2048
  dnsNames:
  - dev-app.local
  - localhost
  ipAddresses:
  - 127.0.0.1
  - 192.168.1.100
  issuerRef:
    name: my-ca-issuer
    kind: ClusterIssuer
    group: cert-manager.io
```

### Certificat multi-domaines

Pour couvrir plusieurs services de développement :

```yaml
# certificate-multi-domain.yaml
apiVersion: cert-manager.io/v1
kind: Certificate
metadata:
  name: multi-dev-cert
  namespace: default
spec:
  secretName: multi-dev-tls
  duration: 2160h # 90 jours
  renewBefore: 360h # 15 jours
  privateKey:
    algorithm: RSA
    size: 2048
  dnsNames:
  - api.dev.local
  - web.dev.local
  - admin.dev.local
  - grafana.dev.local
  - prometheus.dev.local
  subject:
    organizationalUnits:
    - "Development Team"
  issuerRef:
    name: my-ca-issuer
    kind: ClusterIssuer
```

### Certificat wildcard auto-signé

Pour une flexibilité maximale en développement :

```yaml
# certificate-wildcard-dev.yaml
apiVersion: cert-manager.io/v1
kind: Certificate
metadata:
  name: wildcard-dev-cert
  namespace: default
spec:
  secretName: wildcard-dev-tls
  duration: 8760h # 1 an (plus long pour le dev)
  renewBefore: 720h # 1 mois
  privateKey:
    algorithm: ECDSA
    size: 256
  dnsNames:
  - "*.dev.local"
  - "*.test.local"
  - "dev.local"
  - "test.local"
  issuerRef:
    name: my-ca-issuer
    kind: ClusterIssuer
```

## Configuration DNS locale

### Fichier /etc/hosts

Pour que vos domaines de développement fonctionnent localement :

```bash
# Ajout dans /etc/hosts (Linux/macOS)
127.0.0.1 dev.local
127.0.0.1 api.dev.local
127.0.0.1 web.dev.local
127.0.0.1 admin.dev.local
127.0.0.1 grafana.dev.local
127.0.0.1 prometheus.dev.local

# Ou avec une IP spécifique si MicroK8s est sur une autre machine
192.168.1.100 dev.local
192.168.1.100 api.dev.local
# etc.
```

### DNS local avec dnsmasq

Pour une approche plus professionnelle, configurez dnsmasq :

```bash
# Installation sur Ubuntu
sudo apt install dnsmasq

# Configuration dans /etc/dnsmasq.conf
address=/.dev.local/127.0.0.1
address=/.test.local/127.0.0.1

# Redémarrage du service
sudo systemctl restart dnsmasq
```

### CoreDNS dans le cluster

Configuration pour résolution interne dans MicroK8s :

```yaml
# coredns-custom-config.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: coredns-custom
  namespace: kube-system
data:
  dev.local.db: |
    $ORIGIN dev.local.
    @    3600 IN SOA ns.dev.local. admin.dev.local. (
                            2023010101 ; serial
                            3600       ; refresh
                            1800       ; retry
                            604800     ; expire
                            86400 )    ; minimum
    @    3600 IN NS  ns.dev.local.
    ns   3600 IN A   127.0.0.1
    api  3600 IN A   10.152.183.1  # IP du service ingress
    web  3600 IN A   10.152.183.1
    *    3600 IN A   10.152.183.1  # Wildcard pour tous les autres
```

## Utilisation avec Ingress

### Ingress basique avec certificat auto-signé

```yaml
# ingress-dev-selfsigned.yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: dev-app-ingress
  annotations:
    cert-manager.io/cluster-issuer: "my-ca-issuer"
    nginx.ingress.kubernetes.io/ssl-redirect: "true"
    # Optionnel : backend protocol si l'app parle aussi HTTPS
    nginx.ingress.kubernetes.io/backend-protocol: "HTTP"
spec:
  ingressClassName: public
  tls:
  - hosts:
    - web.dev.local
    secretName: dev-app-tls
  rules:
  - host: web.dev.local
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: dev-app-service
            port:
              number: 80
```

### Ingress avec certificat wildcard

```yaml
# ingress-wildcard-dev.yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: wildcard-dev-ingress
  annotations:
    nginx.ingress.kubernetes.io/ssl-redirect: "true"
spec:
  ingressClassName: public
  tls:
  - hosts:
    - "*.dev.local"
    secretName: wildcard-dev-tls  # Certificat créé précédemment
  rules:
  - host: api.dev.local
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: api-service
            port:
              number: 80
  - host: admin.dev.local
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

## Installation de la CA racine

### Sur navigateurs web

**Chrome/Edge**
1. Ouvrir les paramètres → Confidentialité et sécurité → Sécurité
2. Gérer les certificats → Autorités → Importer
3. Sélectionner le fichier CA (extraire du secret Kubernetes)
4. Cocher "Faire confiance à cette CA pour identifier les sites web"

**Firefox**
1. Paramètres → Vie privée et sécurité → Certificats
2. Afficher les certificats → Autorités → Importer
3. Sélectionner le fichier CA
4. Cocher "Faire confiance à cette CA pour identifier les sites web"

### Extraction du certificat CA

```bash
# Extraire le certificat CA du secret Kubernetes
microk8s kubectl get secret my-selfsigned-ca-secret \
  -n cert-manager \
  -o jsonpath='{.data.tls\.crt}' | base64 -d > my-lab-ca.crt

# Vérifier le contenu
openssl x509 -in my-lab-ca.crt -text -noout
```

### Sur système d'exploitation

**Ubuntu/Debian**
```bash
# Copier le certificat
sudo cp my-lab-ca.crt /usr/local/share/ca-certificates/my-lab-ca.crt

# Mettre à jour le store de certificats
sudo update-ca-certificates
```

**CentOS/RHEL**
```bash
# Copier le certificat
sudo cp my-lab-ca.crt /etc/pki/ca-trust/source/anchors/

# Mettre à jour le store
sudo update-ca-trust
```

## Gestion du cycle de vie

### Durées recommandées

**Certificat CA racine**
- Durée : 5-10 ans pour un lab personnel
- Renouvellement : manuel avec redistribution
- Impact du renouvellement : tous les certificats enfants

**Certificats de services**
- Durée : 90 jours à 1 an
- Renouvellement : automatique avec cert-manager
- Impact : redémarrage des pods utilisant le certificat

### Rotation des certificats

```yaml
# Certificate avec rotation automatique
apiVersion: cert-manager.io/v1
kind: Certificate
metadata:
  name: auto-rotating-cert
spec:
  secretName: auto-rotating-tls
  duration: 720h     # 30 jours
  renewBefore: 240h  # Renouveler 10 jours avant expiration
  # Force la création d'un nouveau certificat
  privateKey:
    rotationPolicy: Always
  issuerRef:
    name: my-ca-issuer
    kind: ClusterIssuer
```

### Monitoring des expirations

```bash
# Vérifier l'expiration des certificats
microk8s kubectl get certificates -o custom-columns=\
"NAME:.metadata.name,READY:.status.conditions[0].status,EXPIRES:.status.notAfter"

# Détails d'un certificat spécifique
microk8s kubectl describe certificate my-cert

# Vérifier un certificat depuis un secret
microk8s kubectl get secret my-tls-secret -o jsonpath='{.data.tls\.crt}' | \
  base64 -d | openssl x509 -noout -dates
```

## Dépannage courant

### Problèmes fréquents

**Certificat non reconnu par le navigateur**
- Vérifier que la CA racine est installée
- Contrôler que le domaine correspond à celui du certificat
- S'assurer que le certificat n'est pas expiré

**Erreur "x509: certificate signed by unknown authority"**
- La CA racine n'est pas dans le trust store du système
- Le certificat intermédiaire manque dans la chaîne
- Configuration incorrecte de l'application cliente

**Renouvellement automatique non fonctionnel**
- Vérifier les logs cert-manager
- Contrôler la configuration renewBefore
- S'assurer que l'issuer est accessible

### Commandes de diagnostic

```bash
# Tester la connectivité HTTPS
curl -k https://api.dev.local/health

# Tester avec validation du certificat (après installation CA)
curl https://api.dev.local/health

# Vérifier les détails du certificat
openssl s_client -connect api.dev.local:443 -servername api.dev.local

# Vérifier la chaîne de certificats
openssl s_client -connect api.dev.local:443 -showcerts
```

## Bonnes pratiques

### Sécurité

**Protection de la clé privée CA**
- Stockage sécurisé du secret contenant la CA
- Backup chiffré de la clé privée
- Limitation des accès au namespace cert-manager

**Segmentation des environnements**
- CA différentes pour dev/test/staging
- Namespaces séparés pour isoler les certificats
- Policies RBAC appropriées

### Organisation

**Nommage cohérent**
- Convention de nommage pour les secrets et certificats
- Tags et labels pour faciliter la gestion
- Documentation des durées de vie

**Automatisation**
- Scripts de déploiement incluant la configuration SSL
- Tests automatisés de connectivité HTTPS
- Monitoring proactif des expirations

## Transition vers la production

### Migration vers Let's Encrypt

Quand votre développement est prêt pour un environnement accessible publiquement :

1. **Préparer les DNS publics** pointant vers votre cluster
2. **Configurer un ClusterIssuer Let's Encrypt**
3. **Modifier les annotations** des Ingress
4. **Supprimer les anciens secrets** pour forcer la régénération
5. **Tester la connectivité** depuis l'extérieur

### Conservation des certificats auto-signés

Même en production, les certificats auto-signés gardent leur utilité :

- **Services internes** entre microservices
- **APIs d'administration** non exposées publiquement
- **Environnements de développement** parallèles
- **Tests d'intégration** automatisés

## Préparation pour les sections suivantes

Maintenant que vous maîtrisez les certificats auto-signés pour le développement, vous êtes prêt pour :

- Implémenter des certificats wildcard pour simplifier la gestion à grande échelle
- Mettre en place des stratégies de renouvellement automatique robustes
- Diagnostiquer et résoudre les problèmes complexes de certificats

Les certificats auto-signés constituent un outil puissant et flexible pour les environnements de développement, vous permettant de travailler avec HTTPS sans contraintes externes.

⏭️
