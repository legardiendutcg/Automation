# Contexte Projet — Le Gardien du TCG
> Document à coller en premier message dans chaque nouvelle session Claude.
> Dernière mise à jour : Session #6 — 29/05/2026

---

## 0. Instructions pour Claude

- **Tout au long de la discussion** : mettre à jour ce document en arrière-plan au fil des avancées — ajouter les nouveaux milestones complétés, les nouvelles idées de développement à garder pour plus tard, et retirer du backlog les améliorations sur lesquelles on a travaillé et validées.
- **Quand il est opportun d'ouvrir une nouvelle discussion** (contexte trop long, session de travail cohérente terminée) : prévenir, mettre à jour ce document une dernière fois, puis proposer de basculer.
- **En début de chaque session**, lorsqu'une étape du développement est identifiée : indiquer quel modèle Claude est le plus adapté pour cette étape (Haiku / Sonnet / Opus) avec une justification courte.
- **À chaque livraison de fichier** : bumper systématiquement la version (patch/minor/major selon le changement) avant de présenter le fichier. Ne jamais livrer deux fichiers successifs avec le même numéro de version. Quand les deux apps sont modifiées ensemble, elles partagent le même numéro de version pour garder un mapping cohérent entre Sales et Master.
- **Au début de la prochaine session** : les deux features prioritaires de la session #5 (TVA_Vente + acheteur côté Sales) ont été livrées en v1.6.0. Prochaine priorité à challenger avant de coder : **tester la clôture d'événement 🏁** (archive + reset des buffers `ventes_detail` / `anomalies`) puis **les autres configurations cartes** (gradées, articles manuels). Questionner les choix de design, signaler les risques, obtenir une validation explicite avant implémentation.
- **Vigilance sur les caracteres speciaux** lors du traitement de donnees provenant des Google Sheets (locale belge) : les virgules (separateur decimal vs separateur de milliers), les apostrophes (droite ' U+0027 vs courbe ' U+2019 que Google Sheets insere souvent automatiquement), et les accents (e/E vs e/E, etc.) sont des sources frequentes de bugs silencieux (find() qui renvoie -1, parseFloat qui interprete mal le format, etc.). Toujours considerer ces cas lors des comparaisons de strings ou du parsing de nombres.
- Ces cinq instructions s'appliquent automatiquement dans chaque session — inutile de les rappeler.

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
const SHEET_CARDS        = 'Cartes_Stock_&_Transactions';
const SHEET_SEALED       = 'Scelle_Stock';
const SHEET_SEALED_TX    = 'Scelle_Transactions';
```

### Google Sheets (base de données centrale — CENTRAL_GSHEET_ID)
- `Cartes_Stock_&_Transactions` — stock + transactions cartes (structure existante, ne change pas)
- `Scelle_Stock` — état du stock scellé actuel (header dynamique : ligne 26 actuellement, mais le code détecte automatiquement la première ligne dont col C = "Key")
- `Scelle_Transactions` — historique de toutes les ventes/mouvements scellés
- Onglet compta — agrège toutes transactions, calcule TVA et marges (formules pointent vers `Cartes_Stock_&_Transactions` et `Scelle_Transactions`)

### Google Sheets événements (EVENTS_GSHEET_ID)
- Onglets temporaires créés à chaque convention : `ventes_detail`, `heartbeat`, etc.
- Archivés et vidés à la clôture de l'événement (bouton 🏁)

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
- Usage : ouvert sur le téléphone de chaque vendeur pendant une convention
- Flow : connexion OAuth → chargement cache → scan barcode → panier → validation → écriture `ventes_detail`

### TCG Master (app maître — gérant)
- Fichier : `tcg-master.html`
- Usage : ouvert sur l'iPad ou ordinateur du gérant pendant la convention
- Flow : écoute `ventes_detail` (heartbeat) → synchronise le stock → gère l'événement

### Communication inter-apps
Les deux apps ne se parlent **pas directement** — elles communiquent via Google Sheets :
- Sales écrit dans `ventes_detail` (EVENTS_GSHEET_ID)
- Master lit `ventes_detail` et met à jour les onglets stock (CENTRAL_GSHEET_ID)

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

> ⚠️ **À revérifier (session #6)** — cette table (Scelle_Stock) est antérieure aux corrections et présente un décalage. Faits **vérifiés** : `TVA_Vente` est en **AG** sur les onglets scellés (liste autorisée en `Scelle_Transactions!AG8:AG23`), et en **AF** sur `Cartes_Stock_&_Transactions`. La ligne de **transaction** écrite par Master a une colonne **A = « x »** en tête, et son ordre exact (différent de cette table : Key/Code Série notamment) est défini par `txRow` dans `applySealedSale` — **le code fait foi**. Les positions A→AF de cette table restent à confirmer contre le sheet live.

### Scelle_Transactions (historique, même structure de colonnes)
- Même colonnes que Scelle_Stock
- 1 ligne par mouvement : vente (Statut=vendu), vol, ouverture, échange, transfert, cadeau
- Alimenté manuellement (historique) + automatiquement par TCG Master à chaque vente convention

---

## 4. Flow Scellé dans les Apps

### Cache (`loadStockCache` — Sales + Master)
- Lit `Scelle_Stock!A:AF`
- Détecte dynamiquement la ligne de header (col C = "Key")
- Indexe par colonne C (Key = barcode EAN) — articles sans Key ignorés
- Filtre : statut ≠ vendu/x ET qty > 0

### Vente scellée (Sales)
- Scanner EAN (`/^\d{8,14}$/`) → lookup `stockCacheSealed`
- **Verrou** : si l'article a `TVA_Vente = "réservée"` (lu col AG, normalisé accents), le scan est bloqué (pas d'ajout au panier)
- Sélecteur **TVA_Vente** (liste `Scelle_Transactions!AG8:AG23`, « réservée » exclue, défaut "Régime normal") + champ **Acheteur** (lecture seule si Master publie un acheteur par défaut, sinon saisie vendeur obligatoire)
- Ajout au panier → validation → ligne dans `ventes_detail` (A:N) avec `source='scelle'`, `TVA_Vente` (col M) et `Acheteur` (col N)

### Sync (Master — `applySealedSale`)
1. Lookup par Key dans `stockCacheSealed`
2. Lecture ligne complète : `Scelle_Stock!A{row}:AG{row}`
3. Vérification overdraw
4. Décrémentation qty dans `Scelle_Stock`
5. Calcul Pa proportionnel : `Pa_lot × (qtySold / currentQty)`
6. Écriture ligne dans `Scelle_Transactions` (colonnes A:AG)
   - Acheteur (col Q) = `ventes_detail.N` si rempli, sinon `currentEventName`
   - TVA_Achat (col AF) = lue depuis le lot source
   - **TVA_Vente (col AG)** = `ventes_detail.M` (défaut "Régime normal") — *écriture AG ajoutée en v1.6.0, elle était auparavant omise*
   - Facture de vente / Facture_Vente (cols U / AD) = `currentEventName` (référence session, découplée de l'acheteur)

---

## 5. État Actuel — Fin Session #6 (29/05/2026)

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

### 🔧 À tester en session #7
- **v1.6.0 end-to-end** : sélecteur TVA_Vente (scellé + carte), verrou « réservée », acheteur Sales (mode Master-fournit vs mode saisie vendeur), écriture effective de AG
- Clôture événement avec bouton 🏁 (créer tab archive, vider les buffers)
- Autres configurations cartes : cartes gradées, articles manuels

### 🔑 Douchettes en attente
- À la réception : séance de scan pour associer un vrai EAN à chaque article en stock
- Même procédure à chaque nouvel article entrant en stock

---

## 6. Prochaines Étapes (dans l'ordre)

### ✅ Livré en session #6 (v1.6.0)
1. ~~**[Sales] Choix du type de vente (TVA_Vente)**~~ — fait. Source = `Scelle_Transactions!AG8:AG23`, écrit en AG (scellé) / AF (carte).
2. ~~**[Sales] Choix de l'acheteur**~~ — fait via modèle Master-autoritaire (acheteur par défaut publié dans `app_state` col D ; sinon saisie vendeur). La détection Levenshtein d'uniformisation est **descopée** (le modèle Master-autoritaire supprime la divergence à la source) → reléguée au backlog si un jour besoin.

### 🔥 Priorité haute — à challenger en début de prochaine session
3. **Tester clôture événement** 🏁 : archive (création tab dédié) + reset des buffers `ventes_detail` / `anomalies`. Vérifier au passage la précaution de migration v1.6.0 (vider `ventes_detail` avant upgrade).
4. **Autres configurations cartes** : cartes gradées, articles manuels.

### Reste du backlog ordonné
5. **Workflow boutique / vente comptoir** (après convention)

---

## 7. Backlog Améliorations

| # | Amélioration | Contexte |
|---|---|---|
| 1 | Auto-reload cache slave à la reconnexion | Cache à 0 après re-auth OAuth |
| 2 | Refresh silencieux token OAuth | Token expire ~1h, reconnexion manuelle |
| 3 | Toggle "Sync actif sur ce device" | Risque double-sync si 2 masters ouverts |
| 4 | Scanner caméra 📷 en fallback | Si douchette en panne ou code-barres abîmé |
| 5 | Auto-création des onglets manquants | Master affiche erreur si onglet absent |
| 6 | Correction arrondi 1¢ sur "Ajuster total" | Ex: 56€ → 50€ donne 49.99€ |
| 7 | **Panneau de log pliable** (🐞) | Capturer tous les `showToast` + `console.error/warn` avec horodatage — les toasts d'erreur disparaissent trop vite pour debug (vécu en session #4) |
| 8 | Table fournisseurs/vendeurs pros | Auto-déterminer TVA_Achat selon le vendeur |
| 10 | **[Sales] Boutons +/- quantité panier** | Déplacer les boutons +/- vers la gauche de la ligne article pour que leur position reste fixe quand le prix total change de largeur (variation du nb de chiffres/décimales) |
| 9 | Pill `📦 N` reste affiché sur état erreur | Le timer de refresh recalcule le total et écrase "✗ Cache HS" — petit artefact UX |
| 11 | Uniformisation acheteurs (Levenshtein) | Descopé en #6 (modèle Master-autoritaire suffit). À ressortir seulement si on autorise un jour la saisie acheteur libre multi-vendeurs en convention |

### ✅ Sorti du backlog en session #4
- ~~Migration Netlify → GitHub Pages~~ (fait)

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

---

## 10. Versioning des Apps

| Version | Date | App(s) | Changements |
|---------|------|--------|-------------|
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
