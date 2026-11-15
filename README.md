# Cadastre Kédougou

Application de gestion cadastrale du service de Kédougou : dossiers, secrétariat, géomètre, mutations, courrier (arrivées et départs), plan (shapefiles) et évaluation des agents.

Interface en français. Les rôles sont `admin` et `user` (`users.role`).

## Stack

- PHP 8.2, Laravel 11, Eloquent, Pest, Laravel Pint
- Inertia.js, Vue 3, Tailwind CSS, Vite
- MySQL (XAMPP), sessions en base
- Carte : Leaflet, Turf, proj4, `gasparesganga/php-shapefile`

## Prérequis

- PHP 8.2+ avec les extensions Laravel habituelles (`mbstring`, `openssl`, `pdo_mysql`, `tokenizer`, `xml`, `ctype`, `json`, `fileinfo`)
- Composer, Node.js 20+
- MySQL via XAMPP, base `cadastre_kedougou`

## Installation

Depuis la racine du projet (`htdocs/Cadastre_Kedougou`) :

```bash
composer install
cp .env.example .env
php artisan key:generate
```

Renseigner `.env` :

```
DB_CONNECTION=mysql
DB_HOST=127.0.0.1
DB_PORT=3306
DB_DATABASE=cadastre_kedougou
DB_USERNAME=root
DB_PASSWORD=
```

Puis :

```bash
php artisan migrate
php artisan storage:link
npm install
npm run dev
```

L’application est servie par Apache : `http://localhost/Cadastre_Kedougou/public`.

`npm run build` produit les assets de production. `vendor/bin/pint --dirty` formate le PHP touché. `php artisan test` lance Pest.

## Architecture

Chaque cas d’usage suit une seule chaîne. Le contrôleur orchestre. Il ne valide pas, n’applique pas de règle métier et n’écrit pas de requête.

```
Requête HTTP
  → Form Request (Store*Request ou Update*Request)
  → Contrôleur
  → Service (règles métier, transaction)
  → Repository (requêtes Eloquent et eager loading)
  → Modèle
```

| Couche | Dossier | Rôle |
| --- | --- | --- |
| Form Request | `app/Http/Requests/{Domaine}` | `authorize()`, `rules()`, messages français |
| Contrôleur | `app/Http/Controllers` | Appeler le service avec `$request->validated()`, répondre (redirect Inertia ou JSON) |
| Service | `app/Services` | Cas d’usage, `DB::transaction()`, règles (NICAD, doublon, délai, rôle) |
| Repository | `app/Repositories` | Lectures et écritures Eloquent, `with()` / `withCount()` |
| Modèle | `app/Models` | Relations, casts, `$fillable`, scopes locaux |
| Page Vue | `resources/js/Pages` | Affichage. Le nom du fichier est le chemin Inertia |

Les pages de traitement des dossiers restent dans `resources/js/Pages/traitementDissier`.

Eloquent reste l’accès aux données. Un repository ne réécrit pas le query builder : il regroupe les requêtes d’un modèle et les relations à charger, pour qu’un écran ne relance pas une requête par ligne.

### Form Request

Deux classes par ressource qui crée et met à jour, comme dans la [documentation validation](https://laravel.com/docs/11.x/validation#form-request-validation) :

```bash
php artisan make:request Dossier/StoreDossierRequest
php artisan make:request Dossier/UpdateDossierRequest
```

```php
public function store(StoreDossierRequest $request, DossierService $dossiers)
{
    $dossiers->creer($request->validated());

    return redirect()
        ->route('secretariat.create')
        ->with('success', 'Dossier enregistré.');
}

public function update(UpdateDossierRequest $request, Dossier $dossier, DossierService $dossiers)
{
    $dossiers->mettreAJour($dossier, $request->validated());

    return back()->with('success', 'Dossier mis à jour.');
}
```

Le contrôleur reçoit uniquement le tableau validé. `authorize()` porte le droit (`admin` ou propriétaire du dossier). Les messages d’erreur sont en français.

### Service

Un service = un cas d’usage du métier, déjà illustré par `app/Services/AttributionDossierService.php`.

```php
class DossierService
{
    public function __construct(private DossierRepository $dossiers) {}

    public function creer(array $donnees): Dossier
    {
        return DB::transaction(function () use ($donnees) {
            $this->empecherDoublon($donnees['txt_num_dossier']);

            return $this->dossiers->creer($donnees);
        });
    }
}
```

La transaction et les règles (numéro déjà pris, NICAD, compte protégé, pièce PDF à supprimer avec l’enregistrement) vivent ici. Le service appelle le repository. Il ne lit pas `$request`.

### Repository et requêtes N+1

La [documentation Eloquent](https://laravel.com/docs/11.x/eloquent-relationships#eager-loading) charge les relations avec `with()` avant la boucle. Le repository est l’endroit unique de ce chargement.

```php
class DossierRepository
{
    public function paginer(): LengthAwarePaginator
    {
        return Dossier::query()
            ->with([
                'terrain.commune',
                'titulaire',
            ])
            ->withCount('mutations')
            ->latest('id')
            ->paginate(25);
    }

    public function trouver(int $id): Dossier
    {
        return Dossier::query()
            ->with(['terrain.commune', 'titulaire', 'mutations'])
            ->findOrFail($id);
    }
}
```

Règles :

- Toute relation affichée dans une liste est dans le `with()` de la méthode de liste.
- Un total affiché passe par `withCount()`, pas par `$dossier->mutations->count()` dans une boucle.
- Une relation ajoutée après coup utilise `loadMissing()`.
- Aucune requête Eloquent dans un `foreach`, un accesseur ou une prop Inertia calculée ligne à ligne.
- En local, `Model::preventLazyLoading()` dans `AppServiceProvider` fait échouer la requête paresseuse au lieu de la masquer.

### Éviter la duplication

- `Store*Request` et `Update*Request` partagent leurs règles dans une classe abstraite ou un trait du même dossier. La mise à jour ajoute `Rule::unique(...)->ignore($modele->getKey())` et `sometimes` pour les champs absents.
- Une règle métier n’existe que dans le service. Une requête n’existe que dans le repository (ou un scope du modèle si elle tient en une contrainte réutilisable).
- Le front réutilise `resources/js/Components` (`BaseInput`, `BaseSelect`, boutons, `DeleteModal`) avant d’en créer un nouveau.
- Une route nommée existante est étendue. `routes/web.php` ne gagne pas de doublon.
- Les fichiers PDF restent sur le disque `public`. La suppression de l’enregistrement supprime le fichier dans le même service.

## Accès

Les routes authentifiées sont dans le groupe `auth` de `routes/web.php`. L’administration (paramètres, utilisateurs, barèmes, attribution) est derrière `middleware('role:admin')`. Un autre rôle revient au tableau de bord.

Le compte protégé (`User::estProtege()`) ne se modifie pas et ne se supprime pas depuis l’administration.

## Données sensibles

`.env`, mots de passe, codes d’accès hashés et PDF réels de dossiers (`storage`, `public/storage`) restent hors du dépôt. Les réponses Inertia n’exposent pas `password` ni le code d’accès.
