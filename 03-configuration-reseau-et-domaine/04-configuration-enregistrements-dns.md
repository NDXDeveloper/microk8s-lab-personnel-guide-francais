🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 3.4 Configuration des enregistrements DNS

## Introduction

Maintenant que vous possédez un nom de domaine, il est temps de le configurer pour qu'il pointe vers votre lab MicroK8s. Cette section vous guidera à travers la configuration détaillée des enregistrements DNS, depuis les concepts de base jusqu'aux configurations avancées pour un lab professionnel.

## Comprendre le système DNS

### Comment fonctionne la résolution DNS

```
Voyage d'une requête DNS :
━━━━━━━━━━━━━━━━━━━━━━━━━━

Utilisateur tape : app.monlab.com
            ↓
1. Cache navigateur → "Je connais ?" → Non
            ↓
2. Cache OS → "Je connais ?" → Non
            ↓
3. Résolveur FAI → "Je connais ?" → Non
            ↓
4. Root servers → ".com est géré par..."
            ↓
5. TLD servers .com → "monlab.com est géré par..."
            ↓
6. Serveurs du registrar → "app.monlab.com = 82.123.45.67"
            ↓
7. Retour à l'utilisateur avec l'IP

⏱️ Première requête : ~50-200ms
⏱️ Requêtes suivantes : <1ms (cache)
```

### La hiérarchie DNS

```
Structure hiérarchique DNS :
━━━━━━━━━━━━━━━━━━━━━━━━━━━

                    . (root)
                      ↓
        ┌─────────────┼─────────────┐
       .com         .org           .fr
        ↓            ↓              ↓
    monlab.com   example.org   site.fr
        ↓
    ┌───┼───┐
   www api app

Chaque niveau a ses serveurs DNS autoritaires
```

### TTL (Time To Live)

Le TTL détermine combien de temps une réponse DNS est mise en cache :

```
Valeurs TTL courantes :
━━━━━━━━━━━━━━━━━━━━━━━

300 (5 min) :
• Pour tests et changements fréquents
• IP dynamique
• Migration en cours

3600 (1 heure) :
• Bon compromis général
• Configuration standard

86400 (24 heures) :
• Configuration stable
• IP fixe
• Production

Règle : TTL court = propagation rapide mais plus de requêtes
```

## Types d'enregistrements DNS essentiels

### Enregistrement A (Address)

L'enregistrement A est le plus fondamental - il associe un nom à une adresse IPv4.

```
Format enregistrement A :
━━━━━━━━━━━━━━━━━━━━━━━━
Nom          Type  TTL   Valeur
monlab.com   A     3600  82.123.45.67
www          A     3600  82.123.45.67
api          A     3600  82.123.45.67

Cas d'usage :
• Domaine principal vers IP serveur
• Sous-domaines vers IP serveur
• Load balancing (plusieurs A records)
```

### Enregistrement AAAA (IPv6)

Pour les adresses IPv6, de plus en plus importantes :

```
Format enregistrement AAAA :
━━━━━━━━━━━━━━━━━━━━━━━━━━━
Nom          Type  TTL   Valeur
monlab.com   AAAA  3600  2001:db8::1
www          AAAA  3600  2001:db8::1

Note : Votre FAI doit supporter IPv6
      Vérifiez avec : test-ipv6.com
```

### Enregistrement CNAME (Canonical Name)

Un CNAME crée un alias vers un autre nom de domaine :

```
Format enregistrement CNAME :
━━━━━━━━━━━━━━━━━━━━━━━━━━━━
Nom          Type   TTL   Valeur
www          CNAME  3600  monlab.com
blog         CNAME  3600  monlab.com
shop         CNAME  3600  shopify.myshop.com

⚠️ Règles importantes :
• Pas de CNAME sur domaine racine (@)
• Pas d'autres enregistrements avec CNAME
• Éviter chaînes de CNAME
```

### Enregistrement MX (Mail Exchange)

Pour recevoir des emails sur votre domaine :

```
Format enregistrement MX :
━━━━━━━━━━━━━━━━━━━━━━━━━
Nom          Type  Priority  TTL   Valeur
@            MX    10        3600  mail.monlab.com
@            MX    20        3600  backup.monlab.com

Priority : Plus bas = priorité haute
Note : Nécessite serveur mail configuré
```

### Enregistrement TXT

Pour diverses validations et métadonnées :

```
Format enregistrement TXT :
━━━━━━━━━━━━━━━━━━━━━━━━━━
Nom          Type  TTL   Valeur
@            TXT   3600  "v=spf1 -all"
_dmarc       TXT   3600  "v=DMARC1; p=none"
@            TXT   3600  "google-site-verification=..."

Usages courants :
• SPF (anti-spam email)
• Vérification propriété (Google, Let's Encrypt)
• DMARC/DKIM (sécurité email)
```

### Enregistrement CAA

Pour contrôler qui peut émettre des certificats SSL :

```
Format enregistrement CAA :
━━━━━━━━━━━━━━━━━━━━━━━━━━
Nom    Type  TTL   Flags  Tag    Valeur
@      CAA   3600  0      issue  "letsencrypt.org"
@      CAA   3600  0      iodef  "mailto:admin@monlab.com"

Sécurité : Empêche émission frauduleuse de certificats
```

## Configuration pour votre lab MicroK8s

### Configuration minimale

Pour démarrer rapidement votre lab :

```
Configuration DNS minimale :
━━━━━━━━━━━━━━━━━━━━━━━━━━━

Type  Nom  Valeur           TTL   Commentaire
A     @    VOTRE_IP_PUBLIC  3600  Domaine racine
A     *    VOTRE_IP_PUBLIC  3600  Wildcard (tous sous-domaines)

Résultat :
✓ monlab.com → Votre IP
✓ n'importe.quoi.monlab.com → Votre IP
```

### Configuration recommandée

Une configuration plus complète et professionnelle :

```
Configuration DNS recommandée :
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

Type   Nom         Valeur               TTL    Usage
A      @           82.123.45.67         3600   Domaine principal
A      www         82.123.45.67         3600   Site web
A      api         82.123.45.67         3600   API
A      app         82.123.45.67         3600   Application
A      dashboard   82.123.45.67         3600   Dashboard K8s
A      grafana     82.123.45.67         3600   Monitoring
A      *           82.123.45.67         300    Catch-all
TXT    @           "v=spf1 -all"        3600   Anti-spam
CAA    @           0 issue "letsencrypt.org"  3600   Sécurité SSL
```

### Configuration avancée avec sous-zones

Pour organiser vos environnements :

```
Structure avec sous-zones :
━━━━━━━━━━━━━━━━━━━━━━━━━━

# Production
A      @           82.123.45.67    3600
A      www         82.123.45.67    3600
A      api         82.123.45.67    3600

# Développement
A      *.dev       82.123.45.67    300
A      *.test      82.123.45.67    300
A      *.staging   82.123.45.67    300

# Services internes
A      *.internal  82.123.45.67    3600
A      *.tools     82.123.45.67    3600

Exemples d'accès :
• feature-x.dev.monlab.com
• v2.staging.monlab.com
• jenkins.tools.monlab.com
```

## Interface de gestion DNS

### Accès au panneau DNS

Chaque registrar a son interface, mais les principes restent similaires :

```
Navigation typique :
━━━━━━━━━━━━━━━━━━━━━

1. Connexion au registrar
   ↓
2. "Mes domaines" / "Domain list"
   ↓
3. Sélectionner votre domaine
   ↓
4. "Gérer DNS" / "DNS Settings"
   ↓
5. "Enregistrements" / "DNS Records"
```

### Interface Namecheap (exemple)

```
Interface Namecheap :
━━━━━━━━━━━━━━━━━━━━

Advanced DNS → Add New Record

┌─────────────────────────────────────┐
│ Type: [A Record    ▼]               │
│ Host: [@          ]                 │
│ Value: [82.123.45.67    ]           │
│ TTL: [Automatic ▼]                  │
│      [✓ Save]                       │
└─────────────────────────────────────┘

Host spéciaux :
• @ = domaine racine
• * = wildcard
• www = sous-domaine www
```

### Interface Cloudflare (exemple)

```
Interface Cloudflare :
━━━━━━━━━━━━━━━━━━━━━

DNS → Records → Add Record

┌─────────────────────────────────────┐
│ Type: [A         ▼]                 │
│ Name: [app       ]                  │
│ IPv4: [82.123.45.67    ]           │
│ Proxy: [☁️ Proxied ▼]               │
│ TTL: [Auto]                         │
│       [Save]                        │
└─────────────────────────────────────┘

⚠️ Proxy Cloudflare :
• Orange ☁️ = Trafic via Cloudflare (protection DDoS)
• Gris ☁️ = DNS only (direct vers votre IP)
```

## Propagation DNS

### Comprendre la propagation

```
Processus de propagation :
━━━━━━━━━━━━━━━━━━━━━━━━━

T+0min : Vous modifiez l'enregistrement
   ↓
T+5min : Serveurs du registrar mis à jour
   ↓
T+1h : Certains FAI voient le changement
   ↓
T+4h : ~50% d'Internet voit le changement
   ↓
T+24h : ~95% d'Internet voit le changement
   ↓
T+48h : Propagation complète mondiale

Facteurs influençant la vitesse :
• TTL de l'ancien enregistrement
• Cache des résolveurs DNS
• Fréquence de mise à jour des FAI
```

### Vérifier la propagation

```bash
# Outils en ligne de commande

# Linux/Mac - Vérifier résolution locale
dig monlab.com
nslookup monlab.com

# Vérifier avec serveur DNS spécifique
dig @8.8.8.8 monlab.com        # Google DNS
dig @1.1.1.1 monlab.com        # Cloudflare DNS
dig @208.67.222.222 monlab.com # OpenDNS

# Voir le chemin complet de résolution
dig +trace monlab.com

# Windows
nslookup monlab.com 8.8.8.8
```

### Sites web de vérification

```
Outils web recommandés :
━━━━━━━━━━━━━━━━━━━━━━━

whatsmydns.net :
• Vérifie depuis ~30 serveurs mondiaux
• Interface visuelle carte
• Historique de propagation

dnschecker.org :
• Plus de 100 serveurs
• Détails par région

dnsmap.io :
• Propagation en temps réel
• Graphiques de progression

nslookup.io :
• Interface simple
• Multiple types d'enregistrements
```

## Stratégies de migration

### Migration sans interruption

Pour migrer d'un serveur à un autre sans coupure :

```
Stratégie de migration douce :
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

Jour J-2 :
1. Réduire TTL à 300 (5 min)
2. Attendre propagation (24h)

Jour J :
3. Ajouter nouvelle IP (garder ancienne)
4. Tester nouvelle IP directement
5. Supprimer ancienne IP
6. Surveiller les logs

Jour J+2 :
7. Remonter TTL à 3600
8. Migration terminée

✅ Zéro downtime !
```

### Rollback en cas de problème

```
Plan de rollback :
━━━━━━━━━━━━━━━━━━

Si problème détecté :
1. Remettre ancienne IP immédiatement
2. TTL court = rollback en 5 min
3. Investiguer le problème
4. Retenter plus tard

Toujours garder :
• Ancienne config en backup
• Accès à l'ancien serveur 48h
• Logs des deux serveurs
```

## DNS pour environnements multiples

### Approche par sous-domaine

```
Organisation par environnement :
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

Production :
A    monlab.com         82.123.45.67
A    www.monlab.com     82.123.45.67
A    api.monlab.com     82.123.45.67

Staging :
A    staging.monlab.com      82.123.45.67
A    api.staging.monlab.com  82.123.45.67

Développement :
A    dev.monlab.com          82.123.45.67
A    *.dev.monlab.com        82.123.45.67

Avantages :
✅ Séparation claire
✅ Certificats SSL distincts possibles
✅ Règles firewall par environnement
```

### Approche par port (alternative)

```
Si un seul domaine disponible :
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

DNS unique :
A    monlab.com    82.123.45.67

Routage par port :
• monlab.com:443  → Production
• monlab.com:8443 → Staging
• monlab.com:9443 → Dev

⚠️ Moins élégant mais fonctionnel
   Nécessite spécifier les ports
```

## Configuration pour services spécifiques

### Configuration pour GitLab/GitHub webhooks

```
DNS pour CI/CD webhooks :
━━━━━━━━━━━━━━━━━━━━━━━━

A    ci.monlab.com      82.123.45.67  3600
A    webhook.monlab.com 82.123.45.67  3600

Utilisation :
• GitHub webhook → https://ci.monlab.com/hook
• GitLab runner → https://ci.monlab.com
• ArgoCD → https://ci.monlab.com/argo
```

### Configuration pour registry Docker

```
DNS pour registry privé :
━━━━━━━━━━━━━━━━━━━━━━━━

A    registry.monlab.com    82.123.45.67  3600
A    docker.monlab.com      82.123.45.67  3600

Usage après configuration :
docker push registry.monlab.com/mon-image:v1
docker pull registry.monlab.com/mon-image:v1
```

### Configuration pour monitoring

```
DNS pour stack monitoring :
━━━━━━━━━━━━━━━━━━━━━━━━━━

A    grafana.monlab.com     82.123.45.67  3600
A    prometheus.monlab.com  82.123.45.67  3600
A    alerts.monlab.com       82.123.45.67  3600
A    logs.monlab.com         82.123.45.67  3600

Organisation claire des services
```

## Sécurité DNS

### DNSSEC

DNSSEC ajoute une couche de sécurité par signature cryptographique :

```
DNSSEC en bref :
━━━━━━━━━━━━━━━

Avantages :
✅ Authentifie les réponses DNS
✅ Empêche l'empoisonnement DNS
✅ Protection contre MITM

Inconvénients :
❌ Configuration complexe
❌ Pas toujours supporté
❌ Peut casser si mal configuré

Pour un lab : Optionnel
Pour production : Recommandé
```

### Monitoring des changements

```
Surveillance DNS :
━━━━━━━━━━━━━━━━━

Services de monitoring :
• DNSPerf - Performance et uptime
• StatusCake - Alertes changements
• UptimeRobot - Monitoring gratuit

Script de vérification :
━━━━━━━━━━━━━━━━━━━━━━
#!/bin/bash
EXPECTED_IP="82.123.45.67"
CURRENT_IP=$(dig +short monlab.com)

if [ "$CURRENT_IP" != "$EXPECTED_IP" ]; then
    echo "ALERTE: IP changée!"
    # Envoyer notification
fi
```

### Protection contre DNS hijacking

```
Mesures de protection :
━━━━━━━━━━━━━━━━━━━━━━

1. Registrar :
   • 2FA obligatoire
   • Domain lock activé
   • Alertes email changements

2. DNS :
   • Logs des modifications
   • Backup configuration
   • CAA records

3. Monitoring :
   • Vérification régulière
   • Alertes automatiques
```

## Optimisation des performances

### Choix des serveurs DNS

```
Performance par provider :
━━━━━━━━━━━━━━━━━━━━━━━━━

Cloudflare (1.1.1.1) :
• ~10ms moyenne mondiale
• Anycast network
• Privacy-focused

Google (8.8.8.8) :
• ~15ms moyenne
• Très fiable
• Cache important

DNS du registrar :
• Variable (30-100ms)
• Souvent plus lent
• Inclus dans le prix

Recommandation : Cloudflare pour performance
```

### Configuration du cache

```
Optimisation TTL :
━━━━━━━━━━━━━━━━━

Développement actif :
• TTL : 300 (5 min)
• Changements fréquents OK

Production stable :
• TTL : 3600-86400
• Moins de requêtes DNS
• Meilleure performance

Règle : Augmenter TTL progressivement
        quand configuration stable
```

## Troubleshooting DNS

### Problèmes courants

```
Diagnostic rapide :
━━━━━━━━━━━━━━━━━━

"Site inaccessible" :
1. DNS résout ? → dig monlab.com
2. Bonne IP ? → Vérifier configuration
3. Propagation ? → whatsmydns.net
4. Cache local ? → Vider cache

"Mauvais certificat SSL" :
• DNS pointe vers bonne IP ?
• Nom dans certificat = nom DNS ?
• Wildcard nécessaire ?

"Certains utilisateurs seulement" :
• Propagation incomplète
• Cache FAI
• Attendre 24-48h
```

### Commandes de diagnostic

```bash
# Suite de diagnostic complète

# 1. Résolution basique
dig monlab.com

# 2. Résolution avec trace
dig +trace monlab.com

# 3. Tous les enregistrements
dig monlab.com ANY

# 4. Test depuis différents serveurs
for dns in 8.8.8.8 1.1.1.1 208.67.222.222; do
  echo "Test avec $dns:"
  dig @$dns monlab.com +short
done

# 5. Reverse DNS
dig -x 82.123.45.67

# 6. Vérifier SOA (zone)
dig monlab.com SOA

# 7. Vérifier NS (name servers)
dig monlab.com NS
```

## Check-list de configuration

```
Validation de configuration :
━━━━━━━━━━━━━━━━━━━━━━━━━━━━

Basique :
☐ A record pour @ configuré
☐ A record pour www configuré
☐ TTL approprié (300-3600)
☐ IP correcte vérifiée

Avancé :
☐ Wildcard * configuré
☐ CAA record pour sécurité
☐ TXT SPF si email
☐ Sous-domaines services

Tests :
☐ dig monlab.com fonctionne
☐ Propagation vérifiée (whatsmydns.net)
☐ Accès HTTP/HTTPS testé
☐ Certificat SSL valide

Documentation :
☐ Configuration sauvegardée
☐ Changements documentés
☐ Plan de rollback prêt
```

## Préparation pour l'étape suivante

Avec vos DNS configurés, vous êtes prêt pour :

1. **Configurer la redirection de ports** (Section 3.5)
2. **Mettre en place le firewall** (Section 3.6)
3. **Obtenir des certificats SSL** (Section 5)

Vos enregistrements DNS sont la fondation de l'accessibilité de votre lab. Une fois correctement configurés, ils permettront à vos utilisateurs d'accéder à vos services par des noms mémorables plutôt que des IPs.

## Points clés à retenir

1. **Commencez simple** : A record pour @ et *, puis complexifier
2. **TTL court pendant les tests** : Facilite les modifications
3. **Patience pour la propagation** : 24-48h pour propagation complète
4. **Wildcard = flexibilité** : *.monlab.com simplifie tout
5. **Documentez vos changements** : Gardez trace des modifications
6. **Testez depuis l'extérieur** : Utilisez plusieurs serveurs DNS

## Résumé

La configuration DNS transforme votre nom de domaine en passerelle vers votre lab MicroK8s. Avec les bons enregistrements A, votre domaine dirigera le trafic vers votre IP publique. Les wildcards vous donnent la flexibilité d'ajouter des services sans toucher aux DNS. Dans la prochaine section, nous configurerons la redirection de ports pour que ce trafic atteigne réellement votre cluster MicroK8s.

⏭️
