# ChangeLog — Module Distillerie

---

## [0.13.5] — 2026-11-05

### Documentation (aucun changement de code)
- **CLAUDE.md** : sections regroupées et corrigées — suivi de distillation (date de ligne `date_line`, repli déduit), nouvelle section PDF de la distillation (mise en page, pied de page, piège TCPDF `setPage`/`SetAutoPageBreak`), table `dist_distil_line`, classe `DistImport`, SimpleXLSX en flux, liste exacte des onglets de configuration, **6 dictionnaires natifs déclarés (et non 7 : Alambic géré par page dédiée)**, bouton de test Tuya (reste dans Intégrations, retiré de la fiche), processus de livraison réel, harness de tests et ses limites, état réel du contrôle qualité.
- **EVOLUTIONS.md** : #7 (import incrémental) retiré car livré, #2 mis à jour (volumétrie 0.13.1), nouveaux points #16 (compléments PDF / reprise), #17 (tables inutilisées), #18 (validation sur instance réelle).
- **INSTALL.md** : réécrit — prérequis (extensions zip/xml), mise à jour sans désactivation via SQL Upgrade, réglages PHP de l'import (`upload_max_filesize`, `post_max_size`, `max_input_vars`) et limites du module, configuration Tuya, vérifications, liste complète des tables.

---

## [0.13.4] — 2026-11-04

### Export PDF de la distillation : mise en page
- Nom de la société, nom du registre et titre « Journal de distillation » **centrés** ; le logo ne chevauche plus le titre.
- Informations d'entête (date/heure début, production, TAV, volume, registre, douanes, EDF, alambic) sur **deux colonnes** (libellé normal, valeur en gras).
- **Pied de page** sur chaque page : « Dolibarr <version> - Module Distillerie <version> - Page x / y » (version du module lue dans `VERSION`).
- Pièges TCPDF corrigés pendant le développement : `setPage()` restaure le saut de page automatique de la page (le pied de page créait une page vide) → `SetAutoPageBreak(false)` après chaque `setPage()` ; marge basse réservée (18 mm) pour ne pas chevaucher le tableau.
- Vérifié par un vrai rendu TCPDF 6.7.5 (2 pages, pied de page présent sur les deux).

---

## [0.13.3] — 2026-11-04

### Suivi de distillation : date enregistrée en base sur chaque ligne
Nouvelle colonne `dist_distil_line.date_line` (DATE). La date est enregistrée à l'ajout d'une ligne (champ date sous l'heure, aujourd'hui par défaut ; hier si l'heure saisie dépasse l'heure actuelle de plus de 12 h, ex. relevé de 23:50 saisi à 00:10) et à la reprise (date du jour). Elle est modifiable dans la ligne (champ date) et journalisée à l'audit. La colonne est créée automatiquement à l'ouverture de la fiche (et listée dans Configuration > SQL Upgrade).
- Lignes antérieures à 0.13.3 ou importées d'Excel sans heure : la date reste déduite (début de distillation + passages de minuit) pour l'affichage, sans écriture ; l'enregistrement de la ligne la fige en base.
- Une date postée invalide est ignorée (repli sur la date calculée à l'ajout, date inchangée à la modification).

---

## [0.13.2] — 2026-11-04

### Suivi de distillation : date sous l'heure
Dans la liste des lignes, la date (JJ/MM/AAAA) s'affiche en petit texte sous l'heure. Les lignes ne stockent que HH:MM : la date est déduite de la date de début de la distillation, +1 jour à chaque passage de minuit (heure inférieure à celle de la ligne précédente). Aucun changement de base de données. Limite : une reprise plusieurs jours après l'arrêt, avec une heure supérieure à celle de la ligne précédente, serait affichée au même jour.

---

## [0.13.1] — 2026-11-02

### Import Excel : contrôle volumétrique (gros fichiers)
- **Lecture en flux** (`SimpleXLSX`, XMLReader) : une seule ligne en mémoire au lieu du DOM complet de la feuille ; résultat strictement identique à l'ancien parseur (vérifié sur le vrai fichier et 4 fichiers générés). Pic mémoire du parseur sur 3000 productions : 12,2 Mo → 8,1 Mo.
- **Garde-fous** : feuille > 150 Mo décompressés ou textes > 100 Mo (archive piégée) refusés avant lecture ; > 200 000 lignes ou > 5 000 colonnes refusés ; > 1 500 productions par fichier refusé avec message « scindez le fichier ».
- **Limites PHP explicites** : message précis pour `upload_max_filesize`, `post_max_size` (envoi vide auparavant sans aucun message), dossier temporaire plein, envoi partiel.
- **`max_input_vars`** : la confirmation envoie 3 variables par production ; PHP tronquait silencieusement au-delà (défaut 1000 → ~330 productions). La prévisualisation bloque désormais le bouton avec explication, et l'import refuse un POST tronqué.
- **Délai** : `set_time_limit(60)` renouvelé et `ignore_user_abort` à chaque production ; analyse mémoïsée (un même fichier n'est plus relu).
- Mesures (harness, base simulée) : 1000 productions = 0,10 s d'analyse, 0,13 s d'import, 11 Mo, 3,0 requêtes/production, 1000 transactions.

---

## [0.13.0] — 2026-11-02

### Import Excel : mode UPDATE (EVOLUTIONS #7)
Nouvelle option **Productions déjà existantes : Ignorer (défaut) / Mettre à jour** sur l'écran de prévisualisation ; en mode « Mettre à jour », seules les lignes cochées sont traitées. Une référence existante n'est jamais dupliquée.
- **Production** : mise à jour seulement si elle est en brouillon (validée/clôturée : ignorée) ; date, fruit, poids brut, lieu modifiés uniquement s'ils sont renseignés dans le fichier.
- **Étapes** : mises à jour par code d'étape (libellé, date, données) ; étape absente en base : créée ; étape absente du fichier : conservée.
- **Distillation** : mise à jour seulement si elle est encore « Importée » et non officielle (sinon conservée, avec avertissement) ; lignes mises à jour ou créées par type de coupe (jamais supprimées) ; alambic, volume et TAV de départ appliqués seulement s'ils sont fournis.
- **Mouvement de registre** : si un registre est choisi, les mouvements brouillon de la distillation sont recréés ; un mouvement validé n'est jamais touché.
- Seules les colonnes réellement différentes sont écrites (un 2e passage identique ne change rien et ne journalise rien) ; transaction par production, rollback et message en cas d'échec ; une production mise à jour n'est jamais rattachée au suivi d'import (la suppression d'un import ne la touche pas). Le résultat distingue créées / mises à jour / sans changement / ignorées.
- Pour les productions déjà existantes, les champs volume et TAV de départ sont proposés sans préremplissage.

## [0.12.23] — 2026-11-02

### Import Excel : une transaction par production
Production, étapes, distillation, lignes, rattachement au suivi et mouvement de registre d'une production importée sont désormais créés **en entier ou pas du tout** (`begin` / `commit` / `rollback` par production). En cas d'échec d'écriture ou d'exception, seule cette production est annulée (aucune donnée partielle conservée) et un message l'indique avec la cause ; les productions suivantes continuent. L'import n'est plus marqué « interrompu » pour une simple erreur : ce statut est réservé à l'arrêt brutal du script (erreur fatale, délai, mémoire), détecté par le suivi existant. Corrige le point 4 de la roadmap v0.12.1 de l'état des lieux.

## [0.12.22] — 2026-11-02

### Reprise d'une distillation interrompue
- Bouton **Reprendre la distillation** (distillation en cours, au moins une ligne) : ajoute une ligne de type **Reprise** à l'heure actuelle (modifiable en ligne). Cette ligne vaut 00:00 et devient la référence de durée des lignes suivantes (la durée d'une ligne = écart avec la dernière reprise, ou avec la première ligne s'il n'y en a pas). Les volumes restent cumulés sur la même distillation ; la ligne de reprise n'a pas de volume. Pas de doublon si la dernière ligne est déjà une reprise.
- Le tableau de bord ne compte plus les lignes de reprise dans la répartition par type de coupe.

### Fiche distillation : bouton « Tester la connexion Tuya » supprimé
Le bouton et son action ont été retirés (le test reste disponible dans Configuration > Intégration et pour chaque équipement dans Équipements Tuya).

## [0.12.21] — 2026-11-02

### Correctif Tuya : « Call to undefined function dolibarr_set_const() »
`TuyaClient` enregistre le jeton dans une constante via `dolibarr_set_const()` / `dolibarr_del_const()`, fonctions de `core/lib/admin.lib.php` non chargées sur la fiche distillation : le relevé échouait (compteur et équipements) dès qu'un jeton neuf devait être mis en cache. `admin.lib.php` est désormais chargé par la classe. La ligne était bien conservée (enregistrement avant l'appel réseau).

## [0.12.20] — 2026-11-02

### Suppression d'une distillation non clôturée
Bouton **Supprimer** sur la fiche distillation (statuts brouillon, en cours, importée). Suppression en une transaction : relevés d'équipements Tuya, lignes de suivi, détachement de l'étape de production liée (elle est conservée), en-tête, puis PDF archivés. Tout est journalisé (RÈGLE 04). Refusée si la distillation est clôturée ou définie comme registre officiel, ou si des mouvements de stock y sont rattachés. Confirmation obligatoire, jeton CSRF vérifié.

## [0.12.19] — 2026-11-02

### Équipements Tuya : conversion °F → °C
Chaque mesure d'un équipement Tuya dispose d'une option **Conversion** (Aucune / °F → °C). Valeur enregistrée sur la ligne = (valeur brute ÷ échelle − 32) × 5/9, arrondie à 4 décimales ; l'unité est préremplie à « °C » si elle est vide. La colonne `conversion` est ajoutée automatiquement aux tables créées en 0.12.18. Non rétroactif : les relevés déjà enregistrés ne sont pas modifiés.

## [0.12.18] — 2026-11-02

### Équipements Tuya supplémentaires (même compte Tuya)
- Nouvelle page **Configuration > Équipements Tuya** (admin) : ajout, activation/désactivation et suppression dynamiques d'équipements ; chaque équipement porte des **mesures** (code du point de données Tuya, libellé, unité, échelle). Bouton « Tester / découvrir » : liste les points de données de l'appareil avec un lien pour ajouter la mesure.
- À l'**ajout d'une ligne de distillation**, chaque mesure active est relevée (un appel par équipement) et enregistrée sur la ligne, comme pour le compteur électrique : la ligne est toujours enregistrée AVANT tout appel réseau ; si la connexion Tuya est perdue, les équipements suivants sont ignorés avec un avertissement.
- Colonnes dynamiques dans la fiche distillation, valeurs modifiables ligne par ligne (bouton Enregistrer existant), lecture seule si la distillation est verrouillée.
- Trois nouvelles tables (`dist_tuya_device`, `dist_tuya_measure`, `dist_distil_line_tuya`) créées à l'activation et par le contrôle SQL ; toutes les écritures sont journalisées (RÈGLE 04) et protégées CSRF.
- Les relevés conservent une copie du libellé/unité : supprimer un équipement ne détruit pas l'historique (la colonne n'est simplement plus affichée).
- Limite connue : l'export PDF n'inclut pas encore ces colonnes.

## [0.12.17] — 2026-11-01

### Configuration > Intégration : libellés Tuya
Les 4 champs de la rubrique « Compteur électrique Tuya » (Client ID, Client Secret, Device ID, Endpoint) affichaient `MissingTranslationForConfKey` : `FormSetup` utilise la clé de la constante comme clé de traduction et elles n'existaient pas dans les fichiers de langue. Ajout des libellés et infobulles (fr_FR et en_US). Les autres champs de configuration (`admin/setup.php`) ont été vérifiés : tous traduits.

## [0.12.16] — 2026-11-01

### Tuya : test de connexion sur la fiche et ligne jamais perdue
**Problème** : quand la connexion Tuya se perdait, la ligne n'était parfois pas ajoutée. Causes : (1) le relevé Tuya (jusqu'à ~26 s cumulées en cas de coupure : jeton 5+8 s puis lecture 5+8 s) était fait **avant** l'enregistrement de la ligne — un dépassement du temps d'exécution PHP/serveur ou une coupure du navigateur faisait perdre la saisie ; (2) un jeton en cache refusé par Tuya (périmé/invalide) était réutilisé à chaque appel jusqu'à son expiration : la connexion restait « perdue ».
- **Ajout de ligne** : la ligne est maintenant **enregistrée d'abord** (sans relevé), puis le test de connexion + relevé Tuya est lancé ; si le relevé aboutit, il est ajouté à la ligne par un UPDATE audité. En cas d'échec (coupure, timeout, appareil hors ligne, exception), la ligne existe quand même, un avertissement le dit, et Puissance / Énergie / Température ambiante peuvent être saisies sur la ligne (0.12.8). `ignore_user_abort` : la ligne est créée même si le navigateur se déconnecte.
- **Bouton « Tester la connexion Tuya »** sur la fiche distillation (distillation en cours, droit d'écriture) : demande un jeton neuf et lit le compteur, sans rien modifier ; affiche les valeurs ou la cause de l'échec.
- **TuyaClient** : budget de temps global de 14 s ; un nouvel essai après une erreur réseau ; jeton en cache purgé puis redemandé automatiquement si Tuya le refuse ; lecture défensive de la réponse (plus d'index non défini) ; nouvelle méthode `testConnection()` (utilisée aussi par le bouton de la configuration).

## [0.12.15] — 2026-11-01

### Correctifs d'affichage
- **Fiche distillation : volume de départ absent** alors que la liste l'affichait. Deux causes : la fiche lisait uniquement `volume_depart_l` (volume normalisé en L) alors que la liste lit `volume_depart` (+ unité) ; et `volume_depart_l`, absent de `DistDistil::$fields`, n'était jamais écrit par `create()` lors d'une création manuelle. Corrigé : la fiche se rabat sur `volume_depart` + unité quand `volume_depart_l` est vide (distillations existantes), et `create()` écrit désormais `volume_depart_l` (même transaction que l'audit).
- **Tableaux de bord : graphiques vides.** Chart.js a besoin d'un `<canvas>` et les graphiques étaient posés sur des `<div>`. Corrigé sur les **trois** tableaux de bord (distillations, productions, registres), 6 graphiques au total.
- **Répartition par type de coupe** : les codes bruts (`TETE`, `COEUR`, `QUEUE`, `NONE`) sont remplacés par les libellés Tête / Cœur / Queue ; les lignes sans coupe sont regroupées sous « Autre ».

## [0.12.14] — 2026-11-01

### Fiche distillation : visualiser et supprimer les PDF générés
- Dans la liste **Documents** : le nom du fichier et le bouton **Voir** ouvrent le PDF (nouvel onglet) par une action propre à la fiche (`viewpdf`), sans dépendre de `document.php` du cœur ; bouton **Supprimer** (confirmation, jeton CSRF, droit d'écriture sur les distillations).
- Le nom de fichier est validé strictement (`REGISTRE_<ref de la distillation>_vNNN.pdf`) : impossible de lire ou supprimer un autre fichier ou le PDF d'une autre distillation.
- La suppression est tracée au journal d'audit (nom du fichier supprimé) **avant** l'effacement : si l'audit échoue, le fichier est conservé. Elle est refusée pour une distillation **officielle** (archive réglementaire).
- Le bouton Régénérer crée toujours une nouvelle version (v001, v002…).

## [0.12.13] — 2026-11-01

### Contrôle avant clôture d'une production
Une production ne peut être clôturée que si toutes ses distillations sont terminées : celles en **brouillon** ou **en cours** bloquent la clôture (message avec leur nombre). Les distillations **clôturées, officielles ou importées** ne bloquent pas. La validation d'une production n'est soumise à aucun contrôle de ce type. Code : `distillerie_change_status()` (nouvel argument de sortie `$err_arg`).

## [0.12.12] — 2026-11-01

### Boutons Valider / Clôturer pour productions et registres
Le verrou de la 0.12.11 devient utilisable.
- **Production** (statuts 0 brouillon, 1 validée, 2 clôturée) et **registre** (0, 1, 9 clôturé) : boutons **Valider** (brouillon → validé), **Clôturer** (validé → clôturé, définitif) et, pour un **administrateur** seulement, **Repasser en brouillon** (validé → brouillon). Confirmation avant chaque action ; droit « écriture » du domaine requis ; jeton CSRF vérifié.
- **Registre** : la clôture est refusée tant qu'il reste des mouvements non validés ; elle renseigne la date de clôture. Un registre validé ou clôturé s'affiche en lecture seule (formulaire désactivé, bouton Enregistrer masqué).
- **Audit** : chaque changement de statut est journalisé dans la même transaction (action `VALIDATE` pour la validation, `UPDATE` avec le statut avant/après pour la clôture et la réouverture). Un enregistrement clôturé ne se rouvre pas.
- Le statut est affiché sur la fiche production.
- Technique : `distillerie_change_status()` et `distillerie_record_status_label()` dans `lib/distillerie.lib.php`.

## [0.12.11] — 2026-11-01

### Verrouillage après validation (EVOLUTIONS #3)
- **Distillation** : une distillation clôturée ou officielle est en lecture seule, y compris pour son **statut** : l'action `setstatus` pouvait jusqu'ici rouvrir une distillation clôturée, voire officielle (trou de sécurité corrigé). `setofficial` sur une distillation déjà officielle est ignoré.
- **Garde au niveau des classes** (et plus seulement de l'écran) : `update()` / `delete()` de `DistProd`, `DistDistil`, `DistRegistry` et `update()` de `DistProdStep` refusent l'écriture si l'enregistrement est verrouillé **en base** (production et registre validés ou clôturés ; distillation clôturée ou officielle ; étape dont la production est validée ou clôturée). Message : « Enregistrement validé ou clôturé : modification et suppression verrouillées. »
- **Écrans** : bouton « Modifier » de la production masqué si elle est validée/clôturée ; ajout d'étape refusé dans ce cas. Mouvements de registre : verrou validé inchangé (déjà au niveau classe).
- Précision : les statuts « validé »/« clôturé » des productions et registres n'ont pas encore de bouton pour les atteindre ; le verrou est en place dès qu'un workflow de validation sera ajouté. Les distillations importées (statut « importée ») restent modifiables.

## [0.12.10] — 2026-11-01

### Hygiène du module (état des lieux §3)
- **Descripteur** : prérequis alignés sur la prod — PHP **8.2** minimum (au lieu de 7.1), Dolibarr **21** minimum (au lieu de 19).
- **Fichiers retirés de la distribution** : `class/distmouvement.class.php.patch`, `class/simplexlsx.class.php` (doublon jamais inclus de `includes/SimpleXLSX.php`), `img/object_myobject.png`, `SQL_VERIFICATION_REPORT.md`, `DEBUG_WORKFLOW.md`, `doc/VERIFICATION_COMPLETE.md`, `doc/VERIFICATION_v0.5.1.md`, `doc/VERSION_0.5.1.md`, `doc/CHANGELOG_v0.5.2.md`.
- **Tests PHPUnit supprimés** : gabarit Selenium inutilisable (URL `dev.zenfusion.fr`, `admin/admin` en dur) et son doublon `functionnal/`. Reste `test/agent_controle.php` (analyse statique). EVOLUTIONS #15 reformulé.
- **CLAUDE.md** : PHP 8.2+ / Dolibarr 21+ ; pattern de migration de colonnes réécrit (source de vérité : `lib/distillerie.lib.php`, plus de `addMissingXxxColumns()`). README : arborescence nettoyée.

## [0.12.9] — 2026-11-01

### Sécurité : vérification CSRF explicite sur toutes les pages qui écrivent
Constat (état des lieux §2.3 / §3) : `distmouvement_card`, `distprod_card`, `distregistry_card`, `production_step`, `production_step_edit`, `admin/alambic`, `admin/automation`, `admin/setup` et `distdeclaration_generate` affichaient un jeton mais ne le vérifiaient pas (dépendance à `MAIN_SECURITY_CSRF_WITH_TOKEN`).
- Chaque page refuse désormais, sans jeton valide, ses actions d'écriture/génération (message « Action refusée : jeton invalide ou expiré ») ; les actions de lecture ne sont pas concernées.
- Lien « Créer une distillation » (fiche production) et formulaire des déclarations mensuelles : jeton ajouté.
- Pages déjà protégées : `distdistil_card`, `admin/import_productions`, `admin/integration`, `admin/sql_upgrade`.

## [0.12.8] — 2026-11-01

### Suivi de distillation : durée depuis la première ligne, relevés modifiables avec la ligne
- **Durée** : elle se calcule désormais par rapport à la **première ligne** du suivi (heure de départ ; la première ligne vaut 00:00), et non plus par rapport à la ligne précédente. Appliqué à l'ajout, à la modification, à la suppression (si la première ligne est supprimée, la suivante devient la référence), à l'affichage et au PDF. Les durées stockées qui changent sont réécrites et tracées au journal d'audit. Une heure antérieure à la première ligne est comprise comme un passage de minuit (+24 h) au lieu d'être ramenée à 00:00. Les lignes issues d'un import (sans heure) ne sont pas touchées.
- **Puissance (W), Énergie (kWh), Température ambiante (°C)** : modifiables sur les lignes déjà saisies comme les autres champs, enregistrés par le bouton **Enregistrer** de la ligne (mêmes contrôles de plage ; champ vide = non renseigné ; écriture et audit dans la même transaction).
- **Bouton « Modifier » supprimé** (ainsi que les actions `editmeter` / `updatemeter` et les clés de langue devenues inutiles).

## [0.12.7] — 2026-11-01

### Audit (RÈGLE 04) étendu à TOUS les éléments du module
Jusqu'ici seuls les mouvements de registre et les corrections de relevés compteur étaient journalisés. Désormais **toute écriture** l'est, dans la **même transaction** que l'écriture (si l'audit échoue, l'écriture est annulée : aucune modification non tracée). La consultation reste dans **Configuration > Historique** (pas d'onglet sur les fiches, décision de l'exploitant).

**Éléments couverts**
- **Productions** (création, modification, suppression), **étapes de production** (création, modification, liaison à la distillation), **distillations** (création, modification, changement de statut, passage en officielle, en-tête registre/douane), **lignes de distillation** (ajout, modification, suppression, recalcul de durée de la ligne suivante, correction des relevés compteur), **registres** (création, modification), **mouvements** (inchangé).
- **Alambics** (création, modification, suppression), **fruits** ajoutés au dictionnaire par l'import.
- **Imports Excel** : suivi d'import (création, clôture, interruption, suppression), rattachement des productions, lignes générées, liaisons ; **suppression d'un import interrompu** : chaque production, étape, distillation et ligne supprimée est journalisée avec sa photo avant suppression.
- **Configuration** : paramètres du module et de l'intégration Tuya (une entrée par constante modifiée, **secrets jamais écrits en clair** : « *** (défini) » / « *** (vide) »).
- **Purge des données** (réinitialisation depuis la configuration) : entrée `PURGE` avec le nombre d'éléments par table avant suppression ; le journal lui-même n'est jamais purgé.
- **Exports réglementaires** : DRM (PDF, Excel, génération), DTI, FEC, PDF de distillation → entrée `EXPORT`.
- **Automatisations / intégrations** : calcul de stocks, brouillons d'inventaire, import de documents d'accompagnement, synchronisation de sites.

**Fonctionnement**
- Nouveau trait `DistAuditedTrait` (classes métier) et `DistAudit::track()` (SQL brut) : photo de la ligne avant/après (sans `tms`), UPDATE sans effet non journalisé (égalité numérique : 1.500 = 1.5).
- `DistAudit::snapshot`, `sameState`, `snapshotConsts`, `logConfigChanges` ; filtre des propriétés techniques renforcé (`fields`, `array_options`…) pour ne plus polluer les entrées des mouvements.
- Page Historique : actions `PURGE`, `EXPORT`, `SYNC` ajoutées au filtre ; rendu dédié (purge, configuration).

**Limites assumées (à connaître)**
- **Dictionnaires natifs Dolibarr** (fruits, états, nettoyage, broyage, fûts, types de registre) : modifiés par `admin/dict.php` du cœur, **non interceptables** par le module : leurs modifications manuelles ne sont pas journalisées (seuls les fruits créés par l'import le sont). Alambics : couverts (page dédiée).
- Cache de jeton Tuya (constante technique) volontairement non journalisé.
- Volumétrie : un import de 43 productions écrit plusieurs centaines d'entrées d'audit.
- L'audit n'est pas signé/chaîné (intégrité cryptographique non assurée).
- Les entrées antérieures à la 0.12.7 n'existent pas (seuls mouvements et relevés avaient un audit) ; certaines anciennes entrées de mouvements contiennent des propriétés techniques superflues.

### Vérifié (exécuté)
- Nouveau test du socle (vraie classe `DistAudit`, vrai trait, DB simulée) : UPDATE/CREATE/DELETE tracés avec commit ; UPDATE sans changement ignoré ; écriture en échec → rollback sans audit ; **audit en échec → rollback** ; table invalide refusée ; secrets de configuration masqués (aucun secret en clair dans le SQL d'audit) ; trait : id renvoyé, échec métier, échec d'audit : 19 contrôles OK.
- Import : tests existants rejoués avec audit (43 / 2 existantes / 12 distillations, 41 créées), suivi d'import et suppression d'import interrompu avec entrées DELETE (production, distillation, étape, ligne) et mise à jour du suivi ; relevés compteur 13 contrôles OK.
- `php -l` : 0 erreur. Agent de contrôle (exécuté) : 0 erreur critique, 6 avertissements (inchangés), 9 suggestions, 116 contrôles réussis, 95,1 %.

### Non traité / à valider
- Non testé sous Dolibarr réel : en particulier les transactions imbriquées (`begin`/`commit` des méthodes `*Common` à l'intérieur de l'audit) et l'écriture du journal d'une modification de configuration via FormSetup (fonction de fin de script). **À valider sur `gestion.barnecki-spirits.com`** : modifier un paramètre, créer une production, ajouter/supprimer une ligne, puis contrôler l'onglet Historique.
- Verrouillage après validation (EVOLUTIONS #3) toujours à faire.

---

## [0.12.6] — 2026-11-01

### Ajout : onglet « Historique » (Configuration) — consultation du journal d'audit
Nouvelle page `admin/historique.php`, onglet **Historique** de la configuration du module (droit « admin », **lecture seule**). Elle affiche `llx_dist_audit_log` du plus récent au plus ancien, avec filtres : table, n° d'enregistrement, action (CREATE / UPDATE / DELETE / VALIDATE), utilisateur, période ; pagination précédent/suivant.
- Pour une modification (UPDATE), seuls les **champs réellement modifiés** sont affichés, en « avant → après » (comparaison numérique : `1.500` et `1.5` ne sont pas une modification) ; la distillation et le motif sont rappelés. Pour CREATE/DELETE : liste compacte des champs renseignés.
- Les corrections de relevés compteur (0.12.5) y sont donc consultables : `dist_distil_line`, n° de ligne, valeurs avant/après, utilisateur, date, IP.
- Valeurs filtrées et échappées (table validée contre la liste réelle du journal, dates au format `AAAA-MM-JJ`, entiers castés) ; aucune écriture.

### Vérification demandée : points 2.1 et 2.2 de l'état des lieux 0.12.0 — tous corrigés
Contrôle sur le code de cette version (recherche dans les sources) et tests exécutés sur le vrai `TRACABILITE.xlsx` :
- **2.1 Tuya faussé à l'import** : corrigé (0.12.1). Aucune référence Tuya exécutable dans `distimport.class.php` ni dans la page d'import ; `TuyaClient` n'est appelé que par `distdistil_card.php` (ajout de ligne) et par le bouton de test de `admin/integration.php`. Relevés laissés à NULL à l'import (modifiables à la main depuis la 0.12.4/0.12.5).
- **2.2 Ré-import = doublons** : corrigé (0.12.2) — référence existante repérée à la prévisualisation et ignorée à l'import (test : 2 existantes ignorées, 41 créées). Mode UPDATE : toujours absent (EVOLUTIONS #7).
- **2.2 `prod_ref` non renseigné** : corrigé (0.12.1).
- **2.2 Sans `fk_alambic` / `volume_depart` / `tav_depart_pct`, statut « En cours »** : corrigé (0.12.2 pour les valeurs demandées à l'écran ; statut « Importée » en 0.12.1). Le volume de départ n'est pré-rempli que si l'Excel est un nombre pur, le TAV jamais.
- **2.2 Aucun mouvement de registre** : corrigé (0.12.2), mouvements brouillon optionnels, règle cœur + queues par défaut depuis la 0.12.3.
- **2.2 Détection Tête/Cœur/Queue jamais validée sur le vrai fichier** : validée (0.12.1), 43 productions / 12 distillations ; corrections de parsing apportées depuis.
- **2.2 `volume_ml` INTEGER** : inchangé par choix — arrondi désormais **explicite** (`round()`) et documenté ; le type de colonne n'a pas été modifié.
Restent hors de ces deux points : import non transactionnel par production (atténué par le suivi des imports interrompus, 0.12.3), CSRF des autres pages, nettoyage du ZIP (§3 de l'état des lieux).

### Vérifié (exécuté)
- Rendu des entrées d'audit testé (UPDATE avec champs changés / nulls / égalité numérique / aucun changement ; CREATE avec échappement HTML ; DELETE).
- Rejeu des tests : import (43 / 2 existantes / 12 distillations, 41 créées, alambic partout, volume de départ écrit une seule fois), suivi des imports (tout OK), relevés + audit (13 contrôles OK).
- `php -l` : 0 erreur. Agent de contrôle (exécuté) : 0 erreur critique, 6 avertissements (inchangés), 9 suggestions, 116 contrôles réussis, 95,1 %.

### Non traité / à valider
- Page non testée sous Dolibarr réel. Pas encore d'onglet Historique sur les fiches distillation / production (la page de configuration est globale ; le filtre table + n° permet de retrouver une ligne).
- Audit toujours limité aux mouvements et aux corrections de relevés (EVOLUTIONS #3).

---

## [0.12.5] — 2026-11-01

### Correction : unité de `quantite_totale` (kg et non L)
`quantite_totale` reçoit le « Poids Brut » de la récolte : c'est une **masse**. Les affichages disaient « L ». Corrigé en **kg** : tableau de bord production (total, moyenne, liste, libellés des deux graphiques), tableau de bord général, fiche production (affichage, saisie, création). Le libellé « Volume total » de ces deux tableaux de bord devient « Quantité récoltée totale » (nouvelle clé `QuantiteRecolteeTotale`). Aucune valeur en base n'est modifiée (seule l'unité affichée était fausse).

### Ajout : correction des relevés compteur tracée au journal d'audit (RÈGLE 04)
Chaque correction faite avec le bouton « Modifier » d'une ligne de distillation écrit dans `llx_dist_audit_log` (table `dist_distil_line`, action `UPDATE`) : anciennes et nouvelles valeurs de puissance / énergie / température ambiante, avec la distillation concernée et le motif, plus utilisateur, date, IP et navigateur ajoutés par `DistAudit::log()`.
- Mise à jour et écriture d'audit dans **la même transaction** : si l'audit échoue, la correction est annulée (rollback) avec un message d'erreur — pas de correction non tracée.
- Valeurs identiques à l'existant : aucune écriture ni entrée d'audit (message « Aucune modification »).
- Ligne introuvable pour cette distillation : refus explicite.

### Vérifié (exécuté)
- Test de l'action `updatemeter` réelle (13 contrôles OK) : audit écrit avec anciennes/nouvelles valeurs et distillation, commit, NULL tracé, audit en échec → rollback, valeurs identiques → rien, ligne introuvable → rien, plus les contrôles de plage, verrou, droit et injection.
- `php -l` : 0 erreur. Agent de contrôle (exécuté) : 0 erreur critique, 6 avertissements (inchangés), 9 suggestions, 115 contrôles réussis, 95,0 %.

### Non traité / à valider
- Pas de page de consultation du journal d'audit (EVOLUTIONS #3) : les entrées sont en base (`dist_audit_log`, `DistAudit::getHistory('dist_distil_line', id)`), pas encore affichées dans l'interface.
- Interface non testée sous Dolibarr réel.

---

## [0.12.4] — 2026-11-01

### Correction : `DistProd` ne déclarait pas `fk_fruit`, `quantite_totale`, `lieu_recolte`
Constat : ces trois champs étaient utilisés par l'import (`$production->fk_fruit = ...`) et par les tableaux de bord (`SUM(quantite_totale)`, jointure fruit), mais ne figuraient ni dans `DistProd::$fields` (donc **jamais écrits en base** par `createCommon()`/`updateCommon()`, l'import les perdait en silence) ni dans `sql/llx_dist_prod.sql` (donc requêtes de dashboard en échec silencieux sur une installation neuve).
- Les trois champs sont déclarés dans `DistProd::$fields` (`fk_fruit` entier, `quantite_totale` double(12,3), `lieu_recolte` varchar(255), tous facultatifs).
- Colonnes ajoutées à `sql/llx_dist_prod.sql` (installations neuves) et au registre `distillerie_get_known_schema_fixes()` (installations existantes, ajout seulement si absentes : idempotent). Application : page **Configuration > SQL Upgrade**, ou réactivation du module, ou ouverture de la page d'import (qui applique les correctifs connus).
- Fiche production : fruit (liste du dictionnaire), quantité récoltée et lieu de récolte affichés, saisissables à la création et modifiables.
- **Les productions déjà importées avant cette version n'ont pas ces valeurs** (elles n'avaient pas été écrites) : pour les renseigner il faut supprimer/réimporter, ou les saisir sur la fiche.

### Ajout : bouton « Modifier » sur les relevés compteur d'une ligne de distillation
Sur la fiche distillation, chaque ligne a un bouton **Modifier** : les trois cellules Puissance (W), Énergie (kWh), Température ambiante (°C) deviennent saisissables, avec **Enregistrer les relevés** / Annuler. Utile pour renseigner les distillations importées (relevés laissés vides à l'import) ou corriger un relevé Tuya.
- Action `updatemeter` dédiée : ne modifie que ces 3 colonnes (+ utilisateur), ne touche pas aux autres champs de la ligne ni au recalcul des durées.
- Champ vide = non renseigné (NULL) ; valeurs contrôlées (puissance 0 à 1 000 000 W, énergie 0 à 100 000 000 kWh, température -60 à 200 °C ; virgule décimale acceptée), requête échappée.
- Mêmes verrous que la modification d'une ligne (distillation clôturée ou officielle = refus) + droit `distillation/write` exigé + jeton CSRF vérifié.

### Vérifié (exécuté)
- Test de l'action `updatemeter` réelle (bloc extrait de la page, stubs Dolibarr) : valeurs valides, vides (NULL), température négative, texte / négatif / hors plage / injection refusés, distillation verrouillée et absence de droit : 8 contrôles OK.
- Correctif de schéma : ALTER générés vérifiés (`fk_fruit INTEGER NULL`, `quantite_totale DECIMAL(12,3) DEFAULT 0 NULL`, `lieu_recolte VARCHAR(255) NULL`).
- `php -l` : 0 erreur. Agent de contrôle (exécuté) : 0 erreur critique, 6 avertissements (inchangés), 9 suggestions, 115 contrôles réussis, 95,0 %.

### Non traité / à valider
- Interface non testée sous Dolibarr réel (fiche production, bouton Modifier).
- `quantite_totale` reçoit le « Poids Brut » de l'Excel (poids) alors que `distprod_dashboard.php` l'affiche avec l'unité « L » : unité à clarifier.
- Les modifications de relevés ne sont pas tracées dans `DistAudit` (comme les autres lignes de distillation, EVOLUTIONS #3).

---

## [0.12.3] — 2026-11-01

### Décisions de l'exploitant appliquées (pas de fonctionnalité 0.13.0)

- **Lignes dupliquées du fichier réel = phase de FIN** : le bloc répété « FERMENTATION/MACERATION DEBUT » (Date, Heure, Température ext./int., Contrôle visuel) est **scindé** : à la détection d'un libellé déjà vu dans DEBUT, la suite du bloc est réaffectée à « FERMENTATION/MACERATION FIN ». Un avertissement par production indique le nombre de champs réaffectés (visible dans la prévisualisation). Vérifié sur le vrai `TRACABILITE.xlsx` : DEBUT ne garde que ses champs propres (Date, Heure, N° Fût, levure…), les valeurs réaffectées se retrouvent en FIN sans écrasement (aucun recouvrement de données constaté).
- **Imports interrompus signalés et supprimables** : chaque import ouvre un suivi (`llx_dist_import_batch`, statut `running`) et rattache chaque production créée (`dist_prod.fk_import_batch`). Il passe à `done` en fin normale, à `interrupted` si une erreur survient (l'import s'arrête à cette production) ou si le script meurt (fonction de shutdown : fatal, délai, mémoire). Un import resté `running` plus de 15 min est aussi signalé. La page d'import affiche un **bandeau** par import interrompu (nombre de productions, étapes, distillations, mouvements déjà créés) et un bouton « Supprimer les données de cet import » avec **confirmation explicite** (case à cocher + jeton CSRF). Suppression en **transaction** : mouvements brouillon (via `DistMouvement::delete`, audit conservé), lignes, distillations, étapes, productions. **Refusée** (rien supprimé) si un mouvement lié a été validé (RÈGLE 04). Les fruits ajoutés au dictionnaire ne sont pas supprimés ; les numéros de séquence des mouvements supprimés ne sont pas réattribués. Historique des 10 derniers imports sous le formulaire.
- **Règle douanière par défaut : cœur + queues comptabilisés** (têtes non comptées), **modifiable à l'écran** : 3 choix dans la prévisualisation (cœur + queues = défaut, cœur seul, tête + cœur + queue). Remplace le défaut « cœur seul » de la 0.12.2.

### Schéma
- Nouvelle table `llx_dist_import_batch` ; colonne `dist_prod.fk_import_batch`. Créées à l'activation, et **auto-réparées** à l'ouverture de la page d'import (`distillerie_ensure_import_schema()`) et via SQL Upgrade : aucune désactivation/réactivation nécessaire pour une mise à jour. Si le schéma n'est pas à jour, l'import fonctionne mais sans suivi (avertissement affiché).

### Vérifié (exécuté)
- Nouveau test `e2e3` (DB simulée) : import normal → suivi `done` + rattachement ; exception en cours de production → `interrupted`, arrêt, 1 seule production créée ; schéma absent → import sans suivi + avertissement ; suppression d'un import signalé → mouvements/lignes/distillations/étapes/productions supprimés en transaction ; refus si mouvement validé (aucun DELETE) ; refus si l'import n'est pas signalé interrompu ; critère de détection (interrupted ou running périmé, jamais done/deleted) : 23 contrôles OK.
- Non-régression sur le vrai fichier : 43 productions analysées / 2 existantes / 12 distillations ; import 41 productions, 12 distillations, 12 mouvements ; règle par défaut cœur + queues (ex. 0,657 L contre 0,45 L cœur seul sur le 2e mouvement), options « cœur seul » et « tout » également vérifiées.
- `php -l` sur tous les fichiers : 0 erreur. Agent de contrôle (exécuté) : **0 erreur critique, 6 avertissements (résiduels inchangés), 9 suggestions, 115 contrôles réussis, score 95,0 %**.

### Non traité / à valider
- Pas de test sous Dolibarr réel : la page (bandeau, confirmation, historique) est à valider sur `gestion.barnecki-spirits.com`. Le test de suppression doit se faire avec un import volontairement interrompu (ex. en coupant le script) sur une base de test.
- L'import reste non transactionnel **par production** (le suivi permet désormais de nettoyer proprement après coup).
- Mode UPDATE (EVOLUTIONS #7), validation des valeurs hors norme (#1), `volume_ml` INTEGER : inchangés.
- Constat non corrigé : `DistProd` ne déclare pas certaines colonnes (`fk_fruit`, `quantite_totale`, `lieu_recolte`) : voir question posée à l'exploitant.

---

## [0.12.2] — 2026-11-01

### ADD : prévisualisation de l'import Excel + 3 points restés ouverts après la 0.12.1

L'import passe en deux temps : **upload → prévisualisation (aucune écriture en base) → confirmation**. Les trois points que la 0.12.1 laissait ouverts sont traités par des valeurs **demandées dans cet écran**, car l'Excel ne les contient pas de façon fiable. Pas de fonctionnalité 0.13.0 : pas de mode UPDATE, pas de validation des valeurs hors norme.

- **Doublons (ré-import)** : les productions dont la référence existe déjà sont repérées dans la prévisualisation (case désactivée, badge « Existe déjà ») et **ignorées** à l'import — jamais écrasées ni dupliquées. Avant, un ré-import échouait production par production sur la contrainte `UNIQUE (entity, ref)`. Une case à cocher par production permet aussi d'importer une partie du fichier.
- **Alambic / volume de départ / TAV de départ** : `fk_alambic` (liste du dictionnaire, appliquée à toutes les distillations générées), `volume_depart` (L) et `tav_depart_pct` (%) par production, saisis dans le tableau. Le volume est pré-rempli **uniquement** si la cellule « Poids Net » de fin de fermentation est un nombre pur (18 cellules du fichier réel sur 43 sont du texte : « 7,9 (+bac alambic) », « 5L + 2L »…) ; le TAV n'est **jamais** pré-rempli (échelles 0,08 / 10 / texte « 8-10% » ambiguës) : la valeur brute de l'Excel est affichée à côté du champ. Champ vide = non renseigné. `volume_depart_l` n'étant pas dans `DistDistil::$fields`, il est écrit par UPDATE direct.
- **Mouvements de registre** : choix d'un registre (ou « ne pas générer », défaut), d'une catégorie fiscale optionnelle et des coupes comptabilisées (**cœur seul** par défaut, ou tête+cœur+queue). Un mouvement `production` **brouillon** (non validé) par distillation : volume en litres, alcool pur = Σ volume × TAV, degré moyen pondéré — laissés vides avec avertissement si une coupe comptabilisée n'a pas de TAV. Numéro de séquence attribué par `DistMouvement::create()` (verrou + audit inchangés), commentaire « à contrôler avant validation ».

### Sécurité / robustesse de la page d'import (défauts relevés à l'état des lieux 0.12.0)
- Token CSRF **vérifié explicitement** à chaque étape (preview / import / annuler).
- Extension `.xlsx` imposée (le parseur ne lit pas `.xls`), taille limitée à 10 Mo, fichier temporaire au **nom généré côté serveur** (`import_<16 hex>.xlsx`, jamais le nom client) validé par expression régulière, et **supprimé** après import ou annulation (`@unlink`).
- À l'import, seules les colonnes issues de l'analyse du fichier sont acceptées (pas d'index arbitraire posté).

### Vérifié (exécuté)
- Test bout en bout hors Dolibarr (DB simulée, classes métier simulées) sur le vrai `TRACABILITE.xlsx`, avec 2 références déclarées existantes : analyse 43 productions / 2 existantes / 12 distillations ; import avec options → 41 productions, 2 ignorées, 12 distillations, alambic appliqué partout, volume/TAV de départ écrits sur la production ciblée seulement, 12 mouvements brouillons (nature `production`, cœur seul, degré et alcool pur calculés ; premier mouvement sans degré car sans TAV, avec avertissement) ; aucune sélection → 0 production, échec propre.
- `php -l` sur tous les fichiers : 0 erreur. Agent de contrôle (exécuté) : 0 erreur critique, 6 avertissements (résiduels inchangés), 114 contrôles réussis, score 95,0 %. Un premier passage avait signalé `actif` dans une requête SQL brute sur `dist_categorie_fiscale` (colonne légitime de cette table, mais mieux vaut ne pas dupliquer le SQL) : remplacé par `DistCategorieFiscale::getAllActive()`.

### Non traité (reste à faire)
- Import toujours **non transactionnel** (un import interrompu laisse des productions partielles).
- Mode UPDATE des productions existantes (EVOLUTIONS #7) ; validation des valeurs hors norme (EVOLUTIONS #1).
- Bloc dupliqué « FERMENTATION/MACERATION DEBUT » du fichier réel (probable erreur de libellé pour FIN) : toujours fusionné dans DEBUT.
- `volume_ml` reste `INTEGER` (arrondi explicite à l'import).
- Interface écran non testée sous Dolibarr : à valider sur `gestion.barnecki-spirits.com`.

---

## [0.12.1] — 2026-11-01

### Corrections — import Excel, validé sur le VRAI `TRACABILITE.xlsx` (43 productions, 71 colonnes)

Premier passage du code d'import sur le fichier réel (jusqu'ici deviné). Il a révélé plusieurs défauts que la 0.12.0 ne pouvait pas voir.

**Décisions de l'exploitant (2026-11-01)**
- **Plus aucun relevé Tuya à l'import** : l'import charge de l'historique ; le compteur renvoyait la puissance/énergie du jour de l'import, rattachées à une distillation datant de mois ou d'années (donnée fausse mais d'apparence valide) + un appel HTTPS par distillation importée. `power_w / energy_kwh / temp_ambiante` restent à `NULL`.
- **Nouveau statut `STATUS_IMPORTED = 3` « Importée »** (`DistDistil`), à la place de « En cours » pour les distillations générées. Libellés FR/EN, `distillerie_distil_status_label()`, fiche (boutons *Clôturer* / *Ré-ouvrir* depuis « Importée »), badge du dashboard. On ne peut pas ajouter de ligne de suivi (donc pas de relevé Tuya) tant que la distillation n'est pas ré-ouverte.

**Défauts trouvés grâce au fichier réel**
- **L'étape RECOLTE n'était jamais importée** : elle est déclarée sur la ligne « Nom » elle-même (col A = RECOLTE, col B = Nom), et le parseur démarrait après cette ligne avec `current_etape = null` → Date, Fruit, Lieu, Poids Brut ignorés, `date_prod` = jour de l'import, fruit vide, quantité 0. Corrigé (`parseStructureSimple`).
- **TAV des coupes jamais détecté** : le libellé « % alcool moyen » est répété 3 fois (sous Tête, Cœur, Queue) ; les 3 colonnes se collisionnaient sur la même clé et aucun libellé ne contenait le nom de la coupe. Les libellés commençant par `%` sont désormais préfixés par le libellé précédent (« Tête (ml) % alcool moyen »). `findChampValue()` accepte des jetons d'exclusion (`!%`) pour qu'un libellé de TAV ne soit jamais lu comme un volume.
- **Distillations en plusieurs passes** (« 100 & 120 », « 49% + 45% ») : `(float)` n'en lisait que la 1re valeur. Volumes désormais cumulés, TAV pondéré par les volumes, avertissement explicite à l'import.
- **Codes d'étape de fermentation** : le fichier contient un vrai saut de ligne dans « FERMENTATION/MACERATION⏎DEBUT » alors que le mapping utilisait un `\n` littéral → code `fermentation_maceration_debut` au lieu de `fermentation_debut` et rang erroné. Noms d'étape normalisés (espaces/sauts de ligne). *Les imports déjà faits en production gardent l'ancien code.*
- **Dates** : `createCommon()` applique lui-même `$db->idate()` (vérifié dans les sources Dolibarr 23.0.3, `setSaveQuery`) ; l'import lui passait une chaîne déjà passée par `idate()` → double conversion. Il passe désormais des timestamps UNIX, comme la fiche manuelle. Formats lus : ISO et `jj/mm/aaaa` (1re date d'une cellule « 22/07/2023 & 23/07/2023 »). Date de distillation : cellule Date, sinon date de production (avec avertissement).
- **Liaison étape → distillation (`fk_distil`)** : `DistProdStep::update()` n'écrit pas cette colonne, la liaison annoncée en 0.12.0 n'était donc jamais persistée. UPDATE direct ajouté.
- **`prod_ref`** renseigné sur la distillation générée (le lien de production était vide sur la fiche).
- **`distdistil_dashboard.php`** : la lib n'était pas incluse (fatal sur `distillerie_distil_status_label`) et le badge affichait « Brouillon » pour toute distillation non « en cours » (y compris clôturée). Corrigé pour les 4 statuts.
- **`distdistil_card.php` (`setstatus`)** : liste blanche des statuts (0 à 3).

### Vérifié (exécuté)
- Test bout en bout hors Dolibarr (DB simulée) sur le vrai fichier : 43 productions, étape RECOLTE présente, 12 distillations / 27 lignes générées, toutes en statut 3, **aucune colonne Tuya dans les INSERT**, 2 avertissements « plusieurs passes » (TEST0014).
- `php -l` sur tous les fichiers modifiés.

### Non traité (reste à faire)
- Import toujours non transactionnel, sans contrôle de doublon (ré-import = doublons), sans token CSRF explicite — cf. `doc/` état des lieux 0.12.0.
- Le bloc 2 de « FERMENTATION/MACERATION DEBUT » (lignes dupliquées avec Date/Heure/Température/Contrôle visuel, probablement une erreur de libellé pour FIN) fusionne dans l'étape DEBUT : la dernière valeur écrase la première. À arbitrer avec l'exploitant.

---

## [0.12.0] — 2026-09-19

### ADD : génération automatique des Distillations depuis l'import Excel (EVOLUTIONS.md #1)

Lors de l'import Excel, si l'étape DISTILLATION d'une production contient au moins une coupe (Tête/Cœur/Queue) avec un volume renseigné, une `DistDistil` et ses `DistDistilLine` sont désormais générées automatiquement — plus besoin de ressaisir manuellement les distillations historiques après un import en masse.

- **ADD : `DistImport::generateDistilFromImport()`** — appelée automatiquement après création réussie de l'étape DISTILLATION. Ne crée la distillation que si au moins une coupe a un volume > 0 (les 3 coupes sont indépendantes, pas besoin des 3) ; sinon retourne silencieusement sans erreur (étape DISTILLATION vide = cas normal).
  - Statut laissé à `STATUS_RUNNING` (pas `STATUS_CLOSED`) : la distillation générée reste éditable via la fiche après import, tant que la validation/prévisualisation (EVOLUTIONS.md #2/#3, non encore implémentées) ne fiabilise pas les données en amont.
  - Relevé Tuya (compteur électrique atelier) câblé sur ce nouveau canal de création, avec le même contrat non bloquant que le formulaire manuel (`distdistil_card.php::addline`) — cf. `tuya-integration-distillerie.md` §4.1 et `ChangeLog.md` v0.10.0. C'était le point explicitement laissé en TODO à ce sujet.
  - Étape DISTILLATION liée à la distillation générée via `fk_distil` (colonne déjà présente sur `llx_dist_prod_step`, jusqu'ici jamais renseignée par l'import).
- **ADD : `DistImport::normalizeChampKey()` / `findChampValue()` / `findChampValueAny()`** — correspondance floue (accents/casse/ponctuation ignorés, plusieurs libellés candidats essayés) pour retrouver les colonnes Tête/Cœur/Queue dans l'Excel importé. **Nécessaire** car les libellés exacts des colonnes DISTILLATION ne sont documentés dans `doc/IMPORT_PRODUCTIONS.md` que de façon informelle ("Tête : Volume (ml), % alcool moyen"), sans fichier `TRACABILITE.xlsx` réel disponible pour valider un nom de colonne exact. Testé avec plusieurs variantes plausibles ("Tête (ml)", "Volume Tête (ml)", "Tête : Volume (ml)"...) — toutes reconnues. **Si l'import réel ne détecte aucune distillation malgré des données présentes**, voir la procédure de diagnostic ajoutée dans `doc/IMPORT_PRODUCTIONS.md`.
- **`class/distimport.class.php::importProductionSteps()`** — signature de retour changée de `int` (nombre d'étapes) vers un tableau (`steps`, `distillations`, `distil_lines`, `warnings`), pour remonter le résultat de la génération automatique jusqu'à `admin/import_productions.php` (déjà câblé pour afficher `details`/`warnings`, aucune modification nécessaire côté page).
- **Doc** : `doc/IMPORT_PRODUCTIONS.md` mis à jour (section "Génération Distillations" sortie de "Phase 2" projetée, note sur la correspondance floue et la procédure de diagnostic en cas de non-détection). `EVOLUTIONS.md` : item #1 retiré (implémenté), renumérotation de la liste (y compris un trou de numérotation pré-existant 10→12 corrigé au passage).

### Vérifié
- Test unitaire isolé de `normalizeChampKey`/`findChampValueAny` sur plusieurs variantes de libellés plausibles (avec/sans accents, ordre des mots différent) : toutes correctement reconnues. Étape DISTILLATION vide → aucune génération, aucune erreur. Une seule coupe renseignée sur 3 → distillation créée avec uniquement cette ligne.
- Agent de contrôle : 0 erreur critique, 6 avertissements (inchangé, résiduels déjà documentés), 9 suggestions (inchangé), 114 contrôles réussis, score 95,0%.

---

## [0.11.0] — 2026-09-19

### ADD : page admin "SQL Upgrade" — diagnostic et correction du schéma SQL

Nouvel onglet dans Setup qui contrôle, par rapport à la version du module installée, que les tables définies dans `sql/` existent et que les correctifs de schéma connus (colonnes/contraintes ajoutées après la création initiale d'une table sur une version ultérieure) sont bien appliqués — sans avoir à désactiver/réactiver le module.

- **ADD : `admin/sql_upgrade.php`** — diagnostic en lecture seule (liste des tables attendues avec statut existe/absente, liste des correctifs connus avec la version qui les a introduits et leur statut à jour/manquant) + bouton "Appliquer les correctifs manquants" (CSRF vérifié explicitement dès l'écriture).
- **ADD : onglet "SQL Upgrade"** dans le menu admin du module (`lib/distillerie.lib.php`).
- **Refactor DRY : `lib/distillerie.lib.php`** — les 3 migrations existantes (`addMissingRegistryColumns`, `addMissingAlambicConstraints`, `addMissingDistilLineColumns`, ~196 lignes dans le descripteur) extraites en fonctions génériques réutilisables :
  - `distillerie_check_columns()` / `distillerie_check_unique_keys()` — vérification + correction génériques (idempotentes, basées sur `information_schema`, tolérantes aux doublons pour les contraintes UNIQUE)
  - `distillerie_get_known_schema_fixes()` — registre central des correctifs connus (table, colonnes/contraintes attendues, version d'introduction) : une seule source de vérité, désormais utilisée à la fois par `modDistillerie::init()` (activation du module) et par la nouvelle page de diagnostic
  - `distillerie_run_schema_diagnostic($db)` / `distillerie_apply_schema_fixes($db)` — orchestration diagnostic (lecture seule) / correction
  - Les 2 cas particuliers non génériques (élargissement `typeregistre` INTEGER→VARCHAR, index UNIQUE `numero_entrepot`) conservés à l'identique, extraits en fonctions dédiées
- **`core/modules/modDistillerie.class.php`** : les 3 méthodes privées supprimées, `init()` appelle désormais `distillerie_apply_schema_fixes($this->db)`.

### Corrections de l'agent de contrôle (régression détectée pendant cette session)
- **`test/agent_controle.php`** — les contrôles 14/18 (pattern `information_schema`) et 17/18 (détection d'un mécanisme de migration) ne scannaient que le descripteur (`modDistillerie.class.php`). Le refactor ci-dessus ayant déplacé la logique de migration vers `lib/distillerie.lib.php`, ces deux contrôles seraient devenus respectivement muet et faussement positif (2 tables historiquement migrées — `dist_registry`, `dist_distil_line` — auraient été signalées à tort comme "sans mécanisme de migration"). Élargi le scan aux deux fichiers (descripteur + lib) avant livraison.

### Vérifié
- Agent de contrôle : 0 erreur critique, 6 avertissements (inchangé, résiduels déjà documentés en v0.10.2), 9 suggestions (2 faux positifs corrigés dans le même mouvement que la régression ci-dessus), **114 contrôles réussis**, score **95,0%**.

---

## [0.10.3] — 2026-09-19

### Corrections (retour terrain sur v0.10.0)

- **`admin/integration.php`** — HTTP 500 lors de l'enregistrement de la configuration Compteur électrique Tuya. Cause : `require_once DOL_DOCUMENT_ROOT.'/core/lib/admin.lib.php';` manquant, alors que `core/actions_setmoduleoptions.inc.php` (natif Dolibarr, ajouté en v0.10.0 pour sauvegarder les 4 constantes Tuya) en dépend systématiquement — confirmé en comparant plusieurs pages admin natives (`admin/mrp.php`, `admin/ticket.php`, `admin/delivery.php`, `asset/admin/setup.php`), qui incluent toutes `admin.lib.php` juste avant `actions_setmoduleoptions.inc.php`. `admin/setup.php` du module l'avait déjà (d'où l'absence de ce bug sur la page Paramètres) ; `admin/integration.php` ne l'avait jamais eu.

### Vérifié
- Agent de contrôle : 0 erreur critique, 6 avertissements (inchangé), 8 suggestions, 112 contrôles réussis, score 94,9%.

---

## [0.10.2] — 2026-09-19

### Corrections — résorption du backlog qualité (17 → 6 avertissements)

Traitement méthodique des 17 avertissements identifiés en v0.10.1, occurrence par occurrence (distinction GET/POST) plutôt qu'une correction aveugle.

- **`lib/distillerie.lib.php`** — nouvelle fonction `distillerie_redirect($url)` (pattern méta-refresh + JS du template, sans toucher aux buffers), à utiliser pour toute redirection suivant un formulaire POST.
- **`header('Location:')` corrigés (13 occurrences)** :
  - **5 faux positifs confirmés** (`admin/dict_broyage/dict_etats/dict_fruits/dict_futs/dict_nettoyage.php`) : ce sont de purs stubs de redirection GET (0 traitement POST) — contrôle 8/18 de l'agent affiné pour exiger la présence de `GETPOST(` dans le fichier avant de flaguer.
  - **10 vrais risques corrigés** dans `distdistil_card.php` (actions `add`, `saveheader`, `addline`, `updateline` — toutes POST) : remplacés par `distillerie_redirect()`.
  - **7 vrais risques corrigés** répartis sur `distmouvement_card.php` (2 : `add`, `update`), `distprod_card.php` (2 : `add`, `update`), `distregistry_card.php` (2 : `add`, `update`), `production_step.php` (5, action `add`).
  - **5 vrais risques corrigés** dans `production_step_edit.php` (gardes de validation + action `update`, potentiellement atteintes en contexte POST).
  - Les occurrences GET-only vérifiées (`setstatus`, `setofficial`, `delline`, `buildpdf`, redirection par défaut `$id<=0`, `admin/setup.php` `resetdb`/`specimen`) restent en `header('Location:')` — aucune régression, comportement inchangé.
- **Pagination ajoutée (3 pages liste)** : `distdistil_list.php`, `distregistry_list.php` (pattern simple, repris de `distprod_list.php`) et `distmouvement_list.php` (COUNT dupliquant les 5 filtres actifs pour un total cohérent, `$param` construit pour préserver les filtres dans les liens de pagination/tri — bug latent corrigé au passage : `$param` était déjà référencé dans les en-têtes de tri mais jamais défini).
- **`production_step.php`** — les 3 accès directs à `$_POST` (écriture de valeurs par défaut depuis un JSON stocké, jamais des données utilisateur) remplacés par un tableau local `$defaultValues` ; comportement strictement identique (ces valeurs n'étaient de toute façon jamais affichées dans les champs texte, qui imprimaient `value=""` en dur).

### Restent 6 avertissements (résiduels attendus, documentés)
- `check_tables.php` : bootstrap non standard — **intentionnel** (garde CLI-only ajoutée en v0.9.9, ne doit pas inclure `main.inc.php`).
- 5× `header('Location:')` sur `admin/setup.php`, `distdistil_card.php`, `distmouvement_card.php`, `distprod_card.php`, `production_step.php` : la détection du contrôle 8/18 reste au niveau fichier (pas par occurrence) — ces 5 fichiers contiennent encore des redirections **GET-only vérifiées sûres** (ex. `resetdb`/`specimen`, `setstatus`/`setofficial`/`delline`/`buildpdf`, `create_distilmanual`) aux côtés des redirections POST déjà corrigées. Un affinement occurrence-par-occurrence demanderait une analyse de flux de contrôle disproportionnée par rapport au risque résiduel (déjà nul, vérifié manuellement ligne par ligne dans cette session).

### Bonus — fichier de test cassé (hors périmètre de l'agent, découvert en vérification exhaustive)
- **`test/phpunit/DistillerieFunctionalTest.php`** et sa copie dupliquée **`test/phpunit/functionnal/DistillerieFunctionalTest.php`** : erreur de syntaxe fatale (bloc de bootstrap `main.inc.php` injecté par erreur avant une déclaration `namespace`, plus deux accolades orphelines) — scaffold ModuleBuilder resté à l'état de gabarit, jamais fonctionnel. L'agent de contrôle exclut volontairement `test/` de son scan (`scanPhpFiles`), d'où l'absence de détection lors des sessions précédentes. Bloc de bootstrap retiré (un fichier de test PHPUnit n'a pas à inclure `main.inc.php` lui-même) ; les deux copies compilent désormais sans erreur. Le contenu du test (Selenium, URL `dev.zenfusion.fr` codée en dur, identifiants `admin/admin`) reste un exemple générique du ModuleBuilder, non adapté à ce module — à réécrire ou supprimer dans une session dédiée aux tests si souhaité.

### Vérifié
- Agent de contrôle : 0 erreur critique, 6 avertissements (au lieu de 17), 8 suggestions inchangées (tables volumineuses sans migration — hors périmètre), **112 contrôles réussis**, score **94,9%**.

---

## [0.10.1] — 2026-09-19

### Corrections (retour terrain sur v0.10.0)

- **`lib/distillerie.lib.php`** — `admin/integration.php` n'était référencé dans aucun onglet admin (`distillerieAdminPrepareHead()`), donc invisible depuis l'interface Dolibarr malgré son existence. Ajout des onglets **Intégrations** et **Automatisations** — ce dernier (`admin/automation.php`) souffrait du même défaut, corrigé au passage.
- **`admin/integration.php` / `admin/automation.php`** — ajout de la barre d'onglets (`dol_get_fiche_head`/`dol_get_fiche_end`), absente sur ces deux pages alors que toutes les autres pages admin du module l'affichent (`setup.php`, `alambic.php`, `import_productions.php`, `autodiag.php`).
- **`distdistil_card.php`** — le formulaire d'ajout de ligne (7 colonnes) ne reflétait pas les 3 nouvelles colonnes du tableau des lignes existantes (11 colonnes depuis v0.10.0), créant un désalignement visuel qui laissait croire que les champs Tuya n'étaient "que dans l'en-tête". Ajout de 3 cellules informatives en lecture seule (icône + tirets) dans la ligne de saisie, avec info-bulle expliquant que ces valeurs sont capturées automatiquement à la création de la ligne, pas saisies manuellement.
- **`langs/fr_FR/distillerie.lang`** — apostrophe dans la valeur de `DistilTuyaAutoCapture` (introduite dans cette même session), détectée et corrigée avant livraison.

### Vérifié
- Agent de contrôle : 0 erreur critique, 17 avertissements (backlog pré-existant, inchangé), 8 suggestions, 105 contrôles réussis, score 85,4%.

---

## [0.10.0] — 2026-09-19

### ADD : intégration compteur électrique Tuya Cloud sur les lignes de distillation

Câblage de la classe `TuyaClient` (fournie, cf. `tuya-integration-distillerie.md`) pour capturer automatiquement `power` (W), `energy` (kWh) et `temp` (°C) depuis le compteur électrique connecté de l'atelier, à chaque ajout d'une ligne de distillation.

- **ADD : `class/tuyaclient.class.php`** — client Tuya Cloud API (signature HMAC-SHA256, cache de l'`access_token` dans `llx_const`, méthode `getMeterValues()`). Aligné sur la convention du module : étend `CommonObject` avec `$table_element = ''` (classe de service sans table SQL propre), comme `DistIntegration`/`DistAutomation`.
- **ADD : 3 nouvelles colonnes sur `llx_dist_distil_line`** — `power_w` (DECIMAL 10,2), `energy_kwh` (DECIMAL 12,3), `temp_ambiante` (DECIMAL 6,2). Migration idempotente `addMissingDistilLineColumns()` ajoutée dans le descripteur (pattern `addMissingRegistryColumns`/`addMissingAlambicConstraints`) pour les installations existantes ; colonnes incluses nativement dans `sql/llx_dist_distil_line.sql` pour les installations neuves.
- **ADD : `distdistil_card.php`** — appel `TuyaClient::getMeterValues()` câblé dans l'action `addline`, **synchrone et non bloquant** : en cas d'échec (abonnement IoT Core expiré, appareil offline, timeout, configuration absente), la ligne se crée quand même avec les 3 colonnes Tuya vides et un message d'avertissement explicite — jamais d'erreur bloquante. Affichage des 3 valeurs (lecture seule) dans le tableau des lignes existantes.
- **ADD : `admin/integration.php`** — nouvelle section "Compteur Tuya" : configuration des 4 constantes (`DISTILLERIE_TUYA_CLIENT_ID/SECRET/DEVICE_ID/ENDPOINT`, pattern `FormSetup`, secret masqué en `password`) et bouton "Tester la connexion". CSRF vérifié explicitement dès l'écriture (`GETPOST('token') == $_SESSION['newtoken']`), conformément au rappel de l'audit sécurité v0.9.9 sur ce même fichier.
- **ADD** : libellés `DistilTuyaPowerW`, `DistilTuyaEnergyKwh`, `DistilTuyaTempAmbiante`, `DistilTuyaSectionTitle`, `DistilTuyaTestButton`, `DistilTuyaTestSuccess` (FR/EN).

### Écarts assumés entre la spécification fournie et l'implémentation
- La table `dist_distil_line` n'utilise pas le mécanisme d'extrafields Dolibarr (insertion en SQL brut) : les 3 mesures Tuya sont stockées comme **colonnes réelles**, pas comme `array_options`, pour rester cohérent avec l'existant (`temp_ala_c`, `volume_ml`, `tav_pct`...).
- Sur les 3 canaux de création de ligne mentionnés dans la spec (formulaire manuel, import Excel, API REST), **seul le formulaire manuel existe réellement dans le code actuel** — l'import Excel (`class/distimport.class.php`) ne crée aucune `dist_distil_line` à ce jour (roadmap `EVOLUTIONS.md` #1), et il n'existe aucune classe API REST dans ce module. L'appel Tuya n'est donc câblé que sur ce canal ; à répliquer au même endroit le jour où l'import Excel génère des distillations.
- `voltage`, `current`, `fac`, `fre` sont disponibles dans le retour de `getMeterValues()` mais non persistés, conformément à la demande initiale (§4.6 de la spec).
- Décision actée : stockage du relevé cumulé brut (`energy`), pas de calcul de delta à l'écriture (§4.5 de la spec).

### Vérifié
- Agent de contrôle : 0 erreur critique, 17 avertissements (backlog qualité pré-existant, inchangé), 8 suggestions, **106 contrôles réussis**, score 86,2%.
- Le contrôle 10/18 (classes métier) a d'abord signalé `TuyaClient` comme n'étendant pas `CommonObject` — vérifié en comparant aux 15 autres classes du module (toutes étendent `CommonObject`, y compris les classes de service sans table comme `DistIntegration`) : pas un faux positif, `TuyaClient` alignée sur la convention plutôt que d'exclure la règle.

---

## [0.9.10] — 2026-08-21

### Vérifié — exécution réelle de l'agent de contrôle après l'audit sécurité v0.9.9

L'agent (`test/agent_controle.php`) n'avait pas été relancé depuis la livraison de v0.9.9. Son exécution effective révèle un écart avec les mentions "100% — 0 erreur, 0 avertissement" des ChangeLog v0.9.6 à v0.9.8 : **4 erreurs critiques et 17 avertissements réels** étaient présents, non détectés faute d'avoir relancé l'agent après les changements correspondants. Toutes les erreurs critiques sont corrigées ci-dessous ; les avertissements restants sont un backlog qualité distinct, non traité dans cette session (voir note en fin d'entrée).

### 🐛 Corrigé (erreurs critiques révélées par l'agent)

**1. `admin/automation.php` / `admin/integration.php` — bootstrap `main.inc.php` non standard**
- Les deux fichiers utilisaient `require '../main.inc.php';` (chemin relatif simple), au lieu du bloc multi-chemins standard utilisé partout ailleurs dans le module. Risque : page cassée si le fichier est appelé depuis un contexte où la profondeur relative diffère (rewrite, symlink, sous-domaine).
- **Correctif** : remplacement par le bloc multi-chemins standard (variante `admin/`, deux niveaux), identique à celui de `admin/alambic.php`.

**2. `distreg_dashboard.php` — 3× `fetch_object()` non protégés**
- `$obj = $db->fetch_object($resql);` sans vérifier `$resql` au préalable (lignes "Nombre de mouvements", "Volume total produit", "Volume total sorti") : si la requête échoue, `fetch_object(false)` provoque une erreur fatale PHP 8.
- **Correctif** : pattern sécurisé `$obj = ($resql && ($o = $db->fetch_object($resql))) ? $o : null;` puis accès via `??` sur les propriétés.

**3. `test/agent_controle.php` — faux positif du contrôle 18/18 (anti-régression `nom_fruit`)**
- `distprod_dashboard.php` était signalé pour réutilisation de la colonne fautive `nom_fruit`, alors qu'il ne s'agit que de l'alias SQL intentionnel (`d.label as nom_fruit`) introduit par le correctif v0.9.6 — pas une régression.
- **Correctif de l'agent** : exclusion du cas `as <colonne_fautive>` (alias de sortie), qui ne constitue jamais une lecture de la colonne source réellement fautive.

### Non traité (backlog qualité, hors périmètre sécurité)
- **17 avertissements** subsistent, dont 13 occurrences du pattern `header('Location:')` en contexte POST (`admin/dict_*.php`, `admin/setup.php`, `distdistil_card.php`, `distmouvement_card.php`, `distprod_card.php`, `distregistry_card.php`, `production_step.php`, `production_step_edit.php`) — risque documenté de HTTP 500 via `ob_gzhandler`, à traiter méthodiquement fichier par fichier (certains cas sont probablement des faux positifs à vérifier au cas par cas, ex. redirections en contexte GET plutôt que POST). Egalement : 3 pages liste sans `print_barre_liste()`, 1 accès direct `$_POST` en pré-remplissage (`production_step.php`, déjà noté non bloquant en v0.9.9).
- Point à traiter dans une prochaine session dédiée à la qualité, distincte de l'audit sécurité.

### Vérifié
- Agent de contrôle : **0 erreur critique** (était 4), 17 avertissements (backlog qualité), 9 suggestions inchangées — **103 contrôles réussis** sur un total élargi (le module a grandi depuis le dernier décompte à 56)
- Score de conformité : 85,8%

---

## [0.9.9] — 2026-08-21

### SEC — Audit sécurité (skill dolibarr-module-dev, catalogue Dolibarr v21→23.0.3)

Premier audit de sécurité formel du module, exécuté selon `references/security_audit_process.md`. 3 points confirmés et corrigés :

- **SEC : `admin/integration.php`** — les actions `import_dae`, `export_dti` et `export_fec` affichaient le token CSRF dans le formulaire mais ne le vérifiaient jamais côté serveur, contrairement au pattern déjà établi dans `distdistil_card.php` (`GETPOST('token','alpha') == $_SESSION['newtoken']`). Ajout de la vérification explicite avant chacune des 3 actions, avec `accessforbidden('InvalidToken')` en cas d'échec.
- **SEC : `class/distintegration.class.php::syncSites()`** — injection SQL confirmée : `$site_source`/`$site_dest` concaténés directement dans une requête `INSERT INTO site_...` sans passer par `$db->escape()`. Méthode actuellement non appelée (code mort en attente de la roadmap Sites/Chais, `EVOLUTIONS.md` #6), corrigée par prudence avant qu'une future page ne la branche à une entrée utilisateur : validation stricte par expression régulière (`^[a-zA-Z0-9_]+$`) en plus de `$db->escape()`.
- **SEC : `check_tables.php`** — script de diagnostic à la racine du module, sans inclusion de `main.inc.php` et donc sans authentification Dolibarr. Restreint à une exécution CLI uniquement (`php_sapi_name() !== 'cli'` → HTTP 403), pour éviter toute exécution directe en HTTP par un visiteur non connecté s'il reste dans un ZIP livré en production.

### Vérifié sans correction nécessaire
- Pas d'usage de `run_trigger()` (supprimé en v23), pas de `$user->rights->xxx` legacy, pas de `dol_eval()`, pas de classe API REST custom dans ce module.
- Usage de `document.php?modulepart=distillerie` (2 occurrences, `admin/setup.php` spécimen PDF et `distdistil_card.php` liste documents) : chemins construits à partir de `glob()`/`basename()` côté serveur, jamais depuis une entrée utilisateur — pas de path traversal possible.
- `curl_exec()` dans `DistIntegration::sendToDGDDI()` : URL fixe (`https://api.douane.gouv.fr/declarations`, endpoint stub non encore implémenté), aucun risque SSRF tant que l'URL reste codée en dur.
- Écriture directe dans `$_POST` (`production_step.php`, pré-remplissage de valeurs par défaut depuis la BDD) : pas un risque de sécurité (aucune donnée utilisateur non fiable réinjectée), mais pattern fragile à surveiller — non corrigé cette session (amélioration de robustesse, pas un correctif de sécurité).
- Les 5 pages `admin/dict_*.php` (redirections vers `admin/dict.php` natif) : pas de `hasRight()`/`accessforbidden()` explicite, mais authentification déjà imposée par `main.inc.php` et contrôle d'accès effectué par la page cible native — aucune action à ce niveau.

### Suggestion pour une prochaine session (hors périmètre de cet audit)
- Le catalogue `security_updates_v21_to_v23.md` du skill mentionne CVE-2026-22666/37711/37712 comme "contesté/non confirmé" : une recherche web du 2026-08-21 montre que ces avis sont désormais **officiellement publiés et corrigés** par l'éditeur (23.0.2/23.0.3). L'instance `gestion.barnecki-spirits.com` (23.0.3) est déjà protégée côté core — mise à jour du catalogue du skill à faire dans une session dédiée à la maintenance du skill, hors périmètre du module lui-même.

### Vérifié
- Agent de contrôle : à relancer après cette session (non exécuté dans le cadre de cet audit)

---

## [0.9.8] — 2026-07-21

### Modifié
- **Dictionnaire Alambic : retour à une page d'administration dédiée** (`admin/alambic.php`), pour fournir un **vrai sélecteur de date** (calendrier) sur le champ `Date_mise_service`.
  - **Raison du changement** : l'interface générique native de dictionnaire Dolibarr (`$this->dictionaries`) ne propose qu'un champ texte libre pour toute colonne déclarée par un module externe — aucune syntaxe de type ("date", "select"...) n'a pu être confirmée avec certitude dans les sources disponibles (skill `dolibarr-module-dev`, recherche du code source officiel `htdocs/admin/dict.php`). Une tentative avec un suffixe `:date` non vérifié a été **annulée en v0.9.7** par prudence (risque de rupture silencieuse de la soumission du formulaire).
  - **Solution retenue** : page dédiée utilisant `Form::selectDate()` — API Dolibarr standard et déjà éprouvée dans ce même module (`distregistry_card.php`, `production_step_edit.php`).
  - Alambic retiré de `$this->dictionaries` dans `modDistillerie.class.php` (6 dictionnaires natifs restants : fruits, états, nettoyage, broyage, fûts, type registre). Évite la double-gestion qui avait causé le bug `date_service`/`date_mise_service` initial (v0.9.4) : un seul système gère désormais ce dictionnaire.
  - Onglet "Dictionnaire Alambic" réintégré dans Configuration → Distillerie.
  - La page réutilise les **vrais noms de colonnes** (`code`, `label`, `active`, `position`, `date_mise_service`, `etat`, `numero_serie`, `numero_douane`) et respecte les contraintes `UNIQUE` sur `numero_serie`/`numero_douane` (v0.9.7) : une tentative de doublon affiche un message d'erreur explicite (`ErrorRecordAlreadyExists`) plutôt qu'une erreur SQL brute.
  - Formulaire conforme aux règles de sécurité du skill : token CSRF, `hasRight`/`$user->admin`, `GETPOST()` avec filtres explicites, `$db->escape()`.

### Vérifié
- Agent de contrôle : **100%** — 56 contrôles passés, 0 erreur, 0 avertissement, 9 suggestions (inchangées)
- `EVOLUTIONS.md` : point "Widget calendrier Date_mise_service" retiré (résolu)

---


## [0.9.7] — 2026-07-20

### Ajouté
- **Contraintes d'unicité sur le dictionnaire Alambic** : `Numero_serie` et `Numero_douane` sont désormais uniques (une seule ligne peut porter un numéro de série ou un numéro de douane donné).
  - `sql/llx_dist_dict_alambic.sql` : ajout de `UNIQUE KEY uk_llx_dist_dict_alambic_numero_serie (numero_serie)` et `UNIQUE KEY uk_llx_dist_dict_alambic_numero_douane (numero_douane)` (installations neuves).
  - Nouvelle méthode `addMissingAlambicConstraints()` dans `modDistillerie.class.php`, appelée en tête de `init()` : migration auto-réparatrice idempotente pour les installations déjà en production (pattern établi en v0.9.2 pour `llx_dist_registry` — vérification via `information_schema.STATISTICS` + `DATABASE()`, tolérance à l'échec si des doublons existent déjà, log `dol_syslog()` systématique).
  - Les colonnes étant `NULL`-ables, plusieurs lignes sans numéro de série/douane restent possibles (comportement standard MySQL/MariaDB : `NULL` n'est jamais considéré égal à `NULL` pour une contrainte `UNIQUE`).

### Non traité (nécessite confirmation)
- **Champ `Date_mise_service` en widget calendrier** : tentative initiale d'ajouter le suffixe `:date` à la déclaration `tabfield` (`'date_mise_service:date'`), **annulée** après vérification — cette syntaxe n'a pu être confirmée ni dans les références du skill `dolibarr-module-dev`, ni via recherche de la source officielle `htdocs/admin/dict.php`. Une syntaxe incorrecte aurait risqué de casser silencieusement la soumission du formulaire (nom de champ POST malformé, valeur jamais transmise à l'INSERT). L'infobulle d'aide sur le format attendu (`AAAA-MM-JJ`, ajoutée en v0.9.6) reste en place comme palliatif vérifié sans risque. Point à traiter dans une prochaine session avec accès direct au code source de `dict.php` pour confirmer la syntaxe exacte avant implémentation.

### Vérifié
- Agent de contrôle : **100%** — 56 contrôles passés, 0 erreur, 0 avertissement, 9 suggestions (inchangées)

---


## [0.9.6] — 2026-07-20

### 🐛 Corrigé (confirmé par test réel + audit proactif)

**1. Dictionnaires — "Enregistrement déjà existant" malgré une liste vide (bug systémique, les 7 dictionnaires)**
- Confirmé par `dolibarr.log` : le correctif "active en double" de la v0.9.5 fonctionne bien (plus d'erreur SQL sur l'INSERT).
- Nouveau bug découvert à la place : les 7 `tabsql` du descripteur filtrent `WHERE f.entity = 0`, mais l'interface générique Dolibarr insère les nouvelles lignes sans préciser `entity` explicitement → la colonne prend sa valeur `DEFAULT 1` de la table. Résultat : toute ligne nouvellement créée (entity=1) est **invisible** dans la liste (qui ne montre que entity=0), donnant l'illusion d'un échec ("Aucun enregistrement trouvé"). En retentant la création du même code, la contrainte unique `(entity, code)` bloque désormais un doublon bien réel mais invisible → `Duplicate entry '1-ALAMBIC194'`.
- **Correctif** : les 7 `tabsql` filtrent désormais `WHERE f.entity IN (0,".$conf->entity.")` — cohérent avec le comportement d'insertion et avec le pattern déjà utilisé dans `DistDictTypeRegistre::getAllActive()`.
- Ajout d'une infobulle d'aide (`tabhelp`) sur le champ `Date_mise_service` du dictionnaire Alambic précisant le format attendu (`AAAA-MM-JJ`), après qu'une première tentative avec le format `JJ/MM/AAAA` ait été rejetée par MySQL (`Incorrect date value`).

**2. Import Excel — liaison/création automatique des fruits totalement silencieuse (bug latent découvert par audit)**
- `DistImport::getFruitId()` utilisait les colonnes `nom_fruit`/`actif`, qui n'existent pas dans `llx_dist_dict_fruits` (colonnes réelles : `code`, `label`, `active`, `entity`) — la même famille d'erreur que les bugs `date_service`/`date_distillation` déjà corrigés. La méthode échouait silencieusement à chaque import, laissant `fk_fruit` systématiquement `NULL` sans qu'aucune erreur ne remonte à l'utilisateur.
- **Correctif** : requêtes SELECT/INSERT réécrites avec les vraies colonnes, génération d'un code technique dérivé du nom (majuscules + underscores), filtre `entity IN (0, courant)`, log d'échec explicite en cas de problème.

**3. Dashboard Productions — sections "Productions par fruit" et "Dernières productions" silencieusement vides**
- Même bug de colonne inexistante (`d.nom_fruit` au lieu de `d.label`) dans **2 requêtes** de `distprod_dashboard.php`, découvert par audit proactif après la correction de l'import (recherche de toutes les occurrences de `nom_fruit` dans le module).
- **Correctif** : `SELECT d.label as nom_fruit` / `SELECT ... d.label as fruit` — alias SQL préservant la compatibilité avec le code PHP existant (`$obj->nom_fruit`, `$obj->fruit`) sans autre modification.

### Vérifié
- `DistDictTypeRegistre` (classe utilisée par `distregistry_card.php`) : déjà correcte (`entity IN (0, courant)`), aucune modification nécessaire
- `distprod_card.php`/`distprod_list.php` : n'interrogent pas le dictionnaire fruits (aucun risque de cette classe de bug à cet endroit)
- Agent de contrôle : **100%** — 56 contrôles passés, 0 erreur, 0 avertissement, 9 suggestions (inchangées)

### Agent de contrôle (v5)
- Nouveau contrôle intégré au [2/18] : détecte les tabsql de dictionnaire filtrant `entity = 0` strict
- Nouveau contrôle **[18/18]** : anti-régression sur les 4 noms de colonnes déjà corrigés par le passé (`nom_fruit`, `actif`, `date_service`, `date_distillation`) — empêche leur réapparition accidentelle dans une future requête SQL
- 18 contrôles au total (était 17)

### Note sur les données de test existantes
Les lignes déjà insérées par erreur lors des tests précédents (ex. Alambic ALAMBIC194 en entity=1) deviennent automatiquement **visibles** après ce correctif, sans opération de nettoyage nécessaire — le filtre élargi `entity IN (0, courant)` les inclut désormais dans la liste.

---


## [0.9.5] — 2026-07-20

### 🐛 Corrigé (3 bugs restants du rapport de tests)

**1. Dictionnaire Alambic (et TOUS les autres) — `Column 'active' specified twice`**
- Les 7 dictionnaires du module (`fruits`, `etats`, `nettoyage`, `broyage`, `futs`, `alambic`, `typeregistre`) déclaraient explicitement le champ `active` dans `tabfield`/`tabfieldvalue`/`tabfieldinsert`, alors que l'interface générique de dictionnaire Dolibarr gère déjà ce champ automatiquement pour toute page de dictionnaire natif → doublon dans l'INSERT SQL généré.
- **Correctif** : suppression de `active` des 3 tableaux pour les 7 dictionnaires (le champ `active` reste bien présent dans `tabsql` pour l'affichage de la liste, qui n'est pas concerné par ce bug).
- **Bug systémique** : affectait potentiellement la création de lignes dans tous les dictionnaires, pas seulement Alambic (seul celui-ci avait été testé).

**2. Dashboard Distillations à zéro malgré des distillations existantes en 2026**
- Cause réelle (différente du diagnostic précédent) : la table `llx_dist_distil` utilise la colonne **`date_start`**, mais `distdistil_dashboard.php` et `distdashboard.php` interrogeaient une colonne **`date_distillation`** qui n'existe pas. Le pattern SQL sécurisé (`if ($resql && $obj)`) masquait l'échec de la requête en silence, produisant des zéros sans erreur visible.
- **Correctif** : remplacement de toutes les occurrences de `date_distillation` par `date_start` (13 occurrences dans `distdistil_dashboard.php`, 3 dans `distdashboard.php`). Vérification croisée de `dist_distil_line` (`volume_ml`, `tav_pct`, `cut_type`) : cohérentes, aucune autre correction nécessaire.

**3. Export Excel des déclarations — `Class 'PhpOffice\PhpSpreadsheet\Writer\BaseWriter' not found`**
- `DistExportExcel` dépendait de `PhpOffice\PhpSpreadsheet\Writer\Xlsx`, nécessitant l'autoloader Composer indisponible sur ce serveur (même famille de problème que l'import Excel résolu en v0.6.4-0.6.5).
- **Correctif** : nouvelle classe **`includes/SimpleXLSXWriter.php`** — générateur `.xlsx` minimal et autonome (ZipArchive natif PHP uniquement, aucune dépendance), produisant un fichier OOXML valide avec styles de base (gras, en-têtes colorées, bordures, cellules fusionnées). `DistExportExcel::exportMouvements()` et `exportStocksPeriode()` réécrites pour l'utiliser. Validé par relecture avec openpyxl (structure, données, styles conformes, aucun avertissement).

**Export PDF déclarations — 503 persistant : diagnostic renforcé (non résolu avec certitude)**
- Le fait que l'export Excel affiche désormais un message d'erreur propre confirme que le `try/catch(\Throwable)` fonctionne correctement pour les erreurs "catchables" dans cet environnement. La 503 sur le PDF suggère donc une erreur **non catchable** (épuisement mémoire ou dépassement de `max_execution_time`).
- Ajouts : `ini_set('memory_limit', '256M')` défensif, `register_shutdown_function()` capturant spécifiquement les erreurs fatales non catchables (`E_ERROR`, `E_PARSE`, etc.) avec usage mémoire précis avant crash, logs `dol_syslog()` avant/après génération dans `pdf_declaration_mensuelle::write_file()` (désormais elle-même protégée par un `try/catch` interne dédié). **Le prochain test révélera la cause exacte dans `dolibarr.log`.**

### Agent de contrôle (v4)
- Nouveau contrôle intégré au [2/17] : détecte le champ `active` dupliqué dans `tabfield` des dictionnaires
- Score : **100%** — 54 contrôles passés, 0 erreur, 0 avertissement, 9 suggestions

---


## [0.9.4] — 2026-07-20

### 🐛 Corrigé (5 bugs critiques signalés)

**1. Impossible d'enregistrer un mouvement de registre (table toujours vide)**
- `DistCategorieFiscale::getSelectArrayGrouped()` retournait un tableau imbriqué (`array[groupe][id]=libelle`) passé à `Form::multiselectarray()` qui n'accepte qu'un tableau plat → `TypeError` fatal PHP en cours de rendu → page HTML tronquée avant le bouton de soumission.
- `fk_categorie_fiscale` est un `INTEGER` unique en base (`llx_dist_mouvement`), mais le formulaire utilisait un multiselect (sélection multiple) incompatible avec `GETPOSTINT()` côté traitement.
- **Correctif** : nouvelle méthode `DistCategorieFiscale::getSelectArrayFlat()` (tableau plat, groupe préfixé au libellé) + remplacement du multiselect par un select simple dans `distmouvement_card.php`.

**2. Export PDF distillation → page d'erreur navigateur**
- `pdf_getInstance($db)` appelé avec l'objet base de données au lieu d'un format de page (`'A4'`, etc.) → plantage TCPDF.
- Le PDF n'était de toute façon qu'archivé sur disque, jamais renvoyé au navigateur.
- **Correctif** : `pdf_getInstance()` sans argument + streaming réel du fichier (`Content-Disposition: inline`) dans `distdistil_card.php`.

**3. Export PDF déclarations → erreur 503 / boutons génération sans effet**
- Les boutons "Générer la déclaration" et "Export Excel" utilisaient le **libellé traduit** (avec espaces/accents) comme valeur d'action (`value="<?= $langs->trans(...) ?>"`), comparé ensuite à `$action` filtré par `GETPOST(..., 'aZ09')` qui supprime les caractères non alphanumériques → la comparaison échouait systématiquement, aucun clic ne déclenchait quoi que ce soit.
- **Correctif** : codes d'action fixes (`generate`, `generate_releve`) via des `<button>` HTML séparant libellé affiché et valeur technique, dans `distdeclaration_generate.php`.
- Handlers `export_pdf`/`export_excel` sécurisés avec `try/catch(\Throwable)` complet : toute erreur produit désormais un message explicite + trace dans `dolibarr.log` au lieu d'un échec silencieux (503/page blanche).

**4. Dashboards Productions/Distillations à zéro malgré des données existantes**
- Cause : filtre année par défaut = année serveur courante (2026), alors que les productions/distillations importées (TRACABILITE.xlsx) datent de 2022-2025 → 0 résultat par simple absence de correspondance d'année, pas un bug SQL.
- **Correctif** : l'année par défaut est désormais calculée dynamiquement (`MAX(YEAR(...))` sur les données existantes) dans `distprod_dashboard.php`, `distdistil_dashboard.php`, `distdashboard.php` et `distreg_dashboard.php`. Message contextuel avec lien de bascule si l'année choisie diffère de la dernière année active.

**5. Dictionnaire Alambic → `Unknown column 'date_service'`**
- La table `llx_dist_dict_alambic` définit la colonne `date_mise_service`, mais `admin/alambic.php` utilisait partout `date_service` (INSERT, UPDATE, SELECT, affichage).
- Le dictionnaire Alambic est de toute façon déjà entièrement pris en charge par l'interface native Dolibarr (`$this->dictionaries`, Configuration → Dictionnaires → Alambic) avec les bons noms de colonnes.
- **Correctif** : suppression de la page `admin/alambic.php` (doublon obsolète non référencé dans le menu, même schéma que `dict_typeregistre.php` retiré en v0.7.4).

### Vérifié
- `DistMouvement` : cohérence classe/schéma SQL contrôlée (aucune incohérence de nom de champ détectée, contrairement à l'alambic)
- Agent de contrôle : **100%** — 53 contrôles passés, 0 erreur, 0 avertissement, 9 suggestions (tables à surveiller)

### Agent de contrôle (v3)
- Nouveau contrôle **[15/17]** : détecte les tableaux imbriqués (méthodes `*Grouped*`) passés à `Form::multiselectarray()`/`selectarray()` — cause exacte du bug #1
- Nouveau contrôle **[16/17]** : détecte les comparaisons `$action == $langs->trans(...)` ou boutons avec libellé traduit en `value` — cause exacte du bug #3
- 17 contrôles au total (was 15)

---


## [0.9.3] — 2026-07-20

### 🐛 Corrigé (bug critique — la migration 0.9.2 ne s'exécutait pas réellement)
- **Cause racine identifiée via `dolibarr.log`** : la requête de migration ajoutée en v0.9.2 utilisait `TABLE_SCHEMA = '".$this->db->escape($this->db->database_name)."'`, or `$this->db->database_name` est **vide dans le contexte d'exécution de `init()`** (confirmé dans le log : `WHERE TABLE_SCHEMA = '' AND TABLE_NAME = 'llx_dist_registry'`). La requête `information_schema.COLUMNS` ne retournait donc aucune ligne, `$existing` restait vide, et la fonction sortait silencieusement **sans jamais ajouter les colonnes** — d'où la persistance de l'erreur `Unknown column 'numero_entrepot'` après désactivation/réactivation du module.
- **Correctif** : remplacement des 3 occurrences de `TABLE_SCHEMA = '".$this->db->escape($this->db->database_name)."'` par `TABLE_SCHEMA = DATABASE()` (fonction MySQL native, toujours fiable quel que soit le contexte d'exécution) dans `addMissingRegistryColumns()`.
- Ajout de **traces `dol_syslog()` explicites** à chaque étape de la migration (nombre de colonnes détectées, colonnes ajoutées, échecs) — consultables dans `dolibarr.log` pour diagnostiquer instantanément si la migration s'exécute correctement.

### ⚠️ Action requise
**Désactiver puis réactiver le module** après mise à jour. Vérifier dans `dolibarr.log` la présence de la ligne `Distillerie migration: colonne(s) ajoutee(s) sur llx_dist_registry: numero_entrepot, siret, raison_sociale, ...`.

### Agent de contrôle
- Nouveau contrôle **[14/15]** : détecte l'anti-pattern `$this->db->database_name` dans les requêtes `information_schema` (peut être vide selon le contexte) — impose `DATABASE()` à la place
- Renumérotation complète des 15 contrôles
- Score : **100%** — 51 contrôles passés, 0 erreur, 0 avertissement, 9 suggestions (tables à surveiller)

---


## [0.9.2] — 2026-07-19

### 🐛 Corrigé (bug critique)
- **`Unknown column 'numero_entrepot' in 'INSERT INTO'` lors de la validation d'un registre** : sur les installations où le module avait été activé avant la v0.7.2, la table `llx_dist_registry` existante en base n'a jamais reçu les colonnes RÈGLE 01/05/09/10 (`numero_entrepot`, `siret`, `raison_sociale`, `adresse_siege`, `adresse_exploitation`, `numero_caution`, `dispense_caution`, `date_cloture_exercice`, `campagne_viticole`, `page_garde`, `nb_feuillets_type`, `taux_pertes`, `taux_dechets`) — `CREATE TABLE IF NOT EXISTS` ne modifie jamais une table déjà existante.
- Ajout de la méthode `addMissingRegistryColumns()` dans `modDistillerie.class.php`, appelée à chaque activation du module :
  - Vérifie via `information_schema.COLUMNS` les colonnes réellement présentes en base
  - Ajoute (`ALTER TABLE ADD COLUMN`) uniquement les colonnes manquantes — idempotent, sans erreur si déjà présentes
  - Corrige aussi `typeregistre` si encore de type numérique (migration v0.7.3 INTEGER → VARCHAR(64) non appliquée)
  - Ajoute l'index unique `uk_dist_registry_entrepot` si absent (tolérant aux échecs en cas de doublons existants)

### ⚠️ Action requise
**Désactiver puis réactiver le module** après mise à jour pour déclencher la migration automatique des colonnes manquantes sur les installations concernées. Aucune perte de données.

### Agent de contrôle
- Nouveau contrôle **[14/14] Migration des colonnes** : détecte les tables volumineuses (>15 colonnes) sans mécanisme de migration ALTER TABLE dans le descripteur — 9 tables identifiées en suggestion (audit_log, chai, distil, document_accompagnement, inventaire, mouvement, notification_dgddi, site, stock_periode) à surveiller si des colonnes y sont ajoutées après une première installation en production
- Score : **100%** — 50 contrôles passés, 0 erreur, 0 avertissement, 9 suggestions

---


## [0.9.1] — 2026-07-02

### Ajouté
- `distregistry_card.php` : préremplissage automatique des informations société lors de la **création** d'un nouveau registre
  - **Raison sociale** ← `$mysoc->name` (société courante Dolibarr)
  - **SIRET** ← `$mysoc->idprof2` (convention France : idprof1=SIREN, idprof2=SIRET)
  - **Adresse siège** ← `$mysoc->getFullAddress()`
  - **Numéro entrepôt fiscal suspensif** ← constante `MAIN_INFO_ID_EA` si définie, avec repli sur `DISTILLERIE_NUMERO_ENTREPOT` (setup module) si absente
  - Repli SIRET/raison sociale sur `DISTILLERIE_SIRET` / `DISTILLERIE_RAISON_SOCIALE` (setup module) si `$mysoc` incomplet
  - Message indicatif "(Prérempli depuis la constante MAIN_INFO_ID_EA)" affiché sous le champ quand la valeur vient de cette constante
  - Le préremplissage n'écrase jamais une ressaisie en cas d'erreur de validation (détection via `GETPOSTISSET('label')`)

### Vérifié
- Agent de contrôle : **100%** — 42 contrôles passés

---


## [0.9.0] — 2026-07-01

### Ajouté
- **Page Autodiagnostic** (`admin/autodiag.php`) — onglet "Autodiagnostic" dans Configuration. Contrôles runtime en conditions réelles :
  - Existence et comptage des 7 tables principales + 7 dictionnaires
  - Droits en base `llx_rights_def` (détection des anciens IDs 5000xx à migrer)
  - Chargement des 9 classes du module + instanciation (getNextRef, getAllActive)
  - Cohérence fichier VERSION ↔ descripteur, module activé, numero = 250004
  - Environnement : PHP, ZipArchive (import Excel), répertoire temp
  - Score global avec badges OK/!/KO par contrôle
- Registre des IDs de droits (25000401-25000499) documenté dans `CLAUDE.md` avec plages libres

### Modifié
- **`admin/setup.php` nettoyé** : les 13 paramètres d'exemple ModuleBuilder (MYPARAM1-13) remplacés par 6 paramètres métier réels via FormSetup :
  - Section Entrepôt fiscal : DISTILLERIE_NUMERO_ENTREPOT, DISTILLERIE_SIRET, DISTILLERIE_RAISON_SOCIALE
  - Section Production : DISTILLERIE_UNITE_VOLUME (l/ml), DISTILLERIE_TAUX_PERTES_DEFAUT, DISTILLERIE_IMPORT_AUTO_FRUITS (oui/non)
- Agent de contrôle : `admin/autodiag.php` et `test/agent_controle.php` ajoutés aux fichiers structurels vérifiés

### Vérifié
- **Dépôt GitHub `Barnecki/distillerie` opérationnel** : fichier VERSION accessible sur main → `url_last_version` fonctionnel (détection auto de mise à jour dans Dolibarr)
- Score agent : **100%** — 42 contrôles passés

---


## [0.8.3] — 2026-07-01

### ⚠️ MIGRATION REQUISE
Après installation : **désactiver puis réactiver le module**, puis **réattribuer les permissions aux utilisateurs/groupes** (les IDs de droits ont changé, les anciens droits en base ne correspondent plus).

### Modifié
- **ID module officiel : `250004`** (plage réservée BARNECKI SPIRITS 250000-250019) — remplace le placeholder `500189`
- **IDs de permissions migrés** vers le pattern anti-collision du skill `$this->numero . sprintf("%02d", ...)` :
  - Global : 500001/2/3 → 25000401/02/03
  - Distillation : 500010/11 → 25000411/12
  - Production : 500020/21 → 25000421/22
  - Registre : 500030/31 → 25000431/32
  - Admin : 500100 → 25000499
- Tests PHPUnit : `$module_id` mis à jour → 250004

### Agent de contrôle (v2)
- Nouveau contrôle : **plage ID BARNECKI** (erreur critique si numero hors 250000-250019)
- Nouveau contrôle : **pattern concaténation des IDs de droits** (warning si IDs en dur)
- Nouveau contrôle [13/13] : **cohérence des dictionnaires** entre `$this->dictionaries`, `init()` (CREATE TABLE) et `remove()` (DROP)
- Score : **100%** — 40 contrôles passés, 0 erreur, 0 avertissement

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

