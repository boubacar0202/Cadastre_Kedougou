# Cadastre Kédougou — règles pour les agents

Lire `README.md` pour l’installation. Tout code nouveau suit la chaîne Laravel ci-dessous. En modifiant un contrôleur existant, extraire le flux touché vers cette chaîne au lieu d’ajouter de la logique dans le contrôleur.

```
Form Request → Contrôleur → Service → Repository → Modèle
```

## Form Request

Créer `Store{Modele}Request` et `Update{Modele}Request` dans `app/Http/Requests/{Domaine}` :

```bash
php artisan make:request Dossier/StoreDossierRequest
php artisan make:request Dossier/UpdateDossierRequest
```

- `authorize()` refuse sans le rôle attendu (`admin`, ou l’agent propriétaire).
- `rules()` est la seule validation. Le contrôleur consomme `$request->validated()`, jamais `$request->all()`.
- Les règles communes aux deux classes vivent dans une classe abstraite ou un trait du même dossier. `Update*Request` ajoute `Rule::unique()->ignore()` et `sometimes`.
- Messages et attributs en français (`messages()`, `attributes()`).

Le contrôleur tient en quelques lignes : appeler le service, rediriger avec `success` ou `error`, ou renvoyer la page Inertia. Pas de `validate()`, pas de `DB::transaction()`, pas de requête.

## Service

Un fichier `app/Services/{CasUsage}Service.php` par cas d’usage (`DossierService`, `AttributionDossierService`).

- Dépendances reçues par le constructeur (repository, autre service).
- Écriture multi-tables dans `DB::transaction()`.
- Règles métier ici : doublon de numéro, NICAD, délai d’attribution, suppression du PDF avec l’enregistrement, compte `User::estProtege()`.
- Le service ignore `Request` et la session. Il reçoit un tableau validé et des modèles.

## Repository

Un fichier `app/Repositories/{Modele}Repository.php` par modèle agrégat. Il est le seul endroit qui interroge ce modèle pour ce cas d’usage.

Prévenir les requêtes N+1 comme dans la documentation Eloquent :

- `with([...])` pour chaque relation affichée, y compris les relations imbriquées (`terrain.commune`).
- `withCount()` pour un nombre. `loadMissing()` si le modèle est déjà chargé.
- Aucune requête dans un `foreach`, un accesseur, ou le partage Inertia ligne à ligne.
- Colonnes explicites sur les grandes listes (`select`) quand la table porte des fichiers ou du texte long.
- Activer en local dans `AppServiceProvider::boot()` :

```php
Model::preventLazyLoading(! app()->isProduction());
```

Le repository ne contient pas de règle métier et ne renvoie pas de redirection. Le modèle garde les relations, les casts, `$fillable` et les scopes d’une seule contrainte (`scopeEnCours`).

Pas d’interface ni de binding tant qu’il n’existe qu’une implémentation.

## DRY

- Étendre le service, le repository ou la Form Request existants avant d’en créer un deuxième pour le même modèle.
- Réutiliser les composants Vue de `resources/js/Components`.
- Conserver le dossier `resources/js/Pages/traitementDissier` (les pages Inertia y pointent).
- Garder les noms de routes Ziggy. Mettre à jour l’appel Vue dans le même changement.
- Une route admin passe par `middleware('role:admin')`.

## Métier à préserver

- Rôles stockés : `admin`, `user`.
- Identifiants métier (NICAD, numéro de dossier, référence cadastrale) : vérifier avant écriture, ne pas les recalculer en silence.
- Mots de passe et codes d’accès hashés, jamais renvoyés au front ni écrits dans les logs.
- Flash Inertia : `success`, `error`, `warning`, `info`.

## Vérifier

```bash
vendor/bin/pint --dirty
php artisan test
```

Ne pas lancer `migrate:fresh` sur la base de travail. Ne pas committer `.env` ni les PDF de `storage` et `public/storage`.
