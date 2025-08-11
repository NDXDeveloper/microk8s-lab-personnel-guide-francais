🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 3.1 Concepts de base du réseau Kubernetes

## Introduction

Avant de plonger dans la configuration pratique du réseau pour votre lab MicroK8s, il est essentiel de comprendre comment Kubernetes gère le réseau. Cette compréhension vous évitera bien des frustrations et vous permettra de diagnostiquer efficacement les problèmes. Ne vous inquiétez pas si certains concepts semblent abstraits au début - ils deviendront plus clairs avec la pratique.

## Le modèle réseau de Kubernetes

### Principes fondamentaux

Kubernetes impose quatre règles simples mais puissantes pour son modèle réseau :

1. **Chaque Pod obtient sa propre adresse IP** : Pas de NAT entre les Pods
2. **Les Pods peuvent communiquer entre eux** : Sans NAT, peu importe le nœud où ils se trouvent
3. **Les nœuds peuvent communiquer avec tous les Pods** : Et vice-versa, sans NAT
4. **L'IP qu'un Pod voit de lui-même** : Est la même que celle vue par les autres

Ces règles créent un réseau "plat" où chaque composant peut parler directement à n'importe quel autre composant.

### Analogie simple

Imaginez Kubernetes comme un grand immeuble de bureaux :
- **Les Pods** sont des bureaux individuels avec leur propre numéro (adresse IP)
- **Les Services** sont comme la réception qui dirige les visiteurs vers les bons bureaux
- **L'Ingress** est l'entrée principale de l'immeuble avec le portier qui vérifie et oriente les visiteurs
- **Le réseau cluster** est le système de couloirs permettant de circuler dans l'immeuble

## Les composants réseau essentiels

### 1. Pod Network (Réseau des Pods)

Le réseau des Pods est l'espace d'adressage IP interne où vivent tous vos conteneurs.

```
Caractéristiques du Pod Network :
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
• Plage IP : Généralement 10.1.0.0/16 dans MicroK8s
• Chaque Pod : Reçoit une IP unique de cette plage
• Durée de vie : L'IP change si le Pod est recréé
• Visibilité : IPs internes, non routables sur Internet
```

**Exemple concret** : Si vous déployez une application web, elle pourrait recevoir l'IP `10.1.23.45`. Cette IP est temporaire et changera si le Pod redémarre.

### 2. Service Network (Réseau des Services)

Les Services résolvent le problème des IPs changeantes des Pods en fournissant une IP stable.

```
Types de Services Kubernetes :
━━━━━━━━━━━━━━━━━━━━━━━━━━━━
1. ClusterIP (par défaut)
   └── IP interne stable, accessible uniquement dans le cluster

2. NodePort
   └── Ouvre un port sur chaque nœud (30000-32767)

3. LoadBalancer
   └── Demande un load balancer externe (cloud provider)

4. ExternalName
   └── Mappe un service à un nom DNS externe
```

**Pourquoi les Services sont-ils importants ?**
- Ils fournissent une **découverte de service** automatique
- Ils effectuent du **load balancing** entre plusieurs Pods
- Ils offrent une **abstraction stable** malgré la nature éphémère des Pods

### 3. Kube-proxy

Kube-proxy est le composant qui fait fonctionner la magie des Services. Il s'exécute sur chaque nœud et maintient les règles réseau.

```
Rôle de kube-proxy :
━━━━━━━━━━━━━━━━━━━━
• Surveille les Services et Endpoints
• Configure iptables/ipvs pour router le trafic
• Implémente le load balancing
• Gère les health checks
```

### 4. CoreDNS

CoreDNS fournit la résolution de noms à l'intérieur du cluster. C'est lui qui permet d'accéder aux services par leur nom plutôt que par IP.

```
Format des noms DNS dans Kubernetes :
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
<service-name>.<namespace>.svc.cluster.local

Exemple :
mon-app.default.svc.cluster.local
  ↓        ↓      ↓        ↓
service  namespace type  domaine
```

## Le flux du trafic réseau

### Trafic interne (Pod à Pod)

Voyons comment deux Pods communiquent entre eux :

```
Pod A (10.1.0.5) veut parler à Pod B (10.1.0.8)
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

1. Pod A envoie un paquet à 10.1.0.8
       ↓
2. Le plugin CNI route le paquet
       ↓
3. Pod B reçoit directement le paquet
       ↓
4. Pod B répond à 10.1.0.5

✅ Communication directe, pas de NAT !
```

### Trafic via Service

Quand on utilise un Service, le flux devient :

```
Pod A veut accéder au Service "mon-api" (10.152.183.100)
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

1. Pod A résout "mon-api" → 10.152.183.100 (via CoreDNS)
       ↓
2. Pod A envoie le paquet à 10.152.183.100
       ↓
3. iptables (configuré par kube-proxy) intercepte
       ↓
4. Load balancing vers un Pod backend (ex: 10.1.0.15)
       ↓
5. Le Pod backend reçoit et traite la requête

🔄 Le Service agit comme un proxy intelligent
```

### Trafic externe entrant

Pour le trafic venant d'Internet :

```
Utilisateur Internet → Votre Application
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

1. Utilisateur tape : https://monapp.example.com
       ↓
2. DNS public résout vers votre IP publique
       ↓
3. Routeur redirige ports 80/443 vers MicroK8s
       ↓
4. Ingress Controller reçoit la requête
       ↓
5. Ingress examine l'host header "monapp.example.com"
       ↓
6. Route vers le bon Service selon les règles
       ↓
7. Service route vers un Pod backend
       ↓
8. Pod traite et renvoie la réponse

🌐 Tout ce chemin sera configuré dans cette section !
```

## L'Ingress Controller

L'Ingress Controller est votre point d'entrée principal pour le trafic HTTP/HTTPS externe.

### Qu'est-ce qu'un Ingress ?

Un Ingress est un objet Kubernetes qui définit les règles de routage HTTP/HTTPS. L'Ingress Controller (comme NGINX) lit ces règles et les applique.

```yaml
Exemple conceptuel d'Ingress :
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
Si l'utilisateur va sur :
  • blog.monsite.com → Route vers le Service "blog"
  • api.monsite.com  → Route vers le Service "api"
  • monsite.com/shop → Route vers le Service "boutique"
```

### Avantages de l'Ingress

1. **Un seul point d'entrée** : Pas besoin d'ouvrir plusieurs ports
2. **Routage par nom/chemin** : Multiple apps sur les mêmes ports 80/443
3. **Terminaison SSL/TLS** : Gestion centralisée des certificats
4. **Fonctionnalités avancées** : Rate limiting, authentification, etc.

## Spécificités réseau de MicroK8s

### Architecture simplifiée

MicroK8s utilise des choix d'implémentation qui simplifient le réseau pour un usage mono-nœud :

```
Composants réseau MicroK8s :
━━━━━━━━━━━━━━━━━━━━━━━━━━━
• CNI Plugin : Calico (par défaut)
• DNS : CoreDNS (addon dns)
• Ingress : NGINX (addon ingress)
• Load Balancer : MetalLB (addon metallb)
```

### Plages d'adresses par défaut

```
Réseaux dans MicroK8s :
━━━━━━━━━━━━━━━━━━━━━━━
• Pod Network     : 10.1.0.0/16 (65,536 adresses)
• Service Network : 10.152.183.0/24 (256 adresses)
• DNS Service     : 10.152.183.10
• Host Network    : Votre réseau local (ex: 192.168.1.0/24)
```

### Le mode host-network

Dans certains cas, vous pouvez configurer un Pod pour utiliser le réseau de l'hôte directement :

```yaml
Cas d'usage du host network :
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
• Monitoring (Prometheus Node Exporter)
• Ingress Controllers
• Applications nécessitant des performances réseau maximales

⚠️ Attention : Perd l'isolation réseau !
```

## Concepts de sécurité réseau

### Network Policies

Les Network Policies sont comme des règles de firewall pour vos Pods :

```
Exemples de Network Policies :
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
• Bloquer tout trafic entrant sauf depuis certains Pods
• Autoriser uniquement le trafic sur certains ports
• Isoler complètement un namespace
• Permettre le trafic uniquement depuis l'Ingress
```

### Isolation par défaut

Par défaut dans Kubernetes :
- **Tous les Pods peuvent se parler** : Pas d'isolation
- **Sécurité = votre responsabilité** : Utilisez les Network Policies
- **Services exposés = accessibles** : Sécurisez vos endpoints

## Diagnostiquer les problèmes réseau

### Outils de diagnostic essentiels

```bash
Commandes utiles pour le réseau :
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

# Voir les Services
kubectl get svc -A

# Voir les Endpoints (Pods derrière un Service)
kubectl get endpoints

# Tester la résolution DNS depuis un Pod
kubectl run -it --rm debug --image=busybox --restart=Never -- nslookup mon-service

# Voir les règles Ingress
kubectl get ingress -A

# Examiner les logs de l'Ingress Controller
kubectl logs -n ingress nginx-ingress-controller-xxx

# Vérifier la connectivité Pod
kubectl exec mon-pod -- ping autre-pod-ip
```

### Problèmes courants et causes

```
Symptôme → Cause probable
━━━━━━━━━━━━━━━━━━━━━━━━━
Service inaccessible → Pas de Pods healthy / Selector incorrect
DNS ne résout pas → CoreDNS down / Configuration DNS incorrecte
Ingress ne route pas → Règles mal configurées / Certificats invalides
Pods ne communiquent pas → Network Policy bloquante / CNI problème
Port inaccessible externe → Firewall / NAT non configuré
```

## Comprendre les adresses IP multiples

Dans votre lab, vous jonglerez avec plusieurs types d'IPs :

```
Types d'IP dans votre lab :
━━━━━━━━━━━━━━━━━━━━━━━━━━
1. IP Publique (ex: 82.123.45.67)
   └── Vue depuis Internet

2. IP LAN de l'hôte (ex: 192.168.1.100)
   └── Dans votre réseau local

3. IP du Service (ex: 10.152.183.50)
   └── Stable, interne au cluster

4. IP du Pod (ex: 10.1.34.12)
   └── Éphémère, change au redémarrage

5. IP localhost (127.0.0.1)
   └── Boucle locale, chaque Pod a la sienne
```

## Le voyage d'une requête HTTP

Pour bien comprendre, suivons une requête depuis un navigateur jusqu'à votre application :

```
Voyage complet d'une requête :
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

1. Navigateur : "Je veux https://app.monlab.com"
       ↓
2. DNS Public : "C'est 82.123.45.67"
       ↓
3. Internet : Route vers votre IP publique
       ↓
4. Box/Routeur : "Port 443 ? Je redirige vers 192.168.1.100"
       ↓
5. Machine MicroK8s : "Port 443 ? C'est l'Ingress Controller"
       ↓
6. Ingress : "app.monlab.com ? Je connais, c'est le service 'app-service'"
       ↓
7. Service : "J'ai 3 Pods disponibles, j'envoie vers le Pod-2"
       ↓
8. Pod-2 : "Je traite la requête et renvoie la page HTML"
       ↓
9. Retour : La réponse suit le chemin inverse

⏱️ Tout cela en quelques millisecondes !
```

## Points clés à retenir

### Les essentiels

1. **Les Pods sont éphémères** : Leurs IPs changent, utilisez toujours des Services
2. **Les Services sont stables** : C'est votre point d'ancrage pour accéder aux applications
3. **L'Ingress est votre porte d'entrée** : Il route le trafic HTTP/HTTPS externe
4. **Le DNS est crucial** : Il permet la découverte de services par nom
5. **Tout est configurable** : Mais commencez simple et complexifiez progressivement

### Bonnes pratiques

- **Utilisez des noms, pas des IPs** : Plus maintenable et flexible
- **Documentez vos ports** : Gardez une liste des ports utilisés
- **Testez progressivement** : Local → LAN → Internet
- **Surveillez les logs** : La plupart des problèmes y sont visibles
- **Comprenez avant de configurer** : Cette base théorique vous fera gagner du temps

## Préparation pour la suite

Maintenant que vous comprenez les concepts de base du réseau Kubernetes, vous êtes prêt à :

1. **Configurer le DNS local** (3.2) : Pour accéder à vos services par nom en local
2. **Acquérir un domaine** (3.3) : Pour un accès professionnel depuis Internet
3. **Router le trafic** (3.4-3.5) : Pour rendre vos services accessibles
4. **Sécuriser l'ensemble** (3.6) : Pour protéger votre installation

Ces concepts peuvent sembler abstraits maintenant, mais ils deviendront concrets quand vous les mettrez en pratique. N'hésitez pas à revenir à cette section quand vous rencontrerez des problèmes - la solution y est souvent !

## Résumé

Le réseau Kubernetes suit un modèle simple mais puissant : chaque Pod a une IP, les Services fournissent la stabilité, l'Ingress gère l'accès externe, et le DNS lie tout ensemble. MicroK8s simplifie cette architecture pour un usage mono-nœud tout en conservant la compatibilité avec Kubernetes standard.

Dans la prochaine section, nous commencerons la configuration pratique avec le DNS local, première étape vers un lab pleinement fonctionnel.

⏭️
