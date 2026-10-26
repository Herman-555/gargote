# Réponses — Niveau 1

## 1.1 Script de migration

### 1. Quelle cle primaire pour `plat_du_jour` ?

La colonne `date_jour` est choisie comme clé primaire.
Cela garantit qu’une date ne peut apparaître qu’une seule fois dans la table et a comme cle etrangere `id_plat` pour avoir un plat du jour au plus une fois par jour.

### 2. Que perdrait-on avec `est_plat_du_jour BOOLEAN` dans `plat` ?

1. On ne pourrait pas conserver l’historique des plats du jour.

2. Plusieurs plats pourraient avoir `est_plat_du_jour = TRUE` en même temps, et on ne pourrait pas associer le plat à une date.

### Vérification exécution

Le script de migration n’est pas ré-exécutable car les colonnes, contraintes et la table existent déjà.

C'est normal pour une migration a exécuter une seule fois sur un schéma donné.
