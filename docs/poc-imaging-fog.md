# POC — Serveur FOG : capture et déploiement d'une image Windows 11

> Objectif de ce POC : lever l'incertitude listée dans [ROADMAP.md](../ROADMAP.md) §11 —
> confirmer (ou infirmer) que FOG Project fonctionne correctement sur une cible
> **Windows 11, UEFI, Secure Boot** — avant tout engagement d'architecture sur
> le module Imaging.

## 1. Prérequis — environnement de lab isolé

**Ne jamais faire ce POC sur un réseau de production.** FOG a besoin de répondre
aux requêtes PXE/DHCP, ce qui peut entrer en conflit avec le serveur DHCP/AD
existant d'une collectivité.

- Un réseau/VLAN de test isolé (ou un commutateur virtuel dédié dans un
  hyperviseur), sans accès au DHCP de production.
- **1 VM serveur FOG** : Ubuntu Server 22.04 ou 24.04 LTS (distributions
  officiellement supportées par FOG), 2 vCPU / 4 Go RAM / 60 Go disque minimum
  (le stockage des images dépendra de leur taille réelle).
- **1 VM « machine de référence »** : Windows 11 (23H2 ou plus récent), en
  **UEFI** (pas de BIOS legacy — Windows 11 l'exige), Secure Boot activable/
  désactivable, TPM 2.0 virtuel activé.
- **1 à 2 VM « cibles »** supplémentaires pour tester le déploiement, idéalement
  avec un profil matériel légèrement différent de la référence (pour tester
  l'injection de pilotes).
- Accès à la console hyperviseur pour forcer le boot PXE/réseau sur les VM de
  test (ou configuration du firmware UEFI virtuel en boot réseau prioritaire).

## 2. Installation du serveur FOG

```bash
sudo apt update && sudo apt upgrade -y
git clone https://github.com/FOGProject/fogproject.git
cd fogproject/bin
sudo ./installfog.sh
```

Pendant l'installeur interactif :

- **Mode d'installation** : « Normal Server » (installation complète : serveur
  web, DHCP/TFTP, base de données).
- **Interface réseau** : sélectionner celle du réseau de lab isolé.
- **Rôle DHCP** — point important pour la suite du projet StationPilot :
  - Sur ce POC isolé, laisser FOG gérer le DHCP est le plus simple.
  - **Mais noter dès maintenant** : dans une collectivité réelle, le DHCP est
    quasiment toujours déjà géré par un contrôleur de domaine ou un routeur/
    firewall. Il faudra installer FOG en **mode « DHCP non géré » / proxyDHCP**
    (FOG répond uniquement aux requêtes PXE via les options 66/67, le DHCP
    existant continue de distribuer les IP). À tester dans une itération
    ultérieure du POC, sur un second scénario.
- Laisser l'installeur créer la base de données et démarrer les services.

Vérifier à la fin : la console web est accessible sur `https://<ip-serveur-fog>/fog/management/`.

## 3. Préparation de la machine de référence Windows 11

1. Installer Windows 11 normalement sur la VM de référence (UEFI, disque GPT).
2. Faire toutes les mises à jour Windows, installer les applications de base
   que doit contenir l'image master.
3. **Installer l'agent FOG Client** (téléchargeable depuis la console FOG,
   menu *FOG Configuration > FOG Client Installer* ou directement depuis
   `https://<ip-serveur-fog>/fog/client/`), en indiquant l'adresse du serveur
   FOG. Cela enregistre automatiquement l'hôte dans l'inventaire FOG.
4. Dans la console FOG (*Host Management*), vérifier que la machine de
   référence apparaît, et lui associer une définition d'image (*Basic Disk
   Deployment* pour une seule partition redimensionnable — recommandé pour
   Windows 10/11 pour gérer les disques cibles de tailles différentes).
5. **Généraliser l'image avant capture** : lancer `sysprep` avec un fichier de
   réponses (`unattend.xml`) en mode `/generalize /oobe /shutdown`.
   - C'est une étape **obligatoire** : sans sysprep, tous les postes déployés
     partageraient le même SID machine et le même identifiant d'installation
     Windows, ce qui pose des problèmes AD et de licence à l'échelle d'un parc.
   - La VM s'éteint à la fin du sysprep — ne pas la redémarrer dans Windows
     après ça, sous peine de devoir recommencer.

## 4. Capture de l'image

1. Dans la console FOG, sur la fiche de l'hôte de référence : *Task Scheduler
   > Capture > Start Task* (ou depuis le menu PXE directement).
2. Démarrer la VM en **boot réseau (PXE/UEFI)** — elle doit afficher le menu
   iPXE de FOG, puis entrer automatiquement dans l'environnement de capture
   (Linux/partclone) grâce à la tâche planifiée.
3. **Point d'attention Windows 11 n°1 — Secure Boot** : l'environnement de
   capture de FOG (iPXE + noyau Linux) n'est généralement pas signé pour le
   Secure Boot de Windows 11. Il faut **désactiver Secure Boot dans le
   firmware UEFI de la VM** avant le boot PXE pour ce POC. À noter comme
   limitation dans la roadmap si confirmé : il faudra documenter une procédure
   de réactivation du Secure Boot après déploiement (ou étudier un shim signé),
   point sensible vis-à-vis des recommandations ANSSI sur le Secure Boot.
4. Laisser la capture se terminer ; l'image est stockée sur le serveur FOG
   (chemin visible dans *Image Management*).

## 5. Déploiement sur une machine cible

1. Enregistrer la/les VM(s) cible(s) dans FOG (soit en les bootant en PXE pour
   un enregistrement rapide *Quick Registration/Full Registration and
   Inventory*, soit en pré-créant l'hôte manuellement par adresse MAC).
2. Associer l'image capturée à chaque hôte cible, puis planifier une tâche de
   déploiement (*Task Scheduler > Deploy*).
3. Démarrer la cible en boot réseau (Secure Boot désactivé pour ce POC, comme
   pour la capture).
4. **Point d'attention Windows 11 n°2 — matériel différent** : si la cible a un
   chipset/contrôleur de stockage différent de la référence, le premier
   démarrage après déploiement peut échouer (écran bleu, pilote de stockage
   manquant) car Windows n'a pas les bons pilotes génériques pour ce matériel.
   FOG propose une gestion de pilotes par modèle de machine (*Host Management
   > Associated Drivers* / fonctionnalité « Host Driver Management ») à tester
   si ce cas se présente.
5. Au premier démarrage après déploiement, Windows doit repasser par le mini-
   setup OOBE (conséquence du sysprep) — vérifier qu'il se déroule normalement
   (ou automatiquement si un `unattend.xml` de déploiement a été fourni).

## 6. Critères de validation du POC

À documenter dans ce fichier (ou dans une note de suivi) une fois le POC joué :

- [ ] Capture réussie d'une image Windows 11 UEFI/GPT.
- [ ] Déploiement réussi sur au moins une machine cible, démarrage OOBE propre.
- [ ] Comportement avec Secure Boot activé vs désactivé (documenter précisément
      ce qui bloque si Secure Boot reste activé pendant le PXE).
- [ ] Test du mode proxyDHCP (FOG sans gérer le DHCP) — pertinent pour un
      déploiement réel en collectivité où le DHCP existe déjà.
- [ ] Temps de capture / déploiement observés (pour dimensionner les attentes
      côté collectivité : combien de postes en combien de temps).
- [ ] Comportement sur matériel hétérogène (test de l'injection de pilotes).

**Décision à l'issue du POC** : si ces points sont validés sans blocage majeur,
FOG Project est confirmé comme moteur du module Imaging (cf. ROADMAP.md §11).
En cas de blocage dur sur Secure Boot ou sur un point structurant, basculer
l'évaluation vers Clonezilla/partclone en solution de repli.

## Notes / limites connues à garder à l'esprit

- FOG est un outil tiers réutilisé « en coulisses » : à terme, StationPilot ne
  doit pas exposer directement la console FOG à l'administrateur final — elle
  sert ici à valider le moteur, pas à devenir l'interface définitive (cf.
  ROADMAP.md §1, principe du « un seul agent/une seule console »).
- La gestion des mises à jour Windows (WSUS ou équivalent) est un besoin
  distinct, traité dans le module Déploiement de logiciels (cf. ROADMAP.md
  §6), pas dans ce POC imaging.
