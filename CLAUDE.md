# CIREPCI — Application de production (PWA)

## Ce qu'est ce projet
Application web **mobile-first**, **hors-ligne**, **mono-fichier**, sans backend, pour le suivi
de production d'une biscuiterie (CIREPCI, francophone, Côte d'Ivoire). Elle remplace un fichier
Excel (PowerPivot/PowerQuery/VBA). Les données restent **sur l'appareil** (localStorage) ; aucune
donnée n'est stockée sur l'hébergeur.

- Hébergée sur **GitHub Pages** (déploiement automatique à chaque commit sur `main`).
- Le fichier servi est **`index.html`** : tout le HTML + CSS + JS est dans ce seul fichier.
- C'est aussi une **PWA** : `manifest.webmanifest`, `sw.js` (service worker, cache-first), et les
  icônes `icon-192.png`, `icon-512.png`, `icon-512-maskable.png`.

## Langue & style
- Toute l'interface et toutes les réponses sont en **français**.
- Style visuel sobre/industriel : encre `#13202b`, bleu acier `#1b5faa` (foncé `#15487f`),
  vert `#1f7a4d`, rouge `#b0455a`, fond papier `#f4f5f2`. Chiffres en police **monospace tabulaire**.

## Règles d'or (à ne jamais casser)
1. **Ne jamais re-render tout l'écran sur la frappe d'un champ** (`oninput`) : ça fait perdre le
   focus. Mettre à jour la donnée + l'affichage dérivé **en place** ; re-render global uniquement
   sur un changement de structure (ajout/suppression de ligne, changement de type).
2. **Tout est relié par CODE, jamais par nom** (articles, batches, produits).
3. **Pas de doublons** : dans toute liste où l'on choisit un article/ingrédient (entrée MP,
   composition de batch, emballages d'une recette, articles d'un fournisseur), un code déjà
   utilisé est retiré des autres menus. La fonction `grpOpts(arts, sel, hideCode, exclude)` gère
   l'exclusion ; `usedCodes(arr, field, exceptIdx)` calcule les codes déjà pris.
4. **Code défensif** : tolérer les valeurs nulles, `try/catch` sur stockage et partage.
5. **Arrondis fixés** (souvent 2 décimales) ; helper `nf(valeur, décimales)`.
6. **localStorage ne marche pas dans l'aperçu d'artefact**, mais marche une fois hébergé (HTTPS).

## Déploiement & PWA
- Après **toute modification visible** de `index.html`, **incrémenter la version du cache** dans
  `sw.js` (`cirepci-v1` → `cirepci-v2`, etc.) sinon les utilisateurs gardent l'ancienne version.
- Pas de librairies externes : l'app est 100 % autonome. Ne pas ajouter de CDN sans raison.
- Toujours **regrouper** les changements : éviter de multiplier les commits/déploiements inutiles.

## Modèle de données (clés de STATE / localStorage)
- `articles` {code, designation, categorie (Matière/Emballage), sous_categorie}
- `batches` {code, type (Pâte/Crème), nom, poids_std, poids_ref, composition[{code, nom, qte}], au_catalogue}
- `produits` {nom, categorie, code_pate, code_creme, code_creme_2, pct_creme_1/2, pct_pate,
  hum_farine_vrac, pct_farine_pate, net_cartons, pertes_matiere, pertes_emballage, emballages[{code,nom,qte,sous_cat}]}
- `previsionnel` {annee, version, produit, mensuel[12], date_saisie}  (Budget = 12 mois ; révisions = mois restants)
- `inventaires` (sessions par localisation) {date, localisation (Atelier/Dépôt), lignes[{code, qte}]}
- `entrees_mp` (sessions) {date, lignes[{code, qte}]}
- `commandes` {id, date, fournisseur, code, qty_commandee, date_estimee, receptions[{date, qte}]}
- `fournisseurs` {id, nom, contact, delai_moyen_jours}
- `fournisseur_article` {fournisseur, code, prix, devise, qty_min, delai_jours}
- `production` {date, equipes[{n, productions[{produit, cartons, dechet_film, dechet_carton}],
  pates[{code, nb_batchs, ajustement, restant}], cremes[...]}]}
- `parametres` {marge_securite_jours, seuil_cmd_recues, delai_defaut_jours, prev_cutoff_jour}
- `jours_ouvres` [12 valeurs]

## Logique métier essentielle (à préserver)
- Formules pâte/crème : `R = 1 − Pct_Farine_Pate × Humidite_Farine_Vrac` ;
  `K = (1−Pct_Pate)/Pct_Pate` (0 si Pct_Pate=1) ; `Poids_Pate = Qté_Cartons × Net_Cartons/(R+K)` ;
  `Poids_Creme = Poids_Pate × K`. Le besoin matière passe TOUJOURS par le poids du batch.
- **Saisie production** : on saisit les **produits** (cartons + déchets) ; les **pâtes et crèmes
  consommées sont déduites automatiquement** des produits (fonction `syncDeclas`), on ne peut pas
  en ajouter manuellement. L'utilisateur ne renseigne que le réel (batchs/ajust/restant) pour les pertes.
- **Pertes** : Réel engagé = Nb×poids_std + ajustement − restant ; Perte = Réel − Théorique.
  Comparer le % aux seuils de la recette (Pertes_Matiere / Pertes_Emballage). Ne jamais mélanger
  pâte et crème sur un même graphe.
- **Besoins (appro)** : horizon 18 mois glissants, stock physique + planifié (commandes injectées
  au mois d'arrivée), jours de stock, période creuse, stock de sécurité, 5 niveaux d'alerte.
  Le besoin appro est calculé au niveau **matière première** (les batches sont éclatés en MP).
- **Goulots** : capacité = cartons encore fabricables avec le stock actuel estimé
  (dernier inventaire + entrées − sorties théoriques), et la MP qui limite.
- **Conso réelle** = Stock initial + Entrées − Stock final entre deux inventaires.

## Architecture du code source (information)
À l'origine, `index.html` était **généré** par un script Python `build_app.py` qui injecte les
données (`seed.json`) dans un template. Dans ce dépôt, on travaille **directement sur `index.html`**
(pas de build). Faire des éditions ciblées et vérifier l'équilibre des balises `<script>`/`</script>`.

## Pour toute modification
- Vérifier la **syntaxe JS** avant de livrer (équilibre des accolades, des balises).
- Respecter la **charte ergonomique** : gros boutons tactiles, claviers adaptés
  (`inputmode`), Entrée → champ suivant, sélection auto au focus, sections pliables avec compteur
  d'avancement coloré, aperçu lisible avant export, confirmation avant action destructrice.
- Après une modif visible : **bump du cache `sw.js`**.
