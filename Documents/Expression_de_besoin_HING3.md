# Expression de besoin — Projet HING3

## 1. Objet du document

Le présent document formalise les besoins de **Sec-Digi**, PME spécialisée en cybersécurité, pour la conception et la réalisation d’une infrastructure informatique sécurisée, performante et exploitable sur trois sites géographiques.

Il décrit les objectifs métier, les utilisateurs, les contraintes, les exigences fonctionnelles et non fonctionnelles ainsi que les critères généraux de réussite. Les choix détaillés de mise en œuvre sont traités dans le dossier d’architecture technique.

## 2. Contexte

Sec-Digi emploie **60 personnes** réparties sur trois sites :

| Site | Effectif | Population |
|---|---:|---|
| Nantes — siège | 30 | 10 pentesteurs, 10 auditeurs, 1 DG, 5 commerciaux, 4 alternants |
| Rennes | 15 | 10 auditeurs, 5 pentesteurs |
| Paris | 15 | 5 commerciaux, 5 auditeurs, 5 pentesteurs |

L’administration informatique est assurée à temps partiel par les auditeurs et pentesteurs de Nantes, avec une capacité cible de **6 heures par semaine** hors incident majeur. La solution doit donc être centralisée, documentée, largement automatisée, simple à administrer et peu chronophage au quotidien.

Toutes les activités dépendent du système d’information pour les rapports, les outils métier et les infrastructures de test. Les outils quotidiens comprennent Microsoft 365, Teams, Keeper, les outils d’audit et de pentest, une infrastructure virtuelle, des applications hébergées sur Debian et des données hébergées sur Windows Server. L’annuaire Active Directory existant date de 2015 et doit être audité puis migré sans recréation non maîtrisée des identités, groupes et droits.

Le client vise une certification **ISO 27001**. La présente architecture doit en faciliter la préparation par la gestion des risques, des actifs, des habilitations, des journaux, des incidents, de la continuité et des preuves, sans prétendre constituer à elle seule un SMSI certifiable.

## 3. Enjeux

Les principaux enjeux sont les suivants :

- permettre aux collaborateurs de travailler depuis chacun des trois sites ;
- protéger des données métier sensibles ;
- fournir des environnements isolés pour reproduire des infrastructures clientes ;
- préserver la disponibilité des services en cas de panne matérielle ;
- limiter le risque d’attaque externe, de mouvement latéral et de fuite interne ;
- assurer la sauvegarde, la restauration et l’externalisation des données ;
- permettre une exploitation réaliste avec des ressources IT limitées.

## 4. Périmètre

### 4.1 Inclus dans le périmètre

- réseaux locaux de Nantes, Rennes et Paris ;
- interconnexion sécurisée des trois sites ;
- accès Internet et filtrage périmétrique ;
- segmentation des réseaux ;
- virtualisation des serveurs ;
- gestion centralisée des identités ;
- stockage et partage des données métier et R&D ;
- environnements de test et de simulation ;
- sauvegarde, restauration et reprise après incident ;
- supervision, centralisation des journaux et alertes ;
- sécurisation des postes, serveurs et accès administratifs ;
- procédures courantes d’exploitation.

### 4.2 Hors périmètre ou à traiter séparément

- choix contractuel d’un second opérateur Internet ;
- achat définitif des licences et abonnements ;
- téléphonie, contrôle d’accès physique et vidéosurveillance ;
- hébergement de services publics destinés aux clients ;
- mise en œuvre complète d’un SMSI et obtention de la certification ISO 27001, tout en préparant les mesures techniques et les preuves utiles ;
- ouverture effective de nouveaux sites, même si l’architecture doit permettre ultérieurement Bordeaux, Lyon ou Lille.

## 5. Matériel disponible

### 5.1 Nantes

- deux serveurs **HP DL380 G6**, chacun doté de deux processeurs et de 128 Go de RAM ;
- réseau HPE à 1 Gbit/s basé sur des switches ProCurve 1810 ;
- un accès Internet Free grand public à 400 Mbit/s ;
- quatre disques SAS de 900 Go en RAID 5 dans chaque serveur, soit environ 2,2 To utiles annoncés par serveur ;
- un contrôleur RAID HPE avec batterie, de référence et de compatibilité HBA/passthrough inconnues.

La volumétrie déclarée est d’environ 500 Go de VM, 100 Go de rapports de pentest, 500 Go de R&D et 1 To de sauvegardes et snapshots. Une saturation ou un problème de performance stockage a déjà eu lieu. L’état, les performances, la marge réelle, le contrôleur et la compatibilité ZFS doivent être vérifiés avant tout choix définitif.

### 5.2 Rennes et Paris

Chaque site dispose de :

- un NAS Synology DS918+ ;
- deux disques de 4 To en RAID 1 ;
- un réseau HPE à 1 Gbit/s basé sur un switch ProCurve 1810 ;
- un accès Internet Free grand public à 400 Mbit/s.

Le matériel date principalement de 2015, n’est plus maintenu et son renouvellement peut être proposé. Chaque site dispose d’un local technique climatisé d’environ 9 m². Des coupures électriques et Internet ont déjà été constatées : onduleurs, arrêt propre, second accès Internet et contrats de support doivent être chiffrés par variantes. Le matériel complémentaire indispensable à la sécurité, à la haute disponibilité et à la sauvegarde pourra être ajouté ou représenté dans la maquette.

## 6. Besoins fonctionnels

### BF-01 — Travail multisite

Les utilisateurs doivent pouvoir s’authentifier et accéder aux ressources autorisées depuis Nantes, Rennes ou Paris, sous réserve de la disponibilité des liaisons réseau.

### BF-02 — Gestion des identités

Les comptes, groupes et droits doivent être administrés de manière centralisée. Les autorisations doivent respecter le principe du moindre privilège et permettre une séparation par fonction et par sensibilité des données.

### BF-03 — Stockage des données métier

La solution doit héberger et partager de manière sécurisée :

- les rapports d’audit ;
- les rapports de pentest ;
- les contrats ;
- les documents commerciaux et administratifs ;
- les autres données métier sensibles.

### BF-04 — Données de R&D

Les données et services de R&D doivent être séparés logiquement des usages bureautiques et accessibles uniquement aux personnes habilitées.

### BF-05 — Environnements de test

Les pentesteurs et auditeurs doivent pouvoir créer des machines virtuelles et des réseaux de test reproduisant les infrastructures clientes. La capacité cible est celle des VM de production **plus 10 VM de test simultanées**, sous quotas et après dimensionnement mesuré. Ces environnements doivent être totalement isolés du système d’information interne, supprimables ou restaurables rapidement par snapshot sans effet sur la production, accessibles à distance par des utilisateurs habilités et disposer d’un accès Internet filtré et journalisé pour les mises à jour et recherches de vulnérabilités.

### BF-06 — Interconnexion des sites

Les échanges entre sites doivent être chiffrés et authentifiés. Ils doivent permettre l’accès aux services internes, la supervision et les flux de sauvegarde autorisés.

### BF-07 — Sauvegarde et restauration

La solution doit assurer :

- des sauvegardes régulières des systèmes et données critiques ;
- au moins une copie hors site ;
- une copie non modifiable ou déconnectée ;
- la restauration d’un fichier, d’une VM et d’un service complet ;
- la vérification périodique de la restaurabilité.

### BF-08 — Supervision et traçabilité

Les événements des équipements, serveurs et postes compatibles doivent être centralisés. Les administrateurs doivent être alertés en cas de panne, d’échec de sauvegarde ou de comportement suspect.

### BF-09 — Administration sécurisée

Les interfaces d’administration ne doivent pas être accessibles directement depuis les réseaux utilisateurs. Les actions sensibles doivent utiliser des comptes nominatifs dédiés et une authentification multifacteur lorsqu’elle est disponible.

### BF-10 — Outils collaboratifs, secrets et échanges sensibles

L’architecture doit prendre en compte Microsoft 365 et les espaces Teams par spécialité, conserver Keeper comme canal approuvé pour les mots de passe et échanges documentaires sensibles, et définir le cycle de vie des groupes, propriétaires et accès. Les documents du projet ne doivent contenir aucun secret.

### BF-11 — Accès distant

Les collaborateurs habilités doivent disposer d’un accès distant flexible mais sécurisé au moyen d’un VPN nomade ou d’une solution ZTNA, avec MFA, poste géré, contrôle des droits, journalisation et révocation centralisée. L’accès d’administration reste limité au bastion.

### BF-12 — Habilitations métier

Chaque profil accède à son espace Teams et à son répertoire de spécialité. Les alternants disposent des mêmes droits que les internes de la fonction à laquelle ils sont affectés, sans privilège supplémentaire lié à leur statut. Les données appartenant directement aux clients ne sont pas hébergées ; les rapports et livrables sensibles restent néanmoins protégés comme données confidentielles.

## 7. Exigences non fonctionnelles

### ENF-01 — Sécurité

- segmentation des usages et refus par défaut entre zones ;
- chiffrement des flux inter-sites et des postes ;
- MFA pour les accès administratifs ;
- protection des endpoints et gestion centralisée des correctifs ;
- séparation des comptes bureautiques et administratifs ;
- journalisation des accès et opérations sensibles ;
- isolement strict des environnements de test.

### ENF-02 — Disponibilité

La panne d’un serveur de virtualisation ne doit pas entraîner la perte définitive des services critiques. Les machines prioritaires doivent pouvoir redémarrer sur le nœud restant.

Le client fixe une **durée maximale d’interruption de 4 heures** pour toute activité critique et exprime une tolérance de perte de données **nulle**. Cette dernière constitue un objectif métier à traduire par service :

- **RTO contractuel maximal : 4 heures** pour les services critiques ;
- **RTO technique cible : 15 à 30 minutes** après une panne simple d’un nœud ;
- **RPO cible : 0** pour les données critiques lorsque la technologie retenue permet une écriture répliquée ou journalisée sans perte ;
- **RPO technique maximal provisoire : 15 minutes** pour les VM répliquées de façon asynchrone ; tout écart au RPO 0 doit être mesuré, documenté et accepté par le DG/DSI ;
- restauration des données selon la fréquence et la rétention définies dans l’architecture.

L’objectif métier est de rendre l’indisponibilité d’un site transparente. Le matériel initial ne permet pas de garantir cet objectif en cas de perte complète de Nantes ou de son unique accès Internet. Une capacité de reprise hors Nantes et un second opérateur constituent donc la cible de production ; à défaut, l’écart et le mode dégradé doivent faire l’objet d’une dérogation formelle du DG/DSI, et non d’une conformité implicite.

### ENF-03 — Performance

- réseau local à 1 Gbit/s ;
- absence de goulot d’étranglement majeur dans le cœur de réseau ;
- accès inter-sites compatible avec les usages bureautiques courants ;
- planification des sauvegardes pour limiter leur impact ;
- mesure du débit, de la latence et des temps d’ouverture de fichiers lors de la recette.

### ENF-04 — Exploitabilité

- administration centralisée autant que possible ;
- documentation des procédures récurrentes ;
- alertes compréhensibles et exploitables ;
- inventaire des équipements, services et dépendances ;
- limitation du nombre de technologies différentes.

### ENF-05 — Évolutivité

La solution doit absorber une hausse d’effectif de **10 % sous deux ans** et permettre l’ajout de VM, de capacité de stockage, de VLAN, d’un second accès Internet et d’un nouveau site éventuel à Bordeaux, Lyon ou Lille sans refonte complète. La capacité est revue au moins trimestriellement.

## 8. Contraintes et hypothèses

- Le matériel existant, ancien, doit être validé avant mise en production : état des disques, compatibilité ZFS, firmwares et performances.
- Les ProCurve 1810 assurent principalement des fonctions de niveau 2 ; le routage inter-VLAN doit être assuré par les pare-feu.
- La haute disponibilité des pare-feu de Nantes dépend des possibilités offertes par l’accès Free : adressage, mode bridge, absence de CGNAT et câblage WAN.
- Les sites distants et l’accès Internet de Nantes conservent des points uniques de défaillance.
- L’accès centralisé aux fichiers dépend de Nantes et des VPN inter-sites.
- La capacité utile de chaque NAS distant est d’environ 4 To avant prise en compte du formatage et des réserves système.
- Toute réplication de sauvegarde Proxmox vers un NAS Synology doit employer une méthode prise en charge et testée ; un NAS seul ne constitue pas automatiquement un serveur Proxmox Backup Server distant.

## 9. Priorités

| Priorité | Exigences |
|---|---|
| Critique | identités, données métier, segmentation, VPN, sauvegarde restaurable, isolement des tests |
| Haute | disponibilité des VM, supervision, patching, chiffrement, administration sécurisée |
| Moyenne | continuité locale lors d’une coupure WAN, automatisation avancée, second accès Internet |
| Basse | optimisations et fonctions de confort non indispensables à la maquette |

## 10. Critères généraux d’acceptation

Le projet sera considéré comme recevable si :

1. les trois sites sont représentés et interconnectés par des tunnels chiffrés ;
2. les utilisateurs autorisés accèdent aux ressources prévues depuis chaque site ;
3. les VLAN sont opérationnels et les flux interdits sont effectivement bloqués ;
4. la Sandbox ne peut pas joindre les réseaux internes ;
5. les services d’identité, DNS et fichiers sont opérationnels ;
6. la panne simulée d’un nœud permet la reprise des VM critiques selon les objectifs annoncés ;
7. une sauvegarde hors production est créée et une restauration est démontrée ;
8. les journaux et alertes prioritaires sont visibles dans la supervision ;
9. les procédures essentielles d’exploitation sont documentées ;
10. les limites résiduelles et points uniques de défaillance font l’objet d’une dérogation explicite du DG/DSI ;
11. le RTO de 4 heures est démontré pour chaque service critique et le RPO réel est mesuré, tout écart au RPO 0 étant formellement arbitré ;
12. les VM de production et 10 VM de test fonctionnent simultanément selon la charge et les quotas convenus ;
13. l’accès distant utilisateur, Microsoft 365, Teams, Keeper et les applications Debian sont intégrés aux habilitations et à l’exploitation ;
14. la migration de l’AD 2015 est testée avec un plan de retour arrière ;
15. la charge planifiée est compatible avec 6 heures par semaine hors incident ;
16. la capacité couvre la croissance de 10 % à deux ans et l’ajout d’un site selon le modèle retenu ;
17. les protections électriques, l’état des locaux et la stratégie de renouvellement sont validés ;
18. le DG/DSI valide la variante budgétaire, le planning, les risques et la réception.

## 11. Livrables attendus

- expression de besoin ;
- dossier d’architecture technique ;
- maquette fonctionnelle ;
- cahier et procès-verbal de recette ;
- dossier d’exploitation ;
- schémas logique et physique ;
- inventaire et matrice des flux ;
- sauvegarde des configurations de la maquette.
