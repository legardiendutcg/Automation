# Contexte Projet — Le Gardien du TCG
> Document à coller en premier message dans chaque nouvelle session Claude.
> Dernière mise à jour : **04/06/2026 — tcg-integrator v0.1.0 livré** · **Versions actuelles : Sales 1.14.0 / Master 1.14.0 (roll-back de vente, #27) / Intake 1.2.1 (ex-`tcg-converter`) / Integrator 0.1.0 (NOUVEAU)**
> · **Cadrage CardMarket approfondi + ossature multi-canal — 04/06/2026** (voir **§6bis**)
>
> 🔁 **Renommage acté (04/06/2026) :** `tcg-converter.html` → **`tcg-intake.html`** (titre interne « TCG Intake », logo « TCG INT » conservé). Le nom **« TCG Integrator » est désormais réservé à la NOUVELLE app de publication `tcg-integrator`** (sortie multi-canal — voir §6bis). Les apps du projet sont donc : **Sales** (vendeurs), **Master** (gérant), **Intake** (entrée de stock, ex-converter), **Integrator** (sortie/publication vers canaux — à construire).
>
> ⚠️ **Lecture prioritaire — Session #13 : 3 jalons coordonnés livrés** (1.12.0 auth GIS · 1.13.0 vidage panier + durcissement auth + panneau comptoir · 1.14.0 roll-back de vente #27), **tous `node --check` OK mais AUCUN testé en navigateur.** 👉 **Checklist de test navigateur complète en fin de §0** (à faire avant prod ; déployer **Sales 1.14.0 + Master 1.14.0 ENSEMBLE**, Ctrl+Shift+R). Détails dans « ## 5. État Actuel — Session #13 ».
>
> 🎯 **Priorité du moment — construire `tcg-integrator` (publication multi-canal), CardMarket en premier.** Cadrage technique approfondi le 04/06/2026 sur fichiers réels (catalogue CM 70 144 singles + export stock 4 095 cartes dispo). **Découverte structurante : stock en FRANÇAIS vs catalogue CM en ANGLAIS → 75,7 % des cartes Pokémon ne matchent PAS par nom seul.** Le pont retenu = **série interne + numéro de collection → Poképédia/TCGdex (multilingue, indexé numéro) → nom EN → `idProduct` CM.** 👉 **Détail complet en §6bis.** *(Le backlog Sales/Master/Intake reste en pause — cf. §5.)*

---

## 0. Instructions pour Claude

- **Tout au long de la discussion** : mettre à jour ce document en arrière-plan au fil des avancées — ajouter les nouveaux milestones complétés, les nouvelles idées de développement à garder pour plus tard, et retirer du backlog les améliorations sur lesquelles on a travaillé et validées.
- **Quand il est opportun d'ouvrir une nouvelle discussion** (contexte trop long, session de travail cohérente terminée) : prévenir, mettre à jour ce document une dernière fois, puis proposer de basculer.
- **En début de chaque session**, lorsqu'une étape du développement est identifiée : indiquer quel modèle Claude est le plus adapté pour cette étape (Haiku / Sonnet / Opus) avec une justification courte.
- **Au début de chaque nouvelle discussion** : lire les 3 fichiers HTML (`tcg-sales.html`, `tcg-master.html`, `tcg-intake.html`) pour assimiler l'intégralité du code, puis faire une passe de revue systématique — triple-check à la recherche de bugs, repérage des incohérences et du code/variables devenus inutiles (code mort), et analyse des opportunités d'amélioration de **performance** (en convention/événement, la **réactivité** est critique). Remonter les trouvailles avant de coder.
- **À chaque livraison de fichier** : bumper systématiquement la version (patch/minor/major selon le changement) avant de présenter le fichier. Ne jamais livrer deux fichiers successifs avec le même numéro de version. Quand les deux apps sont modifiées ensemble, elles partagent le même numéro de version pour garder un mapping cohérent entre Sales et Master.
- **Bilan de la session #9** : revue de **sécurité** des 3 apps (OWASP 2025 + doc « Quand l'application grandit »), puis correctifs validés.
  - **Intake 1.1.0** : ajout `esc()` (corrige XSS DOM via CSV/scraping) + `safeForSheets()` (corrige injection de formules) ; **+ correction du renommage onglet** `cartes_stock_et_transactions` (priorité #9.1 — il cassait dessus).
  - **Option 1 (Master 1.9.0 + Sales 1.9.0)** : Master publie une vue stock **sûre** (`STOCK_PUBLIC` dans EVENTS, sans coûts/marges) ; Sales lit le stock de là et **n'accède plus du tout à CENTRAL** → les vendeurs n'ont plus besoin d'accès à CENTRAL (marges jamais exposées). Confirmé : 2 GSheets en « Restreint ».
  - ⚠️ **Versioning** : l'Option 1 ayant modifié Sales **et** Master ensemble, ils partagent **1.9.0** (réalignement des pistes). Du coup **l'auth GIS, qui visait 1.9.0, glisse en `1.10.0`.** Versions à jour : **Sales 1.9.0 / Master 1.9.0 / Intake 1.1.0**.
  - ✅ **Test E2E #9 concluant** : Option 1 validée de bout en bout (Master 1.9.0 publie `STOCK_PUBLIC`, Sales 1.9.0 lit le stock sans accès CENTRAL, ventes carte + scellé, sync Master) **jusqu'à la clôture d'événement** 🏁 (archive + **vidage des buffers** `ventes_detail`/`anomalies`) — la clôture était en attente depuis #8, elle est désormais validée. **Option 1 prête à la mise en ligne.**
- **Au début de la prochaine session** :
  - **Priorité 1 — auth Google Identity Services (v1.12.0, Sales + Master)** : fin de la déconnexion ~1 h ; jeton en mémoire seule (plus en `localStorage`). ⚠️ Config GCP : origine `https://legardiendutcg.github.io`. *(couvre R4 ; recalé de 1.10.0 → 1.11.0 → 1.12.0, les minors étant consommés par les paliers #11 et #18.)*
  - **Priorité 2 — module commun `tcg-common.js`** (#24) : extraire les helpers dupliqués (`esc`, `safeForSheets`, `a1`, formateurs, `HttpError`…) dans un fichier partagé. ⚠️ **Décision à valider** : passe de **3 fichiers autonomes à 4 fichiers interdépendants** (le commun doit être déployé à côté). Diff des 3 copies de chaque helper avant fusion + re-test navigateur. *(Distinct de l'Option 3, qui est de la sécurité, pas du rangement.)*
  - **Priorité 3 — passe Master (suite)** : colonnes cartes dynamiques côté **écriture** (#19) ; refonte **centimes** entiers complète (#20, le gros reste après l'acompte #10).
  - **Priorité 4 — roll-back de vente (Sales + Master, jalon coordonné, #27)** : écriture compensatoire append-only (`txId` `-RB`, négatifs) ; **spécifier Master d'abord**.
  - **Puis** : cartes gradées / articles manuels ; **Option 3 — passerelle Apps Script** (#25, gros chantier d'architecture, **à faire APRÈS l'auth GIS** car ils interagissent) ; R5 (scope `drive.file`+Picker) ; SRI (seulement si un jour une lib via CDN).
  - 🧪 **À TESTER EN NAVIGATEUR — livraisons Session #13 (RIEN n'a encore été testé)** · *déployer Sales 1.14.0 + Master 1.14.0 ENSEMBLE, Ctrl+Shift+R* :
    - **A. Auth GIS + CSP (1.12.0)** — (1) connexion Google sur **Sales** (tap pastille G) + une **vente réelle** ; (2) connexion **Master** + un **tick de sync réel** ; (3) **console F12 = AUCUN blocage CSP** (`script-src`/`frame-src accounts.google.com`) ; (4) **reload** → la popup ne s'auto-déclenche plus, connexion **au tap** ; (5) **app ouverte > 1 h** → **refresh silencieux**, pas de déconnexion ni popup intempestive ; (6) **popup bloquée** → message actionnable, pas d'échec silencieux ; (7) **Master compte non autorisé** → « accès refusé » + « Changer de compte ».
    - **B. Panier (1.13.0)** — (8) reload **< 30 min** → panier restauré, **> 30 min** → vide ; (9) **nouvelle version** déployée → panier vidé au reload ; (10) **changer le nom d'event** côté Master pendant qu'un panier est rempli → panier **vidé** (snapshot 60 s restaurable) ; (11) **logout** → panier vide.
    - **C. Panneau comptoir Master (1.13.0)** — (12) Master **AVEC** nom d'event → « Faits marquants » ; **effacer** le nom d'event → « 🏠 Récap comptoir » + **swipe** Articles du jour ⇄ Clients du jour (points cliquables).
    - **D. Roll-back (1.14.0)** — (13) **vente carte** → ↶ → carte de nouveau dispo, **Q = prix listé d'origine** (PAS le prix remisé), AF « pas vendu » ; (14) **vente scellée** → ↶ → ligne de transaction **supprimée** + **qty du lot remontée** ; (15) **↶ AVANT sync Master** → **paire neutralisée**, CENTRAL intact (anomalies `RB_CANCELLED`, **pas** de `ROLLBACK_APPLIED`) ; (16) **même article scellé vendu 2×** au même event dont **une remisée** → annuler la remisée → la **BONNE ligne** part (txId estampillé en `AA`) ; (17) **confirmer que la colonne `AA`** de `Scelle_Transactions` n'est pas lue par la compta (sinon changer la constante `SEALED_TX_TXID_COL`).
    - **E. Reliquats #12 (jamais testés)** — (18) **dédup txId** : dupliquer une ligne `ventes_detail` (même txId+clé) → `DUP_SKIPPED` + anomalie `DUPLICATE_TXID` ; (19) **acheteur imposé** : event rempli → champ verrouillé pour tous, vide → champ libre obligatoire ; (20) **Intake 1.2.1** : un import test écrit bien dans `cartes_stock_et_transactions` ; (21) **verrou 2 appareils** : Master sur 2 appareils → lease/TTL, « Forcer la reprise », anti-ping-pong.
  - 🔧 **Hardening recommandé (n'affecte pas le fonctionnement)** : protéger l'onglet `STOCK_PUBLIC` (Données > Protéger → seul le propriétaire édite) et ajouter les vendeurs avec leur **propre compte Google** (Éditeur EVENTS, aucun accès CENTRAL) plutôt qu'un login master partagé.
- 🧪🏠 **Test de simulation maison — samedi 13 juin 2026 (répétition générale, AVANT conditions réelles)** :
  - **Cadre** : test « pour du beurre » à domicile avec **2 amis** → **3 appareils, plusieurs vendeurs connectés en même temps**, **ventes fictives**, sur un **fichier de stock DÉDIÉ** (surtout PAS la GSheet officielle dont dépend toute l'activité) → on peut tout casser sans risque. Objectif : éprouver le **process complet** en conditions réalistes et se mettre volontairement dans un maximum de situations différentes. **C'est de fait la première validation navigateur des livraisons #13** (checklist A–E ci-dessus) → s'appuyer dessus.
  - **Prérequis & smoke-test** : base EVENTS nette (`ventes_detail` + `anomalies` vidés, `STOCK_PUBLIC` à jour via un refresh cache Master, `app_state` prêt) ; **déployer Sales 1.14.0 + Master 1.14.0 ENSEMBLE** ; **la veille ou le matin** (à froid) : connexion Google **au tap** sur chacun des 3 appareils + une première vente simple qui atterrit, après `Ctrl+Shift+R`. Si ça passe sur les 3 téléphones, le gros du risque #13 est levé avant l'arrivée des testeurs.
  - **⚠️ À NE PAS prendre pour des bugs (conflits attendus)** : le stock vendeur vient du cache `STOCK_PUBLIC`, rafraîchi seulement quand Master republie (~5 min, ou après un roll-back). Donc 2 vendeurs **peuvent** scanner la même carte unique / le même dernier scellé depuis leurs caches respectifs ; c'est **le Master qui tranche à la sync** (1er gagne, le perdant tombe en anomalie `P_NON_BLANK` pour une carte / `SEALED_OVERDRAW` pour un scellé). **Comportement anti-survente voulu** → à provoquer exprès, c'est le meilleur test du filet anti-doublon.
  - **Scénarios à provoquer délibérément** (couvrent pile A–E) :
    - **Conflits concurrents** — 2 vendeurs vendent la même carte unique ; 2 vendeurs vident la dernière unité d'un même scellé → on doit voir **1 vente OK + 1 anomalie visible** côté Master.
    - **Roll-back, ses 4 états** — ↶ **AVANT** sync Master (paire neutralisée, `RB_CANCELLED`, CENTRAL intact, **pas** de `ROLLBACK_APPLIED`) ET ↶ **APRÈS** sync (inversion réelle, `ROLLBACK_APPLIED`, article de nouveau vendable) ; **une fois carte, une fois scellé** → valide aussi l'estampille txId en col `AA`. *(items 13–16)*
    - **Verrou d'instance** — ouvrir le Master sur **2 appareils** → le 2nd passe en lecture seule + overlay, « Forcer la reprise » bascule proprement (piège classique : dashboard ouvert sur téléphone *et* iPad). *(item 21)*
    - **Cycle de vie panier + acheteur** — logout → panier vidé ; changer/effacer le nom d'event pendant un panier rempli → panier **vidé** (snapshot 60 s restaurable) ; tester les **deux modes** : event rempli (acheteur **imposé**, identique aux 3) **vs** event vide (chacun saisit son client + panneau **« 🏠 Récap comptoir »** au lieu des « Faits marquants »). *(items 8–12, 19)*
    - **Réseau dégradé** — couper le wifi d'un téléphone en pleine vente → la vente part en **file offline** et se **flushe au retour**, **sans perte ni doublon** (relecture par txId pour le cas « envoyé mais réponse perdue »).
  - **Convention réelle (1res vraies ventes) prévue les 21 et 22 juin 2026** → le 13 juin est la répétition générale ; tout 🔴/🟠 **bloquant** repéré le 13 doit être corrigé dans la fenêtre du 14 au 20 juin.
  - **Priorité produit actuelle** : les améliorations restantes (priorités 2–4 ci-dessus, refactor perf `applySealedSale`, bug cosmétique de l'icône du journal des ventes #18/#27, code mort) sont **EN PAUSE** au profit de l'**ouverture du canal CardMarket** (canal de lancement : 3 % frais pro, audience EU dominante). Le dev reprendra après le lancement et/ou selon les retours du test du 13 juin.
- **Vigilance sur les caracteres speciaux** lors du traitement de donnees provenant des Google Sheets (locale belge) : les virgules (separateur decimal vs separateur de milliers), les apostrophes (droite ' U+0027 vs courbe ' U+2019 que Google Sheets insere souvent automatiquement), et les accents (e/E vs e/E, etc.) sont des sources frequentes de bugs silencieux (find() qui renvoie -1, parseFloat qui interprete mal le format, etc.). Toujours considerer ces cas lors des comparaisons de strings ou du parsing de nombres.
- Ces instructions permanentes s'appliquent automatiquement dans chaque session — inutile de les rappeler.

---

## 1. Qui & Pourquoi

Vendeur TCG (Trading Card Game) en Belgique (Wallonie), sous le nom **Le Gardien du TCG**.

**Produits vendus :**
- Cartes TCG individuelles (Magic, Pokémon, etc.)
- Articles scellés : boosters, displays, coffrets, decks

**Canaux de vente :**
- Conventions (événements ponctuels, plusieurs vendeurs sur le stand)
- Boutique / vente comptoir (à venir — workflow dédié à construire)
- Ventes à domicile / RMP (rendez-vous privés, client nommé)
- **Canaux externes via `tcg-integrator`** (voir §6bis) : **CardMarket** (priorité 1, ~3 % frais pro, modèle catalogue+`idProduct`), puis envisagés **Whatnot** (enchères live), **Vinted** et **Marketplace Facebook** (annonces texte libre, pas de catalogue/ID produit → modèle de publication différent — cf. §6bis).

**Comptabilité belge :**
- Articles achetés chez un **pro** → TVA classique (Régime normal)
- Articles achetés chez un **particulier** (d'occasion) → TVA sur la marge : base taxable = Pv − Pa (nécessite de connaître le Pa exact du lot)
- Comptable externe qui reçoit les données calculées par le système

---

## 2. Architecture Technique

### Constantes clés (présentes dans les deux apps HTML)
```javascript
const OAUTH_CLIENT_ID    = '774473792747-6ogjf5msksq995s0kssqo61b3co5rj96.apps.googleusercontent.com';
const CENTRAL_GSHEET_ID  = '1i6Ch5L9z1zQeye1BQ2mAfqSfxbrXijDJ4EX1isNbEKk';
const EVENTS_GSHEET_ID   = '1WxTPFgVUDGd94vWXAO6i4BSS0jePqiSwi890hb8c2_E';
const SHEET_CARDS        = 'cartes_stock_et_transactions'; // ⚠️ renommé en #8 (était 'Cartes_Stock_&_Transactions')
const SHEET_SEALED       = 'Scelle_Stock';
const SHEET_SEALED_TX    = 'Scelle_Transactions';
const SHEET_STOCK_PUBLIC = 'STOCK_PUBLIC';   // (#9) vue stock sûre publiée par Master dans EVENTS, lue par Sales
```
> **Helper `a1(sheet, rng)` (depuis #8, Sales + Master)** : encadre TOUJOURS le nom d'onglet de quotes simples (échappement A1). Toutes les plages passent par lui → robuste à un futur renommage avec caractère spécial. C'est le renommage de `SHEET_CARDS` qui a corrigé le `400 "Unable to parse range"` apparu après modification du nom d'onglet dans la GSheet.
> ✅ **`tcg-intake.html` corrigé en #12** — ⚠️ et **non en #9** comme l'indiquait par erreur ce document : le renommage (priorité #9.1) avait été **documenté comme livré mais jamais appliqué au fichier**, qui restait sur `Cartes_Stock_&_Transactions` → l'Intake était **cassé** (400 « Unable to parse range »). Réparé en **1.2.0** : `SHEET_NAME` → `cartes_stock_et_transactions` **et** helper `a1()` porté à **toutes** les plages (cartes + onglets LOG/BACKUP), comme Sales/Master.
> **(#9, Option 1)** Sales ne lit **plus** CENTRAL : Master y reste seul et publie un sous-ensemble sûr du stock dans `EVENTS › STOCK_PUBLIC`, que Sales consomme (voir « Communication inter-apps »).

### Google Sheets (base de données centrale — CENTRAL_GSHEET_ID)
- `cartes_stock_et_transactions` — stock + transactions cartes (renommé en #8 ; structure inchangée)
- `Scelle_Stock` — état du stock scellé actuel (header dynamique : ligne 26 actuellement, mais le code détecte automatiquement la première ligne dont col C = "Key")
- `Scelle_Transactions` — historique de toutes les ventes/mouvements scellés
- Onglet compta — agrège toutes transactions, calcule TVA et marges (formules pointent vers `cartes_stock_et_transactions` et `Scelle_Transactions`)
- `anomalies` (EVENTS_GSHEET_ID) — **schéma 11 colonnes A:K** : `Timestamp · TX_ID · Vendeur · Clé/Barcode · Nom article · Source · Type anomalie · Détails · Synced_From_Row · Resolved · Notes`. Écrit par Master (anomalies de réconciliation) **et** par Sales depuis #8 (échec de vente : une ligne/article, `Type anomalie = SALES_NOT_SYNCED`, payload JSON complet dans `Notes` de la 1ʳᵉ ligne pour rejeu manuel).

### Google Sheets événements (EVENTS_GSHEET_ID)
- Onglets temporaires créés à chaque convention : `ventes_detail`, `heartbeat`, etc.
- Archivés et vidés à la clôture de l'événement (bouton 🏁)
- **`STOCK_PUBLIC` (#9, permanent)** — vue stock **sûre** publiée par Master (cartes + scellés disponibles, **sans coûts/marges**) : `Key · Kind(card/sealed) · CodeSerie · NomSerie · Nom · Type · Etat · PrixVente · Qty · Reserved` (A:J) + liste TVA_Vente brute en col L. **Source du stock côté Sales.** À **protéger** (seul le propriétaire édite — les vendeurs sont Éditeurs sur EVENTS).

### Hébergement & déploiement
- **GitHub Pages** (gratuit, déploiements illimités, HTTPS natif)
- Repo public : `https://github.com/legardiendutcg/Automation`
- URLs des apps :
  - `https://legardiendutcg.github.io/Automation/tcg-sales.html`
  - `https://legardiendutcg.github.io/Automation/tcg-master.html`
  - `https://legardiendutcg.github.io/Automation/tcg-intake.html`  *(ex-`tcg-converter.html`)*
  - `https://legardiendutcg.github.io/Automation/tcg-integrator.html`  *(NOUVELLE app de publication multi-canal — à construire, voir §6bis)*
- Workflow de mise à jour : Add file → Upload files → écrase l'ancien → Commit. Rebuild auto ~1 min. **Ctrl+Shift+R** sur l'app pour bypass cache navigateur.

### TCG Sales (app slave — vendeurs)
- Fichier : `tcg-sales.html`
- Usage : ouvert sur le **téléphone** de chaque vendeur pendant une convention (UI optimisée smartphone depuis #8 ; zoom autorisé)
- **(#10, refonte UI 1.9.2→1.9.5)** Interface **3 zones tenant dans un écran sans scroll global** (`100dvh`, `overflow:hidden`) : **HAUT** = logo carré (PNG embarqué) + titre « TCG SALES » (version collée dessous, alignée à droite) + pastilles rondes d'état **G** (Google) / **📦** (stock) cerclées vert/rouge/ambre (toucher = (dé)connexion / refresh ; détail en info-bulle) ; **MILIEU** (seule zone scrollable) = **carrousel swipe panier ⇄ caméra** (2 points), barre de scan fine, lignes article = 🗑 + quantité tappable (à gauche) + nom + **prix unitaire tappable** ; doublon = même ligne ×N, contour orange (carte ancienne clé) / **rouge bloquant** (carte clé neuve) ; **BAS** = client + type de vente (placeholders), **total tappable**, gros bouton « Valider la vente » épinglé (police Syne, identique au titre). **Saisie au clavier** (plus de rouleaux) pour quantité / prix unitaire d'un article / total panier (`inputmode` numeric/decimal). Le **prix unitaire** est éditable par ligne (corrige une côte GSheet périmée sans toucher les autres articles) ; le total reste éditable à part (répartition proportionnelle au centime). Icônes = emojis. *(Le roll-back de vente, esquissé puis retiré, est reporté en jalon coordonné — backlog #27.)*
- Flow : connexion OAuth → chargement cache **depuis `EVENTS › STOCK_PUBLIC`** → scan barcode → panier → validation → **outbox** → écriture `ventes_detail`
- **(#9, Option 1) N'accède plus à CENTRAL** (ni lecture ni écriture) : le stock est lu dans `STOCK_PUBLIC` (une lecture `A2:L`), publié par Master. Le vendeur n'a besoin que d'EVENTS. Le verrou « réservée » et le filtre TVA sont conservés (flag `Reserved` publié par Master).
- **Outbox idempotent (depuis #8)** : toute vente est d'abord mise en **file** (`pending`), puis envoyée ; l'entrée n'est retirée que sur **confirmation**. Avant de renvoyer une entrée dont l'issue est inconnue (`sent`), on **relit `ventes_detail!D` (colonne `txId`)** pour savoir si l'envoi a abouti → pas de doublon si la réponse réseau s'est perdue, pas de perte si l'envoi a échoué (≈ « exactement une fois »). Échec 4xx ou après `MAX_SALE_RETRIES` (5) → bascule `anomalies` ; si même ça échoue → **dead-letter localStorage** + bandeau rouge « Réessayer ». `txId` (col D de `ventes_detail`) = **clé d'idempotence** (Master devra la dédupliquer).

### TCG Master (app maître — gérant)
- Fichier : `tcg-master.html`
- Usage : ouvert sur l'iPad ou ordinateur du gérant pendant la convention
- Flow : lit CENTRAL (stock) → **publie `STOCK_PUBLIC`** (#9) → écoute `ventes_detail` (heartbeat) → synchronise le stock CENTRAL → gère l'événement
- **(#9) `publishStockView()`** : projette la vue sûre dans `STOCK_PUBLIC` à chaque `loadStockCache` (bootstrap, auto ~5 min, refresh manuel) ; capture le flag `reserved` (TVA_Vente : scellé AG / carte AF) ; non bloquant (un échec n'affecte pas le cache).

### Communication inter-apps
Les deux apps ne se parlent **pas directement** — elles communiquent via Google Sheets :
- **Stock (Master → vendeurs)** : Master lit CENTRAL et publie une vue **sûre** dans `EVENTS › STOCK_PUBLIC` ; Sales la lit. Aucune donnée de coût n'atteint les vendeurs.
- **Ventes (vendeurs → Master)** : Sales écrit dans `ventes_detail` (EVENTS) ; Master lit `ventes_detail`, résout par clé/`txId` et met à jour le stock CENTRAL.

### Modèle d'accès Google Drive (#9)
| | CENTRAL | EVENTS |
|---|---|---|
| Master (`legardiendutcg`) | Éditeur | Éditeur |
| Chaque vendeur (compte propre) | **Aucun accès** | **Éditeur** (dépose ses ventes ; lit `STOCK_PUBLIC`) |

> 2 GSheets en « Restreint » (vérifié #9). `STOCK_PUBLIC` à **protéger** (seul le propriétaire édite) ; la vue se ré-aligne de toute façon à chaque publication. Vendeurs à ajouter avec leur **propre compte** (pas le login master partagé → sinon jeton plein scope sur chaque appareil, cf. R4/R5 du bilan).

---

## 3. Structure des Onglets Scellés

### Scelle_Stock (état courant)
**Header dynamique** : actuellement en ligne 26 (lignes 1–25 = autre contenu type sections/totaux). Le code scanne automatiquement la première ligne dont col C = "Key" et démarre l'itération à la ligne suivante.

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
| AA | Code Technique | Ancien identifiant (buyer\|\|date) |
| AB | Date_Vente | |
| AC | Facture_Vente | (référence session/event) |
| AD | Pays_Vente | |
| AE | TVA_Achat | (position historique de cette table — à confirmer, voir note) |
| AF | — | (à confirmer contre le sheet live) |
| AG | **TVA_Vente** | Régime normal / vente intracommunautaire / cadeau / **réservée** (= verrou, §4). Liste curée des valeurs : `Scelle_Transactions!AG8:AG23` |

> ⚠️ **À revérifier (session #6)** — cette table (Scelle_Stock) est antérieure aux corrections et présente un décalage. Faits **vérifiés** : `TVA_Vente` est en **AG** sur les onglets scellés (liste autorisée en `Scelle_Transactions!AG8:AG23`), et en **AF** sur `cartes_stock_et_transactions`. La ligne de **transaction** écrite par Master a une colonne **A = « x »** en tête, et son ordre exact (différent de cette table : Key/Code Série notamment) est défini par `txRow` dans `applySealedSale` — **le code fait foi**. Les positions A→AF de cette table restent à confirmer contre le sheet live.

### Scelle_Transactions (historique, même structure de colonnes)
- Même colonnes que Scelle_Stock
- 1 ligne par mouvement : vente (Statut=vendu), vol, ouverture, échange, transfert, cadeau
- Alimenté manuellement (historique) + automatiquement par TCG Master à chaque vente convention

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

## 5. État Actuel — Session #13 (31/05/2026)

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


## 5bis. État Actuel — Session #12 (31/05/2026)

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

---

## 5ter. État Actuel — Session #11 (30/05/2026)
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

---

## 5quater. État Actuel — Fin Session #9 (29/05/2026)

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

---

## 5quinquies. État Actuel — Fin Session #6 (29/05/2026)

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

---

## 6. Prochaines Étapes (dans l'ordre)

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

### 🔥 Priorités suivantes (à challenger en début de session)
1. ~~**🟡 Acheteur IMPOSÉ côté Sales**~~ ✅ **fait en #12 (Sales 1.10.1)** — lecture seule quand le nom d'event est rempli (`#buyerLocked`, `getResolvedBuyer()` = nom d'event) ; vide → éditable + obligatoire.
2. ~~**#18 dédup `txId` (Master)**~~ ✅ **fait en #12 (Master 1.11.0)** — signature `txId|clé|source`, `DUP_SKIPPED` + anomalie `DUPLICATE_TXID`, cartes/scellés. *Filet, pas mutex (le bail reste la défense primaire). « Ignorer `SALES_NOT_SYNCED` » déjà satisfait par l'archi.*
3. ~~**Auth Google Identity Services (v1.12.0, Sales + Master)**~~ ✅ **fait en #13** — migration flux implicite → modèle token GIS ; jeton en mémoire seule (couvre R4) ; refresh silencieux 5 min avant expiration (fin de la déconnexion ~1 h) ; popup au lieu de redirection ; allow-list master + overlays + release verrou conservés. CSP : +`accounts.google.com` (script-src/frame-src). **⚠️ à valider navigateur** (connexion + refresh + 2 appareils). *(backlog #1, #2, #21 ; recalé 1.10.0 → 1.11.0 → 1.12.0.)*
4. **Module commun `tcg-common.js`** (#24) — extraire les helpers dupliqués. ⚠️ 3 → 4 fichiers (décision à valider) ; diff des copies avant fusion. *(Refactor, ≠ Option 3.)*
5. **Passe Master (suite)** : colonnes cartes dynamiques côté écriture (#19) ; refonte centimes (#20).
6. **Roll-back de vente (Sales + Master, backlog #27)** — append-only (`txId` `-RB`, négatifs) ; spécifier Master d'abord.
7. **Cartes gradées / articles manuels.** Puis **Option 3 — passerelle Apps Script** (#25, gros chantier, **après l'auth GIS**) ; R5 (scope `drive.file`+Picker) ; SRI si lib CDN un jour. 🔧 **Hardening Option 1** (protéger `STOCK_PUBLIC` + `app_state` ; vendeurs en comptes propres). *Cleanup mineur : CSS mort `.stat-*`/`.stats-grid`/`.split-*`/`.lb-*` dans Master (panneaux retirés en 1.10.1).*
> ✅ **#23 CSP** : faite en #12 sur les 3 apps (à valider navigateur). **SRI** : N/A (aucun `<script src>` tiers).

### Reste du backlog ordonné
9. **Workflow boutique / vente comptoir** (après convention)
10. **Nettoyage code mort + perfs réactivité** (backlog #12-16) — purge CSS/constantes mortes, rendu panier incrémental, lecture O(n) `Scelle_Transactions!B:B`.

---

## 6bis. `tcg-integrator` — Publication multi-canal (CardMarket en premier) 🎯

> 🎯 **Priorité du moment.** Construire **`tcg-integrator`** : l'app de **sortie** qui pousse les articles du stock vers des **canaux de vente externes**. **CardMarket** est le 1ᵉʳ canal ; l'ossature est pensée pour accueillir ensuite **Whatnot**, **Vinted**, **Marketplace Facebook**. App distincte de `tcg-intake` (entrée de stock) — on ne mélange pas entrée et sortie. Cadrage technique fait sur **fichiers réels** le 04/06/2026 ; **aucun code livré à ce stade**.

### Architecture multi-canal — ne pas sur-spécialiser sur CardMarket
Les canaux ne fonctionnent **pas** pareil, et ça structure tout le design :
- **CardMarket (et TCGplayer plus tard)** = **catalogue + identifiant produit** (`idProduct`). Le travail dur = **résoudre quel `idProduct` correspond à ma carte** (problème de matching, voir plus bas).
- **Vinted / Marketplace Facebook** = **annonces en texte libre**. Pas de catalogue, pas d'ID produit : on génère un **titre + description + photos + prix**. Aucune résolution d'ID — c'est de la **génération d'annonce**.
- **Whatnot** = **enchères/lots en live**. Pas de mapping article→catalogue ; logique de **lots** et présence caméra.

👉 **Ossature réutilisable (commune à tous les canaux)** : `sélectionner un lot du stock → PRÉPARER le payload spécifique au canal → revue humaine → submit (humain tant qu'il n'y a pas d'API) → marquer « publié sur <canal> » + stocker la référence canal`. La **résolution d'`idProduct` n'est QUE l'implémentation "préparer" de CardMarket** ; Vinted/FB auront une impl. "préparer" = génération d'annonce. Le publisher expose une **interface de canal** (prepare / generate / mark-published), les impls. changent.

### CardMarket — contexte & contrainte API
- **Pas d'API CardMarket.** Candidatures fermées → remplacées par des **fichiers publics** (catalogue produits + price guide), téléchargeables par tous, MAJ quotidienne, libres d'usage.
- **Lecture publique** (catalogue, prix) mais **écriture derrière l'API fermée** → **on publie sans API**, via l'interface CardMarket, avec **submit humain**.

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

### Périmètre — à acter
- **Multi-jeux dans le stock.** ~**79 cartes (1,9 %)** en stock sont du **One Piece** (codes `OP06-023`, `ST02-004`, `PRB01`, `EB03`…). Le fichier `products_singles_6.json` est **Pokémon only**. ⇒ **Phase 1 = Pokémon singles uniquement** ; `tcg-integrator` doit **segmenter par jeu** et écarter le non-Pokémon (catalogue One Piece = fichier séparé, même ossature, plus tard).
- **Scellés ≠ singles.** L'utilisateur veut aussi publier des **items scellés**. ⚠️ Le catalogue singles ne les couvre pas (catalogue **sealed** CM séparé) **et** Poképédia/TCGdex indexent des **cartes**, pas des boosters/displays/ETB. ⇒ **Scellés = piste parallèle** (catalogue sealed CM + normalisation set+type de produit), **après** les singles.
- **216 codes série internes** à mapper vers les 753 `idExpansion` (table `série interne → idExpansion`, grain set). Buckets pénibles : **« Promos » (565 cartes)** et **WOTC**.

### Structure de l'onglet cartes (stock) — relevé le 04/06/2026
- En-tête dynamique **ligne 143** (le code la détecte). Colonnes utiles : **D = Key** (SKU interne, ex. `___STutankaferEx076182NM1.5`) · **G = Code Série** (`EV4.0`, `WOTC1.0`, `Promos`…) · **H = Nom de la série** · **I = Nom de la Carte** (numéro inclus, FR) · **J = Type_Carte** · **K = Etat** (NM/EXC/GD…) · **P = Statut Carte** (**vide = en stock**) · **Q = Prix de vente** · **V = Prix d'achat (Pa)**.
- Volumétrie stock dispo : **4 095 cartes** (statut vide) ; vendues 7 757 ; échangées 228 ; volées 111.

### QR codes — décision (inchangée)
- ❌ **Pas l'`idProduct` comme clé QR** (timing : résolu plus tard ; granularité : édition ≠ exemplaire ; spécifique canal). ✅ Le QR encode un **identifiant interne** (unité de stock / SKU = la **Key** col D). `idProduct` = **référence externe** sur la définition de carte, à côté des futurs `whatnot_id`, `vinted_id`, etc.

### Résolution de l'`idProduct` (matching CardMarket)
- **Une fois par ÉDITION** (cache sur la définition de carte), pas par exemplaire → effort borné et décroissant.
- **Pipeline** : segmenter Pokémon → résoudre identité canonique via **série+numéro → Poképédia/TCGdex** → obtenir **nom EN** → filtrer `idExpansion` → match nom EN → **auto-lien si unique**, sinon **désambiguïsation humaine**, sinon **file d'exceptions** (extension non mappée, set récent, libellé divergent, jeu non Pokémon).
- Persistance sur la définition de carte : **`cardmarket_idProduct`** (nullable, backfill au fil de l'eau) + flag **`verified`** (auto-lié vs confirmé humain).

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

### 🔑 À garder pour plus tard (backlog `tcg-integrator`)
- Catalogue **One Piece** (autre fichier) + autres jeux ; **scellés** (catalogue sealed CM + normalisation produit).
- Canaux **Whatnot / Vinted / FB Marketplace** (impls. "préparer" = génération d'annonce, pas de matching ID).
- **Price guide** CM pour le **repricing** des annonces (pas nécessaire pour lister).
- **Photo par article** (recto/verso collage), notifications de vente temps réel, MAJ rapide CENTRAL anti-survente en convention.

---

## 7. Backlog Améliorations

| # | Amélioration | Contexte |
|---|---|---|
| 1 | Auto-reload cache slave à la reconnexion | Cache à 0 après re-auth OAuth — *sera adressé avec l’auth GIS (#9)* |
| 2 | Refresh silencieux token OAuth | Token expire ~1h → **planifié v1.10.0 (auth GIS)** *(recalé de 1.9.0, pris par l'Option 1 #9)* |
| 3 | Toggle "Sync actif sur ce device" | Risque double-sync si 2 masters ouverts |
| 4 | ~~Scanner caméra 📷 en fallback~~ ✅ | **Fait en #10 (Sales 1.9.2)** : carrousel swipe panier ⇄ caméra, décodage natif `BarcodeDetector` (codes-barres + QR), repli gracieux. *À valider sur le téléphone (non testable hors navigateur).* |
| 5 | Auto-création des onglets manquants | Master affiche erreur si onglet absent |
| 8 | Table fournisseurs/vendeurs pros | Auto-déterminer TVA_Achat selon le vendeur |
| 10 | ~~**[Sales] Boutons +/- quantité panier**~~ ✅ | **Fait/dépassé en #10 (Sales 1.9.2→1.9.3)** : les +/- sont remplacés par un **chip quantité tappable à gauche** (position fixe, → saisie clavier) + une **🗑** de suppression à gauche. Plus de décalage quand le prix change de largeur. |
| 9 | ~~Pill `📦 N` reste affiché sur état erreur~~ ✅ | **Résolu/obsolète en #10 (Sales 1.9.4)** : la pilule texte est remplacée par une **pastille ronde 📦** dont l'**anneau** porte l'état (vert/ambre/rouge) et le détail passe en info-bulle → plus d'artefact « ✗ Cache HS » écrasé par le timer. |
| 11 | Uniformisation acheteurs (Levenshtein) | Descopé en #6 (modèle Master-autoritaire suffit). À ressortir seulement si saisie acheteur libre multi-vendeurs un jour |
| 12 | ~~**[GUI/Sales] Rendu incrémental du panier**~~ ✅ | **Fait en #10 (Sales 1.9.1)** : maj DOM par ligne (clé `uid` + signature anti-réécriture), réordonnancement `appendChild` → préserve focus/scroll. |
| 13 | ~~**[GUI/Sales] Consolider les barres d'état empilées**~~ ✅ | **Fait en #10 (Sales 1.9.1)** : regroupées dans `#statusZone`, spacing géré par la zone (barres masquées = 0 hauteur) + compactage mobile. |
| 14 | ~~**[GUI/Sales] Feedback de scan plus visuel**~~ ✅ | **Fait en #10 (Sales 1.9.1)** : flash vert/rouge + bascule éphémère de l'icône ✓/✗ sur la zone de scan. |
| 15 | **[GUI/Master] Sections repliables + bandeau sync sticky** | Replier les sections et garder le bandeau sync/heartbeat visible (sticky) pendant le scroll |
| 17 | ~~**[Intake] Renommage onglet cartes + `a1()`**~~ ✅ | **Fait en #12 (1.2.0)** — ⚠️ et **non en #9** comme indiqué auparavant (jamais appliqué → l'Intake était cassé). `SHEET_NAME` → `cartes_stock_et_transactions` **et** `a1()` porté à toutes les plages. |
| 18 | ~~**[Master] Dédup `txId` + ignorer `SALES_NOT_SYNCED`**~~ ✅ | **Fait en #12 (Master 1.11.0)** : signature `txId\|clé\|source`, `Set` des appliqués reconstruit par tick, doublon → `DUP_SKIPPED` + anomalie `DUPLICATE_TXID` (cartes/scellés). « Ignorer `SALES_NOT_SYNCED` » déjà satisfait (ces lignes sont dans l'onglet `anomalies`, pas `ventes_detail`). *Filet, pas mutex — le bail reste primaire.* |
| 19 | **[Master] Colonnes cartes dynamiques** | Transposer la détection par nom d'en-tête à Master, **y compris les écritures** (P/P:T/AF par lettre détectée) — chemin d'écriture, à faire avec test |
| 20 | **[Sales+Master] Refonte « centimes entiers »** *(partiel #10)* | **Acompte #10 (Sales 1.9.1)** : col K `ventes_detail` arrondie 2 déc. à l'écriture (`Math.round(x*100)/100`). Reste le **gros** : calculs internes en centimes-entiers partout (pervasif → étape dédiée). |
| 21 | **[Sécu R4] OAuth GIS + PKCE** | Flux implicite (déprécié) + jeton en `localStorage` (Master/Sales) → vol possible via XSS. Migrer vers Google Identity Services / Authorization Code + PKCE ; jeton hors `localStorage`. **= priorité #1 prochaine session (v1.12.0).** *(bilan §A)* |
| 22 | **[Sécu R5] Réduire le scope OAuth** | `auth/spreadsheets` = accès à TOUTES les Sheets du compte → envisager `drive.file` + Google Picker (moindre privilège). *(bilan §A)* |
| 23 | ~~**[Sécu] CSP**~~ ✅ / **SRI N/A** | **CSP faite en #12** sur les 3 apps (`<meta>`, `default-src 'self'`, `connect-src` restreint, inline autorisé car apps mono-fichier). ⚠️ **à valider en navigateur** (OAuth + scraping). **SRI : N/A** — aucun `<script src>` tiers (à ressortir si lib via CDN un jour). |
| 24 | **[Archi] Module commun** *(candidat prochaine session)* | Helpers (`esc`, `safeForSheets`, `a1`, formateurs, `HttpError`…) dupliqués dans les 3 fichiers → extraire `tcg-common.js` (une seule source de vérité). ⚠️ Passe de **3 → 4 fichiers** (le commun doit être déployé à côté) ; **diff des copies avant fusion**. **= rangement du code, distinct de l'Option 3 (#25, sécurité).** *(bilan §A)* |
| 25 | **[Archi] Option 3 — passerelle Apps Script** | Évolution « propre » : web app Apps Script (exécute en tant que propriétaire) sert le stock filtré + reçoit les paniers → vendeurs sans jeton plein scope ni accès Drive. Ferait tomber R4/R5 côté vendeur. *(bilan §A)* |
| 26 | **[Perf] Republier `STOCK_PUBLIC` après vente** | Actuellement republié au rythme du cache Master (~5 min). Pour une dispo quasi temps réel côté vendeur, republier juste après l'application d'une vente. *(#9)* |
| 27 | **[Sales+Master] Roll-back de vente** *(jalon coordonné)* | Annuler une vente déjà validée/envoyée (acheteur qui repart). Approche **append-only** retenue (on n'efface jamais une ligne partie chez Master) : écriture **compensatoire** dans `ventes_detail` — `txId` suffixé **`-RB`**, **quantités et total négatifs**, mêmes articles/acheteur — poussée via le **même outbox idempotent**. ⚠️ **Conséquence Master** : doit reconnaître les `txId` `-RB` / quantités négatives (re-créditer le stock, annuler le total, tracer). **Spécifier Master d'abord, puis câbler le bouton côté Sales** (état « vente validée » vert + bouton ↶, fenêtre de rétractation). *Ébauche commencée puis **retirée** en #10 pour livrer une 1.9.5 cosmétique propre (rien d'actif livré).* |

| 28 | ~~**[Sales] Acheteur imposé**~~ ✅ | **Fait en #12 (Sales 1.10.1)** : lecture seule quand le nom d'event est rempli (`app_state` col B → `getResolvedBuyer()` = nom d'event, pastille `#buyerLocked`) ; vide → éditable + obligatoire. Uniformité multi-vendeurs. |

### ✅ Sorti du backlog
- ~~Migration Netlify → GitHub Pages~~ (#4)
- ~~#6 Correction arrondi 1¢ sur « Ajuster total »~~ → fait en **#8** (méthode du plus grand reste, euros entiers, total exact)
- ~~#7 Capture des toasts / log~~ → **journal `logEvent` persistant + `exportLog()`** livré en #8 (Sales) / #8 (Master). *Reste optionnel : un panneau de log pliable dans l'UI.*
- ~~#16 Purge code mort (Sales)~~ → fait en **#8** : `switchTab()` + CSS d'onglets/stats/leaderboard retirés, `SYNC_POLL_MS` retiré, `HEARTBEAT_POLL_MS` câblé, `SHEET_ANOMALIES` désormais **utilisé** (dead-letter). *Restent : `sheetsUpdate`/`sheetsBatch` non appelés, quelques champs `sealedColIdx` non lus — purge mineure si besoin.*

---

## 8. Infrastructure

| Élément | Détail |
|---------|--------|
| Hébergement | **GitHub Pages** — déploiements illimités, gratuit, HTTPS natif, repo `legardiendutcg/Automation` (public) |
| OAuth | Projet GCP `774473792747` (nom GCP historique : « TCG Integrator » — non renommé côté Google, c'est juste un libellé), client `774473792747-...rj96`, type Application Web. ⚠️ `tcg-integrator` (nouvelle app) pourra réutiliser ce client OU avoir le sien — à décider au moment de l'auth GIS du publisher. |
| Scopes | `https://www.googleapis.com/auth/spreadsheets` (sensible, doit être déclaré dans Data Access du Google Auth Platform) |
| URIs OAuth | Redirect URIs = les 3 URLs github.io complètes (sales/master/converter), origine JS = `https://legardiendutcg.github.io` |
| Audience | Mode **Testing** avec emails listés comme utilisateurs de test (évite la vérif Google complète) |
| APIs activées | Google Sheets API (à activer manuellement dans tout nouveau projet GCP — non activé par défaut) |
| Appareils | Sales sur téléphones vendeurs, Master sur iPad/ordi du gérant |

### Pièges OAuth rencontrés (à se rappeler)
- **403 "insufficient authentication scopes"** : scope manquant dans Data Access OU token cached côté Google avec ancien grant → solution : ajouter le scope + révoquer l'app sur `myaccount.google.com/permissions` → re-consentement frais
- **403 "API has not been used... or it is disabled"** : Google Sheets API à activer dans le nouveau projet (APIs & Services → Bibliothèque)
- **Access blocked: ... has not completed the Google verification process** : compte de connexion absent de la liste des Test users en mode Testing
- **OAuth client supprimé** : Google supprime auto les clients inactifs 6 mois ; restaurables 30j via la page "Deleted credentials"

---

## 9. Notes de Session

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

---

## 10. Versioning des Apps

| Version | Date | App(s) | Changements |
|---------|------|--------|-------------|
| 1.14.0 | 31/05/2026 | Sales + Master | **(#27, roll-back d'une vente — jalon coordonné)** **Sales** : bouton ↶ Annuler pendant 90 s ; annulation directe si la vente est encore dans l'outbox non envoyée, sinon ligne compensatoire `-RB` (qty négative ; col K = `prixOriginal` pour les cartes) via le même outbox idempotent. **Master** : **(#27, roll-back d'une vente — côté Master)** Inversion d'une vente appliquée par **modification des lignes existantes** dans CENTRAL (jamais de qty négative) : **carte** → vide P/R/S/T, remet `Q` = prix d'origine (porté par Sales en col K de la ligne `-RB`), `AF` = « pas vendu » ; **scellé** → supprime la ligne de transaction (retrouvée par **txId estampillé en col `AA`**) + ré-incrémente la qty du lot dans `Scelle_Stock`. `masterSyncTick` reconnaît les lignes `ventes_detail` à txId `…-RB`, **garde à 4 états** (inverse seulement si l'originale fut appliquée ; en attente → **neutralisation de la paire**, CENTRAL intact ; en échec → no-op ; introuvable → anomalie). `applySealedSale` estampille le txId en `AA`. Dédup #18 étendue aux `-RB`. Trace `ROLLBACK_APPLIED` dans `anomalies`. **NON testé navigateur ; déployer avec Sales 1.14.0** (Master 1.14.0 sûr seul : estampille inoffensive, gestion `-RB` dormante). |
| 1.13.0 | 31/05/2026 | Sales + Master | **(#13, jalon coordonné)** **Sales** : (a) **vidage du panier persistant** — panier estampillé `{items, savedAt, version, eventName}`, **jeté au boot** si âge > 30 min OU version ≠ app OU (1ère confirmation Master) event différent ; vidé aussi au **logout** et au **changement/clôture d'event** détecté via heartbeat (lecture confirmée seulement → immunisé aux blips) ; tout vidage auto garde un **snapshot restaurable 60 s**. (b) **Auth** : plus de demande de jeton **auto au boot** (évite la popup bloquée = échec silencieux) → connexion au **tap** ; `onAuthError` rend visibles popup bloquée + échec de refresh. **Master** : (a) **panneau « 🏠 Récap comptoir »** permutant avec « Faits marquants » quand **pas de nom d'event** (carrousel swipe : articles du jour ⇄ clients du jour ; source `ventes_detail`). (b) même durcissement auth (pas d'auto-popup, erreurs visibles). **Déployer les 2 ensemble. `node --check` OK ; NON testé navigateur.** |
| 1.12.0 | 31/05/2026 | Sales + Master | **(#13, auth Google Identity Services — #21/R4, jalon coordonné)** Migration du flux **OAuth implicite codé main** (redirection pleine page, `response_type=token`, jeton en `localStorage`, aucun refresh) vers le **modèle token de GIS** : chargement de `accounts.google.com/gsi/client` + `initTokenClient()`. **Jeton EN MÉMOIRE SEULE** (`persistToken()`/restauration LS retirés ; purge proactive de l'ancien jeton disque au boot → couvre **R4**). **Refresh silencieux** programmé 5 min avant expiration (`requestAccessToken({prompt:''})`) → **fin de la déconnexion ~1 h**. Connexion = **popup** (au tap) ; au reload, tentative silencieuse (transparente si session active, sinon bouton/overlay de connexion). `logout()` → `revoke()` + clear mémoire (Master : release du verrou d'abord, allow-list `isAuthorizedMaster` + overlays conservés). **CSP** : `script-src` + `https://accounts.google.com`, ajout `frame-src https://accounts.google.com`. **SRI** : N/A — le script GIS ne doit pas être épinglé. ⚠️ Prérequis GCP (origine `https://legardiendutcg.github.io`) déjà en place. **Déployer les 2 ensemble. `node --check` OK ; NON testé navigateur.** |
| 1.11.1 | 31/05/2026 | Master | **(#12, CSP — Master seul)** En-tête CSP `<meta>` : `default-src 'self'`, inline script/style autorisés, `connect-src` = Sheets + OAuth Google, `object-src 'none'`, `base-uri 'self'`. ⚠️ à valider navigateur. SRI N/A. |
| 1.11.0 | 31/05/2026 | Master | **(#12, dédup `txId` à la sync — #18, Master seul, PRÉ-REQUIS event réel)** `masterSyncTick` : `saleSignature()` = `txId\|clé\|source` (cartes/scellés ; `''` pour manuel/sans clé) · `Set` des appliqués (Synced ∈ {OK,OK_MANUAL}) reconstruit par tick depuis la feuille + complété au fil du tick · doublon → `DUP_SKIPPED` + anomalie `DUPLICATE_TXID` (non ré-appliqué). Scellé devient idempotent ; carte l'était déjà (pré-check `P_NON_BLANK`). « Ignorer `SALES_NOT_SYNCED` » déjà satisfait (onglet `anomalies`). *Filet, pas mutex (bail = défense primaire). Vérifié par simulation (5 scénarios). Sales inchangé.* |
| 1.10.2 | 31/05/2026 | Sales | **(#12, CSP — Sales seul)** En-tête CSP `<meta>` identique à Master (connect-src Sheets + OAuth Google). ⚠️ à valider navigateur. SRI N/A. |
| 1.10.1 | 31/05/2026 | Sales | **(#12, acheteur IMPOSÉ — Sales seul)** Nom d'event publié par Master (`app_state` col B) → champ acheteur **lecture seule** (pastille `#buyerLocked`), valeur = nom d'event pour tous les vendeurs (uniformité multi-vendeurs) ; vide → éditable + obligatoire. `getResolvedBuyer()` priorise le nom d'event ; `updateBuyerField()` bascule input ⇄ pastille. Fin du défaut écrasable de la 1.10.0. *Master inchangé (besoin Master ≥1.10.0).* |
| 1.10.1 | 30/05/2026 | Master | **(#11, dashboard maquette v6 — Master seul)** En-tête : sous-titre retiré, **version à droite du titre**. Bandeau event compact `.event-strip` + **aide « ? »**. Ligne 2 col. : **« Synchronisation des ventes »** (Sync now / Pause auto à droite) | **« Clôture »** (aide « ? »). Titre « Stats… » retiré. **`.dash-grid`** : 4 KPI compacts **auto-shrink** (`fitKpis`/`data-fit`) · 2 **donuts** catégorie+licence (`pathLength=100`, **% en légende**, `CAT_COLORS`/`LIC_COLORS`+fallback, clé normalisée) · **⭐ Faits marquants** (`computeFacts`, 9 lignes) · **CA/heure horizontal** (`computeHours`/`renderHours`, 10h→17h, montant dans la barre, vides hachurés). Données agrégées de `ventes_detail` au refresh (rien stocké). `renderSplit`/`renderLeaderboard` retirés ; **Journal + Anomalies conservés** dessous. `toggleHelp()` (ferme au clic extérieur). *Sales inchangé (1.10.0). CSS mort `.stat-*`/`.split-*`/`.lb-*` laissé (cleanup mineur). NON testé navigateur.* |
| 1.10.0 | 30/05/2026 | Sales + Master | **(#11, palier coordonné)** **Master** : refonte en-tête (logo embarqué, titre, largeur 1240px, panneaux stock/Google), **page floutée tant que Google non connecté** + overlay accès refusé · **verrou d'instance lease/TTL** (`app_state` C/D = `lock_uuid`/`lock_label` ; acquisition écrire→jitter→relire ; éviction détectée au battement ; « Forcer la reprise » ; reprise auto si bail >35 s ; anti-ping-pong ; migration purge C2/D2) · **sync gated sur le bail** (plus sur l'event) · champs « acheteur défaut »/« label appareil » retirés · cleanup écritures mortes U/AD (scellé). **Sales** : statut master par **détection de changement** d'horodatage (warning **non bloquant**, ventes toujours envoyées) · arrêt lecture `app_state` col D comme acheteur. ⚠️ **Déployer les 2 ensemble.** **NON testé navigateur.** 🟡 **Reste à coder** (validé maquette v6) : **acheteur imposé** côté Sales (le dashboard Master est fait en 1.10.1). | Version collée sous le titre, **alignée à droite** (bord droit = celui du titre) · bouton « Valider » en **police du titre** (Syne 800, `-.05em`), alertes en Syne aussi (plus de Space Mono) · **retrait de l'ébauche de roll-back** (bouton ↶/modale/styles) → reporté en jalon coordonné (#27). Purement cosmétique. *Master & Intake inchangés.* |
| 1.9.4 | 29/05/2026 | Sales | **(#10, en-tête compact — patch Sales seul)** Titre « TCG SALES » **non tronqué** (ligne dédiée) · connexions en **pastilles rondes** : `G` (Google) + `📦` (stock) cerclées **vert/rouge/ambre** (toucher = (dé)connexion / refresh ; détail compte/réf./âge en info-bulle) · version lisible. `updateAuthUI`/`updateCachePill` pilotent les anneaux (pilules texte conservées masquées pour compat). |
| 1.9.3 | 29/05/2026 | Sales | **(#10, ergonomie saisie — patch Sales seul)** **Rouleaux supprimés → saisie clavier** (`inputmode`) pour **quantité**, **prix unitaire** et **total** · **prix unitaire éditable par ligne** (corrige une côte GSheet périmée sans répartir sur tout le panier ; affiché « modifié ») · **🗑 à gauche** de chaque ligne (suppression directe ; qty 0 reste possible) · prix affiché = **prix unitaire** · bouton « Valider » agrandi (police Syne). |
| 1.9.2 | 29/05/2026 | Sales | **(#10, refonte UI 3 zones — patch Sales seul)** Écran smartphone **sans scroll global** (`100dvh`/`overflow:hidden`) ; HAUT logo PNG + titre ; MILIEU **carrousel swipe panier ⇄ caméra** (2 points) + **scan caméra `BarcodeDetector`** (codes-barres + QR, repli gracieux) + barre de scan fine ; lignes = chip quantité à gauche, nom pleine largeur, prix flotté ; **doublon = même ligne ×N**, contour orange (carte ancienne clé) / **rouge bloquant** (carte clé neuve) ; BAS champs sans libellés + total tappable + bouton Valider épinglé. Icônes = emojis ; logo client embarqué (base64). *Caméra/swipe non testés hors navigateur.* |
| 1.9.1 | 29/05/2026 | Sales | **(#10, lot polish — patch Sales seul ; 1.10.0 réservé à l'auth GIS)** Arrondi col K `ventes_detail` à 2 déc. `Math.round(x*100)/100` (#20 **partiel**) · feedback de scan flash ✓/✗ + pulse vert/rouge (#14) · `renderCart()` **incrémental** par ligne (clé `uid`, signature anti-réécriture, réordonnancement `appendChild`) → préserve focus/scroll (#12) · bandeaux d'état regroupés dans `#statusZone`, spacing géré par la zone + compactage mobile (#13) · purge code mort `sheetsUpdate`/`sheetsBatch` + vars `cardHeaderRow`/`cardColIdx`/`sealedColIdx` (#16 reliquat). *Master & Intake inchangés ; nécessite toujours Master 1.9.0 (`STOCK_PUBLIC`).* |
| 1.9.0 | 29/05/2026 | Sales + Master | **(#9, Option 1)** Master `publishStockView()` projette une vue stock **sûre** dans `EVENTS › STOCK_PUBLIC` (Key/Kind/Code/Nom série/Nom/Type/Etat/PrixVente/Qty/Reserved + liste TVA, **sans coûts/marges**) à chaque `loadStockCache` ; capture du flag `reserved` (lecture scellés `A:AF`→`A:AG`, +`iTvaVente`) ; helpers `normAccent`/`isReservedValue` portés · **Sales** lit le stock depuis `STOCK_PUBLIC` (une lecture `A2:L`) et **n'accède plus à CENTRAL** ; retrait code mort `findCardsHeaderRow`/`findColByName`. **Numéro partagé** (les 2 apps modifiées ensemble) — Master re-tagué depuis 1.6.3 ; l'auth GIS visée passe à 1.10.0. *Nécessite onglet `STOCK_PUBLIC` publié + protégé. Test E2E requis avant prod.* |
| 1.8.1 | 29/05/2026 | Sales | Fix : `moveToAnomalies` écrit dans le vrai schéma `anomalies` A:K (1 ligne/article, `SALES_NOT_SYNCED`, payload JSON en `Notes`) |
| 1.8.0 | 29/05/2026 | Sales | **Outbox idempotent** : file → envoi → confirmation par relecture `ventes_detail!D` (txId) · échec → `anomalies` puis dead-letter local · zéro perte silencieuse · journal du cycle de vie |
| 1.7.0 | 29/05/2026 | Sales | Renommage onglet + helper `a1()` · colonnes cartes dynamiques · remise « plus grand reste » euros entiers · formateur Intl fr-BE · journal `logEvent` · purge code mort (`switchTab`/CSS) · zoom + optim. smartphone · `HEARTBEAT_POLL_MS` câblé · `HttpError` · `parseInt`/`init().catch`/`txId` unique |
| 1.6.3 | 29/05/2026 | Master | Renommage onglet `cartes_stock_et_transactions` (corrige `400`) + helper `a1()` partout (lecture/écriture) · `HttpError` · journal `logEvent` · formateur Intl UI · `parseInt`/`init().catch`. **NON transposé** : colonnes dynamiques (écritures par lettre fixe) et format archive clôture |
| 1.6.3 | 29/05/2026 | Sales | Fix : « pas vendu » exclu du menu déroulant TVA_Vente (statut, pas un régime) — `isNotForSaleStatus()`, à côté de l'exclusion existante « réservée » |
| 1.6.2 | 29/05/2026 | Master | Correctifs revue #7 : 🔴 `applyCardSale` (bloc `copyPaste` parasite réf. `nextTxRow` hors scope → `ReferenceError` à chaque vente carte) supprimé · 🟠 index `Synced` `r[12]`→`r[14]` (col O) dans `renderSalesLog` + `loadCloseSummary` · commentaires A:AF→A:AG |
| 1.6.1 | 29/05/2026 | Sales | Fix : liste TVA_Vente lue dans `Scelle_Transactions!AG8:AG23` (pointait par erreur sur `Scelle_Stock`, d'où le menu réduit à « Régime normal ») |
| 1.6.0 | 29/05/2026 | Sales + Master | Sélecteur TVA_Vente (source `Scelle_Transactions!AG8:AG23`, écrit AG scellé / AF carte) · fix bug AG jamais écrit (scellé) · verrou « réservée » au scan · acheteur Sales↔Master (`app_state` col D) avec découplage session/acheteur · `ventes_detail` → A:N (M=TVA_Vente, N=Acheteur) |
| 1.5.1 | 28/05/2026 | Sales | Panneau de scan : retrait du prix (affiché uniquement dans le panier) |
| 1.5.0 | 28/05/2026 | Sales + Master | sealedColIdx entièrement dynamique · iPrix = Côte/u (col M) · txRow 33 cols (col A = "x") · append A1 · retry ×3 avant anomalie · version badge dans UI |
| 1.4.0 | 28/05/2026 | Sales + Master | Migration GitHub Pages · nouveau projet OAuth · fix constantes SHEET_SEALED/_TX · détection dynamique header scellé |
| 1.3.0 | 28/05/2026 | Sales + Master | Split Scelle_Stock/Scelle_Transactions · col C Key · applySealedSale · cache scellé indexé par Key |
| 1.2.0 | — | Sales + Master | Fix message "ajouté (0€)" · corrections diverses |
| 1.1.0 | — | Sales + Master | Architecture de base · flow cartes |

**Règle de versioning :**
- **Patch (x.x.+1)** : fix mineur, UX, cosmétique
- **Minor (x.+1.0)** : nouvelle feature ou fix fonctionnel significatif
- **Major (+1.0.0)** : refonte architecture ou breaking change

### Versioning de l'Intake (piste distincte, back-office)
L'outil `tcg-intake.html` (TCG Intake) est désormais versionné séparément de la paire Sales/Master.

| Version | Date | Changements |
|---------|------|-------------|
| 1.2.1 | 31/05/2026 | **(#12)** `SAFETY_GSHEET_ID` **renseigné** (feuille dédiée logs/backups d'injection → journal + backups opérationnels) · **CSP** ajoutée (`<meta>`, `default-src 'self'`, `connect-src` = Sheets/OAuth Google + les 5 proxies de scraping, inline autorisé). ⚠️ CSP à valider navigateur. SRI N/A. |
| 1.2.0 | 31/05/2026 | **(#12, RÉPARATION 🔴)** L'Intake était **cassé** : `SHEET_NAME` était resté sur `Cartes_Stock_&_Transactions` (le renommage #9.1 documenté n'avait **jamais** été appliqué au fichier) → 400 « Unable to parse range » partout. Corrigé → `cartes_stock_et_transactions` · **+ helper `a1()`** porté à **toutes** les plages (cartes + onglets LOG/BACKUP ; références d'onglet aussi quotées dans les formules `COUNTIF`/`SUMIF`). Comportement inchangé pour des données normales. |
| 1.1.0 | 29/05/2026 | **(#9)** Portage des garde-fous de Sales/Master : `esc()` sur tous les `innerHTML` (corrige XSS DOM via CSV importé / valeurs scrapées) · `safeForSheets()` sur les colonnes texte du CSV dans `buildCardRow` (corrige injection de formules) · `buildLotRow` double les guillemets dans `COUNTIF`/`SUMIF`. ⚠️ **Le renommage `SHEET_NAME` annoncé ici n'a en réalité pas été appliqué** (l'Intake est resté cassé jusqu'au correctif #12 — voir 1.2.0). |
| 1.0.0 | 29/05/2026 | Premier stamp formel. Correctifs revue #7 : 🟡 `parseCSV` normalise CRLF/CR → LF · 🟡 détection dynamique de l'en-tête cartes (`findCardsHeaderRow`/`ensureCardHeaderRow`) au lieu de `143` codé en dur · `parsePrix` locale belge. `SAFETY_GSHEET_ID` reste un placeholder à renseigner (garde-fou explicite déjà en place). |
