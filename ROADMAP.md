# Roadmap — StationPilot

> Suite libre de gestion de parc Windows pour les collectivités territoriales :
> création & déploiement d'images système, déploiement de logiciels,
> prise de main à distance, migration de profils utilisateurs (type ReACL).
> Objectif : offrir une alternative souveraine, auditable et pérenne aux
> solutions propriétaires ou aux briques libres dont la gouvernance se ferme
> (cf. évolution de WAPT Community/Enterprise).

## 1. Décisions actées

| Sujet | Décision | Justification |
|---|---|---|
| Ordre des modules | 1. Imaging (type MDT) → 2. Déploiement logiciel (type WAPT) → 3. Prise de main à distance → 4. Migration de profils (type ReACL) | L'imaging est jugé le besoin le plus structurant pour démarrer un parc, avant même le déploiement applicatif. |
| Stratégie technique | S'appuyer sur des briques open source existantes pour un MVP rapide, dans une architecture modulaire qui permette de réécrire une brique plus tard si elle s'avère inadaptée (licence, sécurité, maintenabilité) | Vitesse pour le MVP sans s'enfermer dans un choix technique irréversible. |
| Périmètre OS (V1) | Postes clients Windows uniquement | Cible = parc Windows des collectivités ; cohérent avec MDT/WAPT historiques. |
| Stack technique (socle agent/serveur) | **Python**, à confirmer/creuser en phase 0/1 | Seul langage maîtrisé par le porteur du projet parmi les options envisagées ; écosystème riche pour AD/ACL (pywin32) et pour du scripting Windows. C#/.NET écartés par manque de compétence actuelle. |
| Gouvernance | Solo / petit noyau au démarrage, recherche de portage institutionnel (ADULLACT, DINUM, association d'élus, etc.) une fois un POC solide | Permet d'avancer vite sans dépendre d'un accord collectif, tout en gardant l'objectif de légitimité institutionnelle. |
| Licence | **AGPL-3.0** | Empêche qu'un tiers referme une version modifiée ou hébergée en SaaS (le risque exact qui s'est matérialisé avec WAPT Enterprise/Community). |
| Nom du projet | **StationPilot** | Retenu, disponible. |

## 2. Principes directeurs

- **Souveraineté réelle, pas seulement « open source »** : auto-hébergeable, pas de dépendance à un cloud ou service tiers non européen, auditable de bout en bout.
- **Architecture modulaire** : chaque module (imaging, déploiement logiciel, prise de main, migration de profils) doit pouvoir être développé, testé et remplacé indépendamment, derrière une interface commune (inventaire, authentification, journalisation, console d'administration).
- **Réutiliser avant de réécrire**, mais documenter dès le départ les critères qui justifieraient une réécriture (licence incompatible AGPL, gouvernance fermée, dépendance non européenne, dette technique).
- **Cibler l'existant réel des collectivités** : postes Windows joints à un AD on-prem ou Azure AD/Entra ID, GPO, parfois flotte hétérogène de versions Windows.

## 3. Phase 0 — Analyse de l'existant (en cours)

Objectif : cartographier précisément les briques candidates avant d'écrire la moindre ligne de code d'architecture.

- [x] WAPT (Tranquil IT) : fonctionnalités, statut de licence Community vs Enterprise (point flou à reconfirmer sur la documentation officielle avant toute décision de fork).
- [x] ReACL : pas d'équivalent libre identifié (le terme est associé à Quest Migrator Pro, propriétaire) ; script communautaire SetACL comme seule base libre connue.
- [x] Prise de main à distance libre : RustDesk (AGPL-3.0, auto-hébergeable) retenu comme candidat prioritaire ; MeshCentral écarté par défaut (signalé non européen et parfois détourné en RAT), à ne pas exclure totalement mais à auditer plus strictement si reconsidéré.
- [ ] **Imaging type MDT** (nouvelle brique, à auditer) : étudier les candidats libres — MDT/WinPE (Microsoft, pas libre mais référence fonctionnelle), **FOG Project**, **Clonezilla / Clonezilla Server Edition**, **WAPT WADS** (fonctionnalité Enterprise, donc non réutilisable telle quelle), OSDCloud (communauté PowerShell), Windows ADK/WinPE comme socle technique commun possible.
- [ ] Déploiement logiciel : confirmer précisément ce qui est encore librement réutilisable dans l'écosystème WAPT (packages, format de paquet, client) vs ce qui est devenu propriétaire ; étudier aussi **opsi (open pc server integration)** comme alternative allemande déjà utilisée dans le secteur public.
- [ ] Inventaire/supervision : étudier **GLPI + FusionInventory/Glpi-Agent** comme brique d'inventaire potentiellement réutilisable et déjà répandue dans les collectivités (via le Comptoir du Libre).
- [ ] Revue juridique/licences : vérifier la compatibilité AGPL-3.0 de chaque brique candidate (RustDesk = AGPL ✅, Clonezilla = GPL ✅ a priori, FOG = GPL ✅ a priori — à confirmer précisément module par module).
- [ ] Revue sécurité/souveraineté : exigences RGS/ANSSI, hébergement en France/UE, dépendances tierces (CDN, télémétrie) à bannir dès l'architecture.
- [ ] Benchmark utilisateurs : identifier 2-3 collectivités ou DSI publiques prêtes à discuter de leurs besoins réels (pilote potentiel) — utile pour prioriser les détails fonctionnels du MVP.

**Livrable de fin de phase 0** : document d'architecture cible (choix de briques par module, schéma agent/serveur, format de paquet, modèle de données inventaire) + liste des risques de licence/gouvernance par brique retenue.

## 4. Phase 1 — Socle technique commun

- Serveur central (API + base de données) : authentification admin, gestion des machines/groupes, journalisation des actions (traçabilité = important pour le secteur public).
- Agent Windows unique, modulaire : un seul agent installé sur le poste, qui expose des capacités activables par module (imaging au boot/PXE à part, déploiement logiciel, prise de main, migration de profil).
- Inventaire matériel/logiciel de base (probablement via GLPI-Agent ou équivalent, à trancher en phase 0).
- Console d'administration web minimale (lister les postes, lancer une action, voir le statut).
- Intégration Active Directory de base (lecture des OU/groupes pour cibler les déploiements).

## 5. Phase 2 — Module Imaging (type MDT)

- PXE boot + environnement de pré-installation (WinPE ou équivalent) pour déployer une image Windows de référence.
- Création/capture d'image de référence (master) avec personnalisation (pilotes, applications de base).
- Déploiement en masse (multicast ou HTTP selon la brique retenue en phase 0).
- Scénarios cibles collectivité : renouvellement de parc, remasterisation annuelle, salle informatique (écoles/médiathèques).

## 6. Phase 3 — Module Déploiement de logiciels (type WAPT)

- Format de paquet applicatif (à définir : réutiliser un format existant compatible AGPL, ou en définir un simple).
- Dépôt de paquets, ciblage par groupe/OU, planification, rapports de déploiement.
- Gestion des mises à jour Windows (optionnel en V1, peut être une itération ultérieure).

## 7. Phase 4 — Module Prise de main à distance

- Intégration/durcissement de RustDesk auto-hébergé (serveur relais + signalisation en interne, pas de passage par les serveurs publics RustDesk).
- Authentification centralisée (SSO avec la console d'administration plutôt que mot de passe par poste).
- Traçabilité des sessions de prise en main (qui s'est connecté, quand, sur quel poste) — exigence de confiance pour le secteur public.

## 8. Phase 5 — Module Migration de profils (type ReACL)

- Cas d'usage : migration de domaine AD (fusion de collectivités, changement de prestataire d'annuaire), ré-ACLage des profils locaux/fichiers après changement de SID utilisateur.
- S'appuyer sur SetACL ou équivalent libre comme brique bas niveau, construire l'orchestration (mapping ancien compte → nouveau compte, prévisualisation, rollback) par-dessus.
- Brique la plus sensible (opérations destructrices sur les profils) : prévoir dry-run et sauvegarde/rollback obligatoires avant toute action en prod.

## 9. Phase 6 — Structuration et portage institutionnel

- Une fois un POC solide sur au moins 2 modules (a minima imaging + déploiement logiciel), engager la discussion avec l'ADULLACT (Comptoir du Libre) et/ou la DINUM pour un portage institutionnel.
- Évaluer une présence candidate sur le Comptoir du Libre dès que le projet est utilisable en conditions réelles, même en version bêta.
- Documenter un mode de gouvernance ouvert (contribution, prise de décision) pour préparer l'élargissement au-delà du porteur initial.

## 10. Risques et points de vigilance

- **Dérive de licence d'une brique amont** : une brique réutilisée (ex. RustDesk) pourrait changer de modèle économique/licence à l'avenir, comme WAPT — prévoir dans l'architecture la possibilité de la remplacer sans tout casser (principe déjà acté en §1).
- **MeshCentral** : écarté par défaut pour la brique prise de main (signalé non européen + abus en RAT par des attaquants), à ne reconsidérer qu'après audit sécurité approfondi si RustDesk s'avère insuffisant.
- **Brique imaging** : plus complexe techniquement que prévu si le choix se porte sur un environnement PXE/WinPE à maintenir soi-même ; prévoir du temps dédié en phase 0 pour un vrai POC avant de s'engager.
- **Module ReACL** : opérations à fort risque (perte de profils utilisateurs) — ne jamais livrer sans dry-run/rollback, même en MVP.
- **Charge solo** : 4 modules + socle commun est un périmètre large pour un porteur seul ; accepter un MVP volontairement restreint par module plutôt que de viser l'exhaustivité dès la V1.

## 11. Décisions encore ouvertes (à trancher en phase 0/1)

- Choix définitif de la brique imaging (FOG Project vs Clonezilla vs autre).
- Confirmation du choix Python pour l'agent/serveur après un premier POC technique (notamment sur l'imaging, le plus exigeant côté bas niveau Windows).
- Choix de la brique d'inventaire (GLPI-Agent vs développement interne minimal).
- Modalités précises de packaging/format de paquet pour le module déploiement logiciel.
- Nom de compte/organisation GitHub et paramètres du dépôt public (visibilité, branche par défaut, etc.).
