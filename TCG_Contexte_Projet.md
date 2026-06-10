# Contexte Projet — Le Gardien du TCG
> Document à coller en premier message dans chaque nouvelle session Claude.
> **Dernière mise à jour : 10/06/2026 — session #21** · **Versions actuelles : Sales 1.15.0 / Master 1.16.0 / Intake 1.4.4 / Integrator 0.4.2**

---

## 0. Instructions permanentes pour Claude

- **Tout au long de la discussion** : mettre à jour ce document en arrière-plan au fil des avancées — ajouter les nouveaux milestones complétés et les nouvelles idées à garder pour plus tard, retirer du backlog les améliorations validées.
- **Au début de chaque nouvelle discussion** : lire les fichiers HTML (`tcg-sales.html`, `tcg-master.html`, `tcg-intake.html`, `tcg-integrator.html`) pour assimiler l'intégralité du code, puis faire une passe de revue systématique — triple-check à la recherche de bugs, repérage des incohérences et du code mort, analyse des opportunités d'amélioration de **performance** (en convention/événement, la **réactivité** est critique). Remonter les trouvailles avant de coder.
- **En début de chaque session**, lorsqu'une étape de développement est identifiée : indiquer quel modèle Claude est le plus adapté (Haiku / Sonnet / Opus) avec une justification courte.
- **À chaque livraison de fichier** : bumper systématiquement la version avant de présenter le fichier. Ne jamais livrer deux fichiers successifs avec le même numéro. Quand deux apps sont modifiées ensemble pour une même fonctionnalité, elles partagent le même numéro (mapping cohérent Sales↔Master). **Règle de versioning** : **patch** (x.x.+1) = fix mineur / UX / cosmétique · **minor** (x.+1.0) = nouvelle feature ou fix fonctionnel significatif · **major** (+1.0.0) = refonte d'architecture ou breaking change.
- **À la fin de chaque session, AVANT de mettre à jour ce document** : toujours demander **de quel numéro de session** il s'agissait.
- **Quand il est opportun d'ouvrir une nouvelle discussion** (contexte trop long, session de travail cohérente terminée) : prévenir, mettre à jour ce document une dernière fois, puis proposer de basculer.
- **Vigilance caractères spéciaux** (locale belge) lors du traitement des données Google Sheets : virgules (séparateur décimal vs milliers), apostrophes (droite `'` U+0027 vs courbe `'` U+2019 insérée souvent par Sheets), accents — sources fréquentes de bugs silencieux (`find()` qui renvoie -1, `parseFloat` qui interprète mal). Toujours considérer ces cas lors des comparaisons de strings et du parsing de nombres.
- Ces instructions s'appliquent automatiquement à chaque session — inutile de les rappeler.

---

## 1. À faire / En attente

### 🥇 Priorité #1 prochaine session — Refresh GIS non silencieux
L'auth Google Identity Services (#13, 1.12.0) **a été testée** : la connexion et le fonctionnement nominal sont **OK**. **MAIS** le **refresh au bout d'~1 h n'est PAS silencieux** : un **popup s'ouvre et demande de se reconnecter**. Le `requestAccessToken({prompt:''})` programmé avant expiration ne renouvelle pas le jeton de façon transparente comme prévu. → **À corriger en priorité** (Sales + Master) : objectif = renouvellement réellement silencieux, sans interruption de session en convention. *(Analyse #20 : sur iPhone le refresh silencieux GIS est structurellement peu fiable — iframe cookies tiers bloquée par WebKit. Pistes : reconnexion maîtrisée au tap plutôt que refresh proactif ; à terme passerelle Apps Script #25.)*

### 🎨 Améliorations esthétiques (toutes apps — ✅ LIVRÉES en #22)
Cosmétiques pures, aucun impact métier. **Faites le 10/06/2026** (décidées en #21). ⚠️ **NON testées navigateur** (à vérifier au passage des tests du 13, notamment l'en-tête intake/integrator sur petit écran — `flex-wrap` ajouté pour que la version repasse sous le titre si la place manque).

| App | Amélioration | État |
|-----|-------------|------|
| **tcg-master** | Panneaux « Journal des ventes » et « Anomalies » → bouton « ▸ Détails / ▾ Masquer », **défaut = caché** (`card-body collapsed` + helper `toggleCardDetails()`) | ✅ |
| **tcg-master** | Logo agrandi 54→84 px (60 px en mobile) | ✅ |
| **tcg-sales** | « TCG SALES » plus gros (`clamp(2rem,12vw,3.2rem)`), prend toute la largeur jusqu'à droite ; version dessous alignée à droite (inchangée) | ✅ |
| **tcg-intake** | Logo société ajouté ; version (`#appVer`) collée à droite du titre « TCG INTAKE » ; « by Cassius » supprimé | ✅ |
| **tcg-integrator** | Logo société ajouté ; version (`#ver`) collée à droite du titre « TCG INTEGRATOR » *(le code disait bien « TCG INTEGRATOR », pas « TCG INTAKE » — pas de coquille à corriger)* ; « by Cassius » supprimé | ✅ |

### 🎯 Résolution de scan à 3 branches (cas 1 / cas 2)
Objectif : quand un vendeur scan un code, distinguer **3 issues** au lieu de « trouvé / non trouvé ».
- ✅ **Cas 2 — code totalement inconnu → ajout forcé (FAIT en #20, Sales+Master 1.15.0).** Modale « ❓ Code non reconnu » : libellé = code brut (modifiable), **prix obligatoire** → article ajouté en `source='inconnu'` (contour ambré + ❓). Part dans `ventes_detail` via l'outbox ; Master l'enregistre (`OK_UNKNOWN`) **et** écrit une anomalie **`SCAN_INCONNU`** (réconciliation le soir). Roll-back d'un inconnu = no-op stock.
- ⏳ **Cas 1 — article reconnu MAIS déjà vendu sur un autre canal → info + blocage (À FAIRE, jalon coordonné).** Aujourd'hui Sales ne peut pas distinguer « vendu ailleurs » de « inconnu » : `STOCK_PUBLIC` ne contient que le stock **disponible** → l'article vendu sur CardMarket en est simplement absent. **Décision de périmètre (validée #20)** : Master publiera dans `STOCK_PUBLIC` les **clés connues-mais-indisponibles qui portent un `idProduct`** (= cross-listées CardMarket : col L cartes / « CM » scellés) avec un **statut + nom de canal générique** (`CM` d'abord, extensible Vinted/Whatnot/FB plus tard sans réécrire Sales). Sales affichera alors « déjà vendu — listé sur : CardMarket → vente bloquée ». ⚠️ Ne PAS republier les ~7 757 cartes vendues : se limiter aux cross-listées indisponibles (ensemble borné). **Spec courte à rédiger avant de toucher Master.** *(À planifier après les améliorations esthétiques.)*


### 🟠 Backlog améliorations (items ouverts)
| # | Amélioration | Contexte |
|---|---|---|
| 1 | Auto-reload cache slave à la reconnexion | Cache à 0 après re-auth — à revérifier depuis l'auth GIS (#13) |
| 3 | Toggle « Sync actif sur ce device » | Risque double-sync si 2 masters ouverts (le verrou d'instance #11 couvre déjà l'essentiel) |
| 5 | Auto-création des onglets manquants | Master affiche une erreur si un onglet est absent |
| 8 | Table fournisseurs/vendeurs pros | Auto-déterminer TVA_Achat selon le vendeur |
| 11 | Uniformisation acheteurs (Levenshtein) | Descopé en #6 (modèle Master-autoritaire suffit). À ressortir si saisie acheteur libre multi-vendeurs un jour |
| 15 | [Master] Sections repliables + bandeau sync sticky | Replier les sections, garder le bandeau sync/heartbeat visible au scroll |
| 19 | [Master] Colonnes cartes dynamiques côté **écriture** | Transposer la détection par nom d'en-tête aux écritures (P/P:T/AF par lettre détectée) — chemin d'écriture, à faire avec test |
| 20 | [Sales+Master] Refonte « centimes entiers » | Acompte #10 fait (col K arrondie 2 déc.). Reste le gros : calculs internes en centimes-entiers partout (pervasif) |
| 22 | [Sécu R5] Réduire le scope OAuth | `auth/spreadsheets` = accès à TOUTES les Sheets → envisager `drive.file` + Google Picker (moindre privilège) |
| 24 | [Archi] Module commun `tcg-common.js` | Helpers dupliqués (`esc`, `safeForSheets`, `a1`, formateurs, `HttpError`…) → une source de vérité. ⚠️ passe de 3→4 fichiers (commun déployé à côté) ; diff des copies avant fusion. **Rangement de code, ≠ Option 3.** |
| 25 | [Archi] Option 3 — passerelle Apps Script | Web app Apps Script (exécute en tant que propriétaire) sert le stock filtré + reçoit les paniers → vendeurs sans jeton plein scope ni accès Drive. Ferait tomber R4/R5 côté vendeur. **Gros chantier.** |
| 26 | [Perf] Republier `STOCK_PUBLIC` après vente | Actuellement au rythme du cache Master (~5 min) ; republier juste après une vente appliquée pour une dispo quasi temps réel |

> **Faits (détail en §8/§9)** : #2 refresh token (auth GIS #13, ⚠️ voir Priorité #1) · #4 scanner caméra (#10) · #9 pastille cache (#10) · #10 boutons quantité (#10) · #12 rendu incrémental panier (#10) · #13 barres d'état (#10) · #14 feedback scan (#10) · #17 renommage onglet Intake (#12) · #18 dédup txId (#12) · #21 OAuth GIS (#13) · #23 CSP (#12) · #27 roll-back (#14) · #28 acheteur imposé (#12). **SRI : N/A** (aucun `<script src>` tiers).

### ⏳ Autres tâches ouvertes
- **Mode vacances CardMarket (convention)** : processus opérationnel validé en #21 — **Manuel uniquement** (API CM fermée). Ordre obligatoire : ① mettre CM en mode vacances, ② reporter les dernières ventes CM non saisies dans la GSheet, ③ rafraîchir le cache Master. Master 1.16.0 embarque un popup checklist à l'activation de l'event + un rappel à la clôture. L'automatisation via l'API CM est dans le backlog #25 (passerelle Apps Script).
- **Write-back idProduct scellé** — non branché (l'intégrateur ne matche que les singles). Cible : onglet **`scelle_stock`** *(nom + casse confirmés en #22 ; ce n'est PAS `stock_scelle`)*, en-tête en **ligne 2**, **Key col C**, idProduct **col K « CM »** *(positions confirmées sur l'extract #22 ; ⚠️ le layout a bougé — nouvelle col `x` en A + col `License` en D — donc TOUJOURS résoudre par **libellé d'en-tête**, jamais par lettre figée)*.
- **Confirmer que la colonne `AA`** de `Scelle_Transactions` n'est **pas lue par la compta** (sinon changer `SEALED_TX_TXID_COL`).
- **Durcissement Option 1** : protéger les onglets `STOCK_PUBLIC` et `app_state` (seul le propriétaire édite) ; donner aux vendeurs leurs **propres comptes Google** sur EVENTS (Éditeur), pas un login master partagé.

### 🗓️ Jalons à venir
- **Simu maison — samedi 13 juin 2026 (répétition générale)** : test « pour du beurre » à domicile avec **2 amis** → **3 appareils, plusieurs vendeurs en même temps**, **ventes fictives**, sur un **fichier de stock DÉDIÉ** (surtout PAS la GSheet officielle). Objectif : éprouver le **process complet** en conditions réalistes. Scénarios à provoquer : conflits concurrents (2 vendeurs même carte / dernier scellé → 1 vente OK + 1 anomalie) ; roll-back dans ses 4 états (avant/après sync, carte + scellé) ; verrou d'instance (Master sur 2 appareils → « Forcer la reprise ») ; cycle de vie panier + acheteur ; réseau dégradé (couper le wifi en pleine vente → file offline → flush au retour, sans perte ni doublon). ⚠️ **Conflits attendus (pas des bugs)** : le stock vendeur vient du cache `STOCK_PUBLIC` (~5 min de fraîcheur) → 2 vendeurs peuvent scanner le même article ; c'est le Master qui tranche à la sync (anomalie `P_NON_BLANK` / `SEALED_OVERDRAW`).
- **Convention réelle (1res vraies ventes) : 21–22 juin 2026.** Tout 🔴/🟠 bloquant repéré le 13 doit être corrigé entre le 14 et le 20 juin.

### 🔑 Cartographie EAN du scellé (spec figée en #22 — À CODER)
Séance de scan pour associer un vrai **code EAN** à chaque article scellé en stock, et même procédure à chaque nouvel entrant. **Spécification complète arrêtée en #22** (conception validée point par point). À implémenter en **session de dev dédiée numérotée** (nouvelle vue Intake : scan + filtres cascade + dédup + batch d'écriture + détection de conflits — ampleur réelle, ne PAS greffer à chaud). Priorité : après les retombées des tests du 13 ; derrière la Priorité #1 (GIS) et la checklist #13.

- **Portée : `scelle_stock` UNIQUEMENT.** `scelle_transactions` = historique → **hors scope** (beaucoup d'objets n'y sont plus détenus, EAN physiques indisponibles ; remplir reviendrait à inventer des données). Confirmé en #22 : `scelle_stock` = stock vivant, `scelle_transactions` = historique.
- **Onglets : noms confirmés `scelle_stock` / `scelle_transactions`** (et non `stock_scelle`). Même gabarit (extract #22 : **38 colonnes**, en-tête en **ligne 2**, pas ligne 1).
- **Layout réel `scelle_stock` (extract 10/06/2026) — résolution par LIBELLÉ, jamais par lettre figée** (détection dynamique de la ligne d'en-tête = celle contenant `Origine`+`Key`, comme côté cartes) : col A=`x` (technique, vide) · **B=`Origine`** · **C=`Key`** (cible d'écriture EAN) · **D=`License`** *(libellé réel à l'anglaise, 1 seul « c » — nouvelle colonne)* · **E=`Code Série`** · **F=`Nom Série`** · **G=`Type_Item`** · **H=`Nom_Item`** · I=`Quantité` · … (vente Q→AH, stocks AI→AL).
- **Clé produit (regroupeur, PAS identifiant de ligne)** : `License + Code Série + Nom Série + Nom_Item`. Le nom est stable à travers les époques → discriminant fiable. `n.a.` (séries de coffrets) traité comme **valeur légitime**, pas comme un trou.
- **Filtres en cascade, dérivés dynamiquement des données** : License → Série (paire Code+Nom, restreinte à la licence) → `Type_Item` (restreint à la série). `Origine` = filtre additionnel.
- **Shortlist = produits DÉDUPLIQUÉS** (un objet acheté en plusieurs lots n'apparaît **qu'une fois**), avec nb de lignes + qté totale concernées, dépliage possible pour vérifier le détail. Sélection de **1 ou 2 produits** possible pour un même EAN (cas du misclic, assumé).
- **Scan** : `BarcodeDetector` réutilisé de Sales (+ repli saisie manuelle). Helpers `sheetsGet`/`sheetsBatchUpdate`/`a1()` déjà présents dans Intake.
- **Écriture DIFFÉRÉE** : matchs `{ean, produits[], lignesCibles[]}` accumulés en mémoire pendant la session ; **un seul batch** en clôture écrit l'EAN en col `Key` de toutes les lignes du/des produits choisis.
- **Conflits** : ligne portant **déjà un EAN différent** → signalée ligne par ligne pour arbitrage avant commit. EAN **identique** déjà présent = **pas un conflit** (cas normal). ⚠️ l'EAN n'est **jamais** une clé d'unicité de ligne — c'est une étiquette physique apposée.
- **Hors scope, noté pour plus tard** : automatisation du « cassage » display/coffret → répartition du prix d'achat au prorata sur les unités, et **rattachement de l'EAN des boosters à la ligne post-break** (pour qu'un scan booster en event identifie le bon produit). Géré **manuellement** pour l'instant.

> ⚠️ **Savoir terrain iOS (établi en #20)** : quand la douchette **Netum CS7501 est appairée en Bluetooth** sur l'iPhone, iOS la voit comme un **clavier physique** et **masque le clavier logiciel** sur TOUS les champs de saisie. Ce n'est pas un bug applicatif — aucun correctif code ne le change. Pour saisir un prix (édition de ligne, ± total, article inconnu) douchette connectée : **double-clic rapide sur la gâchette** du scanner → le clavier iOS réapparaît (fonction native du CS7501, spécifique à l'appairage BLE iOS). Quand la douchette est éteinte/déconnectée, le clavier s'affiche normalement. *(Piste d'amélioration future, non bloquante : pavé numérique intégré dans les modales prix/qté/total, rendu par l'app → saisie indépendante du clavier iOS, douchette connectée ou non.)*

---

## 2. Qui & Pourquoi

Vendeur TCG (Trading Card Game) en Belgique (Wallonie), sous le nom **Le Gardien du TCG**.

**Produits vendus :**
- Cartes TCG individuelles (Magic, Pokémon, etc.)
- Articles scellés : boosters, displays, coffrets, decks

**Canaux de vente :**
- Conventions (événements ponctuels, plusieurs vendeurs sur le stand)
- Boutique / vente comptoir (à venir — workflow dédié à construire)
- Ventes à domicile / RMP (rendez-vous privés, client nommé)
- **Canaux externes via `tcg-integrator`** (voir §6) : **CardMarket** (priorité 1, ~3 % frais pro, modèle catalogue+`idProduct`), puis envisagés **Whatnot** (enchères live), **Vinted** et **Marketplace Facebook** (annonces texte libre, pas de catalogue/ID produit → modèle de publication différent — cf. §6).

**Comptabilité belge :**
- Articles achetés chez un **pro** → TVA classique (Régime normal)
- Articles achetés chez un **particulier** (d'occasion) → TVA sur la marge : base taxable = Pv − Pa (nécessite de connaître le Pa exact du lot)
- Comptable externe qui reçoit les données calculées par le système

---

## 3. Architecture Technique

### Constantes clés
```javascript
const OAUTH_CLIENT_ID    = '774473792747-6ogjf5msksq995s0kssqo61b3co5rj96.apps.googleusercontent.com';
const CENTRAL_GSHEET_ID  = '1i6Ch5L9z1zQeye1BQ2mAfqSfxbrXijDJ4EX1isNbEKk';
const EVENTS_GSHEET_ID   = '1WxTPFgVUDGd94vWXAO6i4BSS0jePqiSwi890hb8c2_E';
const SHEET_CARDS        = 'cartes_stock_et_transactions'; // renommé en #8 (était 'Cartes_Stock_&_Transactions')
const SHEET_SEALED       = 'Scelle_Stock';
const SHEET_SEALED_TX    = 'Scelle_Transactions';
const SHEET_STOCK_PUBLIC = 'STOCK_PUBLIC';   // (#9) vue stock sûre publiée par Master dans EVENTS, lue par Sales
```
> **Helper `a1(sheet, rng)` (Sales + Master + Intake)** : encadre TOUJOURS le nom d'onglet de quotes simples (échappement A1). Toutes les plages passent par lui → robuste à un futur renommage avec caractère spécial.
> **(#9, Option 1)** Sales ne lit **plus** CENTRAL : Master y reste seul et publie un sous-ensemble sûr du stock dans `EVENTS › STOCK_PUBLIC`, que Sales consomme (voir « Communication inter-apps »).

### Google Sheets — base centrale (CENTRAL_GSHEET_ID)
- `cartes_stock_et_transactions` — stock + transactions cartes *(structure détaillée en Annexe A)*
- `Scelle_Stock` — état du stock scellé actuel *(structure détaillée en Annexe A)*
- `Scelle_Transactions` — historique de toutes les ventes/mouvements scellés
- Onglet compta — agrège toutes transactions, calcule TVA et marges (formules pointent vers `cartes_stock_et_transactions` et `Scelle_Transactions`)

### Google Sheets — événements (EVENTS_GSHEET_ID)
- Onglets temporaires créés à chaque convention : `ventes_detail`, `heartbeat`, etc. — archivés et vidés à la clôture (bouton 🏁).
- **`STOCK_PUBLIC` (#9, permanent)** — vue stock **sûre** publiée par Master (cartes + scellés dispo, **sans coûts/marges**) : `Key · Kind(card/sealed) · CodeSerie · NomSerie · Nom · Type · Etat · PrixVente · Qty · Reserved` (A:J) + liste TVA_Vente brute en col L. **Source du stock côté Sales.** À **protéger** (seul le propriétaire édite).
- **`anomalies`** — schéma 11 colonnes A:K : `Timestamp · TX_ID · Vendeur · Clé/Barcode · Nom article · Source · Type anomalie · Détails · Synced_From_Row · Resolved · Notes`. Écrit par Master (réconciliation) et par Sales (échec de vente : `SALES_NOT_SYNCED`, payload JSON dans `Notes`).

### Les apps
**TCG Sales** (`tcg-sales.html`) — app **vendeur**, ouverte sur le **téléphone** de chaque vendeur en convention (UI smartphone). Flow : OAuth → cache stock **depuis `STOCK_PUBLIC`** → scan barcode → panier → validation → **outbox idempotent** → écriture `ventes_detail`. **(#9)** N'accède **plus** à CENTRAL. **Outbox** : toute vente passe en file (`pending`) puis est envoyée ; idempotence par relecture de `ventes_detail!D` (txId) avant tout renvoi → ni doublon ni perte (« ≈ exactement une fois ») ; échec → `anomalies` puis dead-letter local.

**TCG Master** (`tcg-master.html`) — app **gérant**, ouverte sur iPad/ordi en convention. Flow : lit CENTRAL → **publie `STOCK_PUBLIC`** → écoute `ventes_detail` (heartbeat) → synchronise le stock CENTRAL → gère l'événement. Verrou d'instance lease/TTL (anti-double-master) + dashboard (KPI, donuts, faits marquants, CA/heure). `publishStockView()` republie la vue sûre à chaque `loadStockCache`.

**TCG Intake** (`tcg-intake.html`, ex-`tcg-converter.html`) — app **back-office d'entrée de stock**. Importe des CSV (catalogue / scraping), normalise (parser FR, locale belge, `esc()`/`safeForSheets()`) et **injecte dans CENTRAL** (`cartes_stock_et_transactions`) avec aperçu, backup (`SAFETY_GSHEET_ID`) et journal. Depuis #14, porte aussi le **write-back de l'`idProduct` CardMarket** (onglet « idProduct CM ») : charge le fichier `assignations_cm` produit par l'Integrator et réinjecte les idProduct dans le stock singles (aperçu obligatoire, backup, journal).

**TCG Integrator** (`tcg-integrator.html`) — app **back-office de sortie / publication multi-canal**. Lecture seule du stock CENTRAL. Matche les singles vers CardMarket **par numéro de collection** (catalogue TCG Power Tools) et produit deux fichiers : `assignations_cm` (→ Intake, write-back) et `cardmarket_upload_pt` (→ TCG PowerTools → CardMarket). Détail complet en §6.

### Communication inter-apps
Les apps ne se parlent **pas directement** — elles communiquent via Google Sheets :
- **Stock (Master → vendeurs)** : Master lit CENTRAL et publie une vue **sûre** dans `STOCK_PUBLIC` ; Sales la lit. Aucune donnée de coût n'atteint les vendeurs.
- **Ventes (vendeurs → Master)** : Sales écrit dans `ventes_detail` ; Master lit, résout par clé/`txId` et met à jour le stock CENTRAL.

### Modèle d'accès Google Drive (#9)
| | CENTRAL | EVENTS |
|---|---|---|
| Master (`legardiendutcg`) | Éditeur | Éditeur |
| Chaque vendeur (compte propre) | **Aucun accès** | **Éditeur** (dépose ses ventes ; lit `STOCK_PUBLIC`) |

> 2 GSheets en « Restreint ». `STOCK_PUBLIC` à **protéger** (la vue se ré-aligne à chaque publication). Vendeurs à ajouter avec leur **propre compte** (pas le login master partagé).

### Hébergement, OAuth & infrastructure
| Élément | Détail |
|---------|--------|
| Hébergement | **GitHub Pages** — déploiements illimités, gratuit, HTTPS natif, repo `legardiendutcg/Automation` (public) |
| URLs | `…/tcg-sales.html` · `…/tcg-master.html` · `…/tcg-intake.html` · `…/tcg-integrator.html` (base `https://legardiendutcg.github.io/Automation/`) |
| Déploiement | Add file → Upload files → écrase l'ancien → Commit. Rebuild auto ~1 min. **Ctrl+Shift+R** pour bypass cache navigateur. |
| OAuth | Projet GCP `774473792747` (libellé GCP historique « TCG Integrator »), client `774473792747-…rj96`, type Application Web. Auth = **flux GIS token** (popup, refresh — ⚠️ refresh non silencieux, cf. §1 Priorité #1) sur les 4 apps. |
| Scopes | `https://www.googleapis.com/auth/spreadsheets` (sensible, à déclarer dans Data Access) |
| URIs OAuth | Origine JS = `https://legardiendutcg.github.io` (les apps GIS n'ont plus besoin des redirect URIs de l'ancien flux implicite) |
| Audience | Mode **Testing** avec emails listés comme utilisateurs de test (évite la vérif Google complète) |
| APIs activées | Google Sheets API (à activer manuellement dans tout nouveau projet GCP) |
| Appareils | Sales sur téléphones vendeurs, Master sur iPad/ordi du gérant |

**Pièges OAuth à se rappeler** : *403 « insufficient authentication scopes »* → scope manquant OU token caché côté Google → ajouter le scope + révoquer l'app sur `myaccount.google.com/permissions` → re-consentement · *403 « API has not been used / disabled »* → activer Google Sheets API dans le projet · *« Access blocked: …has not completed the Google verification process »* → compte absent des Test users · *OAuth client supprimé* → Google supprime auto les clients inactifs 6 mois (restaurables 30 j via « Deleted credentials »).

---

## 4. Flow Scellé dans les Apps

### Cache (`loadStockCache`)
- **Master** : lit `Scelle_Stock!A:AG` + cartes (`C…:AF`) dans CENTRAL · détecte dynamiquement le header scellé (col C = "Key") · indexe par Key · filtre statut ≠ vendu/x ET qty > 0 · **capture le flag `reserved`** (TVA_Vente scellé AG / carte AF) · puis **publie `STOCK_PUBLIC`** (#9).
- **Sales (#9)** : lit `EVENTS › STOCK_PUBLIC!A2:L` (une seule lecture) et reconstruit ses caches cartes/scellés + la liste TVA. **Ne lit plus CENTRAL.** Verrou « réservée » et filtre TVA conservés (flag `Reserved` publié par Master).

### Vente scellée (Sales)
- Scanner EAN (`/^\d{8,14}$/`) → lookup `stockCacheSealed`
- **Verrou** : si l'article a `TVA_Vente = "réservée"` (lu col AG, normalisé accents), le scan est bloqué (pas d'ajout au panier)
- Sélecteur **TVA_Vente** (liste `Scelle_Transactions!AG8:AG23`, « réservée » exclue, défaut "Régime normal") + champ **Acheteur**. **Modèle 1.10.0** : si le **nom d'event** est publié (`app_state` col B) → **imposé en lecture seule** à tous les vendeurs (cohésion journée) ; sinon saisie vendeur **obligatoire**. *(⚠️ « imposé » validé en #11 mais **pas encore codé** : le code 1.10.0 livré a encore l'acheteur en défaut **écrasable** — à corriger.)*
- Ajout au panier → validation → ligne dans `ventes_detail` (A:N) avec `source='scelle'`, `TVA_Vente` (col M) et `Acheteur` (col N)

### Sync (Master — `applySealedSale`)
1. Lookup par Key dans `stockCacheSealed`
2. Lecture ligne complète : `Scelle_Stock!A{row}:AG{row}`
3. Vérification overdraw
4. Décrémentation qty dans `Scelle_Stock`
5. Calcul Pa proportionnel : `Pa_lot × (qtySold / currentQty)`
6. Écriture ligne dans `Scelle_Transactions` (colonnes A:AG)
   - Acheteur (col Q) = `ventes_detail.N` si rempli, sinon le **nom d'event** courant (défaut)
   - TVA_Achat (col AF) = lue depuis le lot source
   - **TVA_Vente (col AG)** = `ventes_detail.M` (défaut "Régime normal") — *écriture AG ajoutée en v1.6.0, elle était auparavant omise*
   - Facture de vente / Facture_Vente (cols U / AD) = **NON écrites** (depuis 1.10.0) : ces colonnes portent une **formule recopiée** depuis la ligne du dessus (`copyPaste`), elles ne sont pas dans `staticCols`. *(Avant 1.10.0, Master y écrivait `currentEventName` — retiré.)*

---

## 5. État Actuel des Apps

- **Sales 1.15.0** — stable, déployé en #20. Aucun changement en #21.
- **Master 1.16.0** — livré en #21. Popup checklist « mode vacances » à l'activation d'un event (transition vide→nom rempli uniquement) : ① mettre CM en mode vacances, ② reporter les ventes CM non saisies, ③ rafraîchir le cache Master — le nom d'event n'est activé qu'après validation humaine de la checklist. Rappel « réactiver les canaux » affiché après chaque clôture 🏁 réussie. Correction des étiquettes changelog (#14→#20 dans les entrées 1.15.0 et 1.14.x). ⚠️ **À déployer** (Sales reste en 1.15.0, pas de jalon coordonné).
- **Intake 1.4.4 / Integrator 0.4.2** — stables, déployés et **validés en navigateur de bout en bout** pour la chaîne CardMarket (06/06/2026) : matching par numéro → write-back de l'`idProduct` dans CENTRAL → export TCG PowerTools → **upload réel sur CardMarket réussi**. Détail en §6.

### 🟠 Points ouverts (non bloquants)
- Cache scellé indexé **par clé** (1 ligne / code-barres) alors que le modèle autorise **plusieurs lignes-lots** par code-barres (limitation pré-existante signalée).
- Clôture event 🏁 : chemin nominal **validé E2E (#9, réaffirmé #13)**. Les 2 cas-limites #12 (vidage si drain échoue ; archive limitée aux colonnes A→G) sont des durcissements **optionnels**, pas des bugs.

---

## 6. `tcg-integrator` — CardMarket & multi-canal 🎯

App de **sortie** (publication du stock vers des canaux externes), distincte de `tcg-intake` (entrée de stock). CardMarket est le 1ᵉʳ canal ; l'ossature est pensée pour accueillir Whatnot, Vinted, Marketplace Facebook. **Livré & opérationnel (session #14).**

### ✅ Stratégie de matching RETENUE — par NUMÉRO de collection
Le matching se fait par **numéro**, pas par nom → la barrière FR↔EN disparaît (le numéro est agnostique à la langue). *(L'ancien pont nom→EN via Poképédia/TCGdex est **abandonné** — voir §9 Archive.)*
- **Catalogue** = export **TCG Power Tools** (`cm-catalogue.csv`, **82 615 lignes** : `cardmarketId · name · collectorNumber · rarity · expansion · expansionCode · scryfallId · tcgplayerId`). Il **contient** le numéro de collection.
- **Pont série→extension AUTOMATIQUE** : const embarquée `SERIE_BRIDGE` (**194 entrées**, copie « slim » du MAPPING de l'intake) résout `code_série → expansionCode` ; ambiguïté des sets éclatés levée par `nom_série` (Foudre Noire→BLK, Flamme Blanche→WHT) + préférence du code canonique. Japonais filtrés par `rarity === 'Fixed'`. **~90 % résolus uniques** (auto-remplis, statut « ✓ auto », modifiables), **~10 % collisions** → panneau candidats + **deep-link fiche CM** `?idProduct=<id>`.
- **Variantes** (shadowless / oversized / stamps) **indistinguables dans le texte du catalogue** (seul l'`idProduct` les sépare) → stockées en **col O « Variantes »** du stock singles ; jointure write-back par **Key** → variante respectée implicitement.

### Deux fichiers produits par l'intégrateur
1. **`assignations_cm`** (Key + idProduct + contexte, `;`) → vers tcg-intake pour le **write-back**.
2. **`cardmarket_upload_pt`** (`Quantity · Cardmarket ID · Language · Condition · Price · Name`, virgule) → vers **TCG PowerTools → « Publish to Cardmarket »** = **la voie de publication retenue** (API CM fermée). Langue & état recopiés tels quels depuis CENTRAL ; quantité = 1/single.

### Write-back de l'`idProduct` (dans tcg-intake)
- L'intégrateur reste **lecture seule** ; l'écriture vit dans l'intake (scope écriture + filet `SAFETY_GSHEET_ID`).
- Charge `assignations_cm`, **résout les colonnes par ENTÊTE**, **aperçu obligatoire** (dry-run : à écrire / écrasements / identiques / introuvables), **backup** + **journal**, écriture par lots `values:batchUpdate`. Cible singles : onglet `cartes_stock_et_transactions`, jointure **Key col D**, idProduct **col L**.
- **Auto-détection du format déposé** : `Key + idProduct` → write-back ; sinon export PT → mode ajout. Parsing **agnostique au délimiteur** (`;`/`,`).
- **Scellé : non branché** (pas de source d'`idProduct` scellé). Cible prête : onglet `stock_scelle`, **Key col B**, idProduct **col I « CM »** (⚠️ casse du nom d'onglet à confirmer — cf. §1).

### Vignette CM — abandonnée (lien seul)
Tentative d'afficher le scan CM (`og:image`) : **impossible côté client** (proxies CORS bloqués + mur Cloudflare). Désactivée (`CM_IMG_PROXIES=[]`), mécanique conservée ; ré-activable via un proxy maîtrisé ou le **gateway #25**. Le lien fiche CM suffit à trancher.

### Corrections notables (session #14)
- **Table MAPPING (intake = source de vérité)** : `LG→LC` (Legendary Collection), `UL=Déchaînement` (HGSS2.0, length 95), `UD=Indomptable` (HGSS3.0), `CL2` retiré.
- **Auth intake** migrée vers le **flux GIS token** (identique à l'intégrateur) → basée sur l'origine JS déjà autorisée, **aucune modif console Google** (corrige l'`Error 400 redirect_uri_mismatch` de l'ancien flux implicite).
- **`SAFETY_GSHEET_ID`** repointé sur une **Google Sheet NATIVE** (`1axA5pJeu2PLzrBnd_HLox3AEck5AjbCkFpx19rRtmJg`) — l'ancien `.xlsx` importé était refusé par l'API (`FAILED_PRECONDITION`).
- **Badge de version à l'écran** ajouté à l'intake (`APP_VERSION`).

### Architecture multi-canal — ne pas sur-spécialiser sur CardMarket
Les canaux ne fonctionnent **pas** pareil :
- **CardMarket (et TCGplayer plus tard)** = **catalogue + identifiant produit** (`idProduct`). Travail dur = résoudre quel `idProduct` correspond à ma carte (matching).
- **Vinted / Marketplace Facebook** = **annonces texte libre**. Pas de catalogue ni d'ID : on génère titre + description + photos + prix (**génération d'annonce**).
- **Whatnot** = **enchères/lots en live**. Logique de lots + présence caméra.

👉 **Ossature réutilisable** : `sélectionner un lot → PRÉPARER le payload spécifique au canal → revue humaine → submit (humain tant qu'il n'y a pas d'API) → marquer « publié sur <canal> » + stocker la référence canal`. La résolution d'`idProduct` n'est QUE l'impl. "préparer" de CardMarket ; Vinted/FB auront une impl. "préparer" = génération d'annonce.

### CardMarket — contrainte API
- **Pas d'API CardMarket** (candidatures fermées) → remplacée par les **fichiers publics** (catalogue + price guide, MAJ quotidienne, libres d'usage) en lecture ; **écriture derrière l'API fermée** → on publie sans API, via TCG PowerTools, **submit humain**.

### Périmètre
- **Phase 1 = Pokémon singles uniquement.** ~79 cartes (1,9 %) du stock sont du **One Piece** → l'intégrateur segmente par jeu et écarte le non-Pokémon (catalogue One Piece = fichier séparé, même ossature, plus tard).
- **Scellés = piste parallèle** (catalogue sealed CM séparé), après les singles.
- **Qualité de données** : 198 cartes ont un numéro tronqué `NNN/` (dénominateur jamais saisi) → matching OK via numérateur+série, mais à **compléter à la source (Intake)** idéalement.

### QR codes — décision
- ❌ Pas l'`idProduct` comme clé QR (timing, granularité édition≠exemplaire, spécifique canal). ✅ Le QR encode un **identifiant interne** (SKU = la **Key** col D). `idProduct` = **référence externe** sur la définition de carte, à côté des futurs `whatnot_id`, `vinted_id`, etc.

### 🔑 Backlog `tcg-integrator`
- Catalogue **One Piece** + autres jeux ; **scellés** (catalogue sealed CM + normalisation produit).
- Canaux **Whatnot / Vinted / FB Marketplace** (impls. "préparer" = génération d'annonce).
- **Price guide** CM pour le **repricing** (pas nécessaire pour lister).
- **Photo par article**, notifications de vente temps réel, MAJ rapide CENTRAL anti-survente.
- **Gateway Apps Script (#25)** : ré-active la vignette CM (lecture serveur sans CORS) + l'API CM si elle rouvre.

---

## 7. Notes de Session

| Session | Date | Résumé |
|---------|------|--------|
| #1 | — | Mise en place architecture de base, flow cartes |
| #2 | — | Corrections diverses dont message "ajouté (0€)" |
| #3 | 28/05/2026 | Split Scelle_Stock/Scelle_Transactions, col C Key, applySealedSale, renommage sans accent (onglets), document de contexte |
| #4 | 28/05/2026 | Migration GitHub Pages, nouveau projet OAuth Google Cloud, fix constantes SHEET_SEALED/_TX, détection dynamique header scellé, cache fonctionnel à 4155 items |
| #5 | 28/05/2026 | Flow scellé end-to-end validé (master 1.5.10 / sales 1.5.3) ; durcissements 1.5.x (arrondi unitPa, fail explicite sheetId, UNFORMATTED_VALUE, locale parsePrix) |
| #6 | 29/05/2026 | Challenge + implémentation features TVA_Vente & Acheteur (v1.6.0) ; fix bug AG jamais écrit (scellé) ; verrou « réservée » ; découplage session/acheteur ; correction off-by-one table §3 ; non-bug cartes (TVA_Vente=AF) confirmé. Fix v1.6.1 (Sales) : liste TVA_Vente lue dans `Scelle_Transactions` (et non `Scelle_Stock`). Mapping versions divergent assumé : **Sales 1.6.1 / Master 1.6.0** (patch Sales seul). |
| #7 | 29/05/2026 | **Revue de code des 3 apps, PUIS correctifs livrés après validation.** Bugs corrigés : 🔴 bloquant Master `applyCardSale` (bloc `copyPaste` parasite réf. `nextTxRow` hors scope → `ReferenceError` à chaque vente carte, fausse anomalie `CARD_NOT_FOUND`) ; 🟠 régression index `Synced` (col M lu au lieu de col O dans `renderSalesLog`/`loadCloseSummary` → résumé de clôture faussé) ; 🟡 Intake (CRLF non normalisé, en-tête cartes dynamique, `parsePrix` locale). Livrés : **Master 1.6.2**, **Intake 1.0.0** (1ʳᵉ mise sous versioning). Non traités → backlog : code mort (Sales `switchTab`+CSS, constantes/fonctions inutilisées, #16) et perfs réactivité (#12). `SAFETY_GSHEET_ID` laissé en placeholder (garde-fou explicite déjà présent). Ajout instruction §0 : revue de code systématique en début de chaque discussion. Petit fix complémentaire **Sales 1.6.3** : « pas vendu » retiré du menu TVA_Vente. **Test E2E (ventes cartes + scellé) concluant** sur les nouvelles versions. Détail complet en §5. |
| #8 | 29/05/2026 | **Revue stricte de `tcg-sales`, lot de durcissements, PUIS outbox idempotent.** Déclencheur : le renommage de l'onglet cartes en `cartes_stock_et_transactions` (côté GSheet) a cassé Master (`400 Unable to parse range`). **(a) Durcissements** — **Sales 1.7.0** (renommage `SHEET_CARDS` + helper `a1()` partout ; **colonnes cartes en détection dynamique** par nom d'en-tête avec repli sur offsets ; remise « ± Total » refaite en **plus grand reste / euros entiers** sur tous les articles, total exact ; **formateur monétaire Intl fr-BE** ; **journal `logEvent`** persistant + `exportLog()` ; reset acheteur après vente hors-convention ; purge `switchTab`/CSS morts ; **zoom + media query smartphone** ; `HEARTBEAT_POLL_MS` câblé, `SYNC_POLL_MS` retiré ; `HttpError`, `parseInt` base 10, `init().catch`, `txId` rendu unique) et **Master 1.6.3** (même renommage + `a1()` partout (lecture ET écriture) + `HttpError` + journal + Intl UI + `parseInt`/`init().catch`). NON transposé à Master : détection dynamique des colonnes (Master **écrit** par lettre fixe P/P:T/AF → trop risqué sans test) et format des nombres de l'archive de clôture. **(b) Outbox idempotent** — **Sales 1.8.0** : toute vente mise en file (`pending`) puis envoyée ; **idempotence par relecture `ventes_detail!D`** (txId) avant tout renvoi → ni doublon (réponse perdue) ni perte (envoi échoué) ; échec définitif (4xx) ou après `MAX_SALE_RETRIES` → onglet `anomalies` ; si l'écriture anomalies échoue aussi → **dead-letter localStorage** + bandeau rouge « Réessayer » (**zéro perte silencieuse**) ; **journal du cycle de vie**. Machine à états vérifiée en simulation (7 scénarios). **Sales 1.8.1** : `moveToAnomalies` aligné sur le **vrai schéma A:K** de l'onglet `anomalies` (une ligne/article, `SALES_NOT_SYNCED`, payload JSON en `Notes`). **`tcg-intake` non touché** (référence encore l'ancien nom → à corriger en #9). Mise à jour de ce document. *(NB : les changelogs internes des fichiers livrés étiquetaient l'outbox « Session #9 » ; à recaler en « #8 » si tu veux une cohérence parfaite.)* |
| #9 | 29/05/2026 | **Revue de sécurité (OWASP 2025) + Option 1.** **Intake 1.1.0** : `esc()` (corrige XSS DOM via CSV/scraping) + `safeForSheets()` (corrige injection de formules) + **renommage onglet** `cartes_stock_et_transactions` (priorité #9.1 — il cassait dessus). **Option 1 (Master 1.9.0 + Sales 1.9.0)** : Master publie une vue stock sûre `STOCK_PUBLIC` dans EVENTS (cartes+scellés dispo, **sans coûts/marges**, + flag `Reserved` + liste TVA) ; Sales lit le stock de là et **ne lit plus CENTRAL** (fonctions mortes `findCardsHeaderRow`/`findColByName` retirées) → vendeurs sans accès CENTRAL, marges jamais exposées. Confirmé : 2 GSheets « Restreint ». **Versioning** : Option 1 modifiant les 2 apps ensemble, elles partagent **1.9.0** (Master re-tagué 1.6.3→1.9.0) ; l'auth GIS visée glisse en **1.10.0**. **Test E2E concluant** : Option 1 validée jusqu'à la **clôture d'événement** 🏁 (archive + vidage des buffers `ventes_detail`/`anomalies`) — la clôture, en attente depuis #8, est validée. **Hardening recommandé restant** : protéger `STOCK_PUBLIC` + ajouter vendeurs (comptes propres). Backlog sécurité ouvert : R4/R5, CSP/SRI, module commun, Option 3 Apps Script. Détail : `Bilan_risques_TCG.md`. |
| #10 | 29/05/2026 | **Revue §0 d'ouverture → constat que tout le « plan » était déjà livré, PUIS lot polish Sales.** La revue a révélé que les apps étaient en **1.9.0** (doc d'ouverture périmé en #7) : l'« augmentation du panier » et les « articles manuels » discutés étaient **déjà implémentés** (Sales `± Total` bidirectionnel + `+ Manuel` / Master `applyManualSale` no-op stock), et l'arrondi #6 déjà corrigé en #8. **Vérifié par exécution** que la règle de répartition « ± Total » (proportionnelle, euros entiers, plus grand reste, total exact) marche **à la hausse comme à la baisse** ; seul résidu = micro-dérive flottante ≤ 5,7e-14 € sur col K quand qty>1 (jamais visible). **Lot livré — Sales 1.9.1** (patch, **Sales seul** ; ⚠️ **1.10.0 reste réservé à l'auth GIS** Sales+Master) : hardening arrondi col K `Math.round(x*100)/100` (#20 partiel) · feedback de scan flash ✓/✗ (#14) · `renderCart()` **incrémental** par ligne (#12) · bandeaux d'état regroupés `#statusZone` + compactage mobile (#13) · purge reliquats code mort `sheetsUpdate`/`sheetsBatch` + vars `cardHeaderRow`/`cardColIdx`/`sealedColIdx` (#16 reliquat). Master & Intake **non touchés**. Refonte centimes complète (#20) toujours ouverte. **PUIS refonte UI Sales complète (1.9.2 → 1.9.5, patch Sales seul, itérée sur maquettes)** : interface **3 zones** tenant dans un écran smartphone sans scroll global ; en-tête (logo PNG embarqué + titre « TCG SALES » entier + version collée dessous à droite + **pastilles rondes d'état** G/📦 cerclées vert/rouge/ambre, détail en info-bulle) ; milieu = **carrousel swipe panier ⇄ caméra** avec **scan caméra natif `BarcodeDetector`** (codes-barres + QR, repli gracieux — **#4 fait**, à valider sur téléphone) ; lignes article = 🗑 + **chip quantité tappable à gauche** (position fixe — **#10 fait/dépassé**) + nom + **prix unitaire éditable par ligne** (corrige une côte périmée sans toucher les autres) ; **doublon = même ligne ×N** (contour orange ancienne clé / **rouge bloquant** clé neuve) ; bas = client/type/total/**bouton Valider épinglé en Syne** (= police du titre). **Rouleaux remplacés par saisie clavier** (quantité/prix/total, `inputmode`). Pastille cache = anneau coloré (**#9 résolu**). **Roll-back de vente esquissé puis RETIRÉ** (bouton ↶/modale/styles) pour livrer une **1.9.5 propre** : il touche aux données déjà chez Master → **reporté en jalon coordonné (backlog #27)**, à spécifier côté Master d'abord. ⚠️ Caméra/swipe/claviers **non testés** (pas de navigateur en atelier). Aucune logique métier touchée. |
| #11 | 30/05/2026 | **Palier coordonné 1.10.0 (Sales+Master) + dashboard Master 1.10.1.** **Master 1.10.0** : refonte en-tête (logo, titre, 1240px, panneaux stock/Google), page floutée tant que Google non connecté + overlay accès refusé · **verrou d'instance lease/TTL** (`app_state` C/D = `lock_uuid`/`lock_label`, acquisition écrire→jitter→relire, éviction détectée au battement, « Forcer la reprise », reprise auto si bail >35 s, anti-ping-pong, migration purge C2/D2) · sync gated sur le bail (plus sur l'event) · champs « acheteur défaut »/« label » retirés · cleanup écritures mortes U/AD (scellé). **Sales 1.10.0** : statut master par détection de changement d'horodatage (warning non bloquant, ventes toujours envoyées) · arrêt lecture `app_state` col D comme acheteur. ⚠️ déployer les 2 ensemble. Puis **dashboard Master one-screen maquette v6 (Master 1.10.1)** : 4 KPI auto-shrink, 2 donuts catégorie/licence, ⭐ faits marquants, CA/heure horizontal, aides « ? ». Reste acheteur imposé côté Sales (validé maquette). *À l'époque NON testé navigateur (l'a été depuis — cf. #12).* |
| #12 | 31/05/2026 | **Revue systématique des 3 apps (instruction §0), PUIS livraisons.** **Trouvailles de revue** : 🔴 l'**Intake était cassé** (`SHEET_NAME` resté sur l'ancien nom — le renommage #9.1 documenté n'avait jamais été appliqué) ; 🟠 la clôture vide `ventes_detail` même si le drain échoue (pending>0 → ventes non appliquées perdues) et l'archive ne garde que les colonnes A→G des lignes brutes — *signalés ; ⚠️ **cas-limites / question de scope, PAS un dysfonctionnement** : le **chemin nominal de clôture (drain OK → archive + vidage des buffers) est testé et validé E2E** (cf. #9 — voir aussi §5ter et §6, « Clôture événement 🏁 ✅ validée en #9 » ; **réaffirmé en #13**). Le durcissement éventuel (refuser le vidage si pending>0 ; archiver les colonnes complètes au-delà de A→G) reste un item **backlog optionnel, non bloquant**.* ; CSS mort `.stat-*`/`.split-*`/`.lb-*` (Master) confirmé. **Livraisons** : **Master 1.11.0** — #18 dédup `txId` à la sync (signature `txId\|clé\|source`, `Set` reconstruit par tick, `DUP_SKIPPED` + anomalie `DUPLICATE_TXID`, cartes/scellés ; scellé devient idempotent ; « ignorer SALES_NOT_SYNCED » déjà satisfait ; **filet, pas mutex** ; vérifié par simulation 5 scénarios). **Sales 1.10.1** — acheteur **imposé** (event rempli → lecture seule pour tous ; vide → éditable obligatoire). **Intake 1.2.0** — réparation : `SHEET_NAME` → `cartes_stock_et_transactions` + helper `a1()` partout. **Intake 1.2.1** — `SAFETY_GSHEET_ID` renseigné (feuille logs/backups) + **CSP**. **CSP sur les 3 apps** (Sales **1.10.2** / Master **1.11.1** / Intake **1.2.1**) ⚠️ à valider navigateur (seul vrai point de test). **SRI : N/A** (aucun `<script src>` tiers). **Clarifications** : module commun (#24, rangement code) ≠ Option 3 (#25, sécurité, **gros chantier reporté après l'auth GIS**). Versioning : #18 a pris Master 1.11.0 → **auth GIS glisse en 1.12.0**. Correction : l'UI #11 a depuis été **validée en navigateur** (dette de test #11 levée). Tout `node --check` OK. Mise à jour de ce document. |
| #13–#19 | — | *(non historisées dans cette table — voir §9 Archive pour les snapshots #13/#14 ; sessions #15–#19 à reconnecter si nécessaire.)* |
| #21 | 10/06/2026 | **Popup checklist vacances CM + rappel clôture (Master 1.16.0).** Deux fonctionnalités livrées. **(a) Popup checklist à l'activation d'un event** : déclenchée **uniquement** à la transition vide→nom rempli (pas sur re-save identique ni au reload) ; checklist ordonnée ① mettre CM en mode vacances, ② reporter les ventes CM non saisies, ③ rafraîchir le cache Master ; l'application effective du nom d'event est **différée** jusqu'à validation humaine (`confirmVacationCheck()` → `_applyEventName()`). **(b) Rappel réactivation canaux** après clôture 🏁 réussie (`openVacationReactivateModal()`). Correction des étiquettes changelog embarquées dans le fichier (« Session #14 » → « Session #20 » sur les entrées 1.15.0/1.14.x ; les 1.13.0/1.12.0 vraiment livrées en #13 restent inchangées). Tout `node --check` OK. **Décisions d'architecture** : mode vacances CardMarket = **Manuel** uniquement (API CM candidatures fermées ; apps statiques GitHub Pages incompatibles avec les secrets OAuth 1.0a) ; automatisation vraie = backlog #25. Processus opérationnel validé (ordre des 3 étapes). **Décision** : améliorations esthétiques des 4 apps planifiées (décrites en §1 — PENDING, à coder en prochaine session). Cas 1 différé après les esthétiques. |
| #20 | 10/06/2026 | **Tests iPhone réels → 2 bugs iOS + cas 2.** **(a) Douchette** : après chaque scan, `renderCart()` déplaçait `#scanZone` → l'input HID se défocusait (WebKit) → scan suivant perdu ; corrigé par re-`focusScan()` en fin de `processScan` (**Sales 1.14.1**) puis **filet global** `focusout` qui ré-arme la douchette après toute action (**Sales 1.14.4**). **(b) Clavier iOS** : longue chasse (1.14.1→1.14.4 : focus synchrone, suppression du vol de focus, masquage opacity au lieu de `display:none`, champs modale à 16px) — **la vraie cause était externe au code** : la **douchette Bluetooth connectée = clavier physique pour iOS → clavier logiciel masqué**. Solution terrain = **double-clic gâchette** (cf. §1). Les correctifs focus/16px restent de vrais progrès. **(c) Cas 2 (jalon coordonné Sales+Master 1.15.0)** : code scanné non reconnu → modale d'ajout forcé (libellé + prix obligatoire) → `source='inconnu'` → vente via outbox + anomalie `SCAN_INCONNU` côté Master (`applyUnknownSale`, statut `OK_UNKNOWN`) ; roll-back inconnu = no-op. **Validé sur iPhone réel.** Reste : cas 1 (vendu ailleurs, cf. §1) + Priorité #1 (GIS). Tout `node --check` OK. |
| #22 | 10/06/2026 | **(a) Améliorations esthétiques des 4 apps LIVRÉES** (décidées #21, cosmétiques pures, **NON testées navigateur**) : tcg-master logo 54→84 px + panneaux Journal/Anomalies repliables `collapsed` par défaut (`toggleCardDetails()`) ; tcg-sales titre agrandi pleine largeur (`clamp(2rem,12vw,3.2rem)`) ; tcg-intake + tcg-integrator logo société ajouté + version collée à droite du titre + « by Cassius » supprimé. *(Vérifié au passage : le code integrator disait bien « TCG INTEGRATOR » — pas la coquille « TCG INTAKE » redoutée.)* HTML parse-check OK sur les 4. **(b) Spec « Cartographie EAN scellé » FIGÉE** (détail complet en §1) — conception validée point par point : portée **`scelle_stock` uniquement** (transactions = historique hors scope) ; clé produit `License + Code Série + Nom Série + Nom_Item` ; filtres cascade dynamiques (License→Série→Type) ; shortlist **dédupliquée** (1 produit même multi-lots), sélection 1-2 produits/EAN ; écriture **différée en batch** col `Key` ; conflits (EAN différent) signalés avant commit. Layout réel confirmé (extract 10/06 : 38 col, en-tête **ligne 2**, Key=**C**, License=**D**, CM=**K** — résolution par libellé obligatoire). À CODER en session dédiée, après tests du 13. Aucune logique métier touchée cette session. |

---

## 8. Versioning des Apps

> Une table par app. Quand une fonctionnalité touche deux apps lors d'une même session (jalon coordonné), elles partagent le numéro — la ligne apparaît alors dans les deux tables, décrite côté app. *(Règles de versioning : voir §0.)*

### TCG Sales
| Version | Date | Changements |
|---------|------|-------------|
| 1.15.1 | 10/06/2026 | **(#22, cosmétique pure — NON testé navigateur)** Titre « TCG SALES » agrandi et étiré sur toute la largeur jusqu'au bord droit (`.brand-title-block` width:100%, `h1` `clamp(2rem,12vw,3.2rem)`) ; version inchangée dessous, alignée à droite. Aucune logique métier touchée. |
| 1.15.0 | 10/06/2026 | **(#20, cas 2 — jalon coordonné Master 1.15.0)** Code scanné non reconnu → modale « ❓ Code non reconnu » (libellé = code brut modifiable, **prix obligatoire**) → ajout en `source='inconnu'` (pas de regroupement, contour ambré + ❓). Part dans `ventes_detail` via l'outbox ; Master l'enregistre + anomalie `SCAN_INCONNU`. *Nécessite Master ≥ 1.15.0.* |
| 1.14.4 | 10/06/2026 | **(#20)** Champs de modale à **16px** (sous 16px, iOS zoome au focus et SUPPRIME le clavier sur focus programmatique). Filet global `focusout` → ré-arme la douchette après toute action. *(NB : le masquage clavier observé venait surtout de la douchette Bluetooth connectée — cf. §1.)* |
| 1.14.3 | 10/06/2026 | **(#20)** Retrait des animations d'entrée des modales (l'input était à opacity 0 à l'instant du focus). + fichier `diagnostic-clavier-ios.html` (révèle que la cause était externe : douchette connectée). |
| 1.14.2 | 10/06/2026 | **(#20)** Anti-vol de focus (stopPropagation prix/qté + garde « modale ouverte ») ; modales masquées via opacity (pas `display:none`). |
| 1.14.1 | 10/06/2026 | **(#20)** Douchette : re-`focusScan()` en fin de `processScan` (enchaînement des scans) ; focus synchrone des modales (sans `setTimeout`). |
| 1.14.0 | 31/05/2026 | **(#27, roll-back)** Bouton ↶ Annuler pendant 90 s ; annulation directe si la vente est encore dans l'outbox non envoyée, sinon ligne compensatoire `-RB` (qty négative ; col K = `prixOriginal` pour les cartes) via le même outbox idempotent. *Déployer avec Master 1.14.0.* |
| 1.13.0 | 31/05/2026 | **(#13)** Vidage du panier persistant (`{items,savedAt,version,eventName}`, jeté au boot si >30 min / version ≠ / event différent ; vidé au logout et au changement/clôture d'event ; snapshot restaurable 60 s) · durcissement auth (plus de demande de jeton auto au boot → connexion au tap ; popup bloquée + échec de refresh rendus visibles). |
| 1.12.0 | 31/05/2026 | **(#13, #21/R4, jalon coordonné)** Migration vers le modèle token GIS ; jeton **en mémoire seule** ; refresh programmé avant expiration ; connexion popup au tap. CSP +`accounts.google.com`. ⚠️ refresh non silencieux constaté (cf. §1). |
| 1.10.2 | 31/05/2026 | **(#12)** En-tête CSP `<meta>` (connect-src Sheets + OAuth Google). |
| 1.10.1 | 31/05/2026 | **(#12)** Acheteur **imposé** : event rempli → champ lecture seule (`#buyerLocked`) = nom d'event pour tous ; vide → éditable + obligatoire. |
| 1.10.0 | 30/05/2026 | **(#11, jalon coordonné)** Statut master par détection de changement d'horodatage (warning non bloquant, ventes toujours envoyées) · arrêt lecture `app_state` col D comme acheteur. *Déployer avec Master 1.10.0.* |
| 1.9.4 | 29/05/2026 | **(#10)** En-tête compact : titre non tronqué, connexions en **pastilles rondes** G/📦 cerclées vert/rouge/ambre. |
| 1.9.3 | 29/05/2026 | **(#10)** Rouleaux → **saisie clavier** (quantité/prix/total) · prix unitaire éditable par ligne · 🗑 à gauche de chaque ligne. |
| 1.9.2 | 29/05/2026 | **(#10)** Refonte UI 3 zones (écran sans scroll global) · carrousel swipe panier ⇄ caméra + scan `BarcodeDetector` · doublon = même ligne ×N. |
| 1.9.1 | 29/05/2026 | **(#10)** Lot polish : arrondi col K 2 déc. (#20 partiel) · feedback scan ✓/✗ (#14) · `renderCart()` incrémental (#12) · `#statusZone` (#13) · purge code mort. |
| 1.9.0 | 29/05/2026 | **(#9, Option 1, jalon coordonné)** Lit le stock depuis `STOCK_PUBLIC` (une lecture `A2:L`) et **n'accède plus à CENTRAL** ; retrait code mort `findCardsHeaderRow`/`findColByName`. |
| 1.8.1 | 29/05/2026 | `moveToAnomalies` aligné sur le vrai schéma `anomalies` A:K. |
| 1.8.0 | 29/05/2026 | **Outbox idempotent** : file → envoi → confirmation par relecture `ventes_detail!D` (txId) · échec → `anomalies` puis dead-letter local · zéro perte silencieuse. |
| 1.7.0 | 29/05/2026 | Renommage onglet + `a1()` · colonnes cartes dynamiques · remise « plus grand reste » euros entiers · formateur Intl fr-BE · journal `logEvent` · purge code mort · `HttpError`. |
| 1.6.3 | 29/05/2026 | « pas vendu » exclu du menu TVA_Vente (`isNotForSaleStatus()`). |
| 1.6.1 | 29/05/2026 | Liste TVA_Vente lue dans `Scelle_Transactions!AG8:AG23` (corrige le menu réduit). |
| 1.6.0 | 29/05/2026 | **(jalon coordonné)** Sélecteur TVA_Vente · verrou « réservée » au scan · acheteur Sales↔Master · `ventes_detail` → A:N. |
| 1.5.1 | 28/05/2026 | Panneau de scan : retrait du prix (affiché uniquement dans le panier). |
| 1.5.0 | 28/05/2026 | **(jalon coordonné)** sealedColIdx dynamique · iPrix = Côte/u · retry ×3 · badge version UI. |
| 1.4.0 | 28/05/2026 | **(jalon coordonné)** Migration GitHub Pages · nouveau projet OAuth · détection dynamique header scellé. |
| 1.3.0 | 28/05/2026 | **(jalon coordonné)** Split Scelle_Stock/Scelle_Transactions · col C Key · cache scellé indexé par Key. |
| 1.2.0 | — | **(jalon coordonné)** Fix message « ajouté (0€) » · corrections diverses. |
| 1.1.0 | — | **(jalon coordonné)** Architecture de base · flow cartes. |

### TCG Master
| Version | Date | Changements |
|---------|------|-------------|
| 1.16.1 | 10/06/2026 | **(#22, cosmétique pure — NON testé navigateur)** Logo agrandi 54→84 px (`.logo-mark`, 60 px sous 520 px) ; panneaux **« 📜 Journal des ventes »** et **« ⚠ Anomalies »** rendus **repliables**, **masqués par défaut** (`.card-body.collapsed` + bouton « ▸ Détails / ▾ Masquer » par carte, helper `toggleCardDetails()`). Le JS qui peuple `#salesLog`/`#anomalies` (innerHTML seul) n'est pas affecté. Aucune logique métier touchée. |
| 1.16.0 | 10/06/2026 | **(#21)** Popup checklist activation event : déclenchée **uniquement** à la transition vide→nom rempli (pas sur re-save identique, pas au reload) ; liste ordonnée ① CM mode vacances, ② reporter ventes CM, ③ rafraîchir cache Master ; nom d'event **différé** jusqu'à `confirmVacationCheck()` → `_applyEventName()`. `saveEventConfig` refactorisé : `_applyEventName()` extraite comme noyau commun. Rappel réactivation canaux (`openVacationReactivateModal()`) après clôture 🏁 réussie. Correction étiquettes changelog embarquées (1.15.0/1.14.x : « Session #14 » → « Session #20 »). `APP_VERSION` bumpé à `1.16.0`. |
| 1.15.0 | 10/06/2026 | **(#20, cas 2 — jalon coordonné Sales 1.15.0)** Nouvelle source `inconnu` : `applyUnknownSale()` ne touche aucun stock et écrit une anomalie `SCAN_INCONNU` (code + libellé + prix) ; statut `OK_UNKNOWN` (compté comme traité). Roll-back d'un `inconnu` = no-op stock (`OK_MANUAL_RB`). Branche isolée, aucun impact sur les flux carte/scellé/manuel. *Nécessite Sales ≥ 1.15.0.* |
| 1.14.0 | 31/05/2026 | **(#27, roll-back)** Inversion par **modification des lignes existantes** dans CENTRAL (jamais de qty négative) : carte → vide P/R/S/T, `Q` = prix d'origine, `AF` « pas vendu » ; scellé → supprime la transaction (retrouvée par **txId estampillé en col `AA`**) + ré-incrémente la qty. Garde à 4 états ; neutralisation de la paire si undo avant sync ; trace `ROLLBACK_APPLIED`. *Master sûr seul.* |
| 1.13.0 | 31/05/2026 | **(#13)** Panneau **« 🏠 Récap comptoir »** permutant avec « Faits marquants » quand pas de nom d'event (swipe articles ⇄ clients du jour) · même durcissement auth. |
| 1.12.0 | 31/05/2026 | **(#13, #21/R4, jalon coordonné)** Migration GIS (jeton mémoire seule, refresh avant expiration, popup ; release du verrou avant logout ; allow-list `isAuthorizedMaster` conservée). ⚠️ refresh non silencieux (cf. §1). |
| 1.11.1 | 31/05/2026 | **(#12)** En-tête CSP `<meta>`. |
| 1.11.0 | 31/05/2026 | **(#12, #18)** Dédup `txId` à la sync : `saleSignature()` = `txId\|clé\|source` · `Set` des appliqués reconstruit par tick · doublon → `DUP_SKIPPED` + anomalie `DUPLICATE_TXID`. Scellé devient idempotent. *Filet, pas mutex.* |
| 1.10.1 | 30/05/2026 | **(#11)** Dashboard maquette v6 : 4 KPI auto-shrink · 2 donuts catégorie/licence · ⭐ Faits marquants · CA/heure horizontal · aides « ? ». Données agrégées de `ventes_detail` (rien stocké). |
| 1.10.0 | 30/05/2026 | **(#11, jalon coordonné)** Refonte en-tête + page floutée tant que Google non connecté · **verrou d'instance lease/TTL** (`app_state` C/D = uuid/label, « Forcer la reprise », anti-ping-pong) · sync gated sur le bail · cleanup écritures mortes U/AD. |
| 1.9.0 | 29/05/2026 | **(#9, Option 1, jalon coordonné)** `publishStockView()` projette la vue sûre `STOCK_PUBLIC` à chaque `loadStockCache` ; capture le flag `reserved`. Re-tagué depuis 1.6.3. |
| 1.6.3 | 29/05/2026 | Renommage onglet `cartes_stock_et_transactions` (corrige 400) + `a1()` partout (lecture/écriture) · `HttpError` · journal · Intl UI. |
| 1.6.2 | 29/05/2026 | Correctifs revue #7 : 🔴 `applyCardSale` (bloc `copyPaste` parasite → `ReferenceError`) supprimé · 🟠 index `Synced` `r[12]`→`r[14]` (col O). |
| 1.6.0 | 29/05/2026 | **(jalon coordonné)** Sélecteur TVA_Vente · fix bug AG jamais écrit (scellé) · découplage session/acheteur. |
| 1.5.0 | 28/05/2026 | **(jalon coordonné)** sealedColIdx dynamique · txRow 33 cols · append A1 · retry ×3. |
| 1.4.0 | 28/05/2026 | **(jalon coordonné)** Migration GitHub Pages · nouveau projet OAuth · détection dynamique header scellé. |
| 1.3.0 | 28/05/2026 | **(jalon coordonné)** Split Scelle_Stock/Scelle_Transactions · applySealedSale. |
| 1.2.0 | — | **(jalon coordonné)** Corrections diverses. |
| 1.1.0 | — | **(jalon coordonné)** Architecture de base · flow cartes. |

### TCG Intake
| Version | Date | Changements |
|---------|------|-------------|
| 1.4.5 | 10/06/2026 | **(#22, cosmétique pure — NON testé navigateur)** Logo société ajouté dans l'en-tête ; version (`#appVer`, désormais un `<span>`) **collée à droite du titre** « TCG INTAKE » (`.logo-title-row`) ; mention « by Cassius » supprimée du sous-titre. `flex-wrap` sur la ligne titre+version (la version repasse sous le titre si la place manque sur petit écran — à vérifier). Aucune logique métier touchée. |
| 1.4.4 | 06/06/2026 | **(#14)** `SAFETY_GSHEET_ID` repointé sur une **Google Sheet native** (l'ancien `.xlsx` importé était refusé par l'API) → backup/log opérationnels. |
| 1.4.3 | 06/06/2026 | **(#14)** Message d'erreur explicite si le classeur de backup/log ou la cible est un `.xlsx` importé (l'API Sheets refuse → `FAILED_PRECONDITION`) ; guide vers une Sheet native. |
| 1.4.2 | 06/06/2026 | **(#14)** Badge de version à l'écran (`APP_VERSION`). |
| 1.4.1 | 06/06/2026 | **(#14)** Fix auth (`redirect_uri_mismatch`) : migration de l'ancien flux implicite vers le **flux GIS token popup** (identique à l'Integrator) → basé sur l'origine JS déjà autorisée, **aucune modif console Google**. + lib GIS + CSP + refresh auto. |
| 1.4.0 | 06/06/2026 | **(#14)** Auto-détection du format déposé (`Key + idProduct` → write-back ; sinon export PT → mode ajout) + parsing **agnostique au délimiteur** (`;`/`,`) + strip BOM (corrige la lecture de `assignations_cm`). |
| 1.3.0 | 06/06/2026 | **(#14)** **Write-back des idProduct CardMarket** (onglet « idProduct CM ») : charge `assignations_cm`, cible résolue par ENTÊTE (Key col D / idProduct col L), aperçu obligatoire (dry-run), backup + journal. |
| 1.2.1 | 31/05/2026 | **(#12)** `SAFETY_GSHEET_ID` renseigné (feuille logs/backups) + **CSP**. |
| 1.2.0 | 31/05/2026 | **(#12, RÉPARATION 🔴)** L'Intake était **cassé** (`SHEET_NAME` resté sur `Cartes_Stock_&_Transactions`, jamais renommé) → 400 partout. Corrigé → `cartes_stock_et_transactions` + `a1()` partout. |
| 1.1.0 | 29/05/2026 | **(#9)** Garde-fous : `esc()` (XSS DOM) + `safeForSheets()` (injection de formules). ⚠️ le renommage annoncé ici n'avait **pas** été appliqué (resté cassé jusqu'à 1.2.0). |
| 1.0.0 | 29/05/2026 | Premier stamp formel (revue #7) : `parseCSV` normalise CRLF/CR · détection dynamique de l'en-tête cartes · `parsePrix` locale belge. |

### TCG Integrator
| Version | Date | Changements |
|---------|------|-------------|
| 0.4.3 | 10/06/2026 | **(#22, cosmétique pure — NON testé navigateur)** Logo société ajouté dans l'en-tête ; version (`#ver`) **collée à droite du titre** « TCG INTEGRATOR » (`.logo-title-row`) ; mention « by Cassius » supprimée du sous-titre. *(Confirmé : le code disait bien « TCG INTEGRATOR » — la coquille « TCG INTAKE » redoutée en #21 n'existait pas.)* Aucune logique métier touchée. |
| 0.4.2 | 06/06/2026 | **UX matching** : pont série→extension éditable **inline** dans la colonne de chaque carte (datalist, surcharge → applique à toute la série) ; colonnes resserrées (1 carte/ligne) ; code couleur vert=résolu auto / bleu=surcharge / orange=ambigu. *(version réelle du fichier déployé)* |
| 0.4.1 | 06/06/2026 | UX : correspondances **uniques pré-remplies auto** (« ✓ auto », modifiables ; un champ vidé à la main n'est pas re-rempli). Seules les collisions restent à choisir. |
| 0.4.0 | 06/06/2026 | **Second export** : fichier d'import **TCG PowerTools** (`cardmarket_upload_pt` : Quantity · Cardmarket ID · Language · Condition · Price · Name). L'intégrateur produit donc 2 fichiers (`assignations_cm` + `cardmarket_upload_pt`). Lecture seule. |
| 0.3.2 | 06/06/2026 | Vignette CM **désactivée** par défaut (`CM_IMG_PROXIES=[]`) : proxies publics bloqués (CORS + mur Cloudflare). Retour au lien « fiche CM » seul ; mécanique conservée (ré-activable via proxy maîtrisé / gateway #25). |
| 0.3.1 | 06/06/2026 | Vignette CM : robustesse (chaîne de proxies de secours, état visible par candidat ⏳/∅/photo). |
| 0.3.0 | 06/06/2026 | Vignette CardMarket par candidat (og:image via proxy CORS configurable) pour désambiguïser visuellement les variantes. |
| 0.2.4 | 06/06/2026 | Lien « fiche CM » par candidat (désambiguïsation des variantes). |
| 0.2.3 | 06/06/2026 | **Pont série→extension AUTOMATIQUE** via table de correspondance (`SERIE_BRIDGE`). |
| 0.2.2 | 04/06/2026 | **Matching par NUMÉRO** (stratégie locale, sans réseau ni TCGdex) — supersède l'approche nom→EN. |
| 0.2.1 | 04/06/2026 | Candidats enrichis (idMetacard, nom d'extension connu). |
| 0.2.0 | 04/06/2026 | Matching CM (catalogue JSON + atelier manuel). |
| 0.1.x | 04/06/2026 | Sélection + filtres (cascade) + pagination. |
| 0.1.0 | 04/06/2026 | Coquille mono-page : **auth GIS** + lecture stock CENTRAL (lecture seule) + **parser FR validé** (99,9 %). |

---

## 9. Archive (historique narratif)

> Contenu sorti du flux principal mais conservé pour référence. **La mémoire canonique reste §9 (Notes de Session) et §10 (Versioning).** Ces sections décrivent des états passés ou des approches abandonnées — ne pas les traiter comme l'état courant.

### A. Snapshots « État Actuel » par session (du plus récent au plus ancien)

### Session #13 (31/05/2026)

> ⚠️ **Toutes les livraisons #13 passent `node --check` mais AUCUNE n'a été testée en navigateur.** Checklist de test complète en **§0** (à faire avant prod). 🧪 Validation prévue via le **test de simulation maison du samedi 13 juin 2026** (multi-vendeurs, stock dédié) — plan détaillé en **§0** ; convention réelle les **21–22 juin 2026**. Le backlog d'améliorations est **en pause** au profit du lancement **CardMarket**.

### ✅ Objectifs de session atteints — 3 jalons coordonnés (Sales + Master)
- **1.12.0** — **Migration auth Google Identity Services (#21, R4)** : flux OAuth implicite codé main → **modèle token GIS**, jeton **en mémoire seule** (plus de `localStorage`, purge de l'ancien au boot), **refresh silencieux** ~5 min avant expiration (fin de la déconnexion ~1 h). CSP : ajout `accounts.google.com` (`script-src` + `frame-src`). SRI reste N/A (la lib GIS ne doit PAS être épinglée). Prérequis GCP déjà en place.
- **1.13.0** — trois retours : (a) **vidage du panier persistant** (estampillé `{items,savedAt,version,eventName}`, jeté au boot si > 30 min ou version ≠, réconcilié/vidé sur changement d'event au 1er heartbeat confirmé, **jamais** sur simple coupure réseau ; auto-vidage = snapshot 60 s restaurable) ; (b) **durcissement auth** : plus d'auto-popup au boot (connexion **au tap** = popup autorisée), erreurs `popup_failed_to_open` et échecs de refresh **rendus visibles** ; (c) **panneau « 🏠 Récap comptoir »** (Master) permutant avec « Faits marquants » quand pas de nom d'event (carrousel swipe Articles ⇄ Clients du jour).
- **1.14.0** — **roll-back d'une vente (#27)** : bouton **↶** pendant 90 s côté Sales ; **annulation directe** si la vente est encore dans l'outbox non envoyée, sinon **ligne compensatoire `-RB`** via le même outbox idempotent. Master **inverse dans CENTRAL par modification des lignes existantes** (jamais de qty négative) : **carte** → vide P/R/S/T, Q = prix d'origine, AF « pas vendu » ; **scellé** → supprime la transaction (retrouvée par **txId estampillé en col `AA`**) + ré-incrémente la qty du lot. **Garde à 4 états** ; **neutralisation de la paire** si undo avant sync ; trace `ROLLBACK_APPLIED`. `applySealedSale` estampille désormais le txId en `AA`.

### 📌 Décisions / clarifications de session
- **Clôture event 🏁** : confirmée **testée & validée E2E** (#9, réaffirmé #13). Les 2 constats 🟠 du log #12 (clôture vide `ventes_detail` même si le drain échoue ; archive ne garde que les colonnes A→G) sont des **cas-limites/scope, PAS un dysfonctionnement** → backlog optionnel non bloquant.
- **Roll-back** : approche **modification des lignes** (et non append de lignes négatives) retenue pour une compta propre. **Estampille `txId` en col `AA`** validée (convainc sur les cas remise + prix d'achat différents par lot). **OPEN** : confirmer que `AA` n'est pas lue par la compta (sinon changer `SEALED_TX_TXID_COL`).
- **Déploiement** : pousser **Sales 1.14.0 + Master 1.14.0 ensemble**. Master 1.14.0 est sûr seul (estampille inoffensive, gestion `-RB` dormante).

### 🟠 Backlog ouvert (hors scope #13)
- Durcissement à finir : protéger les onglets `STOCK_PUBLIC` / `app_state` ; donner aux vendeurs leurs **propres comptes Google** sur EVENTS.
- Limitation **pré-existante** (signalée, non traitée) : le cache scellé indexe le stock **par clé** (1 ligne par code-barres) alors que le modèle mental autorise **plusieurs lignes-lots** par code-barres.

### Session #12 (31/05/2026)

> Session de revue + livraisons. **Tout `node --check` OK ; aucun fichier encore testé en navigateur** (l'utilisateur teste lui-même ses déploiements — l'UI #11 a, elle, déjà été validée en navigateur, cf. correction ci-dessous).

### ✅ Objectifs de session atteints — 5 livraisons
- **[Master 1.11.0 — #18, dédup `txId` à la sync]** `masterSyncTick` ne ré-applique plus deux fois la même vente carte/scellé. Signature anti-doublon = **`txId | clé | source`** (`saleSignature()`, définie pour cartes/scellés seulement ; `''` pour le manuel — no-op stock — et toute ligne sans txId/clé). Un `Set` des signatures **déjà appliquées avec succès** (Synced ∈ {`OK`,`OK_MANUAL`}) est **reconstruit depuis la feuille à chaque tick** (rien à persister, survit au reload) puis complété au fil du tick. Doublon → **non ré-appliqué**, marqué `DUP_SKIPPED` + anomalie `DUPLICATE_TXID`. Le chemin **scellé** (jusqu'ici sans garde-fou) devient idempotent ; la **carte** l'était déjà partiellement (pré-check `P_NON_BLANK`). ⚠️ **Filet, pas mutex** : le split-brain « deux masters lisent le même snapshot au même instant » reste couvert d'abord par le **bail** (ne pas « Forcer la reprise » pendant une sync active). « Ignorer `SALES_NOT_SYNCED` » : **déjà satisfait** par l'archi (ces lignes vivent dans l'onglet `anomalies`, jamais dans `ventes_detail`). *Vérifié par simulation (5 scénarios).*
- **[Sales 1.10.1 — acheteur IMPOSÉ]** Dernier item du palier 1.10.0. Quand le Master publie un nom d'event (`app_state` col B) → champ **lecture seule** (pastille `#buyerLocked`), valeur = nom d'event pour **tous** les vendeurs (uniformité multi-vendeurs) ; vide → champ éditable + obligatoire. `getResolvedBuyer()` renvoie le nom d'event en priorité ; `updateBuyerField()` bascule input ⇄ pastille. Fin du défaut écrasable de la 1.10.0.
- **[Intake 1.2.0 — RÉPARATION 🔴]** L'Intake était **cassé** : `SHEET_NAME` était resté sur `Cartes_Stock_&_Transactions` (le renommage #9.1 documenté n'avait jamais été appliqué au fichier) → 400 « Unable to parse range » sur toutes ses opérations. Corrigé → `cartes_stock_et_transactions`, **+ helper `a1()`** porté à toutes les plages (cartes + LOG/BACKUP ; références d'onglet aussi quotées dans les formules `COUNTIF`/`SUMIF`).
- **[Intake 1.2.1 — config + CSP]** `SAFETY_GSHEET_ID` **renseigné** (`1c-HD5z…DtI`, feuille dédiée logs/backups) → journal + backups d'injection opérationnels. **CSP** ajoutée.
- **[CSP sur les 3 apps — #23]** Balise `<meta Content-Security-Policy>` en tête de `<head>` (Sales **1.10.2** / Master **1.11.1** / Intake **1.2.1**). `default-src 'self'` ; script/style inline autorisés (apps mono-fichier) ; `connect-src` limité à Sheets + OAuth Google (+ les 5 proxies de scraping côté Intake seulement) ; `object-src 'none'` ; `base-uri 'self'`. Protège surtout contre l'**exfiltration** réseau et l'injection de **script externe**. `frame-ancestors` ignoré en `<meta>` (nécessiterait un en-tête HTTP, indispo sur GitHub Pages).

### 🧪 Reste à tester en navigateur (rien testé en #12) — voir la liste détaillée en §0
**La CSP est le seul vrai point sensible** (peut casser l'OAuth si un endpoint manque) : la console affichera tout blocage CSP, retrait = supprimer la balise. Tester aussi : dédup doublon `txId`, acheteur imposé (event rempli/vide), import Intake, et le reliquat **jamais testé** du #11 : scénario **2 appareils** (verrou + reprise).

### 📌 Décisions / clarifications de session
- **SRI : N/A ici.** Aucune des 3 apps ne charge de `<script src>` tiers (OAuth codé à la main, pas de `gapi`/`gsi`) ; seule ressource externe = la CSS Google Fonts (à ne **pas** épingler). À ressortir seulement si ajout d'une lib via CDN un jour.
- **Module commun (#24) ≠ Option 3 (#25)** — deux chantiers **distincts et indépendants** : le module commun = **rangement du code** (dédupliquer les helpers dans `tcg-common.js`, refactor modéré) ; l'Option 3 = **architecture de sécurité** (passerelle Apps Script côté serveur). Module commun = candidat prochaine session ; **Option 3 = gros chantier, à faire après l'auth GIS**.
- **Versioning** : #18 a pris **Master 1.11.0** (minor, fix fonctionnel significatif, Master seul) → l'**auth GIS glisse en 1.12.0** (jalon coordonné). Patches CSP : Sales 1.10.2 / Master 1.11.1 / Intake 1.2.1.

### Session #11 (30/05/2026)
> *Snapshot figé au #11. MAJ #12 : l'UI Master/Sales décrite ici a depuis été **validée en navigateur** ; l'acheteur imposé et #18, notés « à coder » ci-dessous, sont **livrés** (voir §5 #12).*

> ⚠️ **Versions : Sales 1.10.0 / Master 1.10.0** (palier coordonné, numéro partagé). Le numéro 1.10.0, initialement réservé à l'auth GIS, a finalement été **consommé par ce palier** (refonte UI Master + verrou d'instance + sémantique acheteur). **L'auth GIS passe en 1.11.0.**

### ✅ Codé et livré en #11 — Dashboard Master (Master **1.10.1**, `node --check` OK, **NON testé navigateur**)
> Implémentation de la maquette **v6** validée. `tcg-master.html` 1.10.1 (Master seul ; Sales reste 1.10.0).
- **En-tête** : sous-titre « Sync engine… » retiré ; **version à droite du titre** (petit mono).
- **Bandeau event** compact (`.event-strip`) : icône + texte dynamique + champ + Enregistrer + **aide repliée « ? »** (texte imposé/non-bloquant). `renderEventBanner` bascule `event-strip ok|warn`.
- **Ligne 2 colonnes** : **« 🔄 Synchronisation des ventes »** (Sync now / Pause auto **à droite** de l'en-tête, statut dans le corps) | **« 🏁 Clôture de l'event »** (phrase derrière **« ? »**, bouton `btnCloseEvent` rendu unique). Titre « Stats de l'event en cours » **retiré**.
- **Grille `.dash-grid`** (`kpis kpis hours` / `donuts center hours`) : **4 KPI compacts** (CA total / Articles / Transactions / Panier moyen) avec **auto-réduction de police** (`fitKpis`, `data-fit`, relancé au resize) · **2 donuts** (catégorie puis licence) `pathLength=100`, **% dans la légende collés au nom**, couleurs mappées (`CAT_COLORS`/`LIC_COLORS` + fallback, clé normalisée accents/casse) · **⭐ Faits marquants** (`computeFacts`) : heure de pointe, plus gros panier, article le + cher, articles/transaction, cadence dernière h, meilleure série (fenêtre 15 min), vendeurs actifs, première vente, dernière vente · **CA par heure** (`computeHours`/`renderHours`) **horizontal** 10h→17h, créneaux vides hachurés, **montant dans la barre**.
- **Données** : tout agrégé de **`ventes_detail`** (cat=H, licence=I, prix=K, qty=J, ts=A, vendeur=B, txId=D) au refresh stats — **aucune donnée stockée**. `renderSplit`/`renderLeaderboard` supprimés ; **Journal des ventes** + **Anomalies** conservés (secondaires, sous le dashboard). `toggleHelp()` ferme les popovers au clic extérieur.

### ✅ Codé et livré en #11 (1.10.0 — `node --check` OK, **NON testé en navigateur**)
- **[Master 1.10.0 — UI]** Nouvel en-tête : **logo embarqué** (récupéré de Sales) + grand titre « TCG MASTER », largeur **1240px**, panneau stock (📦 + refresh) + panneau Google à droite. **Page floutée + inactive tant que Google non connecté** (overlay de connexion) ; overlay « accès refusé » (compte non master). Panneau event compacté (bannière dynamique seule). Champs « Acheteur par défaut » et « Label de l'appareil » **supprimés**.
- **[Master 1.10.0 — 🔒 Verrou d'instance (lease/TTL)]** `app_state!A2:D2` **remappé** : `[ts, eventName, lock_uuid, lock_label]` (avant `[ts, eventName, device, default_buyer]`). UUID aléatoire persistant (localStorage) + libellé auto. Acquisition **écrire→jitter→relire** ; le détenteur **relit avant chaque battement** (détection d'éviction → auto-désactivation lecture seule) ; bouton **« Forcer la reprise »** ; reprise auto si bail périmé (>35 s) ; **anti-ping-pong** (une instance évincée ne reprend pas auto). **Migration** : purge auto de l'ancien C2/D2 au 1ᵉʳ lancement. **Sync gated sur la détention du bail** (plus sur le nom d'event) + bouton Pause.
- **[Master 1.10.0 — Cleanup]** `applySealedSale` : retrait des écritures mortes `currentEventName` en `txRow` **U/AD** (colonnes à **formule recopiée** depuis la ligne du dessus via `copyPaste` — hors `staticCols`, donc à NE PAS écrire).
- **[Sales 1.10.0]** Statut master par **détection de changement** de l'horodatage (immunisé au décalage d'horloge ; seuil 30 s, poll 10 s) → bannière **warning NON bloquant** affichée seulement si master absent ; **les ventes partent toujours** (outbox inchangé). Lecture de `app_state` **col D comme acheteur supprimée** (col D = libellé verrou désormais).

### 🟡 Reste à coder du palier 1.10.0 — **un seul item**
> Le dashboard Master est **codé** (1.10.1, ci-dessus). Il ne reste que :
- **[Sales — acheteur IMPOSÉ]** Quand le **nom d'event** est rempli (`app_state` col B) → **imposé en lecture seule** à tous les vendeurs (réactiver `#buyerLocked`, `getResolvedBuyer()` renvoie le nom d'event) → cohésion des ventes sur la journée. Vide → champ éditable et **obligatoire**. *(Validé ; le code Sales 1.10.0 livré a encore l'acheteur en défaut **écrasable** → à corriger, puis bumper Sales.)*

### ⚠️ Précautions de déploiement #11
- **Déployer Sales ET Master ENSEMBLE** : le remap `app_state` C/D (uuid/label) casse l'ancien Sales (qui lisait col D comme acheteur). Master 1.10.0 réécrit les en-têtes `app_state` et **purge C2/D2** au 1ᵉʳ lancement.
- **Re-protéger** `STOCK_PUBLIC` et `app_state` si besoin après coup.
- **#18 dédup `txId` = PRÉ-REQUIS BLOQUANT avant tout event réel** : seul filet couvrant la fenêtre de split-brain (~10 s) lors d'une reprise de verrou. Ne pas utiliser « Forcer la reprise » en prod tant que #18 n'est pas livré.
- **Aucun fichier testé en navigateur** : valider le scénario **2 appareils** (verrou + reprise) et le comportement **acheteur** côté Sales.

### Session #9 (29/05/2026)

### ✅ Terminé en session #9 (sécurité + Option 1)
Revue de sécurité (OWASP 2025) des 3 apps puis correctifs validés. Bilan détaillé : `Bilan_risques_TCG.md` (réorganisé : risques ouverts d'abord, puis couverts).
- **[Sécu 🔴 — Intake 1.1.0] XSS DOM via CSV corrigée.** L'Intake était le seul fichier sans échappement. `esc()` (identique à Sales/Master) appliqué à **tous** les `innerHTML` (table, séries inconnues, prix de lot, prix élevés, récap, étiquettes), y compris attributs `id`/`data-*` dynamiques ; `getElementById` inchangés (cohérence via id décodé).
- **[Sécu 🔴 — Intake 1.1.0] Injection de formules corrigée.** `safeForSheets()` sur les colonnes **texte** du CSV dans `buildCardRow` ; `buildLotRow` double les guillemets dans `COUNTIF`/`SUMIF`.
- **[Fix 🔴 — Intake 1.1.0] Renommage onglet cartes** (priorité #9.1) : `SHEET_NAME` → `cartes_stock_et_transactions`. L'Intake est de nouveau opérationnel sur la GSheet cartes.
- **[Archi/Sécu 🔒 — Master 1.9.0 + Sales 1.9.0] Option 1 : vendeurs sans accès à CENTRAL.** Master publie une vue stock **sûre** (`STOCK_PUBLIC` dans EVENTS, sans coûts/marges) ; Sales lit le stock de là et **ne lit plus CENTRAL** (vérifié : zéro référence exécutable à CENTRAL côté Sales ; fonctions mortes `findCardsHeaderRow`/`findColByName` retirées). Les marges ne sont **jamais** exposées aux vendeurs.
- **[Confirmé]** 2 GSheets en « Restreint ».

### ✅ Test E2E #9 — concluant (Option 1 prête à la prod)
- Validé de bout en bout : déploiement Master 1.9.0 → `STOCK_PUBLIC` créé/peuplé (sans colonnes de coût) → Sales 1.9.0 lit le stock **sans accès CENTRAL** → vente carte + scellé, verrou « réservée », menu TVA, sync Master → **clôture d'événement** 🏁 : archive + **vidage des buffers** `ventes_detail`/`anomalies` OK (la clôture était en attente depuis #8).
- **Fraîcheur** : `STOCK_PUBLIC` republié au rythme du cache Master (~5 min). Un scellé vendu peut rester affiché dispo jusqu'à la prochaine publication (Master rattrape en anomalie). Optimisation possible : republier après chaque vente appliquée (backlog #26).
- 🔧 **Hardening recommandé** (n'affecte pas le fonctionnement validé) : protéger l'onglet `STOCK_PUBLIC` (seul le propriétaire édite) ; ajouter les vendeurs avec leur **propre compte Google** sur EVENTS (pas de login master partagé).

### Backlog sécurité ouvert (bilan §A)
R4 (OAuth implicite + jeton `localStorage` → PKCE/GIS) · R5 (scope `spreadsheets` trop large → `drive.file`+Picker) · CSP/SRI · module commun (helpers dupliqués) · Option 3 (passerelle Apps Script, ferait tomber R4/R5 côté vendeur).

### Sessions #6–#7 (29/05/2026)

### ✅ Terminé en session #6 (apps en v1.6.0)
- **[Feature] Sélecteur TVA_Vente côté Sales** : liste lue dans `Scelle_Transactions!AG8:AG23` (source curée, « réservée » exclue du menu), défaut "Régime normal", valeur jointe à chaque vente via `ventes_detail` col M, écrite par Master en **AG (scellé) / AF (carte)**.
- **[Feature] Verrou « réservée »** : un article dont `TVA_Vente = "réservée"` est bloqué au scan côté Sales (lecture col AG scellé / AF carte, comparaison normalisée accents).
- **[Feature] Acheteur côté Sales** : si Master publie un **acheteur par défaut** (`app_state` col D) → affiché en lecture seule à tous les vendeurs ; sinon le vendeur le saisit (obligatoire, validation bloquée si vide). Transite via `ventes_detail` col N. Découplage **session (gate sync, cols U/AD)** vs **acheteur (col Q/R)**.
- **[Bug corrigé] Scellé : col AG (TVA_Vente) jamais écrite** — `txRow[32]` était calculé mais absent de `staticCols` ⇒ AG restait vide. Ajouté.
- **Schéma `ventes_detail` étendu** : `…L=Source, M=TVA_Vente, N=Acheteur, O=Synced, P=SyncedAt` (Sales écrit A:N ; Synced/SyncedAt retrouvés par nom → auto-recalés). `app_state` passe en A2:D2.
- **Non-bug confirmé** : `applyCardSale` écrit bien TVA_Vente en **AF** pour l'onglet cartes (≠ AG des scellés).

### ⚠️ Précaution de déploiement (à faire AVANT de pousser la v1.6.0)
- **Vider `ventes_detail`** (clôturer l'event en cours / reset buffer) avant de lancer le Master v1.6.0. Raison : `verifyEventsSchema` réécrit l'en-tête ; sur un onglet contenant d'anciennes lignes, l'ancien `Synced` (col M) serait relabellisé `TVA_Vente` et le nouveau `Synced` (col O) serait vide ⇒ **re-traitement des ventes = double décrément stock**. Sur un onglet vide, aucun risque.

### ✅ Hérité sessions #3–#5
- Flow cartes complet ; flow scellé end-to-end validé (ex master v1.5.10 / sales v1.5.3)
- Split `Scelle_Stock` / `Scelle_Transactions`, formules compta OK, cache indexé par Key, détection dynamique du header (col C = "Key"), 4155 items chargés
- Migration GitHub Pages, projet OAuth « TCG Intake »

### 🔍 Revue de code + correctifs (session #7)

Triple-check des 3 apps, puis correctifs livrés après validation. Versions résultantes : **Sales 1.6.3 / Master 1.6.2 / Intake 1.0.0**. Les bugs ci-dessous sont **corrigés** (✅) sauf mention contraire.

**✅ 🔴 Bloquant — Master `applyCardSale` : `nextTxRow` non défini (TOUTES les ventes cartes).**
Un bloc `copyPaste` avait été copié par erreur depuis `applySealedSale` ; il référençait `nextTxRow`, variable hors scope (déclarée seulement dans `applySealedSale`). En `'use strict'` → `ReferenceError`, attrapée par le try/catch par-ligne de `masterSyncTick` ⇒ `errors++`, ligne **non marquée Synced**. Les écritures stock (P:T + AF) réussissaient *avant* l'erreur (stock correct), mais : erreur console + **fausse anomalie `CARD_NOT_FOUND`** au tick suivant + ~8 s de retard par ligne. **C'est ce qui empêchait de valider proprement le flow cartes à travers Master 1.6.0.** → **Corrigé (Master 1.6.2)** : bloc supprimé (les cartes n'appendent pas dans `Scelle_Transactions`).

**✅ 🟠 Régression schéma 1.6.0 — Master lisait `Synced` au mauvais index (col M au lieu de col O).**
`Synced` est à l'index **14** (col O) depuis l'extension `ventes_detail` 1.6.0, mais `renderSalesLog` et `loadCloseSummary` lisaient encore l'index **12** (col M = `TVA_Vente`) ⇒ icône toujours ⏳ et résumé de clôture « ✓ Toutes synchronisées » à tort (**risque de clôturer un event avec des ventes non synchronisées**). Données réelles saines (le drain `ensureSyncDrain` utilise `lastSyncStats.pending`). → **Corrigé (Master 1.6.2)** : `r[14]` aux deux endroits.

**✅ 🟡 Intake — CRLF non normalisé.** `parseCSV` faisait `text.trim().split('\n')` sans retirer `\r`. → **Corrigé (1.0.0)** : `text.replace(/\r\n?/g,'\n')` avant le split.

**✅ 🟡 Intake — `CARD_HEADER_ROW = 143` en dur** (vs détection dynamique dans Sales/Master) ⇒ si l'en-tête bouge, la vérif anti-doublon et les offsets D/F/Q ratent les clés (**risque d'intégrité**). → **Corrigé (1.0.0)** : `findCardsHeaderRow` / `ensureCardHeaderRow` (mirroir de Master), résolus en tête de `injectIntoGSheet` et `loadLaterCards`.

**Mineurs :** ✅ Intake `parsePrix` locale belge (virgule décimale) au lieu de `parseFloat` ; ✅ commentaires Master « A:AF » alignés sur la lecture réelle `A:AG`. **Non-corrigé (volontaire)** : `SAFETY_GSHEET_ID` reste un placeholder **à renseigner par l'utilisateur** — le garde-fou `getSafetySpreadsheetId()` lève déjà une erreur explicite tant qu'il n'est pas configuré (comportement voulu).

**✅ Demande complémentaire (Sales 1.6.3) :** « pas vendu » exclu du menu déroulant TVA_Vente proposé au vendeur (c'est un statut, pas un régime). Filtre `isNotForSaleStatus()` ajouté à côté de l'exclusion « réservée » dans le chargement de `tvaVenteOptions`. Seul le menu est touché ; la sémantique du verrou au scan (`isReservedValue`) est inchangée.

**Incohérences / code mort relevés (NON traités — backlog #16) :**
- *Sales* : `switchTab()` + tout le CSS associé morts (classes `.tabs`, `.tab-btn*`, `.tab-content`, `.stats-grid`, `.stat-*`, `.lb-*`, `.split-*`, `.sales-log`, `.sale-*`, `.sync-*`). Constantes inutilisées `SHEET_ANOMALIES`, `SYNC_POLL_MS`, `HEARTBEAT_POLL_MS` (polling = littéral `20000`). Fonctions `sheetsUpdate` / `sheetsBatch` jamais appelées. Champs `sealedColIdx` calculés non lus (`iCode, iOrigine, iCodeSerie, iEtat, iCoteU, iCoteTot`, + `statutColLetter`).
- *Master* : `CARD_DEFAULT_REGIME_FISC` mort (AF prend `saleTva`), `sealedColIdx.iCoteTot` non lu.
- *Asymétrie (non-bug)* : Sales lit `Scelle_Stock!A:AG`, Master `A:AF` (Master relit la ligne complète `A:AG` à la vente).

**Performance / réactivité (NON traité — backlog #12/perf, la réactivité prime en event) :**
- *Sales* : `renderCart()` reconstruit tout l'`innerHTML` à chaque changement → maj incrémentale (DOM diff par ligne).
- *Master* : `applySealedSale` lit toute la colonne `Scelle_Transactions!B:B` à chaque vente (O(n)) → suivre `nextTxRow` de façon incrémentale.
- *Sales* : câbler la constante `HEARTBEAT_POLL_MS` au lieu du littéral `20000`.

### ✅ Validé en session #7 (test E2E)
- **E2E ventes cartes ET scellé** sur les nouvelles versions (Sales 1.6.3 / Master 1.6.2) : **concluant**. Le 🔴 est levé (flow cartes propre à travers Master), TVA_Vente / verrou « réservée » / acheteur / écriture AG OK.

### Reste à tester
- ~~Clôture événement 🏁 (archive + reset buffers)~~ — ✅ **validée en #9** (flux complet rejoué : archive + vidage des buffers OK). **Réaffirmé #13** : le chemin nominal de clôture est testé et validé — à ne PAS re-classer en bug lors des revues d'ouverture (les remarques #12 sur le drain en échec / l'archive A→G sont des durcissements optionnels, non bloquants).
- Autres configurations cartes : cartes gradées, articles manuels *(toujours en attente)*

### 🔑 Douchettes en attente
- À la réception : séance de scan pour associer un vrai EAN à chaque article en stock
- Même procédure à chaque nouvel article entrant en stock


### B. Blocs « Livré en session… » (ex-§6 Prochaines Étapes)

### ✅ Livré en session #6 (v1.6.0)
1. ~~**[Sales] Choix du type de vente (TVA_Vente)**~~ — fait. Source = `Scelle_Transactions!AG8:AG23`, écrit en AG (scellé) / AF (carte).
2. ~~**[Sales] Choix de l'acheteur**~~ — fait via modèle Master-autoritaire (acheteur par défaut publié dans `app_state` col D ; sinon saisie vendeur). La détection Levenshtein d'uniformisation est **descopée** (le modèle Master-autoritaire supprime la divergence à la source) → reléguée au backlog si un jour besoin.

### ✅ Correctifs livrés en session #7 (revue de code)
3. ~~**🔴 Master `applyCardSale`**~~ — fait (1.6.2). Bloc `copyPaste` parasite supprimé. Débloque le flow cartes.
4. ~~**🟠 Master index `Synced`**~~ — fait (1.6.2). `r[14]` (col O) dans `renderSalesLog` et `loadCloseSummary`.
5. ~~**🟡 Intake**~~ — fait (1.0.0). CRLF normalisé, en-tête cartes dynamique, `parsePrix` locale belge.

### ✅ Validé en session #7
6. ~~**Test E2E (Sales 1.6.3 / Master 1.6.2), ventes cartes + scellé**~~ — **concluant**. 🔴 levé, TVA_Vente / verrou « réservée » / acheteur / écriture AG OK.

### ✅ Livré en session #8
7bis. ~~**#6 Arrondi « ± Total »**~~ — fait (#8, Sales 1.7.0) : plus grand reste, euros entiers, total exact, sur tous les articles.
8bis. ~~**#7 Capture des logs**~~ — fait (#8) : `logEvent` persistant + `exportLog()` (Sales & Master).
9bis. ~~**#16 Purge code mort Sales**~~ — fait (#8).
10bis. ~~**Robustesse Sheets**~~ — fait (#8) : renommage onglet + helper `a1()`, `HttpError`, détection dynamique colonnes cartes (Sales).
11bis. ~~**Outbox idempotent Sales**~~ — fait (#8, 1.8.0/1.8.1) : zéro perte, zéro doublon, bascule `anomalies` + dead-letter.

### ✅ Livré en session #9 (sécurité + Option 1)
- ~~🔴 **Corriger l'Intake (renommage onglet)**~~ — fait (1.1.0). `SHEET_NAME` → `cartes_stock_et_transactions`.
- ~~**Sécurité Intake**~~ — fait (1.1.0). `esc()` (XSS) + `safeForSheets()` (injection formules).
- ~~**Option 1 : vendeurs sans accès CENTRAL**~~ — fait (Master 1.9.0 + Sales 1.9.0). Vue `STOCK_PUBLIC` publiée par Master, Sales ne lit plus CENTRAL.
- ~~🧪 **Test E2E Option 1**~~ — **concluant**, jusqu'à la **clôture d'événement** 🏁 (archive + vidage des buffers). La clôture, en attente depuis #8, est **validée**. Option 1 prête à la prod.

### ✅ Livré en session #10 (refonte UI Sales — 1.9.2 → 1.9.5, patch Sales seul)
- ~~**Refonte UI 3 zones smartphone**~~ — en-tête (logo PNG + titre + pastilles d'état G/📦), milieu carrousel **swipe panier ⇄ caméra**, bas client/type/total/valider épinglé.
- ~~**Scanner caméra 📷 en fallback (#4)**~~ — viseur caméra accessible par swipe, décodage natif `BarcodeDetector` (codes-barres **et** QR), repli gracieux si non supporté (douchette = chemin principal). *Non testable hors navigateur — à valider sur le téléphone (HTTPS GitHub Pages OK).*
- ~~**Quantité/contrôles à gauche de la ligne (#10)**~~ — 🗑 + chip quantité tappable à gauche (position fixe), **prix unitaire éditable** par ligne.
- ~~**Rouleaux → saisie clavier**~~ — quantité / prix unitaire / total au clavier (`inputmode`).
- 🟡 **Roll-back de vente** — esquissé puis **retiré** pour livrer une 1.9.5 propre → reporté en jalon coordonné (voir priorités, backlog #27).

### ✅ Livré en session #11 (palier 1.10.0/1.10.1 — Sales + Master)
- ~~**Refonte UI Master (en-tête, blur connexion, panneaux)**~~ — fait (Master 1.10.0).
- ~~**Verrou d'instance (lease/TTL) anti-double-master**~~ — fait (Master 1.10.0, `app_state` C/D = uuid/label).
- ~~**Heartbeat Sales par détection de changement + warning non bloquant**~~ — fait (Sales 1.10.0).
- ~~**Cleanup cols U/AD (scellé)**~~ — fait (Master 1.10.0).
- ~~**Dashboard Master one-screen (maquette v6)**~~ — fait (**Master 1.10.1**) : KPI auto-shrink, 2 donuts catégorie/licence, faits marquants, CA/heure horizontal, aides « ? ». Voir §5.


### C. CardMarket — approche de matching par NOM (abandonnée le 06/06/2026)

> Remplacée par le matching **par numéro** (cf. §6). Conservé pour comprendre la genèse : analyse du catalogue `products_singles_6.json`, barrière de langue FR↔EN, et le pipeline Poképédia/TCGdex qui devait traduire les noms.

### Catalogue public CM (analysé le 04/06/2026 — `products_singles_6.json`)
- idGame **6 = Pokémon**, **70 144 produits** (singles uniquement), daté 03/06/2026. Schéma : `idProduct · name · idCategory(51=Pokémon Single) · categoryName · idExpansion · idMetacard · dateAdded`.
- ✅ **`idProduct` unique → LA clé.** **753 `idExpansion`** (niveau set, plus fin que les ~150 sets officiels). **16 613 `idMetacard`** (regroupe les réimpressions/variantes d'une même carte).
- ⚠️ **Pas de numéro de collection** dans le fichier. Disambiguateurs entre crochets dans `name` (**75,7 %** des entrées). `dateAdded` inexploitable à **12 %**.
- 🔴 **(name + idExpansion) NON unique : 33,86 %** des produits (9 803 couples en doublon intra-extension). **Vérifié : les collisions partagent le même `idMetacard`** → ce sont des **variantes (rareté/illustration)** que **seul CM départage** → ces cas-là partent en désambiguïsation humaine / deep-link.
- Compagnons à récupérer sur la page **Data** de CM : **price guide** (indexé `idProduct`, pour le repricing plus tard) + **table des extensions** (`idExpansion → nom`).

### 🔴 Découverte structurante — barrière de langue FR ↔ EN
- **Le stock est en FRANÇAIS** (« Pomdrapi V », « Exagide V », « Rugit-Lune », « Spray Dés-Evoluant »…), le **catalogue CM en ANGLAIS** (« Weedle », « Kakuna »…).
- **Mesuré sur les 4 095 cartes en stock dispo** : seules **24,3 %** matchent par nom normalisé dans le catalogue CM ; **75,7 % AUCUN match.** ⇒ Le pipeline « filtre idExpansion → match nom normalisé » **ne marche PAS seul**.
- ✅ **Le pont (retenu) : série interne + numéro de collection → base multilingue indexée par (set+numéro) → nom EN → match dans le bon `idExpansion` → `idProduct`.**
  - **81,6 %** des noms en stock contiennent le **numéro de collection** (ex. « Tutankafer Ex 076/182 ») → le numéro devient la **clé du pont**, pas une simple aide à la vérif.
  - Base candidate : **Poképédia** (déjà interrogée par `tcg-intake` !) et/ou **TCGdex** (multilingue, indexé numéro). `tcg-intake.html` contient déjà `fetchPokepedia` / `buildPokepediaIndex` / `lookupPokepedia` / `buildCardNumber` → **logique à extraire/réutiliser** dans `tcg-integrator`.

### ✅ Validation du parser FR sur données réelles (04/06/2026)
Parser `(série + numéro + base)` testé sur les **4 015 cartes Pokémon en stock** (80 One Piece écartés) :
- **99,9 % des cartes individuelles (4 002/4 006) produisent une clé `(série+numéro)`.** Répartition du numéro : `NNN/NNN` complet **3 348** · promo type `XY163` **452** · `NNN/` (dénominateur manquant à la source) **198** · numéro nu **4**. → **le design du pont tient, FR-side verrouillé.**
- **Règle « lots en vrac » :** 9 entrées « Lot de … » (ex. « Lot de 2.000 co/uncos ») ne sont **pas** des produits CM individuels → l'app les **détecte (`^lot`) et les route hors flux singles** (bundle / exclusion). Catégorie à part, pas une exception.
- **Règle segmentation jeu :** détecteur One Piece à compléter avec le format promo **`(P-NNN)`** (3 cartes fuyaient). Après quoi, ~0 faux Pokémon.
- **Qualité de données (chez LGT) :** **198 cartes** ont un numéro tronqué `NNN/` (dénominateur jamais saisi). Matching OK via numérateur+série, mais à **compléter à la source (Intake)** idéalement.
- Parsing du numéro à gérer : `NNN/NNNN`, `NNNa/NNNN` (lettre, ex. `21a/145`), `NNN/` (partiel), promo `XY163`, numéro nu (`Artikodin ex 032`). Crochets `[Attaque | Attaque]` à retirer avant extraction.

### Résolution de l'`idProduct` (matching CardMarket)
- **Une fois par ÉDITION** (cache sur la définition de carte), pas par exemplaire → effort borné et décroissant.
- **Pipeline** : segmenter Pokémon → résoudre identité canonique via **série+numéro → Poképédia/TCGdex** → obtenir **nom EN** → filtrer `idExpansion` → match nom EN → **auto-lien si unique**, sinon **désambiguïsation humaine**, sinon **file d'exceptions** (extension non mappée, set récent, libellé divergent, jeu non Pokémon).
- Persistance sur la définition de carte : **`cardmarket_idProduct`** (nullable, backfill au fil de l'eau) + flag **`verified`** (auto-lié vs confirmé humain).
- **Politique de validation (décidée 04/06/2026) :** l'`idProduct` est **TOUJOURS affiché et modifiable**, même quand le moteur a trouvé un match certain à 100 %. L'override propose la **liste des `idProduct` CM encore disponibles**. ⚠️ Nuance à cadrer : « disponible » = **non encore assigné à une AUTRE édition** du stock LGT (éviter qu'une même clé serve à deux cartes différentes) — mais **plusieurs exemplaires d'une même édition partagent légitimement le même `idProduct`** (l'assignation est au grain ÉDITION, pas exemplaire). La liste de candidats à l'override doit donc se restreindre au bon `idExpansion` et exclure les `idProduct` déjà liés à d'autres éditions, pas les exemplaires.

### Interface de désambiguïsation (texte, sans images)
- **Gauche = ta carte** (stock) : bloc, code+nom série, **numéro**, variante/type, état, qté, **prix de vente**. **Droite = candidats** (catalogue) : `name` EN avec crochets + nom d'extension CM + `idProduct`.
- Images abandonnées pour l'instant. Résidu vraiment ambigu (variantes même libellé) → **deep-link CM `?idProduct=N`**, choisir sur CM, **récolter l'`idProduct` depuis l'URL**.

### Process de publication CM (sans API)
- **Mapping (in-app) ≠ publication (sur CM)** : le mapping ne transmet rien à CM ; la publication exige une interaction CM.
- Deux voies : **fiche produit directe** (deep-link → vendre → état/prix/qté → submit ; déterministe, 1 carte) · **formulaire bulk ≤ 100 articles** (recherche par nom ; l'`idProduct` sert de **certitude sur la ligne à choisir**). **Submit humain** dans tous les cas.
- **Lots par `idExpansion`** (ergonomie ; seule règle dure = 100 articles). Modes de sélection (découplés du traitement) : lot **spécifique** (achat récent) · **top-N valeur** (ex. 100 plus chères au 1ᵉʳ upload — ⚠️ alt-art/SIR = plus ambiguës ET plus coûteuses → **confirmation renforcée**) · lot **mono-série**.
- **State machine par carte/canal** : `en stock → sélectionnée → (CM) idProduct résolu / (Vinted-FB) annonce générée → publiée sur <canal>` (+ anti-doublon par flag « publié sur <canal> »).

### Modèle de données (définition de carte — à ajouter)
- Références externes : **`cardmarket_idProduct`** + **`verified`** ; prévoir d'emblée le **slot générique** pour `whatnot_id`, `vinted_listing_id`, `fb_marketplace_id`.
- Flags **« publié sur <canal> »** par canal (anti-doublon, scan-blocking pour les items vendus sur CM).
- **Plancher de prix `Pa`** (déjà en stock, col V) pour le monitoring de prix / repricing.
- ⚠️ Commission CM (~3 %) à intégrer à la compta avec tag `source=CM` ; second-writer protocol sur CENTRAL pour ventes CM autonomes ; secrets jamais en clair (repo public).

### Séquencement de construction (API-indépendant d'abord)
0. ✅ **v0.1.0 livré (04/06/2026)** — coquille mono-page + **auth GIS** (modèle token, popup au tap, refresh silencieux — pattern Sales/Master) + **CSP** (Sheets+OAuth uniquement à ce stade) + **lecture LIVE du stock** depuis CENTRAL (`cartes_stock_et_transactions`, détection d'en-tête dynamique ligne ~143, scope **lecture seule**) + **parser FR validé** (segmentation jeu avec correctif `(P-NNN)`, routage des lots, extraction base·numéro·suffixes) + tableau de contrôle (single/lot/autre-jeu/exception). Parser JS re-vérifié sur données réelles : **4 002 singles · 9 lots · 1 exception (« Lucario ») · 83 autres jeux**. Aucune écriture. ⚠️ À tester en navigateur (auth GIS + lecture). **CSP à étendre en v0.2.0** (pokepedia + proxies CORS).
1. **Modèle de données** + table `série interne → idExpansion` + ingestion `products_singles_6.json` en table de référence (idProduct PK). *(catalogue 12,8 Mo : décider du mode de chargement navigateur — asset hébergé trimmé vs upload in-app.)*
2. **Moteur de matching** (pont Poképédia/TCGdex, normalisation FR, segmentation jeu, auto-lien/exceptions) — **le gros morceau, le plus à risque · modèle Opus**.
3. **Interface de désambiguïsation** texte + **plancher de prix** + (plus tard) capture photo.
4. **Génération du formulaire** de publication par `idExpansion`.
5. *(Futur)* passerelle / API CM si elle rouvre → publication programmatique (`POST stock` par `idProduct`), le cache d'`idProduct` rend le basculement instantané. **L'effort d'aujourd'hui n'est jamais perdu.**

---

## Annexe A — Structure des onglets GSheet

> ⚠️ **À vérifier régulièrement** contre le sheet live, **surtout après une session où une colonne a été créée/déplacée**. Les apps ciblent les colonnes par **nom d'en-tête** quand c'est possible (robuste au décalage) ; les positions ci-dessous sont un relevé, pas une garantie. En cas de doute, **le code fait foi**.

### A.1 — `cartes_stock_et_transactions` (onglet cartes, CENTRAL)
**En-tête dynamique** : actuellement **ligne 143** (le code la détecte via `findCardsHeaderRow`, ne pas coder en dur). Colonnes utiles relevées le 04/06/2026 :

| Col | Contenu | Note |
|-----|---------|------|
| D | **Key** | SKU interne / clé de jointure (ex. `___STutankaferEx076182NM1.5`) ; clé du write-back idProduct |
| G | **Code Série** | `EV4.0`, `WOTC1.0`, `Promos`… |
| H | Nom de la série | |
| I | **Nom de la Carte** | numéro de collection inclus (FR) |
| J | Type_Carte | |
| K | Etat | NM / EXC / GD… |
| L | **idProduct CM** | cible du write-back CardMarket (résolu par en-tête, repli col L) |
| O | **Variantes** | shadowless / oversized / stamps… (indistinguables au catalogue, départagés ici) |
| P | **Statut Carte** | **vide = en stock** |
| Q | **Prix de vente** | lu par l'Integrator (figé au chargement) |
| V | **Prix d'achat (Pa)** | plancher de prix / repricing |
| AF | **TVA_Vente** | régime fiscal côté cartes (≠ AG des scellés) |

> Volumétrie (04/06/2026) : **4 095 cartes dispo** (statut vide) ; vendues 7 757 ; échangées 228 ; volées 111. Colonnes A→C, E→F, M→N, R→AE non documentées ici → à relever si besoin.

### A.2 — `Scelle_Stock` (état courant, CENTRAL)
**Header dynamique** : actuellement en ligne 26 (lignes 1–25 = sections/totaux). Le code scanne la première ligne dont col C = "Key" et itère à la suivante.

| Col | Contenu | Note |
|-----|---------|------|
| A | Origine | Sous-compte (Stock_Pro, Collection_Privée, etc.) |
| B | Code Série | |
| C | **Key** | Barcode EAN — clé primaire scan (vide jusqu'à séance scan douchette) |
| D | Nom Série | |
| E | Type_Item | |
| F | Nom_Item | |
| G | **Quantité** | Qty disponible (gérée manuellement) |
| H | Etat | NM, GD, etc. |
| I–K | CM / Vinted / Ebay | Cotes marché |
| L | **Côte/u** | Prix de vente affiché |
| M | Côte Totale | |
| N | Statut | Vide pour les articles en stock |
| O | Prix de vente | |
| P | Acheteur | |
| Q | Source | |
| R | Date_Transaction | |
| S | **Facture d'achat** | Clé du lot (ex: DMA25-A-BE-0018) |
| T | Facture de vente | |
| U | Commentaire | |
| V | Type_Vente | |
| W | **Prix d'achat** | Total Pa du lot restant (proportionnel aux unités restantes) |
| X | Marge | |
| Y | % Marge | |
| Z | (vide) | |
| AA | Code Technique | Ancien identifiant (buyer\|\|date) ; ⚠️ utilisé par le roll-back (`SEALED_TX_TXID_COL`) côté `Scelle_Transactions` |
| AB | Date_Vente | |
| AC | Facture_Vente | (référence session/event) |
| AD | Pays_Vente | |
| AE | TVA_Achat | (position historique — à confirmer) |
| AF | — | (à confirmer contre le sheet live) |
| AG | **TVA_Vente** | Régime normal / intracommunautaire / cadeau / **réservée** (= verrou). Liste curée : `Scelle_Transactions!AG8:AG23` |

> ⚠️ Table antérieure aux corrections, **décalage possible A→AF**. **Vérifié** : `TVA_Vente` est en **AG** (scellés) et en **AF** (cartes). La ligne de transaction écrite par Master a **A = « x »** en tête ; son ordre exact est défini par `txRow` dans `applySealedSale` — **le code fait foi**.

### A.3 — `Scelle_Transactions` (historique, CENTRAL)
- Même colonnes que `Scelle_Stock`. 1 ligne par mouvement : vente (Statut=vendu), vol, ouverture, échange, transfert, cadeau.
- Alimenté manuellement (historique) + automatiquement par TCG Master à chaque vente convention.
- ⚠️ **col `AA`** porte l'estampille `txId` du roll-back (`SEALED_TX_TXID_COL`) — confirmer qu'elle n'est pas lue par la compta (cf. §1).
