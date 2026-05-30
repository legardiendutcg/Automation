# Contexte Projet — Le Gardien du TCG
> Document à coller en premier message dans chaque nouvelle session Claude.
> Dernière mise à jour : **Session #11 — 30/05/2026** · **Versions actuelles : Sales 1.10.0 / Master 1.10.1 / Integrator 1.1.0**
>
> ⚠️ **Lecture prioritaire : voir « ## 5. État Actuel — Session #11 »** — palier 1.10.0 (refonte UI Master + verrou d'instance + sémantique acheteur) **codé** ; **dashboard stats redessiné (maquette v6) codé en Master 1.10.1**. Reste **un seul** item du palier : **acheteur imposé côté Sales** (validé, pas encore codé). L'auth GIS passe en **1.11.0**.

---

## 0. Instructions pour Claude

- **Tout au long de la discussion** : mettre à jour ce document en arrière-plan au fil des avancées — ajouter les nouveaux milestones complétés, les nouvelles idées de développement à garder pour plus tard, et retirer du backlog les améliorations sur lesquelles on a travaillé et validées.
- **Quand il est opportun d'ouvrir une nouvelle discussion** (contexte trop long, session de travail cohérente terminée) : prévenir, mettre à jour ce document une dernière fois, puis proposer de basculer.
- **En début de chaque session**, lorsqu'une étape du développement est identifiée : indiquer quel modèle Claude est le plus adapté pour cette étape (Haiku / Sonnet / Opus) avec une justification courte.
- **Au début de chaque nouvelle discussion** : lire les 3 fichiers HTML (`tcg-sales.html`, `tcg-master.html`, `tcg-converter.html`) pour assimiler l'intégralité du code, puis faire une passe de revue systématique — triple-check à la recherche de bugs, repérage des incohérences et du code/variables devenus inutiles (code mort), et analyse des opportunités d'amélioration de **performance** (en convention/événement, la **réactivité** est critique). Remonter les trouvailles avant de coder.
- **À chaque livraison de fichier** : bumper systématiquement la version (patch/minor/major selon le changement) avant de présenter le fichier. Ne jamais livrer deux fichiers successifs avec le même numéro de version. Quand les deux apps sont modifiées ensemble, elles partagent le même numéro de version pour garder un mapping cohérent entre Sales et Master.
- **Bilan de la session #9** : revue de **sécurité** des 3 apps (OWASP 2025 + doc « Quand l'application grandit »), puis correctifs validés.
  - **Integrator 1.1.0** : ajout `esc()` (corrige XSS DOM via CSV/scraping) + `safeForSheets()` (corrige injection de formules) ; **+ correction du renommage onglet** `cartes_stock_et_transactions` (priorité #9.1 — il cassait dessus).
  - **Option 1 (Master 1.9.0 + Sales 1.9.0)** : Master publie une vue stock **sûre** (`STOCK_PUBLIC` dans EVENTS, sans coûts/marges) ; Sales lit le stock de là et **n'accède plus du tout à CENTRAL** → les vendeurs n'ont plus besoin d'accès à CENTRAL (marges jamais exposées). Confirmé : 2 GSheets en « Restreint ».
  - ⚠️ **Versioning** : l'Option 1 ayant modifié Sales **et** Master ensemble, ils partagent **1.9.0** (réalignement des pistes). Du coup **l'auth GIS, qui visait 1.9.0, glisse en `1.10.0`.** Versions à jour : **Sales 1.9.0 / Master 1.9.0 / Integrator 1.1.0**.
  - ✅ **Test E2E #9 concluant** : Option 1 validée de bout en bout (Master 1.9.0 publie `STOCK_PUBLIC`, Sales 1.9.0 lit le stock sans accès CENTRAL, ventes carte + scellé, sync Master) **jusqu'à la clôture d'événement** 🏁 (archive + **vidage des buffers** `ventes_detail`/`anomalies`) — la clôture était en attente depuis #8, elle est désormais validée. **Option 1 prête à la mise en ligne.**
- **Au début de la prochaine session** :
  - **Priorité 1 — auth Google Identity Services (v1.10.0, Sales + Master)** : fin de la déconnexion ~1 h ; jeton en mémoire seule (plus en `localStorage`). ⚠️ Config GCP : origine `https://legardiendutcg.github.io`. *(couvre R4 du bilan)*
  - **Priorité 2 — passe Master** : ignorer `SALES_NOT_SYNCED` + dédup `txId` (#18), colonnes cartes dynamiques côté écriture (#19), refonte centimes (#20).
  - **Priorité 3 — roll-back de vente (Sales + Master, jalon coordonné, backlog #27)** : annuler une vente déjà envoyée (acheteur qui repart) via une **écriture compensatoire** append-only (`txId` suffixé `-RB`, quantités/total négatifs) — Master doit savoir la reconnaître (re-créditer le stock, annuler le total). **Spécifier d'abord le côté Master**, puis câbler le bouton côté Sales. *(ébauche commencée puis retirée en #10 pour livrer une 1.9.5 propre — voir notes #10.)*
  - **Puis** : cartes gradées / articles manuels ; backlog sécurité (R5 scope, CSP/SRI, module commun, Option 3 Apps Script — bilan §A).
  - 🔧 **Hardening recommandé (n'affecte pas le fonctionnement)** : protéger l'onglet `STOCK_PUBLIC` (Données > Protéger → seul le propriétaire édite) et ajouter les vendeurs avec leur **propre compte Google** (Éditeur EVENTS, aucun accès CENTRAL) plutôt qu'un login master partagé.
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
> ✅ **`tcg-converter.html` corrigé en #9** : `SHEET_NAME` pointe désormais sur `cartes_stock_et_transactions` (le nouveau nom n'a pas de caractère spécial → sûr en A1 ; wrapping `a1()` optionnel restant).
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
  - `https://legardiendutcg.github.io/Automation/tcg-converter.html`
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

## 5. État Actuel — Session #11 (30/05/2026)

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

## 5bis. État Actuel — Fin Session #9 (29/05/2026)

### ✅ Terminé en session #9 (sécurité + Option 1)
Revue de sécurité (OWASP 2025) des 3 apps puis correctifs validés. Bilan détaillé : `Bilan_risques_TCG.md` (réorganisé : risques ouverts d'abord, puis couverts).
- **[Sécu 🔴 — Integrator 1.1.0] XSS DOM via CSV corrigée.** L'Integrator était le seul fichier sans échappement. `esc()` (identique à Sales/Master) appliqué à **tous** les `innerHTML` (table, séries inconnues, prix de lot, prix élevés, récap, étiquettes), y compris attributs `id`/`data-*` dynamiques ; `getElementById` inchangés (cohérence via id décodé).
- **[Sécu 🔴 — Integrator 1.1.0] Injection de formules corrigée.** `safeForSheets()` sur les colonnes **texte** du CSV dans `buildCardRow` ; `buildLotRow` double les guillemets dans `COUNTIF`/`SUMIF`.
- **[Fix 🔴 — Integrator 1.1.0] Renommage onglet cartes** (priorité #9.1) : `SHEET_NAME` → `cartes_stock_et_transactions`. L'Integrator est de nouveau opérationnel sur la GSheet cartes.
- **[Archi/Sécu 🔒 — Master 1.9.0 + Sales 1.9.0] Option 1 : vendeurs sans accès à CENTRAL.** Master publie une vue stock **sûre** (`STOCK_PUBLIC` dans EVENTS, sans coûts/marges) ; Sales lit le stock de là et **ne lit plus CENTRAL** (vérifié : zéro référence exécutable à CENTRAL côté Sales ; fonctions mortes `findCardsHeaderRow`/`findColByName` retirées). Les marges ne sont **jamais** exposées aux vendeurs.
- **[Confirmé]** 2 GSheets en « Restreint ».

### ✅ Test E2E #9 — concluant (Option 1 prête à la prod)
- Validé de bout en bout : déploiement Master 1.9.0 → `STOCK_PUBLIC` créé/peuplé (sans colonnes de coût) → Sales 1.9.0 lit le stock **sans accès CENTRAL** → vente carte + scellé, verrou « réservée », menu TVA, sync Master → **clôture d'événement** 🏁 : archive + **vidage des buffers** `ventes_detail`/`anomalies` OK (la clôture était en attente depuis #8).
- **Fraîcheur** : `STOCK_PUBLIC` republié au rythme du cache Master (~5 min). Un scellé vendu peut rester affiché dispo jusqu'à la prochaine publication (Master rattrape en anomalie). Optimisation possible : republier après chaque vente appliquée (backlog #26).
- 🔧 **Hardening recommandé** (n'affecte pas le fonctionnement validé) : protéger l'onglet `STOCK_PUBLIC` (seul le propriétaire édite) ; ajouter les vendeurs avec leur **propre compte Google** sur EVENTS (pas de login master partagé).

### Backlog sécurité ouvert (bilan §A)
R4 (OAuth implicite + jeton `localStorage` → PKCE/GIS) · R5 (scope `spreadsheets` trop large → `drive.file`+Picker) · CSP/SRI · module commun (helpers dupliqués) · Option 3 (passerelle Apps Script, ferait tomber R4/R5 côté vendeur).

---

## 5ter. État Actuel — Fin Session #6 (29/05/2026)

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
- Migration GitHub Pages, projet OAuth « TCG Integrator »

### 🔍 Revue de code + correctifs (session #7)

Triple-check des 3 apps, puis correctifs livrés après validation. Versions résultantes : **Sales 1.6.3 / Master 1.6.2 / Integrator 1.0.0**. Les bugs ci-dessous sont **corrigés** (✅) sauf mention contraire.

**✅ 🔴 Bloquant — Master `applyCardSale` : `nextTxRow` non défini (TOUTES les ventes cartes).**
Un bloc `copyPaste` avait été copié par erreur depuis `applySealedSale` ; il référençait `nextTxRow`, variable hors scope (déclarée seulement dans `applySealedSale`). En `'use strict'` → `ReferenceError`, attrapée par le try/catch par-ligne de `masterSyncTick` ⇒ `errors++`, ligne **non marquée Synced**. Les écritures stock (P:T + AF) réussissaient *avant* l'erreur (stock correct), mais : erreur console + **fausse anomalie `CARD_NOT_FOUND`** au tick suivant + ~8 s de retard par ligne. **C'est ce qui empêchait de valider proprement le flow cartes à travers Master 1.6.0.** → **Corrigé (Master 1.6.2)** : bloc supprimé (les cartes n'appendent pas dans `Scelle_Transactions`).

**✅ 🟠 Régression schéma 1.6.0 — Master lisait `Synced` au mauvais index (col M au lieu de col O).**
`Synced` est à l'index **14** (col O) depuis l'extension `ventes_detail` 1.6.0, mais `renderSalesLog` et `loadCloseSummary` lisaient encore l'index **12** (col M = `TVA_Vente`) ⇒ icône toujours ⏳ et résumé de clôture « ✓ Toutes synchronisées » à tort (**risque de clôturer un event avec des ventes non synchronisées**). Données réelles saines (le drain `ensureSyncDrain` utilise `lastSyncStats.pending`). → **Corrigé (Master 1.6.2)** : `r[14]` aux deux endroits.

**✅ 🟡 Integrator — CRLF non normalisé.** `parseCSV` faisait `text.trim().split('\n')` sans retirer `\r`. → **Corrigé (1.0.0)** : `text.replace(/\r\n?/g,'\n')` avant le split.

**✅ 🟡 Integrator — `CARD_HEADER_ROW = 143` en dur** (vs détection dynamique dans Sales/Master) ⇒ si l'en-tête bouge, la vérif anti-doublon et les offsets D/F/Q ratent les clés (**risque d'intégrité**). → **Corrigé (1.0.0)** : `findCardsHeaderRow` / `ensureCardHeaderRow` (mirroir de Master), résolus en tête de `injectIntoGSheet` et `loadLaterCards`.

**Mineurs :** ✅ Integrator `parsePrix` locale belge (virgule décimale) au lieu de `parseFloat` ; ✅ commentaires Master « A:AF » alignés sur la lecture réelle `A:AG`. **Non-corrigé (volontaire)** : `SAFETY_GSHEET_ID` reste un placeholder **à renseigner par l'utilisateur** — le garde-fou `getSafetySpreadsheetId()` lève déjà une erreur explicite tant qu'il n'est pas configuré (comportement voulu).

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
- ~~Clôture événement 🏁 (archive + reset buffers)~~ — ✅ **validée en #9** (flux complet rejoué : archive + vidage des buffers OK).
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
5. ~~**🟡 Integrator**~~ — fait (1.0.0). CRLF normalisé, en-tête cartes dynamique, `parsePrix` locale belge.

### ✅ Validé en session #7
6. ~~**Test E2E (Sales 1.6.3 / Master 1.6.2), ventes cartes + scellé**~~ — **concluant**. 🔴 levé, TVA_Vente / verrou « réservée » / acheteur / écriture AG OK.

### ✅ Livré en session #8
7bis. ~~**#6 Arrondi « ± Total »**~~ — fait (#8, Sales 1.7.0) : plus grand reste, euros entiers, total exact, sur tous les articles.
8bis. ~~**#7 Capture des logs**~~ — fait (#8) : `logEvent` persistant + `exportLog()` (Sales & Master).
9bis. ~~**#16 Purge code mort Sales**~~ — fait (#8).
10bis. ~~**Robustesse Sheets**~~ — fait (#8) : renommage onglet + helper `a1()`, `HttpError`, détection dynamique colonnes cartes (Sales).
11bis. ~~**Outbox idempotent Sales**~~ — fait (#8, 1.8.0/1.8.1) : zéro perte, zéro doublon, bascule `anomalies` + dead-letter.

### ✅ Livré en session #9 (sécurité + Option 1)
- ~~🔴 **Corriger l'Integrator (renommage onglet)**~~ — fait (1.1.0). `SHEET_NAME` → `cartes_stock_et_transactions`.
- ~~**Sécurité Integrator**~~ — fait (1.1.0). `esc()` (XSS) + `safeForSheets()` (injection formules).
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
1. **🟡 Acheteur IMPOSÉ côté Sales** *(dernier item du palier 1.10.0)* — lecture seule quand le nom d'event est rempli (réactiver `#buyerLocked`, `getResolvedBuyer()` = nom d'event ; vide → éditable + obligatoire). **Puis bumper Sales** et **déployer Sales+Master ensemble**.
2. **#18 dédup `txId` (Master) — PRÉ-REQUIS BLOQUANT avant tout event réel** : ignorer `SALES_NOT_SYNCED` + **dédupliquer par `txId`** (filet split-brain ~10 s lors d'une reprise de verrou).
3. **Auth Google Identity Services (v1.11.0, Sales + Master)** — fin de la déconnexion ~1 h, jeton en mémoire seule. ⚠️ GCP : origine `https://legardiendutcg.github.io`. *(backlog #1, #2, #21 ; couvre R4 — recalé de 1.10.0, consommé par le palier #11.)*
4. **Passe Master (suite)** : colonnes cartes dynamiques côté écriture (#19) ; refonte centimes (#20).
5. **Roll-back de vente (Sales + Master, backlog #27)** — append-only (`txId` `-RB`, négatifs) ; spécifier Master d'abord.
6. **Cartes gradées / articles manuels.** Puis **backlog sécurité** (R5, CSP/SRI, module commun, Option 3). 🔧 **Hardening Option 1** (protéger `STOCK_PUBLIC` + `app_state` ; vendeurs en comptes propres). *Cleanup mineur : CSS mort `.stat-*`/`.stats-grid`/`.split-*`/`.lb-*` dans Master (panneaux retirés en 1.10.1).*

### Reste du backlog ordonné
9. **Workflow boutique / vente comptoir** (après convention)
10. **Nettoyage code mort + perfs réactivité** (backlog #12-16) — purge CSS/constantes mortes, rendu panier incrémental, lecture O(n) `Scelle_Transactions!B:B`.

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
| 17 | ~~**[Integrator] Renommage onglet cartes**~~ ✅ | **Fait en #9 (1.1.0)** : `SHEET_NAME` → `cartes_stock_et_transactions`. *Reste optionnel : wrapping `a1()` pour cohérence avec Sales/Master.* |
| 18 | **[Master] Dédup `txId` + ignorer `SALES_NOT_SYNCED`** | Master doit ignorer dans sa sync les lignes d'anomalie `SALES_NOT_SYNCED` (ventes jamais arrivées) et **dédupliquer par `txId`** (filet multi-instances, utile pour CardMarket à venir) |
| 19 | **[Master] Colonnes cartes dynamiques** | Transposer la détection par nom d'en-tête à Master, **y compris les écritures** (P/P:T/AF par lettre détectée) — chemin d'écriture, à faire avec test |
| 20 | **[Sales+Master] Refonte « centimes entiers »** *(partiel #10)* | **Acompte #10 (Sales 1.9.1)** : col K `ventes_detail` arrondie 2 déc. à l'écriture (`Math.round(x*100)/100`). Reste le **gros** : calculs internes en centimes-entiers partout (pervasif → étape dédiée). |
| 21 | **[Sécu R4] OAuth GIS + PKCE** | Flux implicite (déprécié) + jeton en `localStorage` (Master/Sales) → vol possible via XSS. Migrer vers Google Identity Services / Authorization Code + PKCE ; jeton hors `localStorage`. **= priorité #2 (v1.10.0).** *(bilan §A)* |
| 22 | **[Sécu R5] Réduire le scope OAuth** | `auth/spreadsheets` = accès à TOUTES les Sheets du compte → envisager `drive.file` + Google Picker (moindre privilège). *(bilan §A)* |
| 23 | **[Sécu] CSP + SRI** | Aucune `Content-Security-Policy` ni `integrity=` sur les 3 fichiers → ajouter (nécessite un smoke test : OAuth + scraping Integrator). *(bilan §A)* |
| 24 | **[Archi] Module commun** | Config + helpers (`esc`, `safeForSheets`, `a1`, `parsePrix`, OAuth…) dupliqués dans les 3 fichiers → extraire un `tcg-common.js` (une seule source de vérité). *(bilan §A)* |
| 25 | **[Archi] Option 3 — passerelle Apps Script** | Évolution « propre » : web app Apps Script (exécute en tant que propriétaire) sert le stock filtré + reçoit les paniers → vendeurs sans jeton plein scope ni accès Drive. Ferait tomber R4/R5 côté vendeur. *(bilan §A)* |
| 26 | **[Perf] Republier `STOCK_PUBLIC` après vente** | Actuellement republié au rythme du cache Master (~5 min). Pour une dispo quasi temps réel côté vendeur, republier juste après l'application d'une vente. *(#9)* |
| 27 | **[Sales+Master] Roll-back de vente** *(jalon coordonné)* | Annuler une vente déjà validée/envoyée (acheteur qui repart). Approche **append-only** retenue (on n'efface jamais une ligne partie chez Master) : écriture **compensatoire** dans `ventes_detail` — `txId` suffixé **`-RB`**, **quantités et total négatifs**, mêmes articles/acheteur — poussée via le **même outbox idempotent**. ⚠️ **Conséquence Master** : doit reconnaître les `txId` `-RB` / quantités négatives (re-créditer le stock, annuler le total, tracer). **Spécifier Master d'abord, puis câbler le bouton côté Sales** (état « vente validée » vert + bouton ↶, fenêtre de rétractation). *Ébauche commencée puis **retirée** en #10 pour livrer une 1.9.5 cosmétique propre (rien d'actif livré).* |

| 28 | **[Sales] Acheteur imposé** *(dernier item du palier 1.10.0)* | Lecture seule quand le nom d'event est rempli (`app_state` col B → `getResolvedBuyer()` ; réactiver `#buyerLocked`) ; vide → éditable + obligatoire. ~~Dashboard Master v6~~ ✅ **fait (Master 1.10.1)**. **= priorité #1 prochaine session**, puis bump Sales + déploiement conjoint. |

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
| OAuth | Projet GCP **TCG Integrator** (`774473792747`), client `774473792747-...rj96`, type Application Web |
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
| #7 | 29/05/2026 | **Revue de code des 3 apps, PUIS correctifs livrés après validation.** Bugs corrigés : 🔴 bloquant Master `applyCardSale` (bloc `copyPaste` parasite réf. `nextTxRow` hors scope → `ReferenceError` à chaque vente carte, fausse anomalie `CARD_NOT_FOUND`) ; 🟠 régression index `Synced` (col M lu au lieu de col O dans `renderSalesLog`/`loadCloseSummary` → résumé de clôture faussé) ; 🟡 Integrator (CRLF non normalisé, en-tête cartes dynamique, `parsePrix` locale). Livrés : **Master 1.6.2**, **Integrator 1.0.0** (1ʳᵉ mise sous versioning). Non traités → backlog : code mort (Sales `switchTab`+CSS, constantes/fonctions inutilisées, #16) et perfs réactivité (#12). `SAFETY_GSHEET_ID` laissé en placeholder (garde-fou explicite déjà présent). Ajout instruction §0 : revue de code systématique en début de chaque discussion. Petit fix complémentaire **Sales 1.6.3** : « pas vendu » retiré du menu TVA_Vente. **Test E2E (ventes cartes + scellé) concluant** sur les nouvelles versions. Détail complet en §5. |
| #8 | 29/05/2026 | **Revue stricte de `tcg-sales`, lot de durcissements, PUIS outbox idempotent.** Déclencheur : le renommage de l'onglet cartes en `cartes_stock_et_transactions` (côté GSheet) a cassé Master (`400 Unable to parse range`). **(a) Durcissements** — **Sales 1.7.0** (renommage `SHEET_CARDS` + helper `a1()` partout ; **colonnes cartes en détection dynamique** par nom d'en-tête avec repli sur offsets ; remise « ± Total » refaite en **plus grand reste / euros entiers** sur tous les articles, total exact ; **formateur monétaire Intl fr-BE** ; **journal `logEvent`** persistant + `exportLog()` ; reset acheteur après vente hors-convention ; purge `switchTab`/CSS morts ; **zoom + media query smartphone** ; `HEARTBEAT_POLL_MS` câblé, `SYNC_POLL_MS` retiré ; `HttpError`, `parseInt` base 10, `init().catch`, `txId` rendu unique) et **Master 1.6.3** (même renommage + `a1()` partout (lecture ET écriture) + `HttpError` + journal + Intl UI + `parseInt`/`init().catch`). NON transposé à Master : détection dynamique des colonnes (Master **écrit** par lettre fixe P/P:T/AF → trop risqué sans test) et format des nombres de l'archive de clôture. **(b) Outbox idempotent** — **Sales 1.8.0** : toute vente mise en file (`pending`) puis envoyée ; **idempotence par relecture `ventes_detail!D`** (txId) avant tout renvoi → ni doublon (réponse perdue) ni perte (envoi échoué) ; échec définitif (4xx) ou après `MAX_SALE_RETRIES` → onglet `anomalies` ; si l'écriture anomalies échoue aussi → **dead-letter localStorage** + bandeau rouge « Réessayer » (**zéro perte silencieuse**) ; **journal du cycle de vie**. Machine à états vérifiée en simulation (7 scénarios). **Sales 1.8.1** : `moveToAnomalies` aligné sur le **vrai schéma A:K** de l'onglet `anomalies` (une ligne/article, `SALES_NOT_SYNCED`, payload JSON en `Notes`). **`tcg-converter` non touché** (référence encore l'ancien nom → à corriger en #9). Mise à jour de ce document. *(NB : les changelogs internes des fichiers livrés étiquetaient l'outbox « Session #9 » ; à recaler en « #8 » si tu veux une cohérence parfaite.)* |
| #9 | 29/05/2026 | **Revue de sécurité (OWASP 2025) + Option 1.** **Integrator 1.1.0** : `esc()` (corrige XSS DOM via CSV/scraping) + `safeForSheets()` (corrige injection de formules) + **renommage onglet** `cartes_stock_et_transactions` (priorité #9.1 — il cassait dessus). **Option 1 (Master 1.9.0 + Sales 1.9.0)** : Master publie une vue stock sûre `STOCK_PUBLIC` dans EVENTS (cartes+scellés dispo, **sans coûts/marges**, + flag `Reserved` + liste TVA) ; Sales lit le stock de là et **ne lit plus CENTRAL** (fonctions mortes `findCardsHeaderRow`/`findColByName` retirées) → vendeurs sans accès CENTRAL, marges jamais exposées. Confirmé : 2 GSheets « Restreint ». **Versioning** : Option 1 modifiant les 2 apps ensemble, elles partagent **1.9.0** (Master re-tagué 1.6.3→1.9.0) ; l'auth GIS visée glisse en **1.10.0**. **Test E2E concluant** : Option 1 validée jusqu'à la **clôture d'événement** 🏁 (archive + vidage des buffers `ventes_detail`/`anomalies`) — la clôture, en attente depuis #8, est validée. **Hardening recommandé restant** : protéger `STOCK_PUBLIC` + ajouter vendeurs (comptes propres). Backlog sécurité ouvert : R4/R5, CSP/SRI, module commun, Option 3 Apps Script. Détail : `Bilan_risques_TCG.md`. |
| #10 | 29/05/2026 | **Revue §0 d'ouverture → constat que tout le « plan » était déjà livré, PUIS lot polish Sales.** La revue a révélé que les apps étaient en **1.9.0** (doc d'ouverture périmé en #7) : l'« augmentation du panier » et les « articles manuels » discutés étaient **déjà implémentés** (Sales `± Total` bidirectionnel + `+ Manuel` / Master `applyManualSale` no-op stock), et l'arrondi #6 déjà corrigé en #8. **Vérifié par exécution** que la règle de répartition « ± Total » (proportionnelle, euros entiers, plus grand reste, total exact) marche **à la hausse comme à la baisse** ; seul résidu = micro-dérive flottante ≤ 5,7e-14 € sur col K quand qty>1 (jamais visible). **Lot livré — Sales 1.9.1** (patch, **Sales seul** ; ⚠️ **1.10.0 reste réservé à l'auth GIS** Sales+Master) : hardening arrondi col K `Math.round(x*100)/100` (#20 partiel) · feedback de scan flash ✓/✗ (#14) · `renderCart()` **incrémental** par ligne (#12) · bandeaux d'état regroupés `#statusZone` + compactage mobile (#13) · purge reliquats code mort `sheetsUpdate`/`sheetsBatch` + vars `cardHeaderRow`/`cardColIdx`/`sealedColIdx` (#16 reliquat). Master & Integrator **non touchés**. Refonte centimes complète (#20) toujours ouverte. **PUIS refonte UI Sales complète (1.9.2 → 1.9.5, patch Sales seul, itérée sur maquettes)** : interface **3 zones** tenant dans un écran smartphone sans scroll global ; en-tête (logo PNG embarqué + titre « TCG SALES » entier + version collée dessous à droite + **pastilles rondes d'état** G/📦 cerclées vert/rouge/ambre, détail en info-bulle) ; milieu = **carrousel swipe panier ⇄ caméra** avec **scan caméra natif `BarcodeDetector`** (codes-barres + QR, repli gracieux — **#4 fait**, à valider sur téléphone) ; lignes article = 🗑 + **chip quantité tappable à gauche** (position fixe — **#10 fait/dépassé**) + nom + **prix unitaire éditable par ligne** (corrige une côte périmée sans toucher les autres) ; **doublon = même ligne ×N** (contour orange ancienne clé / **rouge bloquant** clé neuve) ; bas = client/type/total/**bouton Valider épinglé en Syne** (= police du titre). **Rouleaux remplacés par saisie clavier** (quantité/prix/total, `inputmode`). Pastille cache = anneau coloré (**#9 résolu**). **Roll-back de vente esquissé puis RETIRÉ** (bouton ↶/modale/styles) pour livrer une **1.9.5 propre** : il touche aux données déjà chez Master → **reporté en jalon coordonné (backlog #27)**, à spécifier côté Master d'abord. ⚠️ Caméra/swipe/claviers **non testés** (pas de navigateur en atelier). Aucune logique métier touchée. |

---

## 10. Versioning des Apps

| Version | Date | App(s) | Changements |
|---------|------|--------|-------------|
| 1.10.1 | 30/05/2026 | Master | **(#11, dashboard maquette v6 — Master seul)** En-tête : sous-titre retiré, **version à droite du titre**. Bandeau event compact `.event-strip` + **aide « ? »**. Ligne 2 col. : **« Synchronisation des ventes »** (Sync now / Pause auto à droite) | **« Clôture »** (aide « ? »). Titre « Stats… » retiré. **`.dash-grid`** : 4 KPI compacts **auto-shrink** (`fitKpis`/`data-fit`) · 2 **donuts** catégorie+licence (`pathLength=100`, **% en légende**, `CAT_COLORS`/`LIC_COLORS`+fallback, clé normalisée) · **⭐ Faits marquants** (`computeFacts`, 9 lignes) · **CA/heure horizontal** (`computeHours`/`renderHours`, 10h→17h, montant dans la barre, vides hachurés). Données agrégées de `ventes_detail` au refresh (rien stocké). `renderSplit`/`renderLeaderboard` retirés ; **Journal + Anomalies conservés** dessous. `toggleHelp()` (ferme au clic extérieur). *Sales inchangé (1.10.0). CSS mort `.stat-*`/`.split-*`/`.lb-*` laissé (cleanup mineur). NON testé navigateur.* |
| 1.10.0 | 30/05/2026 | Sales + Master | **(#11, palier coordonné)** **Master** : refonte en-tête (logo embarqué, titre, largeur 1240px, panneaux stock/Google), **page floutée tant que Google non connecté** + overlay accès refusé · **verrou d'instance lease/TTL** (`app_state` C/D = `lock_uuid`/`lock_label` ; acquisition écrire→jitter→relire ; éviction détectée au battement ; « Forcer la reprise » ; reprise auto si bail >35 s ; anti-ping-pong ; migration purge C2/D2) · **sync gated sur le bail** (plus sur l'event) · champs « acheteur défaut »/« label appareil » retirés · cleanup écritures mortes U/AD (scellé). **Sales** : statut master par **détection de changement** d'horodatage (warning **non bloquant**, ventes toujours envoyées) · arrêt lecture `app_state` col D comme acheteur. ⚠️ **Déployer les 2 ensemble.** **NON testé navigateur.** 🟡 **Reste à coder** (validé maquette v6) : **acheteur imposé** côté Sales (le dashboard Master est fait en 1.10.1). | Version collée sous le titre, **alignée à droite** (bord droit = celui du titre) · bouton « Valider » en **police du titre** (Syne 800, `-.05em`), alertes en Syne aussi (plus de Space Mono) · **retrait de l'ébauche de roll-back** (bouton ↶/modale/styles) → reporté en jalon coordonné (#27). Purement cosmétique. *Master & Integrator inchangés.* |
| 1.9.4 | 29/05/2026 | Sales | **(#10, en-tête compact — patch Sales seul)** Titre « TCG SALES » **non tronqué** (ligne dédiée) · connexions en **pastilles rondes** : `G` (Google) + `📦` (stock) cerclées **vert/rouge/ambre** (toucher = (dé)connexion / refresh ; détail compte/réf./âge en info-bulle) · version lisible. `updateAuthUI`/`updateCachePill` pilotent les anneaux (pilules texte conservées masquées pour compat). |
| 1.9.3 | 29/05/2026 | Sales | **(#10, ergonomie saisie — patch Sales seul)** **Rouleaux supprimés → saisie clavier** (`inputmode`) pour **quantité**, **prix unitaire** et **total** · **prix unitaire éditable par ligne** (corrige une côte GSheet périmée sans répartir sur tout le panier ; affiché « modifié ») · **🗑 à gauche** de chaque ligne (suppression directe ; qty 0 reste possible) · prix affiché = **prix unitaire** · bouton « Valider » agrandi (police Syne). |
| 1.9.2 | 29/05/2026 | Sales | **(#10, refonte UI 3 zones — patch Sales seul)** Écran smartphone **sans scroll global** (`100dvh`/`overflow:hidden`) ; HAUT logo PNG + titre ; MILIEU **carrousel swipe panier ⇄ caméra** (2 points) + **scan caméra `BarcodeDetector`** (codes-barres + QR, repli gracieux) + barre de scan fine ; lignes = chip quantité à gauche, nom pleine largeur, prix flotté ; **doublon = même ligne ×N**, contour orange (carte ancienne clé) / **rouge bloquant** (carte clé neuve) ; BAS champs sans libellés + total tappable + bouton Valider épinglé. Icônes = emojis ; logo client embarqué (base64). *Caméra/swipe non testés hors navigateur.* |
| 1.9.1 | 29/05/2026 | Sales | **(#10, lot polish — patch Sales seul ; 1.10.0 réservé à l'auth GIS)** Arrondi col K `ventes_detail` à 2 déc. `Math.round(x*100)/100` (#20 **partiel**) · feedback de scan flash ✓/✗ + pulse vert/rouge (#14) · `renderCart()` **incrémental** par ligne (clé `uid`, signature anti-réécriture, réordonnancement `appendChild`) → préserve focus/scroll (#12) · bandeaux d'état regroupés dans `#statusZone`, spacing géré par la zone + compactage mobile (#13) · purge code mort `sheetsUpdate`/`sheetsBatch` + vars `cardHeaderRow`/`cardColIdx`/`sealedColIdx` (#16 reliquat). *Master & Integrator inchangés ; nécessite toujours Master 1.9.0 (`STOCK_PUBLIC`).* |
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

### Versioning de l'Integrator (piste distincte, back-office)
L'outil `tcg-converter.html` (TCG Integrator) est désormais versionné séparément de la paire Sales/Master.

| Version | Date | Changements |
|---------|------|-------------|
| 1.1.0 | 29/05/2026 | **(#9)** Portage des garde-fous de Sales/Master : `esc()` sur tous les `innerHTML` (corrige XSS DOM via CSV importé / valeurs scrapées) · `safeForSheets()` sur les colonnes texte du CSV dans `buildCardRow` (corrige injection de formules) · `buildLotRow` double les guillemets dans `COUNTIF`/`SUMIF` · **`SHEET_NAME` → `cartes_stock_et_transactions`** (priorité #9.1 — l'onglet renommé en #8 cassait l'Integrator). Comportement inchangé pour des données normales. |
| 1.0.0 | 29/05/2026 | Premier stamp formel. Correctifs revue #7 : 🟡 `parseCSV` normalise CRLF/CR → LF · 🟡 détection dynamique de l'en-tête cartes (`findCardsHeaderRow`/`ensureCardHeaderRow`) au lieu de `143` codé en dur · `parsePrix` locale belge. `SAFETY_GSHEET_ID` reste un placeholder à renseigner (garde-fou explicite déjà en place). |
