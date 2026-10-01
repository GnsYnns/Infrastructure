# Dossier d’architecture technique — Projet HING3

## 1. Objet

Ce dossier décrit l’architecture technique retenue pour l’infrastructure multisite de Sec-Digi. Il traduit l’expression de besoin en composants, zones réseau, services, flux, mécanismes de sécurité et scénarios de disponibilité.

## 2. Principes directeurs

L’architecture applique les principes suivants :

- centralisation des services principaux à Nantes ;
- interconnexion de Rennes et Paris selon un modèle hub-and-spoke ;
- segmentation réseau par niveau de confiance et par usage ;
- moindre privilège et refus par défaut ;
- redondance des composants critiques lorsque le matériel le permet ;
- sauvegardes séparées de la production, externalisées et testées ;
- isolement strict des environnements simulant les clients ;
- administration depuis une zone dédiée ;
- exploitation compatible avec une équipe IT à temps partiel.

## 3. Architecture générale

### 3.1 Topologie

Nantes constitue le site central. Il héberge :

- les deux nœuds Proxmox VE ;
- les contrôleurs de domaine et le DNS interne ;
- le stockage de fichiers central ;
- les services R&D ;
- la supervision et le SIEM ;
- le serveur de sauvegarde ;
- les outils d’administration ;
- une paire de pare-feu OPNsense si les conditions opérateur permettent CARP.

Rennes et Paris disposent chacun :

- d’un accès Internet local ;
- d’un pare-feu OPNsense ;
- d’un switch HPE ProCurve 1810 ;
- d’un VPN IPsec vers Nantes ;
- d’un NAS Synology DS918+ réservé principalement aux copies hors site ;
- de postes clients sur le réseau local.

### 3.2 Schéma logique

```text
                              Internet
                                  |
                       Accès Free 400 Mbit/s
                                  |
                    +-------------------------+
                    | OPNsense Nantes A et B  |
                    | HA sous conditions WAN  |
                    +------------+------------+
                                 |
                   +-------------+-------------+
                   | Switches HPE L2 Nantes    |
                   +-------------+-------------+
                                 |
       +-------------+-----------+-----------+-------------+
       |             |           |           |             |
   Utilisateurs   Serveurs      R&D       Sandbox      Management
                                 |
                    +------------+------------+
                    | 2 nœuds Proxmox VE      |
                    | ZFS local + réplication |
                    +------------+------------+
                                 |
                         VLAN Sauvegarde
                         PBS dédié / média

              VPN IPsec                         VPN IPsec
                  |                                  |
       +----------+---------+              +---------+----------+
       | OPNsense Rennes    |              | OPNsense Paris     |
       | LAN + NAS DS918+   |              | LAN + NAS DS918+   |
       +--------------------+              +---------------------+
```

## 4. Architecture physique

### 4.1 Nantes

| Composant | Quantité | Rôle |
|---|---:|---|
| HP DL380 G6, 2 CPU, 128 Go RAM | 2 | Nœuds Proxmox VE |
| HPE ProCurve 1810 | 2 recommandés | Commutation L2 et trunks VLAN |
| Appliance OPNsense | 2 | Pare-feu, routage, VPN et haute disponibilité conditionnelle |
| QDevice physique indépendant | 1 | Troisième vote du cluster Proxmox |
| Serveur PBS avec stockage dédié | 1 | Sauvegarde des VM et données prises en charge |
| Support déconnectable ou stockage objet | 1 | Copie immuable/hors ligne |

### 4.2 Rennes et Paris

| Composant | Quantité par site | Rôle |
|---|---:|---|
| HPE ProCurve 1810 | 1 | Commutation L2 locale |
| Appliance OPNsense | 1 | Pare-feu, accès Internet et VPN |
| Synology DS918+, 2 × 4 To RAID 1 | 1 | Copie hors site et espace de secours contrôlé |

### 4.3 Câblage et redondance

Les liens des nœuds Proxmox utilisent, selon le nombre de cartes disponibles :

- un bond `active-backup` réparti entre les deux switches pour les réseaux de production ;
- des interfaces ou VLAN dédiés au management, à la réplication et aux sauvegardes ;
- RSTP sur les switches afin d’éviter les boucles.

Le mode LACP entre deux switches n’est utilisé que si les équipements prennent explicitement en charge une agrégation multi-châssis, ce qui ne doit pas être présumé avec les ProCurve 1810. À défaut, le mode `active-backup` est retenu.

## 5. Plan d’adressage proposé

Le plan suivant sert de base à la maquette et doit être adapté à l’existant avant production.

### 5.1 Nantes — `10.10.0.0/16`

| VLAN | Nom | Sous-réseau | Passerelle | Usage |
|---:|---|---|---|---|
| 10 | USERS-NANTES | `10.10.10.0/24` | `10.10.10.1` | Postes utilisateurs |
| 20 | SERVERS | `10.10.20.0/24` | `10.10.20.1` | AD, DNS, fichiers, applications |
| 30 | RND | `10.10.30.0/24` | `10.10.30.1` | R&D |
| 40 | SANDBOX | `10.10.40.0/22` | `10.10.40.1` | Réseaux et VM de test |
| 50 | MANAGEMENT | `10.10.50.0/24` | `10.10.50.1` | Hyperviseurs, pare-feu, switches, SIEM, bastion |
| 60 | BACKUP | `10.10.60.0/24` | `10.10.60.1` | PBS et flux de sauvegarde |
| 70 | IOT-PRINT | `10.10.70.0/24` | `10.10.70.1` | Imprimantes et IoT |
| 80 | GUEST | `10.10.80.0/24` | `10.10.80.1` | Invités, Internet uniquement |
| 90 | REPLICATION | `10.10.90.0/24` | aucune si lien L2 dédié | Réplication Proxmox |

### 5.2 Rennes — `10.20.0.0/16`

| VLAN | Nom | Sous-réseau | Passerelle |
|---:|---|---|---|
| 10 | USERS-RENNES | `10.20.10.0/24` | `10.20.10.1` |
| 50 | MGMT-RENNES | `10.20.50.0/24` | `10.20.50.1` |
| 60 | BACKUP-RENNES | `10.20.60.0/24` | `10.20.60.1` |
| 80 | GUEST-RENNES | `10.20.80.0/24` | `10.20.80.1` |

### 5.3 Paris — `10.30.0.0/16`

| VLAN | Nom | Sous-réseau | Passerelle |
|---:|---|---|---|
| 10 | USERS-PARIS | `10.30.10.0/24` | `10.30.10.1` |
| 50 | MGMT-PARIS | `10.30.50.0/24` | `10.30.50.1` |
| 60 | BACKUP-PARIS | `10.30.60.0/24` | `10.30.60.1` |
| 80 | GUEST-PARIS | `10.30.80.0/24` | `10.30.80.1` |

### 5.4 Adresses structurantes proposées

| Service | Adresse proposée |
|---|---|
| VIP OPNsense Nantes — VLAN serveurs | `10.10.20.1` |
| DC01 / DNS | `10.10.20.10` |
| DC02 / DNS | `10.10.20.11` |
| Serveur de fichiers | `10.10.20.20` |
| WSUS | `10.10.20.30` |
| Wazuh | `10.10.50.20` |
| Bastion | `10.10.50.10` |
| PVE01 | `10.10.50.31` |
| PVE02 | `10.10.50.32` |
| PBS01 | `10.10.60.10` |
| NAS Rennes | `10.20.60.10` |
| NAS Paris | `10.30.60.10` |

## 6. Routage, filtrage et accès Internet

### 6.1 Routage

Les ProCurve 1810 assurent la commutation L2. Les interfaces VLAN des pare-feu OPNsense constituent les passerelles et assurent le routage inter-VLAN.

### 6.2 Politique de filtrage

La politique générale est :

1. refus par défaut entre VLAN ;
2. ouverture des seuls flux justifiés ;
3. journalisation des refus et des flux sensibles ;
4. aucun accès d’administration depuis les VLAN utilisateurs ;
5. aucun accès de la Sandbox vers les réseaux internes ;
6. accès Internet invité sans accès aux réseaux privés ;
7. accès au VLAN sauvegarde uniquement depuis les composants autorisés.

### 6.3 Matrice de flux synthétique

| Source | Destination | Services | Décision |
|---|---|---|---|
| Utilisateurs | DC01/DC02 | DNS, Kerberos, LDAP(S), SMB requis par AD, NTP | Autoriser |
| Utilisateurs | Serveur de fichiers | SMB 445/TCP | Autoriser selon groupes AD |
| Utilisateurs | Management | Tous | Refuser |
| Utilisateurs | R&D | Flux explicitement nécessaires | Autoriser par groupe/besoin |
| R&D | Serveurs | Flux applicatifs documentés | Autoriser au cas par cas |
| Sandbox | Réseaux internes | Tous | Refuser et journaliser |
| Sandbox | Internet | DNS externe, HTTP(S) et besoins de test contrôlés | Autoriser avec filtrage et logs |
| Invités | Réseaux privés | Tous | Refuser |
| Invités | Internet | DNS, HTTP(S) | Autoriser |
| Bastion | Équipements administrés | HTTPS, SSH, RDP, WinRM selon besoin | Autoriser |
| Équipements | Wazuh | Syslog/agent | Autoriser |
| PVE | PBS | Flux Proxmox Backup Server | Autoriser |
| PBS/mécanisme d’export | NAS distants | Flux de copie choisi | Autoriser via VPN |
| Sites distants | Nantes | AD, DNS, SMB et supervision nécessaires | Autoriser via VPN |

La matrice exhaustive devra préciser ports, protocoles, objets source/destination, justification, propriétaire et durée de conservation.

## 7. Interconnexion VPN

Deux tunnels IPsec site-à-site sont établis :

- Nantes ↔ Rennes ;
- Nantes ↔ Paris.

Configuration cible :

- IKEv2 ;
- authentification par certificats lorsque possible, à défaut PSK longue et unique par tunnel ;
- chiffrement moderne pris en charge par tous les équipements, par exemple AES-256-GCM ;
- PFS activée ;
- DPD et reconnexion automatique ;
- routes limitées aux sous-réseaux nécessaires ;
- supervision de l’état des tunnels ;
- documentation sécurisée des paramètres et procédure de renouvellement.

Le trafic Internet courant sort localement sur chaque site. Seuls les flux internes empruntent les tunnels, sauf exigence de filtrage centralisé explicitement retenue.

## 8. Virtualisation et stockage

### 8.1 Cluster Proxmox

Les deux HP DL380 G6 exécutent Proxmox VE et forment un cluster à deux nœuds. Un QDevice physique indépendant apporte un troisième vote.

- `PVE01` et `PVE02` disposent chacun actuellement de quatre disques SAS 900 Go en RAID 5 sur contrôleur HPE à batterie, soit environ 2,2 To utiles annoncés ; un pool ZFS local n’est créé que si l’audit confirme un mode HBA/JBOD fiable. À défaut, le RAID matériel est conservé avec un système de fichiers pris en charge, ou le contrôleur et les disques sont remplacés ;
- les VM critiques sont réparties entre les nœuds ;
- la réplication ZFS est exécutée toutes les 15 minutes ;
- des groupes/règles d’affinité évitent de placer simultanément les deux contrôleurs de domaine sur le même nœud lorsque la version le permet ;
- le watchdog et les mécanismes de fencing sont configurés et testés avant d’activer la HA ;
- la capacité du nœud restant doit suffire aux seules VM prioritaires en mode dégradé.

La réplication n’est pas une sauvegarde : une suppression ou corruption peut être répliquée.

### 8.2 Machines virtuelles proposées

| VM | vCPU | RAM indicative | Stockage indicatif | Priorité |
|---|---:|---:|---:|---|
| DC01 | 2 | 4–8 Go | 80 Go | Critique |
| DC02 | 2 | 4–8 Go | 80 Go | Critique |
| FILE01 | 4 | 16 Go | selon volumétrie | Critique |
| WAZUH01 | 4–8 | 16–32 Go | selon rétention | Haute |
| WSUS01 | 4 | 8–16 Go | 300 Go ou plus | Moyenne |
| BASTION01 | 2 | 4–8 Go | 80 Go | Haute |
| Services R&D | selon besoin | selon besoin | selon projet | Variable |
| VM Sandbox | quotas par projet | quotas par projet | temporaire | Basse |

Ces valeurs doivent être ajustées après mesure. La plateforme doit démontrer le fonctionnement des VM de production et de **10 VM Sandbox simultanées** sous quotas. Une réserve cible de 30 % en stockage et une réserve de CPU/RAM pour les services prioritaires doivent être conservées ; en mode dégradé, les VM Sandbox peuvent être arrêtées pour respecter les priorités. La volumétrie de départ déclarée — 500 Go de VM, 100 Go de rapports, 500 Go de R&D et environ 1 To de sauvegardes/snapshots — impose un plan de capacité avant migration, car elle approche la capacité utile annoncée des serveurs.

### 8.3 Serveur de fichiers

`FILE01` publie les partages SMB. Les droits sont accordés par groupes Active Directory et jamais directement par utilisateur, sauf exception documentée.

Exemple de structure :

- `Audit` ;
- `Pentest` ;
- `Contrats` ;
- `Commercial` ;
- `R&D` ;
- `Commun`.

Les permissions de partage et NTFS appliquent le moindre privilège. L’audit des accès sensibles est activé.

## 9. Identités, DNS et DHCP

### 9.1 Active Directory

Deux contrôleurs de domaine, `DC01` et `DC02`, sont installés sur des nœuds différents. Ils assurent :

- authentification centralisée ;
- groupes et politiques de sécurité ;
- DNS interne ;
- gestion des droits d’accès.

Chaque administrateur possède un compte bureautique et un compte administratif distinct. Les comptes administratifs sont protégés par MFA sur les consoles compatibles.

### 9.2 DNS

Les postes membres du domaine utilisent `DC01` et `DC02`. Les équipements non joints au domaine peuvent utiliser un résolveur OPNsense local si aucune résolution du domaine interne n’est requise.

### 9.3 DHCP

OPNsense assure le DHCP local sur chaque site. Les options DHCP indiquent les DNS Active Directory aux postes du domaine et le résolveur approprié aux réseaux invités ou techniques.

## 10. Sauvegarde et reprise

### 10.1 Stratégie

La cible est une politique 3-2-1-1-0 :

- données actives ;
- sauvegarde primaire sur PBS avec stockage dédié ;
- copie hors site ;
- copie immuable ou déconnectée ;
- vérifications automatiques et restaurations testées.

### 10.2 Politique proposée

- snapshots ZFS horaires pour retour arrière rapide, sans les considérer comme sauvegardes ;
- sauvegardes quotidiennes des VM sur PBS ;
- rétention cible : 7 quotidiennes, 4 hebdomadaires et 12 mensuelles ;
- copie chiffrée hors site vers Rennes et copie des données prioritaires vers Paris ;
- support déconnecté chiffré ou stockage objet avec verrouillage d’objets ;
- restauration de fichier mensuelle ;
- restauration complète de VM trimestrielle.

### 10.3 Intégration des NAS Synology

Un NAS Synology n’est pas assimilé à un serveur PBS distant. La méthode retenue doit être validée par un test de restauration. Les options sont :

1. déployer un PBS distant et utiliser le NAS comme stockage selon une architecture prise en charge ;
2. exporter ou copier les sauvegardes avec un mécanisme documenté, chiffré et cohérent ;
3. utiliser les NAS pour les sauvegardes de fichiers, tandis qu’un vrai datastore PBS distant reçoit les sauvegardes de VM.

Les comptes de sauvegarde sont distincts, sans accès bureautique. Les volumes ne sont pas montés en permanence en écriture par les postes ou le serveur de fichiers.

### 10.4 Scénarios de reprise

| Incident | Réponse cible |
|---|---|
| Panne d’un nœud Proxmox | Redémarrage des VM prioritaires sur le nœud restant |
| Suppression d’un fichier | Restauration depuis snapshot ou sauvegarde selon date |
| Corruption/ransomware | Isolement, restauration depuis copie vérifiée et non modifiable |
| Panne du site Nantes | Reprise limitée ; restauration sur infrastructure de secours à planifier |
| Coupure VPN d’un site | Internet local maintenu, identifiants en cache, ressources centrales indisponibles |

## 11. Supervision et journalisation

Wazuh centralise les événements des sources compatibles :

- postes et serveurs ;
- Proxmox ;
- OPNsense ;
- NAS ;
- switches ;
- services de sauvegarde.

Alertes prioritaires :

- équipement ou service indisponible ;
- tunnel VPN interrompu ;
- échec de sauvegarde ou de vérification ;
- espace disque critique ;
- authentifications anormales ;
- élévation de privilèges ;
- modification d’une configuration sensible ;
- suppression massive de fichiers.

Cible de rétention : six mois en ligne et un an en archive, sous réserve de la volumétrie et des obligations applicables.

## 12. Sécurité des postes et serveurs

- EDR sur chaque système compatible ;
- BitLocker ou chiffrement équivalent ;
- retrait des droits administrateur local ordinaires ;
- gestion des comptes locaux avec Windows LAPS ou équivalent ;
- correctifs Windows via WSUS ;
- dépôts contrôlés et politique centralisée pour Linux ;
- blocage des macros non approuvées et des logiciels non autorisés ;
- revue trimestrielle des droits ;
- durcissement des systèmes selon des référentiels adaptés ;
- sauvegarde des configurations des équipements.

## 13. Administration

Un bastion dans le VLAN Management constitue le point d’entrée d’administration. Son accès exige :

- un compte administratif nominatif ;
- MFA lorsque disponible ;
- journalisation ;
- poste de confiance ;
- accès limité aux administrateurs habilités.

Les consoles Proxmox, OPNsense, Synology, Wazuh et les interfaces des switches ne sont pas exposées sur les VLAN utilisateurs ni sur Internet.

## 14. Disponibilité et limites

### 14.1 Redondance apportée

- deux nœuds de virtualisation ;
- deux contrôleurs de domaine ;
- réplication ZFS ;
- paire de pare-feu à Nantes sous conditions ;
- double commutation au siège si deux switches sont disponibles ;
- copies de sauvegarde hors site.

### 14.2 Points uniques de défaillance résiduels

- accès Internet et équipement opérateur à Nantes ;
- pare-feu et switch uniques à Rennes et Paris ;
- dépendance des fichiers et de l’AD au site de Nantes ;
- QDevice unique, sans impact direct sur les données mais important pour le quorum ;
- serveur PBS unique ;
- matériel HP ancien et disponibilité des pièces.

Un second accès Internet, un pare-feu secondaire par site distant et une capacité de reprise hors Nantes amélioreraient la continuité.

## 15. Intégration de l’existant et des usages client

### 15.1 Active Directory, Microsoft 365 et Teams

L’AD installé en 2015 fait l’objet d’un audit de santé, DNS, réplication, niveaux fonctionnels, GPO, groupes, comptes de service et dépendances applicatives. `DC01` et `DC02` sont ajoutés ou migrés selon le résultat ; les rôles sont transférés, la réplication validée et les anciens DC retirés uniquement après recette et plan de retour arrière. L’identité Microsoft 365 est inventoriée ; la synchronisation, le MFA et les accès conditionnels sont configurés selon les licences disponibles. Les groupes Teams et SMB sont alignés sur les spécialités Audit, Pentest, Commercial et R&D, avec propriétaires et revue trimestrielle. Les alternants reçoivent les droits de leur fonction, sans privilège spécifique.

### 15.2 Keeper et accès distant

Keeper reste le coffre approuvé pour les mots de passe et le canal déclaré pour l’échange de documents sensibles. Les coffres partagés, propriétaires, comptes de secours, révocations et journaux sont intégrés aux procédures. Les collaborateurs nomades utilisent un VPN IKEv2/SSL ou une solution ZTNA avec MFA, certificat d’équipement ou poste géré, EDR à jour, groupes AD, journalisation et révocation centralisée. L’accès distant à la Sandbox est limité aux groupes habilités ; l’administration technique passe toujours par le bastion.

### 15.3 Applications Debian

Les applications métier sous Debian sont inventoriées avec leur propriétaire, dépendances, ports, version, données, sauvegarde, méthode de mise à jour et criticité. Elles sont placées dans le VLAN Serveurs ou R&D selon leur usage et soumises aux mêmes exigences de sauvegarde, supervision et restauration que les services Windows.

## 16. Disponibilité, RPO et continuité de site

| Service/scénario | RTO métier maximal | RPO métier | Réponse technique |
|---|---:|---:|---|
| AD/DNS, fichiers et applications critiques — panne d’un nœud | 4 h | 0 visé | redondance, réplication, journalisation et redémarrage 15–30 min |
| Données de fichiers critiques | 4 h | 0 visé | snapshots fréquents, protection continue/journalisée à sélectionner et sauvegarde ; RPO mesuré |
| VM répliquées | 4 h | 0 visé | réplication asynchrone 15 min : écart maximal de 15 min à faire accepter tant qu’aucune solution synchrone n’est déployée |
| Perte complète de Nantes | 4 h visé | 0 visé | capacité de calcul hors Nantes, identité secondaire, sauvegarde réplicable et second opérateur requis dans la cible |

La réponse « aucune perte » est enregistrée comme exigence RPO 0 et non comme garantie déjà satisfaite. Aucun document ne doit déclarer la conformité avant mesure. Si la solution synchrone ou continue est techniquement ou budgétairement impossible, le DG/DSI signe une dérogation précisant le RPO accepté par service. De même, l’objectif de transparence lors de l’indisponibilité d’un site exige des ressources hors Nantes ; le simple stockage de sauvegardes sur NAS ne suffit pas.

## 17. Capacité, énergie et évolutivité

- la croissance de référence est de 10 % d’effectif sous deux ans ; les licences, ports, plages, stockage et ressources VM conservent cette marge ;
- le modèle de site OPNsense, VLAN, VPN et supervision est réutilisable pour Bordeaux, Lyon ou Lille ;
- les locaux techniques climatisés de 9 m² sont contrôlés pour alimentation, température, accès physique, détection et câblage ;
- des onduleurs supervisés assurent l’autonomie à définir et l’arrêt propre des serveurs, stockages, switches et pare-feu ;
- les équipements de 2015 non maintenus font l’objet de variantes de renouvellement et d’un stock de pièces transitoire ;
- la charge d’exploitation cible est de 6 h/semaine hors incident, suivie pendant le pilote ; les tâches quotidiennes sont automatisées par supervision et alertes.

## 18. Gouvernance, budget et planning

Trois variantes sont remises au DG/DSI : **maquette minimale**, **production sécurisée** et **continuité renforcée**. Chacune chiffre matériel, licences, support, second opérateur, onduleurs, stockage et charge d’exploitation, avec risques résiduels. Le planning proposé suit les phases audit, conception détaillée, maquette, migration pilote, recette, correction puis production. Les décisions techniques, dérogations RPO/RTO et réception finale sont validées par le DG/DSI. Le projet est demandé dès que possible, sans sacrifier audit, sauvegarde ni recette.

## 19. Préparation ISO 27001

Le projet alimente l’inventaire des actifs, la propriété des données, l’analyse de risques, les habilitations, la gestion des fournisseurs, les journaux, les incidents, la continuité et les preuves de contrôle. L’obtention de la certification et la mise en œuvre complète du SMSI restent un chantier de gouvernance distinct.

## 20. Séquencement de mise en œuvre

1. auditer le matériel, les disques, firmwares et cartes réseau ;
2. valider l’adressage public et le mode de raccordement Free ;
3. configurer les switches, VLAN, trunks et RSTP ;
4. installer OPNsense et appliquer les règles minimales ;
5. établir et tester les VPN ;
6. installer Proxmox, ZFS, le cluster et le QDevice ;
7. configurer la réplication et tester la reprise ;
8. déployer AD, DNS, DHCP et les stratégies de sécurité ;
9. déployer fichiers, bastion, Wazuh, WSUS et services R&D ;
10. déployer PBS et la copie hors site ;
11. intégrer les postes et équipements à la supervision ;
12. exécuter le cahier de recette et corriger les écarts ;
13. sauvegarder les configurations et remettre le dossier d’exploitation.

## 21. Décisions à valider avant production

- possibilité réelle de CARP avec l’accès Free ;
- quantité et capacités des interfaces réseau des DL380 ;
- état et mode des contrôleurs RAID pour ZFS ;
- volumétrie réelle des données et des journaux ;
- méthode de copie PBS/Synology ;
- capacité utile des NAS au regard de la rétention ;
- règles détaillées par groupe métier ;
- exigences réglementaires de conservation ;
- acceptation de l’indisponibilité des fichiers lors d’une perte de Nantes ;
- budget d’un second accès Internet et du matériel complémentaire.
