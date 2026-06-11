# ChangeLog — Module Distillerie

---

## [0.8.2] — 2026-05-18

### Ajouté
- **Agent de contrôle qualité autonome** (`test/agent_controle.php`) : 12 familles de contrôles statiques basés sur le skill CLAUDE_template_module_dolibarr_V2 (descripteur, bootstrap, sécurité, pagination, TCPDF, inclusions, redirections, langues, classes, SQL, lint PHP). Score de conformité affiché. Utilisable en CLI : `php test/agent_controle.php`
- Fichier `VERSION` à la racine (source de vérité unique de la version)
- Balises `/* BEGIN/END MODULEBUILDER PERMISSIONS */` et `/* BEGIN/END MODULEBUILDER DICTIONARIES */` dans le descripteur (conformité ModuleBuilder)

### Modifié
- `editor_url` → `https://github.com/Barnecki` (convention BARNECKI SPIRITS du skill)
- `url_last_version` activé → `https://raw.githubusercontent.com/Barnecki/distillerie/main/VERSION` (détection auto de mise à jour Dolibarr)
- `$this->version` lue depuis le fichier `VERSION` (fallback hardcodé)
- `langs/fr_FR/distillerie.lang` : 28 apostrophes droites remplacées par apostrophes typographiques (règle skill : pas d'apostrophe dans les .lang)

### Conformité
- Score agent de contrôle : **100%** (0 erreur, 0 avertissement, 37 contrôles passés)

---


## [0.8.1] — 2026-05-18

### Ajouté
- `EVOLUTIONS.md` : fichier de suivi de toutes les évolutions évoquées non encore implémentées (17 évolutions listées, classées par priorité)

---


## [0.8.0] — 2026-05-18

### Modifié
- Repackaging officiel — numérotation de version alignée sur le cycle de livraison
- `README.md` : version courante mise à jour → 0.8.0
- `CLAUDE.md` : version courante mise à jour → 0.8.0
- `ChangeLog.md` : entrée 0.8.0 ajoutée

---


---

## [0.7.5] — 2026-02-26

### Ajouté
- Requêtes SQL `CREATE TABLE` et `INSERT IGNORE` pour `llx_dist_dict_typeregistre` dans la fonction `init()` de `modDistillerie.class.php` (création automatique à l'activation)
- Table `llx_dist_dict_typeregistre` ajoutée dans la fonction `remove()` (suppression à la désactivation)
- Fichier `README.md` entièrement réécrit (structure, tables, permissions, menus, import, librairies)
- Fichier `CLAUDE.md` créé (guidance Claude Code : conventions, architecture, pièges courants)
- Fichier `ChangeLog.md` mis à jour à chaque version

---

## [0.7.4] — 2026-02-26

### Ajouté
- Dictionnaire `dist_dict_typeregistre` enregistré dans `$this->dictionaries` de `modDistillerie.class.php` (7e dictionnaire du module)

### Supprimé
- Page `admin/dict_typeregistre.php` (remplacée par l'interface standard des dictionnaires Dolibarr)
- Onglet "Types de registres" dans `distillerieAdminPrepareHead()` (`lib/distillerie.lib.php`)

---

## [0.7.3] — 2026-02-26

### Ajouté
- Table `llx_dist_dict_typeregistre` avec champs : rowid, entity, code, label, active, position, tms, fk_user_author, fk_user_modif
- Classe `distdicttyperegistre.class.php` avec méthodes `getAllActive()` et `getLabelByCode()`
- 6 types de registres par défaut : PRODUCTION, DISTILLATION, STOCK, MOUVEMENTS, PERTES, COMPTE_MATIERES
- Script de migration `sql/migrate_typeregistre_0_7_3.sql` (INTEGER → VARCHAR)

### Modifié
- `distregistry_card.php` : champ `typeregistre` utilise maintenant le dictionnaire (select dynamique)
- `distregistry.class.php` : champ `typeregistre` déclaré en `varchar(64)` au lieu de `integer`
- `llx_dist_registry.sql` : colonne `typeregistre` changée de `INTEGER` à `VARCHAR(64)`
- Actions `add` et `update` : `typeregistre` récupéré via `GETPOST('typeregistre', 'alpha')`

---

## [0.7.2] — 2026-02-26

### Ajouté
- `distregistry_card.php` : 14 champs manquants ajoutés dans les formulaires création ET édition
  - **RÈGLE 01** : numero_entrepot, siret, raison_sociale, adresse_siege, adresse_exploitation, numero_caution, dispense_caution
  - **RÈGLE 09** : date_cloture_exercice
  - **RÈGLE 05** : campagne_viticole, page_garde, nb_feuillets_type
  - **RÈGLE 10** : taux_pertes, taux_dechets
- Actions `add` et `update` : récupération complète de tous les champs POST
- Traductions FR/EN pour tous les nouveaux champs

---

## [0.7.1] — 2026-02-22

### Modifié
- `lib/distillerie.lib.php` : étape `PREPARATION` mise à jour avec tous ses champs
  - Ajout : `qte_alcool` (number), `unite_qte_alcool` (select l/ml)
  - Labels explicites pour tous les champs existants
- `production_step_edit.php` : édition autorisée pour `PREPARATION` et `DEBUT_MACERATION`

---

## [0.7.0] — 2026-02-22

### Ajouté
- **Pagination** dans `distprod_list.php` : comptage total, barre de navigation `print_barre_liste()`
- **Rang des étapes** à l'import (`distimport.class.php`) : ordre fixe RECOLTE=10, NETTOYAGE=20, BROYAGE=30, Défécation=40, FERMENTATION DEBUT=50, FERMENTATION FIN=60, DISTILLATION=70, DECHETS=80, RECYCLAGE=90

---

## [0.6.9] — 2026-02-21

### Corrigé
- `distimport.class.php` : champ `json_data` (était `description`), `date_event` (était `date_step`), `fk_user` (était `fk_user_creat`) dans `importProductionSteps()`
- Dates production : utilisation de `$db->idate()` pour format MySQL correct
- Récupération données RECOLTE : `isset()` au lieu de `??` pour compatibilité PHP 7.x

---

## [0.6.8] — 2026-02-21

### Corrigé
- `distimport.class.php` : ajout de `$production->type_prod = 'import'` (champ NOT NULL manquant qui causait l'échec silencieux de `create()`)

---

## [0.6.7] — 2026-02-21

### Modifié
- `distimport.class.php` : ajout de logs de debug dans `importFromExcel()` pour diagnostiquer les problèmes de parsing
- Filtrage noms productions : regex `/^TEST/i` ou `/^[A-Z]+[0-9]+/` au lieu de liste noire

---

## [0.6.6] — 2026-02-21

### Corrigé
- `distimport.class.php` : filtre amélioré pour détecter les vrais noms de productions vs données

---

## [0.6.5] — 2026-02-21

### Ajouté
- `includes/SimpleXLSX.php` : librairie autonome de lecture XLSX (ZipArchive natif PHP, sans Composer)

### Modifié
- `distimport.class.php` : utilisation de `includes/SimpleXLSX.php` embarqué (chemin via `dol_buildpath`)

---

## [0.6.4] — 2026-02-21

### Modifié
- `distimport.class.php` : remplacement de PhpSpreadsheet par SimpleXLSX (abandon de PhpOffice)

---

## [0.6.3] — 2026-02-21

### Corrigé
- `distimport.class.php` : tentative d'autoloader SPL manuel pour PhpSpreadsheet (échec PSR)

---

## [0.6.2] — 2026-02-21

### Corrigé
- `distimport.class.php` : tentative `Bootstrap.php` PhpSpreadsheet (échec Composer)

---

## [0.6.1] — 2026-02-21

### Corrigé
- `admin/import_productions.php` : remplacement de `require '../main.inc.php'` par le bloc multi-chemins standard Dolibarr

---

## [0.6.0] — 2026-02-21

### Ajouté
- Fonctionnalité d'import en masse des productions depuis fichier Excel (TRACABILITE.xlsx)
- Classe `distimport.class.php` avec parsing structure Excel, création productions + étapes, gestion fruits automatique
- Page `admin/import_productions.php` : interface upload, résultats, documentation mapping
- Onglet "Import productions" dans `distillerieAdminPrepareHead()`
- Librairie embarquée `includes/SimpleXLSX.php`

---

## [0.5.9] — 2026-02-21

### Corrigé
- `distdistil_dashboard.php` : sécurisation des 4 requêtes SQL stats (pattern `if ($resql && $obj)`)

---

## [0.5.8] — 2026-02-21

### Corrigé
- `distdistil_dashboard.php` : accolades fermantes manquantes dans les boucles `while` (erreur Parse)
- `distreg_dashboard.php` : même correction sur 4 boucles

---

## [0.5.7] — 2026-02-21

### Corrigé
- `pdf_declaration_mensuelle.modules.php` : ajout du `require_once` manquant pour `CommonDocGenerator`

---

## [0.5.6] — 2026-02-21

### Corrigé
- `distdashboard.php` : 6 requêtes SQL sécurisées (initialisations à 0 + `if ($resql && $obj)`)
- `distprod_dashboard.php` : 5 requêtes + 3 boucles `while` sécurisées

---

## [0.5.5] — 2026-02-21

### Corrigé
- `distexportexcel.class.php` : suppression des instructions `use` dans blocs conditionnels (Parse error PHP)
- Utilisation des noms complets de classes PhpSpreadsheet (`\PhpOffice\PhpSpreadsheet\...`)

---

## [0.5.4] — 2026-02-21

### Corrigé
- `distdeclaration.class.php` : remplacement de `global $db` / `$db->` par `$this->db->` dans `generateDeclarationMensuelle()`
- Toutes les classes services héritent maintenant de `CommonObject` avec `public $fields = array()`

---

## [0.5.3] — 2026-02-21

### Ajouté
- Classes services conformes ModuleBuilder : DistDeclaration, DistAutomation, DistIntegration, DistAudit, DistExportExcel étendent CommonObject
- Automatisations et intégrations externes

---

## [0.5.0] — 2026-02-20

### Ajouté
- Dashboard global `distdashboard.php` (KPI cards, graphiques Chart.js, derniers mouvements)
- Dashboard productions `distprod_dashboard.php`
- Dashboard distillations `distdistil_dashboard.php`
- Dashboard registres `distreg_dashboard.php`
- Génération PDF déclarations mensuelles (`pdf_declaration_mensuelle.modules.php`)
- Export Excel

---

## [0.4.0] — 2026-02-20

### Ajouté
- Interface mouvements (`distmouvement_card.php`, `distmouvement_list.php`)
- Catégories fiscales (`distcategoriefiscale.class.php`)
- Sites de production (`llx_dist_site.sql`)
- Inventaires (`llx_dist_inventaire.sql`)
- Déclarations mensuelles (`distdeclaration_generate.php`)
- Notifications DGDDI

---

## [0.3.x] — 2026-02-16 à 2026-02-20

Voir l'historique complet dans les archives pour les versions 0.3.x (conformité CGI, droits, menus, dictionnaires, corrections multiples).

