🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 3.3 Acquisition d'un nom de domaine

## Introduction

Passer d'un DNS local à un vrai nom de domaine est le moment où votre lab devient véritablement accessible au monde entier. Cette section vous guidera à travers le processus d'acquisition d'un nom de domaine, depuis le choix du nom jusqu'à la configuration initiale, en passant par la sélection du registrar et la compréhension des coûts.

## Pourquoi un nom de domaine public ?

### Différence avec le DNS local

```
DNS Local (section précédente) :
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
✅ Gratuit
✅ Configuration instantanée
✅ Parfait pour le développement
❌ Accessible uniquement sur votre réseau
❌ Pas de certificats SSL valides
❌ Impossible de partager avec l'extérieur

Domaine Public :
━━━━━━━━━━━━━━━
✅ Accessible depuis Internet
✅ Certificats SSL gratuits (Let's Encrypt)
✅ Professionnel et mémorable
✅ Emails personnalisés possibles
❌ Coût annuel (10-50€)
❌ Configuration DNS nécessaire
```

### Cas d'usage pour votre lab

Un domaine public vous permettra de :

```
Possibilités avec un domaine :
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
• Portfolio personnel accessible partout
• Démonstrations clients en temps réel
• Accès à vos services depuis mobile/extérieur
• Webhooks pour CI/CD (GitHub, GitLab)
• Partage de projets avec la communauté
• Environnement de test réaliste
• Apprentissage des DNS/SSL en conditions réelles
```

## Comprendre les noms de domaine

### Anatomie d'un nom de domaine

```
Décomposition d'une URL complète :
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

https://app.lab.example.com:443/page
  ↓      ↓   ↓     ↓      ↓   ↓   ↓
Proto  Sub  Sub   SLD    TLD Port Chemin

• TLD (Top Level Domain) : .com, .org, .fr
• SLD (Second Level Domain) : example
• Sous-domaines : lab, app (vous contrôlez)
• FQDN complet : app.lab.example.com
```

### Types de domaines (TLD)

```
Catégories de TLD :
━━━━━━━━━━━━━━━━━━

Génériques (gTLD) :
• .com : Commercial (le plus populaire)
• .org : Organisation
• .net : Network
• .io  : Tech/Startups (Océan Indien)
• .dev : Développeurs (HTTPS obligatoire)
• .app : Applications (HTTPS obligatoire)

Pays (ccTLD) :
• .fr : France
• .be : Belgique
• .ch : Suisse
• .ca : Canada
• .de : Allemagne

Nouveaux gTLD :
• .cloud, .tech, .lab, .home
• .ninja, .guru, .expert
• Prix souvent plus élevés
```

### Choisir le bon nom

#### Critères de sélection

```
Checklist pour un bon nom :
━━━━━━━━━━━━━━━━━━━━━━━━━━
☐ Court et mémorable (moins de 15 caractères)
☐ Facile à épeler et prononcer
☐ Pas de tirets si possible
☐ Éviter les chiffres (sauf si significatifs)
☐ Vérifier la disponibilité sur réseaux sociaux
☐ Pas de marque déposée
☐ Extension adaptée à l'usage
```

#### Stratégies de nommage pour un lab

```
Approches recommandées :
━━━━━━━━━━━━━━━━━━━━━━━

1. Personnel :
   • prenom-nom.com
   • pseudonyme.dev
   • initiales-lab.io

2. Descriptif :
   • mon-lab-k8s.com
   • home-cluster.net
   • dev-playground.io

3. Créatif :
   • Nom inventé mémorable
   • Combinaison de mots
   • Acronyme personnel

Exemples concrets :
• johnsmith-lab.com
• jsdev.io
• kube-home.net
• devbox42.com
```

## Choisir un registrar

### Qu'est-ce qu'un registrar ?

```
Rôle du registrar :
━━━━━━━━━━━━━━━━━━━
• Intermédiaire agréé pour enregistrer des domaines
• Gère le renouvellement
• Fournit interface de gestion DNS
• Support technique
• Services additionnels (email, hébergement, SSL)

⚠️ Le registrar n'est PAS votre hébergeur
   Vous hébergez sur votre MicroK8s !
```

### Registrars recommandés

#### Pour débutants (interface simple)

```
Namecheap :
━━━━━━━━━━
✅ Prix compétitifs
✅ Interface intuitive
✅ WhoisGuard gratuit (anonymat)
✅ Support réactif
❌ Interface DNS basique
💰 .com : ~10€/an

Google Domains (maintenant Squarespace) :
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
✅ Interface très simple
✅ Intégration services Google
✅ Email forwarding gratuit
❌ Plus cher
💰 .com : ~12€/an
```

#### Pour utilisateurs avancés

```
Cloudflare Registrar :
━━━━━━━━━━━━━━━━━━━━━
✅ Prix coûtant (pas de marge)
✅ Protection DDoS incluse
✅ DNS ultra-rapide gratuit
✅ API puissante
❌ Besoin compte Cloudflare
❌ Transfert depuis autre registrar requis
💰 .com : ~8€/an

OVH :
━━━━━
✅ Registrar européen
✅ Prix corrects
✅ Services additionnels
❌ Interface complexe
💰 .com : ~10€/an

Gandi :
━━━━━━━
✅ Éthique, respect vie privée
✅ Nombreuses extensions
✅ Email gratuit
❌ Plus cher
💰 .com : ~15€/an
```

#### Registrars à éviter pour un lab

```
Moins adaptés pour un lab :
━━━━━━━━━━━━━━━━━━━━━━━━━━━
• GoDaddy : Pratiques commerciales agressives
• 1&1 IONOS : Support difficile, interface confuse
• Registrars gratuits : Souvent limitations cachées
• Revendeurs : Ajoutent une couche de complexité
```

### Comparaison des fonctionnalités

```
Tableau comparatif :
━━━━━━━━━━━━━━━━━━━━

| Registrar    | Prix .com | DNS | Email | API | Support | Débutant |
|--------------|-----------|-----|-------|-----|---------|----------|
| Namecheap    | €€        | ✓   | €     | ✓   | ★★★★    | ★★★★★    |
| Cloudflare   | €         | ★★★ | ✗     | ★★★ | ★★★     | ★★       |
| Google/Sq.   | €€€       | ✓   | ✓     | ✓   | ★★★     | ★★★★★    |
| OVH          | €€        | ✓   | €     | ✓   | ★★      | ★★★      |
| Gandi        | €€€       | ✓   | ✓     | ✓   | ★★★★    | ★★★★     |

Légende : € = économique, ★ = qualité
```

## Processus d'achat

### Étape 1 : Vérification de disponibilité

```
Outils de recherche :
━━━━━━━━━━━━━━━━━━━━
1. Site du registrar choisi
2. Whois lookup : whois.com
3. Instant Domain Search : instantdomainsearch.com
4. Namechk : vérifie aussi réseaux sociaux

💡 Astuce : Préparez 3-5 alternatives
```

### Étape 2 : Configuration initiale

Lors de l'achat, vous devrez fournir :

```
Informations requises :
━━━━━━━━━━━━━━━━━━━━━━
• Informations personnelles (nom, adresse)
  → Seront publiques sans protection WHOIS
• Email valide (important pour renouvellement)
• Méthode de paiement
• Durée (1 an minimum, souvent rabais sur 2-3 ans)

Options à considérer :
━━━━━━━━━━━━━━━━━━━━━
☐ Protection WHOIS (cache vos infos) : Recommandé
☐ Auto-renouvellement : Recommandé
☐ SSL : Pas nécessaire (Let's Encrypt gratuit)
☐ Email : Optionnel
☐ Hébergement : NON (vous avez MicroK8s !)
```

### Étape 3 : Paiement et activation

```
Timeline d'activation :
━━━━━━━━━━━━━━━━━━━━━━
0 min   : Paiement validé
5 min   : Domaine actif chez registrar
15 min  : Accès panel de contrôle
1-2h    : Propagation DNS initiale
24-48h  : Propagation mondiale complète

⚠️ Patience requise pour la propagation !
```

## Configuration DNS de base

### Comprendre les enregistrements DNS

```
Types d'enregistrements essentiels :
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

A Record :
• Pointe un nom vers une IPv4
• Exemple : monlab.com → 82.123.45.67

AAAA Record :
• Pointe un nom vers une IPv6
• Exemple : monlab.com → 2001:db8::1

CNAME Record :
• Alias vers un autre nom
• Exemple : www.monlab.com → monlab.com

MX Record :
• Serveurs de mail
• Exemple : mail.monlab.com (priorité 10)

TXT Record :
• Texte libre (vérifications, SPF, etc.)
• Exemple : "v=spf1 -all"

NS Record :
• Serveurs DNS autoritaires
• Généralement pré-configuré
```

### Configuration minimale pour MicroK8s

Pour commencer, vous avez besoin de seulement deux enregistrements :

```
Configuration DNS basique :
━━━━━━━━━━━━━━━━━━━━━━━━━━

Type | Nom  | Valeur          | TTL
-----|------|-----------------|------
A    | @    | VOTRE.IP.PUBLIQUE | 3600
A    | *    | VOTRE.IP.PUBLIQUE | 3600

@ = domaine racine (monlab.com)
* = wildcard (tous les sous-domaines)

Résultat :
• monlab.com → votre IP
• n'importe.quoi.monlab.com → votre IP
```

### Trouver votre IP publique

```bash
# Méthodes pour trouver votre IP publique :

# Méthode 1 : Via terminal
curl ifconfig.me
# ou
curl ipinfo.io/ip
# ou
wget -qO- icanhazip.com

# Méthode 2 : Sites web
# Visitez : whatismyipaddress.com

# Méthode 3 : Depuis votre box/routeur
# Interface admin → Informations connexion
```

## Gestion des coûts

### Coûts initiaux et récurrents

```
Structure des coûts :
━━━━━━━━━━━━━━━━━━━━

Année 1 (acquisition) :
• Domaine .com : 10-15€
• Protection WHOIS : 0-5€
• Total : 10-20€

Années suivantes :
• Renouvellement : 12-18€/an
• (souvent plus cher que l'acquisition)

Coûts cachés potentiels :
• Transfert vers autre registrar : 0-15€
• Restauration domaine expiré : 50-100€
• Changement de propriétaire : 0-20€
```

### Stratégies d'économie

```
Astuces pour réduire les coûts :
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

1. Codes promo :
   • Recherchez "registrar + coupon"
   • Newsletter = souvent -20% nouveau client
   • Black Friday/Cyber Monday

2. Achat pluriannuel :
   • 2-3 ans = souvent -10-15%
   • Protection contre augmentations

3. Extensions alternatives :
   • .xyz, .site : souvent 1-2€ première année
   • ccTLD moins connus : prix variables

4. Transfert stratégique :
   • Après 60 jours, transférer vers Cloudflare
   • Prix coûtant à vie

⚠️ Attention aux renouvellements !
   .xyz à 1€ peut renouveler à 15€
```

## Sécurité et protection

### Protection WHOIS

```
Sans protection WHOIS :
━━━━━━━━━━━━━━━━━━━━━━
$ whois monlab.com

Registrant:
  Jean Dupont
  123 rue Example
  Paris 75001
  jean@email.com
  +33123456789

⚠️ Infos publiques = SPAM garanti !

Avec protection WHOIS :
━━━━━━━━━━━━━━━━━━━━━━
$ whois monlab.com

Registrant:
  WHOIS Privacy Service
  PO Box [Registrar]
  privacy@registrar.com

✅ Vos infos restent privées
```

### Verrouillage du domaine

```
Sécurités à activer :
━━━━━━━━━━━━━━━━━━━━

Registrar Lock :
• Empêche transferts non autorisés
• TOUJOURS activer
• Désactiver seulement pour transfert voulu

2FA (Two-Factor Authentication) :
• Protection compte registrar
• Utiliser app authenticator (pas SMS)
• Sauvegarder codes de récupération

Auth Code :
• Code secret pour transferts
• Ne jamais partager
• Regenerer après usage
```

### Éviter les arnaques

```
Arnaques courantes :
━━━━━━━━━━━━━━━━━━━

1. Emails de "renouvellement" :
   ❌ "Domain Expiry Notice" d'expéditeur inconnu
   ✅ Toujours passer par votre registrar

2. SEO/Listing services :
   ❌ "Inscrivez votre domaine pour 49€"
   ✅ Inutile, Google indexe automatiquement

3. Transferts forcés :
   ❌ "Urgent: confirmez votre domaine"
   ✅ Vérifier directement chez registrar

4. Typosquatting :
   ❌ godady.com (au lieu de godaddy.com)
   ✅ Toujours vérifier l'URL
```

## Configuration pour IP dynamique

### Le problème de l'IP dynamique

```
Situation typique :
━━━━━━━━━━━━━━━━━━
• Votre FAI change votre IP régulièrement
• DNS pointe vers ancienne IP
• Site inaccessible !

Solutions :
1. IP fixe (5-20€/mois chez FAI)
2. DNS dynamique (gratuit ou 5€/mois)
3. Tunnel (Cloudflare, ngrok)
```

### Services DNS dynamique

```
DynDNS gratuits/économiques :
━━━━━━━━━━━━━━━━━━━━━━━━━━━━

DuckDNS (gratuit) :
• monlab.duckdns.org
• API simple
• Script de mise à jour

No-IP (gratuit avec limitations) :
• monlab.ddns.net
• Confirmation mensuelle requise
• Apps mobiles disponibles

Cloudflare (gratuit) :
• Utiliser API pour update
• Nécessite domaine chez CF
• Protection DDoS incluse

Configuration type :
━━━━━━━━━━━━━━━━━━━
1. CNAME monlab.com → monlab.duckdns.org
2. Script update IP toutes les 5 min
3. TTL court (300 secondes)
```

### Script de mise à jour automatique

```bash
#!/bin/bash
# update-ddns.sh - À exécuter via cron

# Pour DuckDNS
DOMAIN="monlab"
TOKEN="votre-token-duckdns"
IP=$(curl -s ifconfig.me)

curl -s "https://www.duckdns.org/update?domains=$DOMAIN&token=$TOKEN&ip=$IP"

# Pour Cloudflare (avec API)
# ZONE_ID="votre-zone-id"
# RECORD_ID="votre-record-id"
# API_KEY="votre-api-key"
#
# curl -X PUT "https://api.cloudflare.com/client/v4/zones/$ZONE_ID/dns_records/$RECORD_ID" \
#   -H "Authorization: Bearer $API_KEY" \
#   -H "Content-Type: application/json" \
#   --data "{\"type\":\"A\",\"name\":\"monlab.com\",\"content\":\"$IP\"}"
```

## Sous-domaines et organisation

### Stratégie de sous-domaines

```
Organisation recommandée :
━━━━━━━━━━━━━━━━━━━━━━━━━

Production/Public :
• monlab.com           → Site principal
• www.monlab.com       → Redirection vers racine
• api.monlab.com       → API publique
• blog.monlab.com      → Blog personnel

Développement :
• dev.monlab.com       → Environnement dev
• test.monlab.com      → Tests
• staging.monlab.com   → Pré-production

Services techniques :
• grafana.monlab.com   → Monitoring
• jenkins.monlab.com   → CI/CD
• registry.monlab.com  → Docker registry

Projets :
• projet1.monlab.com
• demo.monlab.com
• client.monlab.com
```

### Wildcard vs sous-domaines explicites

```
Approche Wildcard (* ) :
━━━━━━━━━━━━━━━━━━━━━━━
✅ Un seul enregistrement DNS
✅ Flexibilité maximale
✅ Nouveaux services sans toucher DNS
❌ Moins de contrôle
❌ Certificats wildcard plus chers

Approche Explicite :
━━━━━━━━━━━━━━━━━━━
✅ Contrôle précis
✅ Possibilité de routage différent
✅ Meilleur pour SEO
❌ Maintenance DNS pour chaque service
❌ Plus de configuration

Recommandation : Wildcard pour lab
```

## Préparer la configuration des certificats

### Prérequis pour Let's Encrypt

```
Checklist pré-certificats :
━━━━━━━━━━━━━━━━━━━━━━━━━━
☐ Domaine actif et propagé
☐ DNS A record vers votre IP
☐ Port 80 accessible depuis Internet
☐ Port 443 accessible depuis Internet
☐ Ingress controller configuré

Test de préparation :
━━━━━━━━━━━━━━━━━━━━
# DNS résout correctement ?
nslookup monlab.com

# Port 80 ouvert ?
curl http://monlab.com

# Réponse attendue :
# Connexion ou 404, pas timeout
```

### Validation de domaine

Let's Encrypt utilise différentes méthodes :

```
Méthodes de validation :
━━━━━━━━━━━━━━━━━━━━━━━

HTTP-01 (plus simple) :
• Place un fichier sur http://monlab.com/.well-known/
• Nécessite port 80 ouvert
• Parfait pour Ingress

DNS-01 (plus flexible) :
• Ajoute un TXT record au DNS
• Permet wildcard (*.monlab.com)
• Ne nécessite pas ports ouverts
• Plus complexe à automatiser

TLS-ALPN-01 (rare) :
• Validation sur port 443
• Cas d'usage spécifiques
```

## Migration depuis un autre registrar

### Quand migrer ?

```
Raisons de migrer :
━━━━━━━━━━━━━━━━━━
• Prix de renouvellement trop élevé
• Mauvais service/support
• Fonctionnalités manquantes
• Consolidation de domaines

Timing optimal :
• Après 60 jours (minimum légal)
• 1 mois avant renouvellement
• Pas pendant période critique
```

### Processus de transfert

```
Étapes de migration :
━━━━━━━━━━━━━━━━━━━━

1. Chez ancien registrar :
   • Désactiver le lock
   • Obtenir auth code (EPP code)
   • Vérifier email de contact

2. Chez nouveau registrar :
   • Initier transfert
   • Entrer auth code
   • Payer (inclut +1 an)

3. Validation :
   • Email de confirmation
   • Attendre 5-7 jours

4. Finalisation :
   • Vérifier DNS
   • Réactiver lock
   • Configurer auto-renew

⚠️ Le site reste accessible pendant transfert
```

## Check-list post-acquisition

```
À faire après l'achat :
━━━━━━━━━━━━━━━━━━━━━━
☐ Activer registrar lock
☐ Configurer 2FA sur compte
☐ Ajouter enregistrements A
☐ Tester résolution DNS
☐ Configurer auto-renouvellement
☐ Sauvegarder auth code
☐ Noter date expiration
☐ Configurer alertes renouvellement
☐ Faire backup config DNS
☐ Documenter dans votre wiki
```

## Erreurs à éviter

```
Pièges courants :
━━━━━━━━━━━━━━━━

1. Oublier de renouveler
   → Auto-renew + alertes

2. Mauvais DNS dès le début
   → Tester avant propagation

3. Acheter trop d'options
   → Domaine seul suffit

4. Utiliser email jetable
   → Email permanent requis

5. Ignorer la sécurité
   → 2FA + lock obligatoires

6. Choisir nom trop complexe
   → Simple et mémorable

7. Oublier protection WHOIS
   → Spam garanti sans
```

## Points clés à retenir

1. **Le choix du registrar est important** : Privilégiez simplicité et réputation
2. **Un .com reste le meilleur choix** : Universel et mémorable
3. **La protection WHOIS est essentielle** : Évite le spam
4. **IP dynamique n'est pas un blocage** : Solutions gratuites existent
5. **Patience pour la propagation DNS** : 24-48h pour propagation mondiale
6. **Sécurité dès le début** : 2FA + lock + protection WHOIS

## Résumé

L'acquisition d'un nom de domaine transforme votre lab local en service accessible mondialement. Pour 10-20€ par an, vous obtenez une présence professionnelle sur Internet. Le processus est simple : choisir un nom, sélectionner un registrar fiable, configurer les DNS de base, et sécuriser le tout. Dans la prochaine section, nous configurerons ces enregistrements DNS en détail pour router le trafic vers votre lab MicroK8s.

⏭️
