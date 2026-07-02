# Module Distillerie — Dolibarr

Module externe Dolibarr (v21.x+) dédié à la gestion de la production d'alcool artisanale (rhum, eau-de-vie, etc.) avec traçabilité complète des étapes de production, suivi des distillations, gestion des registres douaniers et import de données historiques.

---

## Fonctionnalités principales

### 🏭 Productions
- Création et gestion des lots de production (`TESTxxx`, `BATCHxxxxxx`)
- Suivi des étapes par lot avec données en JSON :
  - RECOLTE, NETTOYAGE, BROYAGE, DÉFÉCATION
  - FERMENTATION/MACÉRATION DEBUT, FERMENTATION/MACÉRATION FIN
  - DISTILLATION, DECHETS, RECYCLAGE, PREPARATION
- Étapes ordonnées par rang (10, 20, 30…) selon le processus de production
- Import en masse depuis fichier Excel (format Traçabilité)
- Pagination de la liste des productions

### 🔥 Distillations
- Gestion des distillations liées aux productions
- Lignes de suivi (Tête, Cœur, Queue : volumes ml et TAV %)
- Dashboard distillation avec graphiques (Chart.js)

### 📋 Registres
- Gestion des registres douaniers officiels
- Champs complets : identification entrepôt fiscal, SIRET, caution, adresses
- Informations réglementaires (RÈGLES 01, 05, 09, 10 DTI+)
- Type de registre lié au dictionnaire `dist_dict_typeregistre`
- Mouvements associés aux registres

### 📊 Dashboards
- Dashboard global (productions + distillations + registres)
- Dashboard productions avec graphiques par fruit et évolution mensuelle
- Dashboard distillations avec statistiques TAV
- Dashboard registres avec mouvements

### 📤 Déclarations mensuelles
- Génération PDF des déclarations récapitulatives mensuelles (DRM)
- Conforme RÈGLE 06 CGI / DTI+ v1.0.23

---

## Installation

1. Copier le dossier `distillerie/` dans `htdocs/custom/`
2. Activer le module dans **Configuration → Modules/Applications**
3. À l'activation : toutes les tables et données par défaut sont créées automatiquement
4. À la désactivation : les tables dictionnaires sont supprimées

> ⚠️ Si vous mettez à jour depuis une version < 0.7.3, exécuter le script `sql/migrate_typeregistre_0_7_3.sql` pour migrer le champ `typeregistre` de INTEGER vers VARCHAR(64).

---

## Structure des fichiers

```
distillerie/
├── admin/
│   ├── about.php               — Page À propos
│   ├── alambic.php             — Gestion alambics
│   ├── automation.php          — Automatisations
│   ├── dict_broyage.php        — Dictionnaire broyage
│   ├── dict_etats.php          — Dictionnaire états
│   ├── dict_fruits.php         — Dictionnaire fruits
│   ├── dict_futs.php           — Dictionnaire fûts
│   ├── dict_nettoyage.php      — Dictionnaire nettoyage
│   ├── autodiag.php            — Autodiagnostic runtime du module
│   ├── import_productions.php  — Import masse depuis Excel
│   ├── integration.php         — Intégrations externes
│   └── setup.php               — Configuration module
├── class/
│   ├── distaudit.class.php            — Journalisation audit
│   ├── distautomation.class.php       — Automatisations
│   ├── distcategoriefiscale.class.php — Catégories fiscales
│   ├── distdeclaration.class.php      — Déclarations mensuelles
│   ├── distdicttyperegistre.class.php — Dictionnaire type registre
│   ├── distdistil.class.php           — Distillations
│   ├── distdistilline.class.php       — Lignes de distillation
│   ├── distexportexcel.class.php      — Export Excel
│   ├── distimport.class.php           — Import productions Excel
│   ├── distintegration.class.php      — Intégrations
│   ├── distmouvement.class.php        — Mouvements registre
│   ├── distprod.class.php             — Productions
│   ├── distprodstep.class.php         — Étapes de production
│   ├── distregistry.class.php         — Registres
│   └── simplexlsx.class.php           — Lecture Excel (legacy)
├── core/modules/
│   ├── modDistillerie.class.php       — Descripteur module
│   └── distillerie/doc/
│       └── pdf_declaration_mensuelle.modules.php
├── includes/
│   └── SimpleXLSX.php          — Librairie lecture XLSX autonome
├── langs/
│   ├── fr_FR/distillerie.lang  — Traductions français
│   └── en_US/distillerie.lang  — Traductions anglais
├── lib/
│   └── distillerie.lib.php     — Fonctions partagées + définition étapes
├── sql/
│   ├── llx_dist_prod.sql
│   ├── llx_dist_prod_step.sql
│   ├── llx_dist_distil.sql
│   ├── llx_dist_distil_line.sql
│   ├── llx_dist_registry.sql
│   ├── llx_dist_mouvement.sql
│   ├── llx_dist_categorie_fiscale.sql
│   ├── llx_dist_dict_fruits.sql
│   ├── llx_dist_dict_etats.sql
│   ├── llx_dist_dict_nettoyage.sql
│   ├── llx_dist_dict_broyage.sql
│   ├── llx_dist_dict_futs.sql
│   ├── llx_dist_dict_alambic.sql
│   └── llx_dist_dict_typeregistre.sql
├── distdashboard.php
├── distprod_list.php
├── distprod_card.php
├── distprod_dashboard.php
├── distdistil_list.php
├── distdistil_card.php
├── distdistil_dashboard.php
├── distregistry_list.php
├── distregistry_card.php
├── distreg_dashboard.php
├── distmouvement_list.php
├── distmouvement_card.php
├── distdeclaration_generate.php
├── production_step.php
└── production_step_edit.php
```

---

## Base de données

### Tables principales

| Table | Description |
|-------|-------------|
| `llx_dist_prod` | Productions (ref, date_prod, type_prod, fk_fruit, quantite_totale, lieu_recolte) |
| `llx_dist_prod_step` | Étapes production (fk_prod, step_code, step_label, rang, date_event, json_data) |
| `llx_dist_distil` | Distillations (fk_prod, date_distillation) |
| `llx_dist_distil_line` | Lignes distillation (fk_distil, cut_type, volume_ml, tav_pct) |
| `llx_dist_registry` | Registres (label, typeregistre VARCHAR(64), numero_entrepot, siret…) |
| `llx_dist_mouvement` | Mouvements registre |
| `llx_dist_categorie_fiscale` | Catégories fiscales |

### Dictionnaires (créés à l'activation, supprimés à la désactivation)

| Table | Description | Valeurs par défaut |
|-------|-------------|-------------------|
| `llx_dist_dict_fruits` | Fruits utilisés | — |
| `llx_dist_dict_etats` | États qualité 1-5 | — |
| `llx_dist_dict_nettoyage` | Types nettoyage | RINCAGE, RINCAGE_EPLUCHAGE, VINAIGRE_BLANC_10, VINAIGRE_BLANC_6 |
| `llx_dist_dict_broyage` | Types broyage | — |
| `llx_dist_dict_futs` | Fûts | — |
| `llx_dist_dict_alambic` | Alambics | — |
| `llx_dist_dict_typeregistre` | Types de registres | PRODUCTION, DISTILLATION, STOCK, MOUVEMENTS, PERTES, COMPTE_MATIERES |

---

## Permissions

| Permission | ID | Description |
|-----------|----|-------------|
| read | 25000401 | Lecture globale |
| write | 25000402 | Écriture globale |
| delete | 25000403 | Suppression globale |
| distillation.read | 25000411 | Lecture distillations |
| distillation.write | 25000412 | Écriture distillations |
| production.read | 25000421 | Lecture productions |
| production.write | 25000422 | Écriture productions |
| registre.read | 25000431 | Lecture registres |
| registre.write | 25000432 | Écriture registres |
| admin | 25000499 | Administration module |

> IDs au pattern `$this->numero . sprintf("%02d", ...)` — module ID officiel **250004** (plage BARNECKI 250000-250019).

---

## Menus

```
Distillerie (top menu)
├── Dashboard
├── Productions
│   ├── Liste des productions
│   └── Nouvelle production
├── Distillations
│   ├── Liste des distillations
│   └── Nouvelle distillation
├── Registres
│   ├── Liste des registres
│   └── Nouveau registre
├── Déclarations mensuelles
└── Configuration
    ├── Paramètres (FormSetup : entrepôt fiscal, unités, taux)
    ├── Import productions
    ├── Autodiagnostic (contrôles runtime)
    └── À propos
```

---

## Import Excel (Traçabilité)

Format attendu : fichier `.xlsx` avec :
- **Ligne 2** col A = Étape, col B = Champ, col C+ = Noms des productions
- Noms productions : format `TESTxxx` ou `BATCHxxxxxxxx`
- Librairie embarquée : `includes/SimpleXLSX.php` (sans dépendance Composer/PhpSpreadsheet)
- Étapes créées avec rang automatique (RECOLTE=10, NETTOYAGE=20, BROYAGE=30…)
- Fruits inconnus créés automatiquement dans `dist_dict_fruits`
- Toutes les données d'étapes stockées en JSON dans `json_data`

---

## Librairies incluses

| Librairie | Usage | Chemin |
|-----------|-------|--------|
| SimpleXLSX | Lecture fichiers .xlsx (ZipArchive natif PHP) | `includes/SimpleXLSX.php` |
| Chart.js v3.9.1 | Graphiques dashboards | CDN |

---

## Versions

Voir [ChangeLog.md](ChangeLog.md) pour l'historique complet.

**Version actuelle : 0.9.0**
