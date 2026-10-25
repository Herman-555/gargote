# Réponses — niveau 0 : prise en main

## 0.1 Lancer l'application

Les serveurs sont :

```
API CodeIgniter : http://localhost:8080
Interface Vue   : http://localhost:5173
```

- [x] <http://localhost:8080/api/plats> affiche du JSON ;
- [x] <http://localhost:5173> affiche l’ardoise avec les plats ;
- [x] J'ai passé une commande depuis l’onglet **Prendre une commande** et la retrouve dans l’onglet **Commandes**.

## 0.2 Lire le schéma relationnel

### 1. Schéma relationnel

- categorie (<u>id</u>, libelle)
- plat (<u>id</u>, nom, prix, #id_categorie)
- commande (<u>id</u>, date_commande, numero_table)
- commande_ligne (<u>id</u>, #id_commande, #id_plat, quantite, prix_unitaire)

### 2. Pourquoi `commande_ligne` contient-elle encore une colonne `prix_unitaire`?

`plat.prix` représente le prix actuel sur la carte, tandis que
`commande_ligne.prix_unitaire` conserve le prix appliqué au moment de la
commande.

Exemple :

1. Le plat `Romazava` coûte 7 000 Ar aujourd'hui.
2. La ligne est enregistrée avec `prix_unitaire = 7000`.
3. Le prix du plat passe ensuite à 8 000 Ar dans `plat.prix`.

L'ancienne commande doit toujours etre a 7 000 Ar par portion. Sans
copie du prix dans `commande_ligne`, l'historique changerait quand le prix de
la carte changerait.

### 3. Suppression d'une commande

Les lignes liées à la commande sont supprimées automatiquement. L'instruction
responsable est :

```sql
id_commande INTEGER NOT NULL REFERENCES commande(id) ON DELETE CASCADE
```

`ON DELETE CASCADE` dit que si cette reference de sa table est supprimee, les ligne qui ont le meme id fk seront supprimees aussi.
`commande_ligne.id_commande`.

### 4. Trois données absurdes actuellement acceptées

Le schéma ne possède pas encore toutes les contraintes métier nécessaires. Il
accepte notamment :

1. un prix négatif, par exemple `prix = -500` ;
2. une quantité nulle ou négative dans `commande_ligne`, par exemple
   `quantite = 0` ou `quantite = -2` ;
3. un numéro de table nul ou négatif, par exemple `numero_table = 0` ou
   `numero_table = -3`.

## 0.3 Interroger l'API à la main

### 5. Pourquoi la requête b retourne 201 et non 200 ?

La requête b est une creation de ressource. Le contrôleur appelle
`ApiController::respondCreated()`, qui renvoie le code HTTP `201 Created`.

Le code `200 OK` convient à une lecture ou une modification reussie,
`201 Created` convient a la creation.

### 6. Qui bloque la suppression du plat ?

C'est PostgreSQL qui bloque la suppression, car `commande_ligne.id_plat`
référence `plat.id` sans `ON DELETE CASCADE`

Dans `backend/app/Controllers/Api/Plats.php`, la conversion de cette erreur en
réponse HTTP 409 se fait dans `delete()`

### 7. Créer la commande de la table 6

```bash
curl -i -X POST "http://localhost:8080/api/commandes" \
  -H "Content-Type: application/json" \
  -d '{"numero_table":6,"lignes":[{"id_plat":4,"quantite":2},{"id_plat":10,"quantite":1}]}'
```

## 0.4 Suivre une requête de bout en bout

### Chemin de création d'un plat

| #   | Étape                                   | Fichier                                                        | Fonction / ligne                              |
| --- | --------------------------------------- | -------------------------------------------------------------- | --------------------------------------------- |
| 1   | Le formulaire est soumis                | `frontend/src/components/PlatForm.vue`                         | `enregistrer()`                               |
| 2   | La requête HTTP est construite          | `frontend/src/api.js`                                          | `requete()`                                   |
| 3   | Vite transmet `/api/...` au port 8080   | `frontend/vite.config.js`                                      | `server.proxy['/api']`                        |
| 4   | CodeIgniter choisit le contrôleur       | `backend/app/Config/Routes.php`                                | route `POST plats` vers `Plats::create`       |
| 5   | Le contrôleur lit le JSON               | `backend/app/Controllers/Api/Plats.php` et `ApiController.php` | `Plats::create()` appelle `donneesJson()`     |
| 6   | Les règles de validation sont vérifiées | `backend/app/Models/PlatModel.php`                             | `PlatModel::$validationRules` avec `insert()` |
| 7   | La ligne est insérée                    | `backend/app/Controllers/Api/Plats.php`                        | `$this->plats->insert($donnees)`, `create()`  |
| 8   | Le formulaire prévient l'écran parent   | `frontend/src/components/PlatForm.vue`                         | `emit('enregistre', resultat)`                |
| 9   | L'écran recharge la liste               | `frontend/src/views/CarteView.vue`                             | `surEnregistrement()` , `charger()`           |

### 8. Où se trouve le code métier d'une commande ?

Il se trouve dans `backend/app/Services/CommandeService.php`, dans la méthode
`creer()`.

Le code n'est pas dans `Commandes.php` car le controleur prend la requete et appelle une service et retourne une reponse. Le service contient les regles metier reutilisable.

## 0.5 Repérer les notions de Vue

| Notion                 | À quoi ça sert                                                | Votre exemple                                                                                                     |
| ---------------------- | ------------------------------------------------------------- | ----------------------------------------------------------------------------------------------------------------- |
| `ref()`                | Une variable dont le changement met l'écran à jour            | `frontend/src/views/CarteView.vue` : `const plats = ref([])` rend la liste reactive par rapport a une mise a jour |
| `computed()`           | Une valeur calculée automatiquement à partir d'autres valeurs | `frontend/src/components/Panier.vue` : `total` recalcule le montant quand une quantite ou plat change.            |
| `v-for`                | Répéter un élément pour chaque élément d'une liste            | `frontend/src/components/PlatList.vue` : `v-for="plat in liste"` affiche chaque plat d'une liste.                 |
| `v-if` / `v-else`      | Afficher conditionnellement un élément                        | `frontend/src/components/Panier.vue` : le message de panier vide avec `v-if`, sinon afficher liste avec `v-else`. |
| `v-model`              | Synchroniser un champ et une variable dans les deux sens      | `frontend/src/components/PlatForm.vue` : `v-model.trim="form.nom"` synchronise le champ avec `form.nom`.          |
| `defineProps`          | Recevoir des données du composant parent                      | `frontend/src/components/PlatForm.vue` reçoit `plat` et `categories` avec `defineProps`.                          |
| `defineEmits` / `emit` | Envoyer un événement au composant parent                      | `frontend/src/components/PlatForm.vue` déclare `enregistre` puis appelle `emit('enregistre', resultat)`.          |
| `onMounted`            | Exécuter du code après l'apparition du composant              | `frontend/src/views/CarteView.vue` appelle `onMounted(charger)` pour charger les plats et categories.             |

### 9. Pourquoi `Panier.vue` émet-il `plus` et `moins` ?

`Panier.vue` est un composant qui reçoit `lignes` en prop et
signale seulement l'action avec les emits `plus` et `moins` pour ne pas dupliquer un etat du panier dans deux composants.

Le composant parent `frontend/src/views/CommandeView.vue` modifie réellement la
quantite.

### 10. `watch` dans `PlatForm.vue`

Dans `frontend/src/components/PlatForm.vue`, `watch` observe
`props.plat`. Quand le parent sélectionne un autre plat, il recopie ses valeurs
dans `form.value` et réinitialise les erreurs et le message.

Sans lui, le formulaire conserverait les anciennes valeurs
des champs quand on clique **Modifier** pour un autre plat.

## 0.6 Premières modifications

### a) Nom de la gargote

- Titre a modifier dans le fichier App.vue.

### b) Ajouter la catégorie « Goûter »

### 11. Pourquoi la catégorie apparaît-elle sans modifier le code ?

Le formulaire recupere les catégories avec `api.categories.lister()`, qui
appelle la route `GET /api/categories` de l'api et `charger()` l'appelle.

---

## Validation du niveau 0

- [x] L’application tourne (API + interface).
- [x] `reponses-niveau-0.md` répond aux 11 questions et contient le tableau 0.4 complet.
- [x] Le nom de la gargote est modifié ; _Goûter_ existe en base.
- [x] Un commit `niveau 0 : prise en main`.

```

```
