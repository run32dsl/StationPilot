# Roadmap — StationPilot

> Suite libre de gestion de parc Windows pour les collectivités territoriales :
> création & déploiement d'images système, déploiement de logiciels,
> prise de main à distance, migration de profils utilisateurs lors des
> migrations de domaine Active Directory.
> Objectif : offrir une alternative souveraine, auditable et pérenne aux
> solutions propriétaires ou aux briques libres dont la gouvernance se ferme
> avec le temps.

## 1. Décisions actées

| Sujet | Décision | Justification |
|---|---|---|
| Ordre des modules | 1. Imaging → 2. Déploiement logiciel → 3. Prise de main à distance → 4. Migration de profils | L'imaging est jugé le besoin le plus structurant pour démarrer un parc, avant même le déploiement applicatif. |
| Stratégie technique | S'appuyer sur des briques open source existantes pour un MVP rapide, dans une architecture modulaire qui permette de réécrire une brique plus tard si elle s'avère inadaptée (licence, sécurité, maintenabilité) | Vitesse pour le MVP sans s'enfermer dans un choix technique irréversible. |
| Périmètre OS (V1) | Postes clients Windows uniquement | Cible = parc Windows actuel de la majorité des collectivités. |
| Stack technique (socle agent/serveur) | **Python**, à confirmer/creuser en phase 0/1 | Seul langage maîtrisé par le porteur du projet parmi les options envisagées ; écosystème riche pour AD/ACL (pywin32) et pour du scripting Windows. C#/.NET écartés par manque de compétence actuelle. |
| Gouvernance | Solo / petit noyau au démarrage, recherche de portage institutionnel (ADULLACT, DINUM, association d'élus, etc.) une fois un POC solide | Permet d'avancer vite sans dépendre d'un accord collectif, tout en gardant l'objectif de légitimité institutionnelle. |
| Licence | **AGPL-3.0** | Empêche qu'un tiers referme une version modifiée ou hébergée en SaaS, et garantit que toute évolution du projet reste accessible à la communauté. |
| Nom du projet | **StationPilot** | Retenu, disponible. |
| Inventaire | **Hors périmètre de StationPilot** : GLPI Agent déjà déployé en parallèle sur le parc | Pas de redondance à construire ; StationPilot pourra, si besoin, consommer les données via l'API GLPI plutôt que redévelopper une brique d'inventaire. |

## 2. Principes directeurs

- **Souveraineté réelle, pas seulement « open source »** : auto-hébergeable, pas de dépendance à un cloud ou service tiers non européen, auditable de bout en bout.
- **Architecture modulaire** : chaque module (imaging, déploiement logiciel, prise de main, migration de profils) doit pouvoir être développé, testé et remplacé indépendamment, derrière une interface commune (inventaire, authentification, journalisation, console d'administration).
- **Réutiliser avant de réécrire**, mais documenter dès le départ les critères qui justifieraient une réécriture (licence incompatible AGPL, gouvernance fermée, dépendance non européenne, dette technique).
- **Cibler l'existant réel des collectivités** : postes Windows joints à un AD on-prem ou Azure AD/Entra ID, GPO, parfois flotte hétérogène de versions Windows.

## 3. Phase 0 — Analyse de l'existant (en cours)

Objectif : cartographier précisément les briques candidates avant d'écrire la moindre ligne de code d'architecture.

- [x] Migration de profils lors d'un changement de domaine : aucun outil totalement libre et confirmé n'a été identifié à ce jour pour l'orchestration complète (mapping compte → compte, ré-ACLage, rollback) ; ce module devra probablement être développé en interne au-dessus de briques bas niveau libres restant à identifier en phase 0.
- [x] Prise de main à distance libre : RustDesk (AGPL-3.0, auto-hébergeable) retenu comme candidat prioritaire ; MeshCentral écarté par défaut (signalé non européen et parfois détourné en RAT), à ne pas exclure totalement mais à auditer plus strictement si reconsidéré.
- [x] Architecture RustDesk confirmée : deux exécutables auto-hébergeables via Docker, `hbbs` (serveur d'ID / rendez-vous) et `hbbr` (relais), chiffrement de bout en bout par clé publique. **Point d'attention** : la console web d'administration (port 21114) est réservée à la version **Pro propriétaire** — la version open source ne l'inclut pas. StationPilot devra développer sa propre console d'administration par-dessus `hbbs`/`hbbr`.
- [x] **Imaging** : deux candidats libres identifiés. **FOG Project** (GPLv3, orienté gestion de parc Windows avec sysprep/snapins/pilotes, mais compatibilité Windows 10 non officiellement confirmée dans la documentation du projet et Windows 11 non documenté à ce jour). **Clonezilla Server Edition** (GPL, clonage bas niveau multicast via DRBL, agnostique à l'OS mais moins orienté « gestion de parc » que FOG). Décision reportée à un POC réel sur un poste Windows 11 (voir §11).
- [x] Déploiement logiciel : **opsi (open pc server integration)** confirmé en **AGPLv3** pour ses composants principaux (serveur Linux + agent `opsi-client-agent` sur postes Windows 32/64 bits). Point de vigilance : le module de gestion des licences logicielles a pu, historiquement, être développé hors du cœur libre dans le cadre d'un financement dédié — à revérifier sur la documentation actuelle d'opsi.org avant de s'appuyer sur cette fonctionnalité. Compatibilité Windows 11/Server 2022+ non confirmée dans les sources trouvées.
- [x] Inventaire/supervision : **hors périmètre de StationPilot**. GLPI Agent (GPL-3.0-or-later, successeur actif de FusionInventory qui est lui abandonné depuis octobre 2022) est déjà déployé en parallèle sur le parc des collectivités cibles — StationPilot n'a pas à redévelopper cette brique, et pourra au besoin s'interfacer avec l'API GLPI plutôt que de la dupliquer.
- [x] Revue juridique/licences : confirmé pour les briques retenues — RustDesk server = AGPL-3.0 (source tierce, **à reconfirmer directement sur le dépôt GitHub officiel `rustdesk/rustdesk-server` avant tout usage en prod**), FOG Project = GPLv3, Clonezilla = GPL, opsi = AGPLv3, GLPI/GLPI Agent = GPL-3.0-or-later. Point de vigilance méthodologique : GLPI a dû changer sa propre licence (GPL-2.0-or-later → GPL-3.0-or-later) après un conflit avec du code AGPL repris de FusionInventory — preuve que la compatibilité entre licences de briques assemblées doit être vérifiée systématiquement, pas une seule fois.
- [ ] Revue sécurité/souveraineté : premier élément confirmé — le RGS s'applique aux collectivités territoriales, mais la qualification **SecNumCloud n'est obligatoire que pour les administrations de l'État** (doctrine « Cloud au centre », 2023) ; pour les collectivités, ce n'est à ce jour qu'une recommandation (CSNP), pas une obligation juridique. StationPilot n'a donc pas d'obligation de certification immédiate, mais l'auto-hébergement en France/UE reste la bonne pratique à documenter. Reste à approfondir : exigences CNIL/RGPD spécifiques à la prise de main à distance et à l'inventaire de postes.
- [ ] Benchmark utilisateurs : toujours ouvert (nécessite un contact direct avec des collectivités/DSI, hors portée d'une recherche documentaire).
- [x] Historique du déploiement d'image dans l'écosystème des outils de référence du secteur (pour mémoire, aucun code réutilisable) : la version réellement libre (GPLv3, dépôt GitHub désormais archivé en lecture seule) de l'outil de déploiement logiciel historique du secteur public français, figée à une version dite « Community » non maintenue par son éditeur depuis le 30/04/2022, ne contenait **aucune fonctionnalité de déploiement d'image système** — elle se limitait au déploiement de paquets logiciels. Le déploiement d'OS n'est apparu que dans une version ultérieure (2.2), introduite en même temps qu'un repositionnement commercial en deux éditions : une payante (Enterprise) et une gratuite mais dont le statut open source n'est pas confirmé (« Discovery », basée sur la Community mais pas republiée sous licence libre). **Conclusion : aucun code de cette lignée n'est légalement réutilisable pour notre module imaging**, ce qui confirme le choix porté sur FOG Project / Clonezilla SE plutôt que sur une réutilisation de cet écosystème.

**Livrable de fin de phase 0** : document d'architecture cible (choix de briques par module, schéma agent/serveur, format de paquet, modèle de données inventaire) + liste des risques de licence/gouvernance par brique retenue.

## 4. Phase 1 — Socle technique commun

- Serveur central (API + base de données) : authentification admin, gestion des machines/groupes, journalisation des actions (traçabilité = important pour le secteur public).
- Agent Windows unique, modulaire : un seul agent installé sur le poste, qui expose des capacités activables par module (imaging au boot/PXE à part, déploiement logiciel, prise de main, migration de profil).
- Console d'administration web minimale (lister les postes, lancer une action, voir le statut). Pas de brique d'inventaire à développer : GLPI Agent est déjà déployé en parallèle sur le parc (voir §1) ; une intégration via l'API GLPI pourra être envisagée plus tard si besoin.
- Intégration Active Directory de base (lecture des OU/groupes pour cibler les déploiements).

## 5. Phase 2 — Module Imaging

- PXE boot + environnement de pré-installation libre (ex. environnement live Linux utilisé par Clonezilla/FOG Project) pour déployer une image Windows de référence, sans dépendance à un outillage propriétaire.
- Création/capture d'image de référence (master) avec personnalisation (pilotes, applications de base).
- Déploiement en masse (multicast ou HTTP selon la brique retenue en phase 0).
- Scénarios cibles collectivité : renouvellement de parc, remasterisation annuelle, salle informatique (écoles/médiathèques).

## 6. Phase 3 — Module Déploiement de logiciels

- Format de paquet applicatif (à définir : réutiliser un format existant compatible AGPL, ou en définir un simple).
- Dépôt de paquets, ciblage par groupe/OU, planification, rapports de déploiement.
- Gestion des mises à jour Windows (optionnel en V1, peut être une itération ultérieure).

## 7. Phase 4 — Module Prise de main à distance

- Intégration/durcissement de RustDesk auto-hébergé (serveur relais + signalisation en interne, pas de passage par les serveurs publics RustDesk).
- Authentification centralisée (SSO avec la console d'administration plutôt que mot de passe par poste).
- Traçabilité des sessions de prise en main (qui s'est connecté, quand, sur quel poste) — exigence de confiance pour le secteur public.

## 8. Phase 5 — Module Migration de profils utilisateurs

- Cas d'usage : migration de domaine AD (fusion de collectivités, changement de prestataire d'annuaire), ré-ACLage des profils locaux/fichiers après changement de SID utilisateur.
- Identifier en phase 0 une brique libre bas niveau pour la gestion des ACL Windows (à défaut, développer ce module en interne) ; construire l'orchestration (mapping ancien compte → nouveau compte, prévisualisation, rollback) par-dessus.
- Brique la plus sensible (opérations destructrices sur les profils) : prévoir dry-run et sauvegarde/rollback obligatoires avant toute action en prod.

## 9. Phase 6 — Structuration et portage institutionnel

- Une fois un POC solide sur au moins 2 modules (a minima imaging + déploiement logiciel), engager la discussion avec l'ADULLACT (Comptoir du Libre) et/ou la DINUM pour un portage institutionnel.
- Évaluer une présence candidate sur le Comptoir du Libre dès que le projet est utilisable en conditions réelles, même en version bêta.
- Documenter un mode de gouvernance ouvert (contribution, prise de décision) pour préparer l'élargissement au-delà du porteur initial.

## 10. Risques et points de vigilance

- **Dérive de licence d'une brique amont** : une brique réutilisée (ex. RustDesk) pourrait changer de modèle économique/licence à l'avenir — prévoir dans l'architecture la possibilité de la remplacer sans tout casser (principe déjà acté en §1).
- **MeshCentral** : écarté par défaut pour la brique prise de main (signalé non européen + abus en RAT par des attaquants), à ne reconsidérer qu'après audit sécurité approfondi si RustDesk s'avère insuffisant.
- **Brique imaging** : plus complexe techniquement que prévu si le choix se porte sur un environnement PXE à maintenir soi-même ; prévoir du temps dédié en phase 0 pour un vrai POC avant de s'engager.
- **Compatibilité Windows 11/versions récentes non confirmée** : ni FOG Project ni opsi n'ont de compatibilité Windows 11/Server 2022+ documentée dans les sources trouvées à ce jour — à valider par un POC réel avant tout engagement d'architecture sur l'une de ces briques.
- **Console RustDesk absente en version libre** : la version open source de RustDesk Server n'inclut pas de console web (réservée à la version Pro) — prévoir le développement d'une console d'administration maison pour piloter `hbbs`/`hbbr`.
- **Module migration de profils** : opérations à fort risque (perte de profils utilisateurs) — ne jamais livrer sans dry-run/rollback, même en MVP.
- **Charge solo** : 4 modules + socle commun est un périmètre large pour un porteur seul ; accepter un MVP volontairement restreint par module plutôt que de viser l'exhaustivité dès la V1.

## 11. Décisions encore ouvertes (à trancher en phase 0/1)

- Choix définitif de la brique imaging : **FOG Project** (plus orienté gestion de parc Windows) vs **Clonezilla Server Edition** (plus robuste/bas niveau, agnostique à l'OS) — à trancher après un POC réel de déploiement d'une image Windows 11 sur les deux candidats.
- Confirmer la compatibilité Windows 11/Server 2022+ de FOG Project et d'opsi directement auprès des projets (dépôts GitHub / documentation officielle, pas seulement des sources tierces).
- Revérifier le statut actuel du module de gestion des licences logicielles d'opsi (libre vs. développement co-financé non publié) sur opsi.org avant de s'appuyer sur cette fonctionnalité.
- Confirmation du choix Python pour l'agent/serveur après un premier POC technique (notamment sur l'imaging, le plus exigeant côté bas niveau Windows).
- Modalités précises de packaging/format de paquet pour le module déploiement logiciel.
