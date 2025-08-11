🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 5.1 Introduction aux certificats

## Qu'est-ce qu'un certificat SSL/TLS ?

Un certificat SSL/TLS est un fichier de données numériques qui lie cryptographiquement une clé publique à l'identité d'un serveur. Pensez à lui comme à une "carte d'identité électronique" qui prouve qu'un site web ou un service est bien ce qu'il prétend être.

### Analogie simple

Imaginez que vous recevez une lettre. Comment savez-vous qu'elle provient réellement de la personne indiquée comme expéditeur ? Dans le monde physique, nous utilisons des signatures, des cachets officiels, ou nous reconnaissons l'écriture. Sur Internet, les certificats SSL/TLS jouent ce rôle d'authentification.

## Les composants d'un certificat

Un certificat SSL/TLS contient plusieurs informations essentielles :

### Informations d'identité
- **Nom du domaine** (exemple : monlab.example.com)
- **Organisation** (optionnel)
- **Localisation géographique** (pays, région, ville)

### Informations techniques
- **Clé publique** : utilisée pour chiffrer les données
- **Algorithme de chiffrement** : méthode utilisée (RSA, ECDSA)
- **Période de validité** : dates de début et de fin
- **Numéro de série** : identifiant unique du certificat

### Informations de certification
- **Autorité de certification (CA)** : qui a émis le certificat
- **Signature numérique** : preuve d'authenticité du certificat
- **Empreinte (fingerprint)** : hash unique du certificat

## Types de certificats

### Certificats selon la validation

**Certificats DV (Domain Validated)**
- Validation la plus simple : prouve seulement que vous contrôlez le domaine
- Idéal pour un lab personnel
- Gratuits avec Let's Encrypt
- Délai d'obtention : quelques minutes

**Certificats OV (Organization Validated)**
- Validation de l'organisation en plus du domaine
- Plus de confiance pour les utilisateurs
- Généralement payants
- Délai d'obtention : quelques jours

**Certificats EV (Extended Validation)**
- Validation étendue de l'organisation
- Affichage spécial dans le navigateur (barre verte historiquement)
- Coûteux et complexes à obtenir
- Principalement pour les sites e-commerce

### Certificats selon la couverture

**Certificats single-domain**
- Protègent un seul domaine (exemple : www.monlab.com)
- Les plus simples à gérer
- Parfaits pour débuter

**Certificats multi-domain (SAN)**
- Protègent plusieurs domaines différents
- Liste des domaines dans le champ "Subject Alternative Names"
- Exemple : api.monlab.com, admin.monlab.com, blog.monlab.com

**Certificats wildcard**
- Protègent tous les sous-domaines d'un domaine
- Notation avec astérisque : *.monlab.com
- Très pratiques pour un lab avec de nombreux services
- Plus complexes à obtenir gratuitement

## Autorités de certification (CA)

### CA publiques
Les autorités de certification publiques sont des organismes reconnus qui émettent des certificats acceptés par tous les navigateurs :

**Let's Encrypt**
- Gratuit et automatisé
- Certificats valides 90 jours
- Parfait pour un lab personnel
- Supporté nativement par cert-manager

**CA commerciales**
- DigiCert, GlobalSign, Sectigo, etc.
- Payantes mais avec support commercial
- Certificats valides généralement 1 à 2 ans
- Fonctionnalités avancées (assurance, support)

### CA privées (auto-signés)
Pour les environnements de développement, vous pouvez créer votre propre autorité de certification :

**Avantages**
- Contrôle total sur le processus
- Gratuit
- Idéal pour les réseaux internes
- Pas de limitation de débit

**Inconvénients**
- Non reconnus par les navigateurs (avertissement de sécurité)
- Nécessite une distribution manuelle du certificat racine
- Gestion manuelle du renouvellement

## Chaîne de certification

### Comment fonctionne la confiance ?

La sécurité des certificats repose sur une "chaîne de confiance" :

1. **Certificat racine** : installé dans le navigateur/système d'exploitation
2. **Certificat intermédiaire** : signé par la racine
3. **Certificat final** : signé par l'intermédiaire

Quand vous visitez un site HTTPS, votre navigateur vérifie cette chaîne complète.

### Exemple pratique

```
Certificat racine Let's Encrypt
    └── Certificat intermédiaire R3
        └── Certificat de votre site (monlab.example.com)
```

## Formats de certificats

### Formats courants

**PEM (Privacy Enhanced Mail)**
- Format texte, lisible
- Extension : .pem, .crt, .cer
- Commence par `-----BEGIN CERTIFICATE-----`
- Le plus utilisé dans Linux/Kubernetes

**DER (Distinguished Encoding Rules)**
- Format binaire
- Extension : .der, .cer
- Plus compact mais non lisible

**P12/PFX (PKCS#12)**
- Format binaire incluant certificat ET clé privée
- Protégé par mot de passe
- Extension : .p12, .pfx
- Utilisé dans Windows

### Dans Kubernetes

Kubernetes utilise principalement le format PEM, stocké dans des objets "Secret" :

```yaml
apiVersion: v1
kind: Secret
type: kubernetes.io/tls
metadata:
  name: mon-certificat-tls
data:
  tls.crt: <certificat encodé en base64>
  tls.key: <clé privée encodée en base64>
```

## Cycle de vie des certificats

### Étapes principales

1. **Génération de la paire de clés** : création des clés publique et privée
2. **Création de la demande (CSR)** : Certificate Signing Request
3. **Validation par la CA** : vérification de l'identité
4. **Émission du certificat** : signature par la CA
5. **Installation et configuration** : déploiement sur les serveurs
6. **Renouvellement** : avant expiration
7. **Révocation** : si compromis

### Durées de vie typiques

- **Let's Encrypt** : 90 jours (renouvellement automatique recommandé)
- **CA commerciales** : 1 à 2 ans
- **Certificats auto-signés** : durée personnalisable (souvent 1 an)

## Sécurité des certificats

### Bonnes pratiques

**Protection de la clé privée**
- Ne jamais partager la clé privée
- Stockage sécurisé (secrets Kubernetes chiffrés)
- Permissions restrictives sur les fichiers

**Monitoring de l'expiration**
- Surveillance proactive des dates d'expiration
- Alertes avant expiration
- Tests réguliers de renouvellement automatique

**Validation régulière**
- Vérification de la chaîne de certification
- Tests de connectivité HTTPS
- Monitoring des erreurs SSL dans les logs

### Risques et menaces

**Certificats expirés**
- Interruption de service
- Perte de confiance des utilisateurs
- Alertes dans les navigateurs

**Clés compromises**
- Nécessité de révocation immédiate
- Émission de nouveaux certificats
- Mise à jour de tous les services

**Certificats invalides**
- Problèmes de chaîne de certification
- Mauvaise configuration des domaines
- Erreurs de déploiement

## Certificats dans l'écosystème Kubernetes

### Niveaux d'utilisation

**Communications internes au cluster**
- Entre pods et services
- API server vers etcd
- Généralement gérés automatiquement par Kubernetes

**Communications externes (Ingress)**
- Clients vers services exposés
- Gestion via Ingress Controller
- C'est ici que cert-manager intervient

**Communications admin**
- kubectl vers API server
- Dashboard Kubernetes
- Outils de monitoring

### Intégration avec MicroK8s

MicroK8s simplifie la gestion grâce aux addons :

- **cert-manager** : automatisation complète
- **ingress** : intégration native avec les certificats
- **dashboard** : interface graphique pour visualiser les certificats

## Préparation pour les sections suivantes

Maintenant que vous comprenez les concepts fondamentaux des certificats SSL/TLS, nous allons voir dans les sections suivantes comment :

- Installer et configurer cert-manager sur MicroK8s
- Obtenir automatiquement des certificats Let's Encrypt
- Générer des certificats auto-signés pour le développement
- Gérer des certificats wildcard
- Automatiser le renouvellement
- Diagnostiquer et résoudre les problèmes courants

Cette base théorique vous permettra de mieux appréhender les aspects pratiques et de comprendre les choix techniques que nous ferons dans la suite du chapitre.

⏭️
