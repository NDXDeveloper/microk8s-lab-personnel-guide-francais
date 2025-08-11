🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 5.2 Configuration de Cert-Manager

## Qu'est-ce que Cert-Manager ?

Cert-Manager est l'outil de référence dans l'écosystème Kubernetes pour automatiser la gestion des certificats SSL/TLS. Il agit comme un "assistant intelligent" qui s'occupe de toutes les tâches fastidieuses liées aux certificats : demande, obtention, installation, renouvellement et même révocation si nécessaire.

### Pourquoi utiliser Cert-Manager ?

**Automatisation complète**
Sans cert-manager, vous devriez manuellement :
- Générer des demandes de certificat (CSR)
- Les soumettre à une autorité de certification
- Récupérer les certificats émis
- Les installer dans vos services
- Surveiller leur expiration
- Les renouveler régulièrement

Avec cert-manager, tout cela se fait automatiquement.

**Intégration native Kubernetes**
Cert-manager comprend les concepts Kubernetes et s'intègre parfaitement avec :
- Les Ingress Controllers
- Les secrets TLS
- Les annotations des ressources
- Le système d'événements Kubernetes

**Support multi-CA**
Cert-manager peut travailler avec différentes autorités de certification :
- Let's Encrypt (gratuit)
- CA privées (auto-signées)
- Venafi (entreprise)
- HashiCorp Vault
- Et bien d'autres

## Architecture de Cert-Manager

### Composants principaux

**Controller Principal**
Le cœur de cert-manager qui surveille les ressources Kubernetes et orchestre les opérations de certification.

**Webhook**
Composant qui valide et modifie les ressources cert-manager avant qu'elles soient stockées dans etcd.

**CA Injector**
Service qui injecte automatiquement les certificats CA dans les ConfigMaps et les Secrets, utile pour les CA privées.

### Concepts clés

**Certificate**
Ressource Kubernetes qui décrit un certificat souhaité. Cert-manager se charge de l'obtenir et de le maintenir à jour.

**Issuer / ClusterIssuer**
Ressource qui définit une autorité de certification. Un Issuer est limité à un namespace, un ClusterIssuer est disponible dans tout le cluster.

**CertificateRequest**
Ressource créée automatiquement par cert-manager pour représenter une demande de certificat spécifique.

**Challenge**
Pour Let's Encrypt, représente un défi à résoudre pour prouver la propriété d'un domaine.

## Installation sur MicroK8s

### Méthode recommandée : Addon officiel

MicroK8s propose cert-manager comme addon officiel, ce qui simplifie grandement l'installation :

```bash
# Activation de l'addon cert-manager
microk8s enable cert-manager
```

Cette commande :
- Installe cert-manager dans le namespace `cert-manager`
- Configure tous les composants nécessaires
- Applique les bonnes pratiques de sécurité
- Met en place les webhooks d'admission

### Vérification de l'installation

Après l'activation, vérifiez que tous les composants sont opérationnels :

```bash
# Vérifier les pods cert-manager
microk8s kubectl get pods -n cert-manager

# Vous devriez voir quelque chose comme :
# NAME                                      READY   STATUS    RESTARTS   AGE
# cert-manager-5d7f97b46d-xyz123           1/1     Running   0          2m
# cert-manager-cainjector-69d7cb5d8-abc456 1/1     Running   0          2m
# cert-manager-webhook-8d7495f4c-def789    1/1     Running   0          2m
```

```bash
# Vérifier les CRDs (Custom Resource Definitions)
microk8s kubectl get crd | grep cert-manager

# Vous devriez voir des ressources comme :
# certificaterequests.cert-manager.io
# certificates.cert-manager.io
# challenges.acme.cert-manager.io
# clusterissuers.cert-manager.io
# issuers.cert-manager.io
# orders.acme.cert-manager.io
```

### Installation manuelle (alternative)

Si vous préférez une installation manuelle ou si l'addon n'est pas disponible :

```bash
# Ajout du repository Helm de cert-manager
microk8s helm3 repo add jetstack https://charts.jetstack.io
microk8s helm3 repo update

# Installation avec Helm
microk8s helm3 install cert-manager jetstack/cert-manager \
  --namespace cert-manager \
  --create-namespace \
  --version v1.13.0 \
  --set installCRDs=true
```

## Configuration des Issuers

### ClusterIssuer vs Issuer

**ClusterIssuer** (recommandé pour un lab)
- Disponible dans tous les namespaces du cluster
- Une seule configuration pour tout le cluster
- Simplifie la gestion
- Idéal quand vous gérez seul votre lab

**Issuer**
- Limité à un namespace spécifique
- Plus de granularité
- Utile dans des environnements multi-tenant
- Sécurité renforcée par isolation

### Issuer pour Let's Encrypt (environnement de test)

Commencez toujours par l'environnement de test (staging) de Let's Encrypt :

```yaml
# clusterissuer-letsencrypt-staging.yaml
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: letsencrypt-staging
spec:
  acme:
    # URL du serveur ACME de test Let's Encrypt
    server: https://acme-staging-v02.api.letsencrypt.org/directory

    # Votre email pour les notifications importantes
    email: votre-email@example.com

    # Secret qui stockera la clé privée ACME
    privateKeySecretRef:
      name: letsencrypt-staging-private-key

    # Resolver pour la validation HTTP-01
    solvers:
    - http01:
        ingress:
          class: public
```

### Issuer pour Let's Encrypt (production)

Une fois les tests concluants, configurez l'environnement de production :

```yaml
# clusterissuer-letsencrypt-prod.yaml
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: letsencrypt-prod
spec:
  acme:
    # URL du serveur ACME de production Let's Encrypt
    server: https://acme-v02.api.letsencrypt.org/directory

    # Votre email pour les notifications
    email: votre-email@example.com

    # Secret qui stockera la clé privée ACME
    privateKeySecretRef:
      name: letsencrypt-prod-private-key

    # Resolver pour la validation HTTP-01
    solvers:
    - http01:
        ingress:
          class: public
```

### Issuer pour certificats auto-signés

Pour le développement local ou les services internes :

```yaml
# clusterissuer-selfsigned.yaml
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: selfsigned-issuer
spec:
  selfSigned: {}
```

## Types de validation ACME

### HTTP-01 Challenge

**Principe**
Let's Encrypt place un fichier temporaire à une URL spécifique sur votre domaine. Cert-manager doit rendre ce fichier accessible pour prouver le contrôle du domaine.

**Avantages**
- Simple à configurer
- Fonctionne avec la plupart des setups
- Pas de configuration DNS nécessaire

**Inconvénients**
- Nécessite que le port 80 soit accessible depuis Internet
- Ne fonctionne pas pour les certificats wildcard
- Chaque domaine doit être validé individuellement

**Configuration type**
```yaml
solvers:
- http01:
    ingress:
      class: public
      # Optionnel : spécifier un ingress controller particulier
      name: nginx
```

### DNS-01 Challenge

**Principe**
Let's Encrypt demande la création d'un enregistrement DNS TXT spécifique. Cert-manager doit pouvoir modifier vos enregistrements DNS automatiquement.

**Avantages**
- Fonctionne pour les certificats wildcard
- Pas besoin d'exposition HTTP
- Validation de domaines internes possible

**Inconvénients**
- Configuration plus complexe
- Nécessite une API DNS supportée
- Délais de propagation DNS

**Configuration type (exemple avec Cloudflare)**
```yaml
solvers:
- dns01:
    cloudflare:
      email: votre-email@example.com
      apiTokenSecretRef:
        name: cloudflare-api-token-secret
        key: api-token
```

## Configuration avancée

### Sélecteurs et contraintes

Pour des configurations plus fines, vous pouvez utiliser des sélecteurs :

```yaml
# Issuer avec sélecteurs
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: letsencrypt-prod-advanced
spec:
  acme:
    server: https://acme-v02.api.letsencrypt.org/directory
    email: votre-email@example.com
    privateKeySecretRef:
      name: letsencrypt-prod-private-key
    solvers:
    # Solver spécifique pour les domaines wildcard
    - dns01:
        cloudflare:
          email: votre-email@example.com
          apiTokenSecretRef:
            name: cloudflare-api-token-secret
            key: api-token
      selector:
        dnsNames:
        - "*.example.com"

    # Solver HTTP pour les autres domaines
    - http01:
        ingress:
          class: public
      selector:
        dnsNames:
        - "api.example.com"
        - "www.example.com"
```

### Gestion des secrets

Les secrets contenant les clés API doivent être créés manuellement :

```yaml
# Secret pour API Cloudflare
apiVersion: v1
kind: Secret
metadata:
  name: cloudflare-api-token-secret
  namespace: cert-manager
type: Opaque
stringData:
  api-token: "votre-token-cloudflare-ici"
```

### Limites de débit

Let's Encrypt impose des limites de débit importantes :

**Environnement de production**
- 50 certificats par domaine enregistré par semaine
- 300 nouveaux ordres par compte par 3 heures
- 5 échecs de validation par compte par heure

**Bonnes pratiques**
- Toujours tester en staging d'abord
- Utiliser des certificats wildcard quand possible
- Monitorer les échecs et erreurs
- Ne pas supprimer les secrets contenant les clés ACME

## Surveillance et monitoring

### Commandes utiles

```bash
# Voir tous les issuers
microk8s kubectl get clusterissuers

# Détails d'un issuer
microk8s kubectl describe clusterissuer letsencrypt-prod

# Voir les certificats
microk8s kubectl get certificates --all-namespaces

# Voir les demandes de certificat
microk8s kubectl get certificaterequests --all-namespaces

# Logs de cert-manager
microk8s kubectl logs -n cert-manager deployment/cert-manager
```

### Événements Kubernetes

Cert-manager génère des événements Kubernetes informatifs :

```bash
# Voir les événements liés aux certificats
microk8s kubectl get events --field-selector reason=Issuing

# Événements d'un certificat spécifique
microk8s kubectl describe certificate mon-certificat
```

## Intégration avec l'écosystème

### Compatibilité Ingress Controllers

Cert-manager fonctionne avec la plupart des Ingress Controllers :

**NGINX Ingress Controller** (recommandé pour MicroK8s)
- Support natif excellent
- Annotations simples
- Gestion automatique des challenges HTTP-01

**Traefik**
- Support natif également
- Configuration via annotations
- Intégration Let's Encrypt native en plus

**HAProxy Ingress**
- Support via annotations
- Configuration légèrement plus complexe

### Annotations importantes

Les annotations permettent de contrôler le comportement de cert-manager :

```yaml
# Dans une ressource Ingress
metadata:
  annotations:
    # Spécifier l'issuer à utiliser
    cert-manager.io/cluster-issuer: "letsencrypt-prod"

    # Forcer le renouvellement
    cert-manager.io/force-renewal: "true"

    # Spécifier la classe d'ingress
    cert-manager.io/ingress-class: "nginx"
```

## Préparation pour les sections suivantes

Maintenant que cert-manager est installé et configuré, vous êtes prêt pour :

- Obtenir vos premiers certificats Let's Encrypt automatiquement
- Gérer des certificats auto-signés pour le développement
- Implémenter des certificats wildcard
- Mettre en place le renouvellement automatique
- Diagnostiquer et résoudre les problèmes courants

Cette configuration de base de cert-manager constitue le fondement de toute stratégie de gestion de certificats robuste dans votre lab MicroK8s.

⏭️
