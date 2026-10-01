# Architecture technique HING3 — Sec-Digi

## 1. Contexte et objectifs

Sec-Digi est une PME de cybersécurité de **60 personnes**, réparties sur trois sites géographiques :

| Site | Rôle | Effectifs | Détail |
|------|------|-----------|--------|
| **Nantes** | Siège | 30 users | 10 pentesteurs, 10 auditeurs, 1 DG, 5 commerciaux, 4 alternants |
| **Rennes** | Site distant | 15 users | 10 auditeurs, 5 pentesteurs |
| **Paris** | Site distant | 15 users | 5 commerciaux, 5 auditeurs, 5 pentesteurs |

L’infrastructure doit répondre à trois impératifs métier :

1. **Monter des architectures de test** pour simuler les infrastructures clientes.
2. **Héberger les données sensibles** : rapports d’audit, rapports de pentest, contrats.
3. **Héberger les données de R&D**.

En outre, l’objectif stratégique est de concevoir une infrastructure :

- **Fluide** : chaque utilisateur doit pouvoir travailler depuis n’importe quel site.
- **Redondante** : tolérance aux pannes matérielles et réseau.
- **Performante** : limitation des goulots d’étranglement, notamment sur les liens inter-sites.
- **Sécurisée** : en tant que société de cybersécurité, l’exposition à une attaque externe ou à une fuite interne serait inacceptable.

L’IT interne est assurée par les auditeurs et pentesteurs du siège de Nantes à temps partiel. L’architecture doit donc rester simple à administrer, centralisée autant que possible, et robuste.

---

## 2. Vue d’ensemble de l’architecture

L’architecture repose sur un modèle **hub-and-spoke** :

- **Nantes** constitue le hub central : il héberge les serveurs physiques, le stockage principal, les firewalls périmétriques haute disponibilité (HA), et les services critiques (annuaire, supervision, sauvegarde).
- **Rennes** et **Paris** sont des sites distants connectés à Nantes par des **tunnels VPN IPsec**.
- Chaque site dispose de son propre accès Internet **Free 400 Mbps** et de son réseau local en **1 Gbps**.

### 2.1 Schéma logique

```
                    WAN/Internet
                         |
                   Free 400 Mbps
                         |
              ┌─────────────────────┐
              │   HA Edge Firewalls │  <- OPNsense en CARP
              └──────────┬──────────┘
                         |
              ┌─────────────────────┐
              │   Core Switches L2    │  <- HPE ProCurve 1810
              │   (double attachement)│
              └──────────┬──────────┘
        ┌────────────────┼────────────────┐
        |                |                |
 Utilisateurs VLAN    R&D VLAN      Testing/Sandbox VLAN
        |                |                |
    User PCs          Données R&D       VMs simulant
                                       les clients
        |                |                |
   ┌─────────────────────────────────────────────┐
   │ Cluster Proxmox VE : 2x HP DL380 G6         │
   │ - ZFS local et réplication toutes les 15 min│
   │ - AD, SMB, Wazuh, WSUS                      │
   │ - QDevice physique indépendant              │
   └─────────────────────────────────────────────┘
        |                                    |
 Management / Serveurs / Sauvegarde VLAN    IPsec VPN
        |                                    |
    Bastion + PBS                       Rennes / Paris
                                     (NAS + LAN + Users)
```

---

## 3. Description détaillée par site

### 3.1 Nantes — Siège

#### 3.1.1 Équipements réseau

- **Accès Internet** : Free 400 Mbps.
- **Firewalls Edge HA** : deux appliances **OPNsense** déployées en redondance active/passive via le protocole **CARP** (Common Address Redundancy Protocol). Les deux pare-feu partagent une **IP virtuelle** ; si le maître tombe, le secondaire prend la main en quelques secondes. La compatibilité de CARP avec l’accès Free (adressage public, mode bridge et absence de CGNAT) devra être validée avant la mise en œuvre.
- **Core Switches** : deux switches **HPE ProCurve 1810** en 1 Gbps, en double attachement pour la résilience. Attention : ce sont des commutateurs **niveau 2** ; ils ne font pas de routage et ne supportent pas VRRP. Le routage inter-VLAN est assuré par les firewalls OPNsense.

#### 3.1.2 Segmentation VLAN

| VLAN | Rôle | Contenu |
|------|------|---------|
| **Utilisateurs** | Postes de travail quotidiens | 30 users (pentesteurs, auditeurs, DG, commerciaux, alternants) |
| **Serveurs** | Services internes | Contrôleurs de domaine, DNS, serveur de fichiers et applications internes |
| **R&D** | Projets internes de recherche | Serveurs et bases de données dédiés aux projets R&D |
| **Testing / Sandbox** | Simulation client | VMs isolées reproduisant des infrastructures clientes |
| **Management** | Administration et sécurité | Hyperviseurs, switches, pare-feu, NAS, SIEM et outils d’administration |
| **Sauvegarde** | Protection des données | Proxmox Backup Server et flux de réplication |
| **Imprimantes / IoT** | Équipements de confiance limitée | Imprimantes et équipements connectés |
| **Invités** | Accès visiteurs | Accès Internet uniquement |

> **Point de vigilance** : le VLAN Testing/Sandbox doit être **strictement isolé** du reste de l’infrastructure. La politique de pare-feu depuis la Sandbox est un **Default Deny** explicite vers tous les VLAN internes. Ce réseau ne doit accéder qu’à Internet de manière filtrée et journalisée, sans résolution DNS interne.

#### 3.1.3 Serveurs et virtualisation

- **2 serveurs physiques HP DL380 G6** : 2 processeurs, 128 Go de RAM chacun.
- **Proxmox VE** est retenu comme hyperviseur sur les deux serveurs.
- Chaque serveur possède actuellement quatre disques SAS de 900 Go en RAID 5 sur contrôleur HPE à batterie, pour environ 2,2 To utiles annoncés. **ZFS local n’est retenu qu’après validation d’un véritable mode HBA/JBOD** ; sinon le RAID matériel est conservé avec un système de fichiers pris en charge, ou le contrôleur et les disques sont remplacés. ZFS ne doit pas être superposé sans validation au volume RAID 5 existant.
- Les VMs critiques sont répliquées entre les deux nœuds toutes les **15 minutes**.
- Un **QDevice**, installé sur un petit équipement physique indépendant à Nantes, fournit le troisième vote nécessaire au quorum. Il ne doit pas être hébergé par le cluster.
- En cas de panne d’un nœud, les VMs critiques sont redémarrées sur le nœud restant. Cette solution vise un **RPO maximal de 15 minutes** et un **RTO de 15 à 30 minutes** ; elle ne garantit pas une continuité sans interruption.
- Les VMs principales hébergées sont :
  - deux contrôleurs de domaine **Windows Server Active Directory**, placés sur des nœuds différents ;
  - un serveur de fichiers central SMB pour les rapports, contrats et données R&D ;
  - un serveur **Wazuh** pour le SIEM et la supervision ;
  - un serveur de correctifs **WSUS** ;
  - les services d’administration nécessaires.

Les contrôleurs de domaine sont placés dans le VLAN Serveurs. Aucun contrôleur de domaine ou RODC n’est hébergé sur les NAS distants, afin de ne pas mélanger les fonctions de production et de sauvegarde.

---

### 3.2 Rennes — Site distant

- **Effectifs corrigés** : **15 users** au total — **10 auditeurs** et **5 pentesteurs**.
- **Accès Internet** : Free 400 Mbps.
- **Pare-feu local** : OPNsense, routeur VPN IPsec vers Nantes.
- **Switch local** : HPE ProCurve 1810 1 Gbps.
- **Postes de travail** : 15 PC connectés au LAN local.
- **NAS Synology DS918+** : 2 disques de 4 To en RAID 1.

### 3.3 Paris — Site distant

- **Effectifs** : **15 users** — 5 commerciaux, 5 auditeurs, 5 pentesteurs.
- **Accès Internet** : Free 400 Mbps.
- **Pare-feu local** : OPNsense, routeur VPN IPsec vers Nantes.
- **Switch local** : HPE ProCurve 1810 1 Gbps.
- **Postes de travail** : 15 PC connectés au LAN local.
- **NAS Synology DS918+** : 2 disques de 4 To en RAID 1.

### 3.4 Matériel complémentaire nécessaire

Le matériel fourni ne suffit pas à mettre en œuvre toutes les fonctions retenues. Les éléments suivants doivent être acquis ou représentés dans la maquette :

- quatre appliances OPNsense : deux à Nantes pour la HA et une sur chaque site distant ;
- un petit équipement physique indépendant pour le QDevice Proxmox ;
- une machine et un stockage dédiés à Proxmox Backup Server ;
- des disques compatibles avec l’usage de ZFS si les DL380 n’en disposent pas ;
- un support de sauvegarde chiffré déconnectable ou un stockage objet avec Object Lock.

Un second accès Internet à Nantes est requis dans la cible de production pour supprimer le point unique de défaillance WAN, mais il n’est pas compris dans le matériel initial. Des onduleurs supervisés doivent protéger les pare-feu, switches, serveurs et stockages et permettre leur arrêt propre. Une capacité de calcul et de restauration hors Nantes est également requise pour satisfaire l’objectif de continuité lors de la perte du siège.

---

## 4. Connectivité inter-sites

### 4.1 Tunnels VPN IPsec

Rennes et Paris sont reliés au siège de Nantes par des **tunnels VPN IPsec site-à-site**. Ces tunnels permettent :

- L’accès au stockage centralisé à Nantes.
- La résolution DNS interne via les contrôleurs de domaine de Nantes.
- La réplication des sauvegardes vers les NAS distants.
- L’administration centralisée et la supervision.

### 4.2 Protocole d’accès aux données centralisées

- **SMB/CIFS (privilégié)** : protocole standard pour le partage de fichiers (rapports, contrats, R&D). Les postes de Rennes et Paris accèdent aux partages de Nantes au travers du VPN IPsec.
- **iSCSI exclus** : opère en mode bloc ; inadapté à un accès simultané par de nombreux utilisateurs sans Clustered File System complexe.
- **NFS non retenu** : bien que performant, il est moins fluide que SMB pour gérer les permissions complexes sur un parc hétérogène Windows/Mac.

### 4.3 Rôle des NAS locaux (Rennes et Paris)

Les NAS DS918+ ne sont pas utilisés pour une synchronisation bidirectionnelle des partages de Nantes. Les utilisateurs accèdent au serveur SMB central au travers des tunnels IPsec. Ce choix évite les conflits de versions, les incohérences de droits et la propagation automatique d’une suppression ou d’un chiffrement malveillant.

Les NAS distants sont réservés aux fonctions suivantes :

- recevoir les copies chiffrées des sauvegardes de Nantes ;
- conserver, si nécessaire, des données strictement locales qui ne sont pas des répliques modifiables du partage central ;
- fournir un espace temporaire contrôlé pendant un incident.

Une coupure Internet rend temporairement les partages centraux indisponibles. Les utilisateurs conservent l’accès à leur poste grâce aux identifiants Windows mis en cache, mais une continuité complète des fichiers exigerait des serveurs distribués supplémentaires. Cette limite est acceptée afin de garder une architecture simple et fiable.

---

## 5. Stratégie de sauvegarde et PRA

Une entreprise de cybersécurité doit appliquer une politique de sauvegarde irréprochable pour se prémunir contre les ransomwares, les erreurs humaines et les sinistres physiques.

### 5.1 Règle du 3-2-1-1-0

| Niveau | Description | Mise en œuvre |
|--------|-------------|---------------|
| **3 copies** | Données actives + sauvegarde locale + copie externalisée | ZFS, Proxmox Backup Server et NAS distant |
| **2 systèmes de stockage** | Production et sauvegarde séparées | DL380 + stockage PBS dédié |
| **1 copie off-site** | Réplication chiffrée hors du siège | NAS de Rennes, avec copie secondaire à Paris |
| **1 copie immuable** | Copie non modifiable par la production | Stockage objet avec Object Lock ; disque chiffré déconnecté pour la maquette |
| **0 erreur** | Contrôle et restauration vérifiés | Vérifications automatiques et tests réguliers |

### 5.2 Détail de la politique

1. **Snapshots à Nantes** : les volumes ZFS prennent des instantanés toutes les heures pour permettre un retour rapide après une erreur. Ces snapshots ne sont pas considérés comme des sauvegardes.
2. **Sauvegarde primaire** : **Proxmox Backup Server**, installé sur une machine et un stockage dédiés, sauvegarde quotidiennement les VMs et les partages de fichiers. La rétention cible est de 7 sauvegardes quotidiennes, 4 hebdomadaires et 12 mensuelles.
3. **Externalisation off-site** : les sauvegardes sont répliquées de manière chiffrée vers le NAS de **Rennes** via IPsec. Le NAS de **Paris** reçoit une copie secondaire des données les plus critiques. Les volumes de sauvegarde ne sont pas accessibles aux utilisateurs et ne sont pas montés en permanence en écriture par la production.
4. **Copie immuable** : une copie est envoyée vers un stockage objet compatible **Object Lock**. Pour la maquette, cette fonction peut être représentée par un disque externe chiffré, déconnecté après la sauvegarde et conservé hors site.
5. **Contrôle** : une restauration de fichier est testée chaque mois et une restauration complète de VM chaque trimestre. Les échecs de sauvegarde, suppressions massives et anomalies déclenchent une alerte.

---

## 6. Services critiques centralisés

### 6.1 Gestion des identités et de l’administration

- Deux contrôleurs de domaine **Windows Server Active Directory**, `DC01` et `DC02`, sont déployés dans le VLAN Serveurs à Nantes, chacun sur un nœud Proxmox différent.
- Les droits sur les fichiers et applications sont attribués par groupes Active Directory selon le principe du moindre privilège.
- Le **MFA** est obligatoire pour les comptes administrateurs, les accès VPN d’administration et les consoles Proxmox, OPNsense, Wazuh et Synology.
- Chaque administrateur possède un compte bureautique et un compte d’administration distincts.
- L’administration des équipements s’effectue depuis un **bastion** placé dans le VLAN Management ; aucun accès direct n’est autorisé depuis les VLAN Utilisateurs.
- Aucun RODC n’est installé sur les NAS distants. En cas de coupure VPN, les ouvertures de session déjà connues utilisent les identifiants Windows mis en cache.

### 6.2 DNS / DHCP

- **DHCP** : géré localement par OPNsense sur chaque site afin de garantir l’attribution des baux pendant une coupure VPN.
- **DNS interne** : assuré par `DC01` et `DC02` pour les postes joints au domaine. Les sites distants les joignent au travers des tunnels IPsec.
- **Résolution Internet de secours** : OPNsense fournit un résolveur local pour les équipements qui n’ont pas besoin de résoudre le domaine Active Directory.

### 6.3 Supervision, SIEM et logs centralisés

La visibilité est primordiale pour une société de cybersécurité. **Wazuh** est retenu comme SIEM et installé sur une VM isolée dans le VLAN Management à Nantes.

Tous les postes clients, serveurs, hyperviseurs, NAS, pare-feu et équipements réseau compatibles envoient leurs journaux vers Wazuh pour :

- détecter les comportements suspects ;
- auditer les accès ;
- répondre aux incidents ;
- alerter sur les échecs de sauvegarde et les modifications administratives.

L’accès aux journaux est limité aux administrateurs habilités. La cible de rétention est de six mois en ligne et un an en archive.

### 6.4 Sécurité des endpoints et des données

- Un agent **EDR** est déployé sur chaque poste et serveur compatible.
- Les postes sont chiffrés avec **BitLocker** ou un équivalent.
- Les utilisateurs ordinaires ne disposent pas de droits d’administration locale ; les comptes locaux sont gérés avec **Windows LAPS** ou un équivalent.
- Un serveur **WSUS** centralise le déploiement des correctifs Windows. Les systèmes Linux utilisent des dépôts contrôlés et une politique de mise à jour centralisée.
- Les macros non approuvées et les logiciels non autorisés sont bloqués.
- Les comptes et volumes de sauvegarde sont séparés des accès bureautiques. Aucun utilisateur ne possède d’accès SMB aux sauvegardes.
- Les droits d’accès sont revus chaque trimestre et les règles inter-VLAN appliquent un refus par défaut, puis n’autorisent que les flux nécessaires.

---

## 7. Prise en compte des réponses client

- **Disponibilité** : RTO maximal métier de 4 heures ; cible technique de 15 à 30 minutes pour une panne simple. La perte de Nantes exige une capacité de reprise hors site et un second lien ; tant qu’ils ne sont pas financés, l’écart doit être accepté par le DG/DSI.
- **Perte de données** : le besoin exprimé est RPO 0. La réplication asynchrone à 15 minutes ne le garantit pas ; les applications et fichiers critiques doivent utiliser journalisation continue, réplication adaptée ou sauvegarde continue, et tout RPO résiduel doit être accepté.
- **Capacité** : la production doit cohabiter avec 10 VM de test simultanées. La volumétrie connue est d’environ 500 Go de VM, 100 Go de rapports, 500 Go de R&D et 1 To de sauvegardes/snapshots. Un test de charge et un plan de capacité avec 30 % de marge conditionnent la production.
- **Croissance** : l’architecture réserve l’adressage et les modèles de configuration pour +10 % d’effectif sous deux ans et un éventuel site à Bordeaux, Lyon ou Lille.
- **Identité** : l’AD de 2015 est audité puis migré vers DC01/DC02 ; les domaines, groupes, GPO, DNS et applications ne sont pas recréés sans plan de migration et retour arrière.
- **Usages** : Microsoft 365, Teams, Keeper et les applications Debian sont intégrés à l’inventaire. Les groupes Teams et SMB suivent les spécialités ; les alternants héritent des droits de leur fonction.
- **Accès distant** : un VPN nomade ou ZTNA avec MFA, poste géré et journalisation est proposé aux collaborateurs ; l’administration demeure exclusivement via bastion.
- **Exploitation** : l’automatisation et les alertes sont conçues pour une charge planifiée de 6 h/semaine hors incident.
- **Sécurité** : Keeper reste le coffre et canal approuvé pour les secrets et échanges sensibles. L’architecture prépare les preuves utiles à l’objectif ISO 27001.
- **Locaux et énergie** : les locaux climatisés de 9 m² sont audités ; des onduleurs supervisés et procédures d’arrêt propre sont inclus dans la cible.
- **Gouvernance** : les variantes de coût, le planning et les dérogations sont soumis au DG/DSI. La recette fonctionnelle constitue le critère de réussite de la maquette.

## 8. Matrice de conformité à la demande initiale et aux réponses client

| Exigence HING3 | Statut | Commentaire |
|----------------|--------|-------------|
| 3 sites (Nantes, Rennes, Paris) | ✅ Conforme | Architecture hub-and-spoke. |
| 30 users à Nantes | ✅ Conforme | 10 pentesteurs + 10 auditeurs + 1 DG + 5 commerciaux + 4 alternants. |
| 15 users à Rennes | ✅ Corrigé | 10 auditeurs + 5 pentesteurs |
| 15 users à Paris | ✅ Conforme | 5 commerciaux + 5 auditeurs + 5 pentesteurs. |
| Simuler infra clientes | ✅ Conforme | VLAN Testing/Sandbox dédié avec VMs isolées. |
| Héberger rapports, pentests, contrats | ✅ Conforme | Stockage central sur les serveurs HP DL380 à Nantes. |
| Héberger données R&D | ✅ Conforme | VLAN R&D + stockage central dédié. |
| 2 serveurs HP DL380 G6 / 128 Go RAM | ✅ Conforme | Cluster Proxmox VE à deux nœuds, réplication ZFS et QDevice indépendant. |
| Réseau HPE 1 Gbps | ✅ Conforme | HPE ProCurve 1810 sur tous les sites. |
| Internet Free 400 Mbps | ✅ Conforme | Présent sur les 3 sites ; la compatibilité CARP reste à valider. |
| NAS Synology DS918+ RAID 1 à Rennes/Paris | ✅ Conforme | Cibles de sauvegarde hors site, sans synchronisation bidirectionnelle des partages. |
| Infrastructure fluide, redondante, performante | ⚠️ Partiel | HA OPNsense et Proxmox ; l’accès Internet et certains équipements réseau restent des points uniques de défaillance. |
| Sécurité renforcée (spécialiste cyber) | ✅ Conforme | Segmentation, MFA, bastion, EDR, Wazuh, patching, chiffrement et sauvegardes 3-2-1-1-0. |
| Interruption maximale de 4 h | ✅ Cible définie | RTO technique inférieur pour panne simple ; PRA hors Nantes à financer et tester. |
| Aucune perte de données | ⚠️ Écart à arbitrer | RPO 0 métier ; réplication asynchrone actuelle à 15 min. Protection continue à sélectionner par service. |
| Production + 10 VM de test | ⚠️ À démontrer | Réservation, quotas et recette de charge obligatoires avant production. |
| Exploitation 6 h/semaine | ⚠️ À mesurer | Automatisation prévue ; charge réelle vérifiée pendant la recette et le pilote. |
| Microsoft 365, Teams et Keeper | ✅ Pris en compte | Gouvernance des groupes, MFA, coffre de secrets et échanges sensibles. |
| AD existant de 2015 | ⚠️ Migration requise | Audit puis migration documentée vers deux DC. |
| Croissance +10 % / nouveaux sites | ✅ Prévu | Réserves de capacité, adressage et modèle de site reproductible. |
| Objectif ISO 27001 | ✅ Préparé | Mesures et preuves techniques ; certification hors périmètre du seul projet. |

---

*Document généré pour le projet HING3 — Sec-Digi.*
