# Cahier de recette — Projet HING3

## 1. Objet

Ce cahier définit les vérifications permettant de confirmer que la maquette HING3 répond à l’expression de besoin et au dossier d’architecture technique.

Chaque test doit être exécuté sur la maquette, documenté et accompagné d’une preuve : capture d’écran, export de configuration, journal, résultat de commande ou photographie du montage.

## 2. Organisation de la recette

### 2.1 Rôles

| Rôle | Responsabilité |
|---|---|
| Responsable de recette | planifie la session, valide les résultats et arbitre les anomalies |
| Exécutant | prépare les données, réalise les tests et collecte les preuves |
| Exploitant | confirme l’exploitabilité et les procédures de reprise |
| Représentant métier | valide les usages utilisateur et les droits fonctionnels |

Une même personne peut tenir plusieurs rôles dans le cadre de la maquette, mais l’auteur du test et le validateur doivent être identifiés.

### 2.2 Statuts

- **À exécuter** : test non commencé ;
- **Réussi** : résultat conforme sans réserve ;
- **Réussi avec réserve** : objectif atteint avec un écart accepté ;
- **Échoué** : résultat non conforme ;
- **Bloqué** : prérequis absent ou incident empêchant l’exécution ;
- **Non applicable** : test hors du périmètre retenu, avec justification obligatoire.

### 2.3 Niveaux d’anomalie

| Niveau | Définition | Effet sur la réception |
|---|---|---|
| Critique | compromet la sécurité, les données ou un service essentiel | réception impossible |
| Majeure | fonction importante absente ou reprise non démontrée | réception conditionnelle ou impossible |
| Mineure | défaut limité avec solution de contournement | réception possible avec plan d’action |
| Observation | amélioration documentaire ou ergonomique | sans blocage |

## 3. Prérequis

Avant la recette :

- les trois sites sont représentés dans la maquette ;
- l’adressage IP et les VLAN sont configurés ;
- les pare-feu et tunnels IPsec sont opérationnels ;
- le cluster Proxmox, le QDevice et la réplication sont configurés ;
- les services AD, DNS, SMB, PBS, Wazuh et bastion sont installés ;
- au moins un poste de test est disponible par site ;
- des comptes de test existent pour chaque profil ;
- des fichiers factices non sensibles sont disponibles ;
- une sauvegarde complète récente est disponible ;
- la date, l’heure et la synchronisation NTP sont correctes ;
- les configurations sont sauvegardées avant les tests de panne.

## 4. Données de test

| Identifiant | Profil | Autorisations attendues |
|---|---|---|
| `test.audit` | Auditeur | lecture/écriture Audit, lecture Commun |
| `test.pentest` | Pentesteur | lecture/écriture Pentest et Sandbox, lecture Commun |
| `test.commercial` | Commercial | lecture/écriture Commercial, accès restreint aux rapports |
| `test.rnd` | R&D | accès R&D et Commun selon politique |
| `test.user` | Utilisateur standard | aucun droit administratif |
| `test.admin` | Administrateur dédié | administration via bastion uniquement |
| `test.guest` | Invité | Internet uniquement |

Les mots de passe ne doivent jamais figurer dans le procès-verbal ou les captures.

## 5. Fiche type de résultat

Pour chaque cas de test, renseigner :

| Champ | Valeur |
|---|---|
| Identifiant | |
| Date et heure | |
| Exécutant | |
| Environnement/version | |
| Résultat obtenu | |
| Statut | |
| Référence de la preuve | |
| Anomalie associée | |
| Commentaire | |

## 6. Tests d’infrastructure physique et réseau

### REC-RES-001 — Inventaire et état des équipements

- **Objectif** : vérifier que les équipements prévus sont présents et identifiés.
- **Procédure** : comparer les numéros d’inventaire, modèles, capacités et versions avec le dossier d’architecture.
- **Résultat attendu** : chaque équipement est identifié ; les écarts sont documentés.
- **Criticité en cas d’échec** : majeure.

### REC-RES-002 — VLAN de Nantes

- **Objectif** : vérifier la création et l’isolation des VLAN.
- **Procédure** : connecter un poste de test successivement aux ports affectés aux VLAN Utilisateurs, Management, R&D, Sandbox, Sauvegarde, IoT et Invités ; relever l’adresse reçue et tester les destinations autorisées.
- **Résultat attendu** : adresse et passerelle conformes ; aucun accès hors matrice de flux.
- **Criticité** : critique si un VLAN non fiable atteint une zone sensible.

### REC-RES-003 — VLAN des sites distants

- **Objectif** : valider les VLAN Utilisateurs, Management, Sauvegarde et Invités à Rennes et Paris.
- **Procédure** : répéter les tests d’adressage et de filtrage sur les deux sites.
- **Résultat attendu** : segmentation conforme au plan d’adressage.
- **Criticité** : majeure.

### REC-RES-004 — Trunks et prévention des boucles

- **Objectif** : vérifier le transport des VLAN et RSTP.
- **Procédure** : contrôler les VLAN tagués/non tagués, l’état RSTP et l’absence de boucle ; désactiver un lien redondant prévu pour observer la convergence.
- **Résultat attendu** : aucun trafic diffusé anormalement ; convergence sans perte durable.
- **Criticité** : majeure.

### REC-RES-005 — DHCP local

- **Objectif** : vérifier l’attribution des paramètres réseau même si le VPN est coupé.
- **Procédure** : renouveler le bail d’un poste sur chaque site, d’abord VPN actif puis VPN interrompu.
- **Résultat attendu** : adresse, masque, passerelle et DNS conformes ; DHCP local disponible.
- **Criticité** : majeure.

### REC-RES-006 — Résolution DNS interne

- **Objectif** : valider le DNS Active Directory.
- **Procédure** : depuis chaque site, résoudre les noms des deux DC et du serveur de fichiers ; effectuer une résolution inverse si configurée.
- **Résultat attendu** : réponses internes correctes via DC01/DC02.
- **Criticité** : majeure.

### REC-RES-007 — Accès Internet local

- **Objectif** : confirmer que chaque site utilise son accès Internet.
- **Procédure** : relever l’adresse IP publique depuis un poste de chaque site et consulter les journaux du pare-feu.
- **Résultat attendu** : sortie locale conforme, sans transit inutile par Nantes.
- **Criticité** : mineure à majeure selon le choix d’architecture.

## 7. Tests VPN inter-sites

### REC-VPN-001 — Tunnel Nantes–Rennes

- **Procédure** : vérifier l’état IKE/IPsec, tester un flux autorisé et consulter les journaux.
- **Résultat attendu** : tunnel établi avec les paramètres cryptographiques prévus ; seuls les réseaux déclarés transitent.
- **Criticité** : critique.

### REC-VPN-002 — Tunnel Nantes–Paris

- **Procédure** : identique à REC-VPN-001 pour Paris.
- **Résultat attendu** : tunnel opérationnel et filtré.
- **Criticité** : critique.

### REC-VPN-003 — Reconnexion automatique

- **Procédure** : interrompre temporairement un tunnel ou l’interface WAN de la maquette, puis rétablir la liaison.
- **Résultat attendu** : reconnexion automatique ; alerte générée ; reprise des flux autorisés.
- **Criticité** : majeure.

### REC-VPN-004 — Étanchéité des routes

- **Procédure** : depuis Rennes et Paris, tenter de joindre des sous-réseaux non autorisés.
- **Résultat attendu** : connexions refusées et journalisées.
- **Criticité** : critique.

## 8. Tests de sécurité et filtrage

### REC-SEC-001 — Isolement de la Sandbox

- **Objectif** : garantir qu’une machine de test compromise ne peut atteindre la production.
- **Procédure** : depuis une VM Sandbox, tenter de joindre les VLAN Utilisateurs, Serveurs, R&D, Management et Sauvegarde sur plusieurs protocoles.
- **Résultat attendu** : tous les accès internes sont bloqués et journalisés ; seuls les flux Internet explicitement autorisés fonctionnent.
- **Criticité** : critique.

### REC-SEC-002 — Isolation du réseau Invités

- **Procédure** : depuis le VLAN Invités, tester Internet puis les plages privées des trois sites.
- **Résultat attendu** : Internet accessible ; tous les réseaux privés inaccessibles.
- **Criticité** : critique.

### REC-SEC-003 — Protection du réseau Management

- **Procédure** : depuis un poste utilisateur, tenter d’ouvrir les consoles OPNsense, Proxmox, Wazuh, Synology et les switches.
- **Résultat attendu** : accès refusé.
- **Criticité** : critique.

### REC-SEC-004 — Administration via bastion

- **Procédure** : se connecter au bastion avec `test.admin`, puis administrer un équipement autorisé ; tenter la même opération depuis un poste utilisateur.
- **Résultat attendu** : opération possible depuis le bastion seulement ; action tracée.
- **Criticité** : majeure.

### REC-SEC-005 — Séparation des comptes

- **Procédure** : utiliser un compte bureautique pour une action administrative, puis le compte dédié.
- **Résultat attendu** : le compte bureautique est refusé ; le compte administratif autorisé est journalisé.
- **Criticité** : majeure.

### REC-SEC-006 — MFA administratif

- **Procédure** : se connecter à chaque console annoncée comme protégée par MFA.
- **Résultat attendu** : le second facteur est exigé ; les éventuelles exceptions sont documentées et compensées.
- **Criticité** : majeure.

### REC-SEC-007 — Droits administrateur local

- **Procédure** : avec `test.user`, tenter une installation et une modification système ; vérifier la gestion LAPS avec un administrateur habilité.
- **Résultat attendu** : utilisateur standard bloqué ; mot de passe local administré et réservé aux habilités.
- **Criticité** : majeure.

### REC-SEC-008 — Chiffrement des postes

- **Procédure** : contrôler l’état BitLocker ou équivalent sur un échantillon représentatif.
- **Résultat attendu** : chiffrement actif et clés de récupération conservées dans un emplacement sécurisé.
- **Criticité** : majeure.

## 9. Tests des identités et des droits

### REC-ID-001 — Authentification depuis les trois sites

- **Procédure** : ouvrir une session avec un utilisateur autorisé à Nantes, Rennes et Paris.
- **Résultat attendu** : authentification réussie et politiques appliquées.
- **Criticité** : critique.

### REC-ID-002 — Redondance DNS et AD

- **Procédure** : arrêter proprement DC01, puis tester ouverture de session, DNS et accès aux fichiers ; répéter avec DC02.
- **Résultat attendu** : services maintenus par le DC restant.
- **Criticité** : critique.

### REC-ID-003 — Permissions des partages

- **Procédure** : avec chaque profil de test, tenter lecture, création, modification et suppression sur chaque partage.
- **Résultat attendu** : opérations conformes à la matrice d’habilitation ; aucun accès indu.
- **Criticité** : critique en cas de divulgation, majeure autrement.

### REC-ID-004 — Identifiants en cache lors d’une coupure VPN

- **Procédure** : ouvrir une première session VPN actif, fermer la session, couper le VPN puis se reconnecter sur le même poste.
- **Résultat attendu** : connexion locale possible avec les identifiants en cache ; ressources centrales indiquées comme indisponibles.
- **Criticité** : mineure.

### REC-ID-005 — Désactivation d’un compte

- **Procédure** : désactiver un compte de test et vérifier ses accès après propagation.
- **Résultat attendu** : nouvelles authentifications refusées ; action visible dans les journaux.
- **Criticité** : majeure.

## 10. Tests du cluster et de la disponibilité

### REC-HA-001 — État du cluster et quorum

- **Procédure** : vérifier les deux nœuds, le QDevice, le quorum et la synchronisation horaire.
- **Résultat attendu** : trois votes visibles, cluster sain et quorate.
- **Criticité** : critique.

### REC-HA-002 — Réplication ZFS

- **Procédure** : créer un fichier horodaté dans une VM critique, attendre une réplication et vérifier la tâche et le volume cible.
- **Résultat attendu** : réplication réussie dans l’intervalle de 15 minutes sans erreur.
- **Criticité** : critique.

### REC-HA-003 — Panne contrôlée d’un nœud

- **Précaution** : confirmer la sauvegarde et la capacité du nœud restant avant le test.
- **Procédure** : arrêter proprement un nœud hébergeant une VM critique et mesurer le temps de redémarrage sur l’autre nœud.
- **Résultat attendu** : cluster quorate ; VM prioritaire disponible ; perte de données inférieure ou égale au RPO annoncé ; RTO mesuré et documenté.
- **Criticité** : critique.

### REC-HA-004 — Perte du QDevice

- **Procédure** : arrêter le QDevice sans arrêter les nœuds.
- **Résultat attendu** : services maintenus ; alerte émise ; état dégradé clairement visible.
- **Criticité** : majeure.

### REC-HA-005 — Capacité en mode dégradé

- **Procédure** : avec un seul nœud, relever CPU, RAM, latence stockage et disponibilité des VM prioritaires.
- **Résultat attendu** : absence de saturation empêchant les services critiques.
- **Criticité** : majeure.

### REC-HA-006 — Bascule des pare-feu Nantes

- **Condition** : applicable seulement si CARP est déployé et compatible avec le raccordement WAN.
- **Procédure** : arrêter le pare-feu maître et mesurer l’interruption des flux internes, VPN et Internet.
- **Résultat attendu** : reprise sur le secondaire ; états et configurations synchronisés ; interruption conforme à la cible retenue.
- **Criticité** : critique si la HA est annoncée comme opérationnelle.

## 11. Tests de stockage et de performance

### REC-STO-001 — Partages depuis chaque site

- **Procédure** : ouvrir, modifier et enregistrer un fichier de test sur `FILE01` depuis chaque site.
- **Résultat attendu** : aucune corruption ; droits respectés ; temps acceptable.
- **Criticité** : critique.

### REC-STO-002 — Mesures inter-sites

- **Procédure** : mesurer latence, débit utile et temps de transfert d’un fichier représentatif, sans données sensibles.
- **Résultat attendu** : résultats consignés et compatibles avec les usages définis ; aucun seuil absolu ne peut être accepté sans mesure de référence.
- **Criticité** : majeure si l’usage courant est impraticable.

### REC-STO-003 — État RAID des NAS

- **Procédure** : vérifier le RAID 1, l’état SMART, les alertes et la capacité utile.
- **Résultat attendu** : volumes sains, alertes configurées et capacité documentée.
- **Criticité** : majeure.

### REC-STO-004 — Quotas Sandbox

- **Procédure** : contrôler ou atteindre un quota de test sur une VM/projet Sandbox.
- **Résultat attendu** : une équipe ne peut pas consommer toute la capacité de production.
- **Criticité** : mineure à majeure.

## 12. Tests de sauvegarde et restauration

### REC-BKP-001 — Sauvegarde quotidienne PBS

- **Procédure** : exécuter une sauvegarde d’une VM et vérifier le journal, la durée et la taille.
- **Résultat attendu** : tâche réussie, point de restauration visible et vérification sans erreur.
- **Criticité** : critique.

### REC-BKP-002 — Rétention

- **Procédure** : examiner les règles de prune et simuler leur effet si l’historique est insuffisant.
- **Résultat attendu** : politique 7 quotidiennes, 4 hebdomadaires et 12 mensuelles configurée ou écart justifié.
- **Criticité** : majeure.

### REC-BKP-003 — Restauration d’un fichier

- **Procédure** : supprimer un fichier factice, le restaurer dans un emplacement contrôlé et comparer son empreinte.
- **Résultat attendu** : contenu identique et permissions restaurées ou réappliquées selon la procédure.
- **Criticité** : critique.

### REC-BKP-004 — Restauration complète d’une VM

- **Procédure** : restaurer une VM dans un réseau isolé, démarrer le système et vérifier le service.
- **Résultat attendu** : VM fonctionnelle, sans collision réseau avec la production ; durée mesurée.
- **Criticité** : critique.

### REC-BKP-005 — Copie hors site

- **Procédure** : déclencher la copie vers Rennes, vérifier chiffrement, journal, volume cible et séparation des comptes.
- **Résultat attendu** : copie cohérente disponible hors Nantes et inaccessible aux utilisateurs ordinaires.
- **Criticité** : critique.

### REC-BKP-006 — Restauration depuis la copie hors site

- **Procédure** : restaurer un élément à partir de la copie de Rennes ou Paris, sans utiliser la sauvegarde locale.
- **Résultat attendu** : restauration réussie ; méthode PBS/Synology démontrée et documentée.
- **Criticité** : critique.

### REC-BKP-007 — Copie immuable ou déconnectée

- **Procédure** : vérifier Object Lock ou la déconnexion physique/logique du support après écriture ; tenter une suppression depuis un compte de production.
- **Résultat attendu** : suppression impossible pendant la rétention ou support inaccessible.
- **Criticité** : critique.

### REC-BKP-008 — Alerte d’échec

- **Procédure** : provoquer un échec contrôlé d’une tâche de sauvegarde non critique.
- **Résultat attendu** : alerte reçue et exploitable, sans exposer de secret.
- **Criticité** : majeure.

## 13. Tests de supervision et de journalisation

### REC-SUP-001 — Remontée des sources

- **Procédure** : générer un événement depuis un poste, un serveur, OPNsense, Proxmox et un NAS.
- **Résultat attendu** : chaque événement apparaît dans Wazuh avec source et horodatage corrects.
- **Criticité** : majeure.

### REC-SUP-002 — Alerte d’authentification suspecte

- **Procédure** : réaliser plusieurs échecs de connexion avec un compte de test.
- **Résultat attendu** : événement détecté et alerte produite selon le seuil défini.
- **Criticité** : majeure.

### REC-SUP-003 — Alerte espace disque

- **Procédure** : abaisser temporairement le seuil sur un volume de test ou utiliser une simulation prise en charge.
- **Résultat attendu** : alerte visible avec hôte, volume et seuil.
- **Criticité** : majeure.

### REC-SUP-004 — Contrôle d’accès aux journaux

- **Procédure** : tenter d’accéder aux journaux avec un utilisateur standard puis un administrateur habilité.
- **Résultat attendu** : accès refusé au premier et autorisé au second.
- **Criticité** : majeure.

## 14. Tests d’exploitation

### REC-EXP-001 — Procédure de création d’utilisateur

- **Procédure** : appliquer le dossier d’exploitation pour créer un compte de test et ses groupes.
- **Résultat attendu** : compte opérationnel avec les seuls droits prévus ; procédure suffisante et reproductible.
- **Criticité** : majeure.

### REC-EXP-002 — Procédure de départ

- **Procédure** : désactiver un compte, révoquer ses sessions et traiter ses données selon la procédure.
- **Résultat attendu** : accès supprimé, traces conservées et actions consignées.
- **Criticité** : majeure.

### REC-EXP-003 — Sauvegarde de configuration

- **Procédure** : exporter les configurations OPNsense, switches, Proxmox et NAS selon les possibilités.
- **Résultat attendu** : archives datées, chiffrées, inventoriées et restaurables.
- **Criticité** : majeure.

### REC-EXP-004 — Procédure d’incident

- **Procédure** : simuler une alerte de sécurité et suivre les étapes d’identification, confinement, conservation des traces et escalade.
- **Résultat attendu** : rôles et actions compris ; aucune preuve détruite ; incident consigné.
- **Criticité** : majeure.

### REC-EXP-005 — Documentation et secrets

- **Procédure** : vérifier que les documents ne contiennent aucun mot de passe, clé privée ou PSK ; contrôler que l’emplacement sécurisé des secrets est référencé.
- **Résultat attendu** : aucun secret dans la documentation ; accès limité au coffre-fort retenu.
- **Criticité** : critique.

## 15. Tests des réponses client et de la capacité

### REC-CLI-001 — RTO métier de 4 heures

- **Procédure** : exécuter les scénarios de panne d’un nœud, restauration d’un service et reprise documentée ; mesurer du début d’incident au retour du service.
- **Résultat attendu** : chaque service critique revient en moins de 4 heures ; la cible de 15 à 30 minutes est évaluée pour la panne simple.
- **Criticité** : critique.

### REC-CLI-002 — RPO et absence de perte

- **Procédure** : générer des écritures horodatées, provoquer une panne contrôlée et comparer la dernière donnée validée après reprise pour les fichiers, applications et VM critiques.
- **Résultat attendu** : RPO mesuré par service ; RPO 0 démontré là où annoncé. Toute perte, notamment la fenêtre asynchrone de 15 minutes, produit un écart soumis au DG/DSI.
- **Criticité** : critique.

### REC-CLI-003 — Dix VM de test simultanées

- **Procédure** : démarrer les VM de production et 10 VM Sandbox représentatives, exécuter une charge convenue et relever CPU, RAM, IOPS, latence, stockage et réseau.
- **Résultat attendu** : production utilisable, quotas respectés, marge documentée et absence d’impact critique.
- **Criticité** : majeure.

### REC-CLI-004 — Accès distant utilisateur

- **Procédure** : depuis un poste géré hors site, tester MFA, accès autorisé, refus des zones non autorisées, journalisation et révocation de session.
- **Résultat attendu** : accès flexible et sécurisé ; aucune administration directe ni accès indu.
- **Criticité** : critique.

### REC-CLI-005 — Microsoft 365, Teams et Keeper

- **Procédure** : tester un utilisateur de chaque spécialité, la propriété des groupes Teams, le MFA, la révocation et un partage sensible factice par Keeper.
- **Résultat attendu** : accès conforme au métier, secrets absents des documents, journaux et propriétaires identifiés.
- **Criticité** : majeure.

### REC-CLI-006 — Migration de l’AD 2015

- **Procédure** : contrôler l’audit initial, la réplication, DNS, GPO, groupes, comptes de service, applications et plan de retour arrière avant retrait d’un ancien DC.
- **Résultat attendu** : aucune identité, permission ou dépendance perdue ; état AD sain.
- **Criticité** : critique.

### REC-CLI-007 — Applications Debian

- **Procédure** : vérifier inventaire, supervision, correctifs, sauvegarde et restauration d’une application Debian représentative.
- **Résultat attendu** : service exploitable et restaurable selon sa criticité.
- **Criticité** : majeure.

### REC-CLI-008 — RAID, HBA et capacité

- **Procédure** : relever modèle du contrôleur, mode réel, disques, SMART, RAID, capacité et performances ; vérifier que ZFS n’est utilisé qu’avec une exposition directe validée des disques.
- **Résultat attendu** : architecture de stockage prise en charge, marge cible documentée et absence de superposition ZFS/RAID non validée.
- **Criticité** : critique.

### REC-CLI-009 — Énergie et locaux

- **Procédure** : contrôler les locaux de 9 m², climatisation, alimentation, onduleurs et alertes ; simuler une perte secteur sans risque pour mesurer autonomie et arrêt propre.
- **Résultat attendu** : équipements protégés, alertes reçues et arrêt propre démontré.
- **Criticité** : majeure.

### REC-CLI-010 — Charge d’exploitation

- **Procédure** : pendant une période pilote représentative, chronométrer contrôles, sauvegardes, correctifs, comptes, incidents simulés et rapports.
- **Résultat attendu** : charge planifiée inférieure ou égale à 6 h/semaine hors incident ; sinon automatisation, périmètre ou ressources ajustés.
- **Criticité** : majeure.

### REC-CLI-011 — Croissance et nouveau site

- **Procédure** : vérifier une projection de +10 % d’utilisateurs et simuler la configuration d’un quatrième site à partir du modèle.
- **Résultat attendu** : capacité, adressage, licences et procédures suffisants sans refonte.
- **Criticité** : majeure.

### REC-CLI-012 — Continuité lors de la perte de Nantes

- **Procédure** : exercice sur table puis test technique de la capacité hors site retenue, sans utiliser les ressources de Nantes.
- **Résultat attendu** : reprise conforme aux objectifs annoncés ; sinon écart, plan financé et dérogation DG/DSI formalisés.
- **Criticité** : critique.

### REC-CLI-013 — Gouvernance et ISO 27001

- **Procédure** : vérifier l’inventaire, propriétaires, risques, preuves, registre de dérogations, variantes budgétaires, planning et validation DG/DSI.
- **Résultat attendu** : décisions traçables et éléments techniques réutilisables dans le futur SMSI.
- **Criticité** : majeure.

## 16. Synthèse d’exécution

| Domaine | Nombre de tests | Réussis | Avec réserve | Échoués | Bloqués | N/A |
|---|---:|---:|---:|---:|---:|---:|
| Réseau | 7 | | | | | |
| VPN | 4 | | | | | |
| Sécurité | 8 | | | | | |
| Identités | 5 | | | | | |
| Haute disponibilité | 6 | | | | | |
| Stockage/performance | 4 | | | | | |
| Sauvegarde | 8 | | | | | |
| Supervision | 4 | | | | | |
| Exploitation | 5 | | | | | |
| Réponses client/capacité | 13 | | | | | |
| **Total** | **64** | | | | | |

## 17. Registre des anomalies

| ID | Test | Description | Niveau | Responsable | Échéance | Statut | Contournement |
|---|---|---|---|---|---|---|---|
| | | | | | | | |

## 18. Critères de réception

La maquette peut être réceptionnée si :

- aucun défaut critique n’est ouvert ;
- tous les tests relatifs à l’isolation, aux identités, aux VPN et à la restauration sont réussis ;
- les anomalies majeures restantes disposent d’un plan d’action accepté ;
- les objectifs RPO/RTO mesurés sont consignés ;
- les limites de disponibilité sont explicitement acceptées ;
- les preuves sont archivées ;
- le dossier d’exploitation a été testé par une personne autre que son rédacteur lorsque cela est possible.

## 19. Procès-verbal de recette

| Champ | Valeur |
|---|---|
| Version de l’architecture | |
| Période de recette | |
| Responsable | |
| Nombre de tests réussis | |
| Nombre de tests avec réserve | |
| Nombre de tests échoués/bloqués | |
| Anomalies critiques ouvertes | |
| Décision | Acceptée / Acceptée avec réserves / Refusée |
| Réserves et échéances | |

### Signatures

| Rôle | Nom | Date | Signature |
|---|---|---|---|
| Responsable de recette | | | |
| Exploitant | | | |
| Représentant métier | | | |
