# SQL sur les données Pokedex
Travaille Pratique pour les cours de SQL de l'UP Organisations des Données 2 du Défi Sociétaux Big Data à EMSE.

Fait pour : Bruno CARNEIRO CAMARA et Thales Vinicius DE LIMA UCHOAS.

## Prise en main

1.1 Vous allez écrire une requête `SELECT` qui permet de construire la table `pokemon_fr`, équivalent à la table `Pokémon` du cours.

```sql
create table pokemon_fr as
select pokemon.species_id, pokemon_species_names.name, pokemon.height, pokemon.weight
from pokemon_species_names 
left join pokemon on pokemon.species_id = pokemon_species_names.pokemon_species_id
where pokemon_species_names.local_language_id=5;
```

1.2 Vous allez maintenant construire la table `pokemon_types_fr` équivalent à `PokémonType` dans le cours. Les types de Pokémons sont donnés dans la table `pokemon_types` et leur noms dans plusieurs langues dans la table  `type_names`. Les types doivent être donnés en français.

```sql
create table pokemon_types_fr as
select pokemon_types.pokemon_id as species_id, type_names.name as type_name
from pokemon_types
left join type_names
on pokemon_types.type_id = type_names.type_id
where type_names.local_language_id = 5;
```

1.3 Définissez les tables `pokemon_fr` et `pokemon_types_fr` comme des vues (avec la commande [`CREATE VIEW`])

```sql
create view pokemon_fr as
select pokemon.species_id, pokemon_species_names.name, pokemon.height, pokemon.weight
from pokemon_species_names 
left join pokemon on pokemon.species_id = pokemon_species_names.pokemon_species_id
where pokemon_species_names.local_language_id=5;


create view pokemon_types_fr as
select pokemon_types.pokemon_id as species_id, type_names.name as type_name
from pokemon_types
left join type_names
on pokemon_types.type_id = type_names.type_id
where type_names.local_language_id = 5;
```

1.4 À partir des vues créées dans la question précédente, écrivez une requête pour lister les Pokémons de type feu (Q1, dans le cours). Ajoutez ensuite la contrainte que le poids des Pokémons doit être inférieur à 100kg (Q2, dans le cours). Ne listez que les [Pokémons de la première génération], allant du rang #1 à #151.

```sql
select pokemon_fr.name
from pokemon_types_fr
join pokemon_fr on pokemon_fr.species_id = pokemon_types_fr.species_id
where pokemon_types_fr.type_name = "Feu";

select pokemon_fr.name
from pokemon_types_fr
join pokemon_fr on pokemon_fr.species_id = pokemon_types_fr.species_id
where pokemon_types_fr.type_name = "Feu" and pokemon_fr.weight < 100  and pokemon_fr.species_id between 1 and 151;
```

## Normalisation

2.1 La vue `pokemon_fr` a-t-elle les dépendances fonctionnelles équivalentes ?

```sql
select species_id, name, count(distinct height)
from pokemon_fr
group by species_id, name
having count(distinct height) > 1;
```

2.2. Créez la vue `pokemon_fr_bis(id, species_id, name)` qui associe à chaque `pokemon.id` l'identifiant de son espèce et son nom en français.

* Quelles sont les dépendances fonctionnelles s'appliquant sur cette vue, selon vous ? Il y a au moins deux dépendances fonctionnelles atomiques élémentaires. Pour chaque dépendance fonctionnelle, prouvez par une requête qu'elle n'est pas violée dans la table.
* La vue que vous avez créée est-elle en forme normale ? Si oui, laquelle (1NF, 2NF, 3NF, BCNF) et pourquoi ?

```sql
create view pokemon_fr_bis as
select pokemon.id, pokemon.species_id, pokemon_species_names.name
from pokemon_species_names 
left join pokemon on pokemon.species_id = pokemon_species_names.pokemon_species_id
where pokemon_species_names.local_language_id=5;

-- vérification de la dépendance fonctionnelle id → species_id
select id, count(distinct species_id)
from pokemon_fr_bis
group by id
having count(distinct species_id) > 1;

-- vérification de la dépendance fonctionnelle id → name
select id, count(distinct name)
from pokemon_fr_bis
group by id
having count(distinct name) > 1;

-- forme normale de Boyce-Codd (BCNF) parce que chaque attribut dépend entièrement de la clé candidate de la vue (qui est {id}). Il n'y a pas de dépendances fonctionnelles partielles ou transitives, ce qui garantit l'absence de redondance et d'anomalies de mise à jour.
```

2.3 Créez la vue `location_areas_en(id, location_id, name)` qui associe à chaque lieu de l'univers Pokémon son nom anglais (les lieux ne sont pas documentés en français). La requête pour le faire est la suivante :

```sql
create view location_areas_en(id, location_id, name) as
select l.id, la.id, case when lap.name is null then ln.name else (ln.name || ' - ' || lap.name) end
from locations l, location_names ln, location_areas la, location_area_prose lap
where l.id = ln.location_id and
      l.id = la.location_id and
      la.id = lap.location_area_id and
	  ln.local_language_id = 9 and
	  (lap.local_language_id = 9 or lap.local_language_id is null);

```

* Expliquez en langage naturel quelle est la différence entre `id` et `location_id` (d'après les données du Pokédex) et comment l'attribut `name` est construit.
* Comme pour la question précédente, faites la liste des dépendances fonctionnelles s'appliquant sur cette vue, avec une requête pour le prouver. La vue est-elle en forme normale ? Si oui, laquelle et pourquoi ?

```sql
-- La différence entre id et location_id est que id est l'identifiant unique d'une zone de localisation dans l'univers Pokémon, tandis que location_id est l'identifiant de la localisation parente associée à la zone de localisation. L'attribut name est construit à partir d'une clause CASE dans la requête SQL, utilisant le nom de la localisation parente en langue anglaise si le nom de la zone de localisation en langue anglaise est nul, sinon les deux noms sont concaténés avec " - " pour former le nom final en anglais.


-- Dépendances fonctionnelles dans cette vue :

---- "id" dépend de "l.id";
---- "location_id" dépend de "l.id";
---- "name" dépend de "ln.name" et de "lap.name"

-- La vue n'est pas en forme normale en raison d'une colonne dérivée "name" dépendant de plusieurs colonnes dans différentes tables. Pour être en forme normale, la vue devrait être décomposée en plusieurs vues ou tables distinctes afin d'éviter les dépendances fonctionnelles complexes.
```

## Réécriture de requête

3.1 Écrivez une requête donnant l'étalement géographique des Pokémons sur leur territoire. La requête doit retourner le nom (ou le rang) du Pokémon, la région dans laquelle il vit et un pourcentage indiquant l'étalement du Pokémon dans la région, à savoir le rapport entre le nombre de lieux sur lesquels on trouve le Pokémon sur le nombre total de lieux dans la région. La principale table à utiliser pour cette requête est `encounters`, qui fait référence aux tables `pokemon` et `location_areas`. Lorsqu'un lieu a plusieurs sous-espaces, ne comptez le Pokémon qu'une seule fois. Cette dernière contrainte nécessite de faire une jointure entre `location_areas` et `locations`. Notez le temps d'exécution de la requête.

```sql
select pokemon_species_names.name as "name_pokemon", region_names.name as "name_region",
       count(distinct location_areas.location_id) as "nombre_de_lieux_pokemon_region",
       round((count(distinct location_areas.location_id) / count(distinct locations.id)) * 100, 2) as "etalement_pokemon"
from encounters
join pokemon on encounters.pokemon_id = pokemon.id
join pokemon_species_names on pokemon.species_id = pokemon_species_names.pokemon_species_id
join location_areas on encounters.location_area_id = location_areas.id
join locations on location_areas.location_id = locations.id
join regions on locations.region_id = regions.id
join region_names on regions.id = region_names.region_id
where pokemon_species_names.local_language_id = 5
group by pokemon_species_names.name, region_names.name
order by region_names.name, pokemon_species_names.name;

-- temps d'exécution = 422 ms
```

3.2 Proposez une écriture alternative de cette requête basée sur une jointure _explicite_. Pour cela, utilisez l'opérateur [`CROSS JOIN`]

```sql
select pokemon_species_names.name as "name_pokemon", region_names.name as "name_region",
       count(distinct location_areas.location_id) as "nombre_de_lieux_pokemon_region",
       round((count(distinct location_areas.location_id) / count(distinct locations.id)) * 100, 2) as "etalement_pokemon"
from encounters
cross join locations
cross join location_areas
cross join pokemon
join pokemon_species_names on pokemon.species_id = pokemon_species_names.pokemon_species_id
join regions on locations.region_id = regions.id
join region_names on regions.id = region_names.region_id
where encounters.pokemon_id = pokemon.id
and encounters.location_area_id = location_areas.id
and location_areas.location_id = locations.id
and pokemon_species_names.local_language_id = 5
group by pokemon_species_names.name, region_names.name
order by region_names.name, pokemon_species_names.name;

-- temps d'exécution = 4278 ms
-- Cela s'explique par le fait que SQLite n'optimise pas les jointures CROSS JOIN et les exécute dans l'ordre indiqué dans la clause FROM.
```

3.3. Écrivez une requête associant une liste d'attaques aux Pokémons sur lesquels ces attaques n'ont pas d'effet. Par exemple, les attaques de type électrique n'ont pas d'effet sur les Pokémons de type sol. Dans le Pokédex, cela se traduit par le tuple `(13, 5, 0)` dans la table `type_efficacy`. Les attributs de cette table sont le type d'attaque (`damage_type_id`), le type de Pokémon (`target_type_id`) et un pourcentage donnant l'efficacité de l'attaque (`damage_factor`). Vous devez donc faire une jointure avec les tables `moves` (sur l'attribut `damage_type_id`) et `pokemon_types` (sur l'attribut `target_type_id`). Restreignez votre requête aux Pokémons de première génération (rang #1 à #151).

```sql
select distinct move_names.name as "name_attaque", pokemon_species_names.name as "name_pokemon"
from moves
join move_names on moves.id = move_names.move_id
join type_efficacy on moves.type_id = type_efficacy.damage_type_id
join pokemon_types on type_efficacy.target_type_id = pokemon_types.type_id
join pokemon_species on pokemon_types.pokemon_id = pokemon_species.id
join pokemon_species_names on pokemon_species.id = pokemon_species_names.pokemon_species_id
where type_efficacy.damage_factor = 0
and pokemon_species.generation_id = 1
order by move_names.name, pokemon_species_names.name;
```

3.4. Proposez au moins trois écritures alternatives de cette requête, de façon à obtenir des temps temps d'exécution distincts. Proposez une procédure pour classer ces réécritures de la plus rapide à la moins rapide, sans avoir à les exécuter.

```sql
-- 1
select distinct move_names.name as "name_attaque", pokemon_species_names.name as "name_pokemon"
from (select * from moves where moves.type_id in (select damage_type_id from type_efficacy where damage_factor = 0)) as moves
join move_names on moves.id = move_names.move_id
join pokemon_types on moves.type_id = pokemon_types.type_id
join pokemon_species on pokemon_types.pokemon_id = pokemon_species.id
join pokemon_species_names on pokemon_species.id = pokemon_species_names.pokemon_species_id
where pokemon_species.generation_id = 1
order by move_names.name, pokemon_species_names.name;

-- 2
select distinct move_names.name as "name_attaque", pokemon_species_names.name as "name_pokemon"
from moves
join move_names on moves.id = move_names.move_id
left join type_efficacy on moves.type_id = type_efficacy.damage_type_id and type_efficacy.damage_factor = 0
join pokemon_types on moves.type_id = pokemon_types.type_id
join pokemon_species on pokemon_types.pokemon_id = pokemon_species.id
join pokemon_species_names on pokemon_species.id = pokemon_species_names.pokemon_species_id
where pokemon_species.generation_id = 1
and type_efficacy.damage_type_id is not null
order by move_names.name, pokemon_species_names.name;

-- 3
select distinct move_names.name as "name_attaque", pokemon_species_names.name as "name_pokemon"
from moves
join move_names on moves.id = move_names.move_id
join type_efficacy on moves.type_id = type_efficacy.damage_type_id and type_efficacy.damage_factor = 0
join pokemon_types on moves.type_id = pokemon_types.type_id
join pokemon_species on pokemon_types.pokemon_id = pokemon_species.id
join pokemon_species_names on pokemon_species.id = pokemon_species_names.pokemon_species_id
where pokemon_species.generation_id = 1
and type_efficacy.damage_type_id = (select damage_type_id from type_efficacy where damage_factor = 0 limit 1)
order by move_names.name, pokemon_species_names.name;
```

3.5 Expliquez ce que signifient les valeurs pour l'attribut `stat`. Donnez en exemple comment son contenu a pu être utilisé dans l'une des deux requêtes précédentes (étalement géographique et attaques sans effet).

```sql
-- L'attribut stat de SQLite est interprété de la manière suivante : Le Bit 0 (valeur 1) indique si les statistiques sont valides, le Bit 1 (valeur 2) indique si les données sont ordonnées en croissant selon l'index, et le Bit 2 (valeur 4) indique si les données sont ordonnées en décroissant selon l'index. Ces données statistiques sont utilisées par l'optimiseur de requêtes de SQLite pour sélectionner les plans d'exécution les plus efficaces en fonction de la distribution des données dans la table. Par exemple, lors d'une requête de diffusion géographique, les statistiques dans sqlite_stat1 peuvent indiquer si les données dans les colonnes de latitude et de longitude sont ordonnées, ce qui peut aider l'optimiseur à choisir un plan d'exécution efficace en utilisant un index sur ces colonnes pour effectuer la jointure spatiale.
```

## Indexage

4.1 Enlevez toutes les contraintes de clé primaire et de clé étrangère des tables `pokemon`, `pokemon_species` et `pokemon_types`. Une fois les contraintes supprimées, refaites la requête de la question 1.4 (Q1). Que constatez-vous ? Pourquoi ?

```sql
select psn.name as "name_pokemon", p.weight, psn_espece.name as "name_species", tn.name as "type"
from pokemon_copia p
join pokemon_species_names psn on p.species_id = psn.pokemon_species_id
join pokemon_species_names psn_espece on p.species_id = psn_espece.pokemon_species_id
join pokemon_types_copia pt on p.id = pt.pokemon_id
join types t on pt.type_id = t.id
join type_names tn on t.id = tn.type_id
where p.weight < 100 -- poids inférieur à 100 kg
and psn.local_language_id = 5 -- langue française
and psn.pokemon_species_id between 1 and 151 -- rang #1 à #151
and t.id in (
  select type_id
  from type_names
  where name = 'fire' and local_language_id = 5
);

-- Après avoir supprimé les contraintes de clé primaire et de clé étrangère des tables "pokemon", "pokemon_species" et "pokemon_types" dans la base de données Pokédex, la requête que vous avez fournie peut retourner des résultats incorrects ou incohérents. Cela est dû au fait que les données dans ces tables peuvent être dupliquées, incohérentes ou invalides, ce qui peut causer des incohérences dans les résultats des requêtes. Dans ce cas, nous avons obtenu 0 résultats, au lieu de 2.
```

4.2. De la même manière, supprimez les contraintes de clé primaire et de clé étrangère sur les tables `encounters`, `location_areas` et `locations` puis re-faites la requête écrite dans la question 3.1 (sur l'étalement géographique des Pokémons). Testez avec jointure explicite et avec jointure implicite. Constatez-vous une différence ? Pourquoi ?

```sql
select pokemon_species_names.name as "name_pokemon", region_names.name as "name_region",
       count(distinct location_areas.location_id) as "nombre_de_lieux_pokemon_region",
       round((count(distinct location_areas.location_id) / count(distinct locations.id)) * 100, 2) as "etalement_pokemon"
from encounters
join pokemon on encounters.pokemon_id = pokemon.id
join pokemon_species_names on pokemon.species_id = pokemon_species_names.pokemon_species_id
join location_areas on encounters.location_area_id = location_areas.id
join locations on location_areas.location_id = locations.id
join regions on locations.region_id = regions.id
join region_names on regions.id = region_names.region_id
where pokemon_species_names.local_language_id = 5
group by pokemon_species_names.name, region_names.name
order by region_names.name, pokemon_species_names.name;

-- Le temps d'exécution a augmenté (922 ms), mais les résultats n'ont pas changé

select pokemon_species_names.name as "name_pokemon", region_names.name as "name_region",
       count(distinct location_areas.location_id) as "nombre_de_lieux_pokemon_region",
       round((count(distinct location_areas.location_id) / count(distinct locations.id)) * 100, 2) as "etalement_pokemon"
from encounters, pokemon, pokemon_species_names, location_areas, locations, regions, region_names
where encounters.pokemon_id = pokemon.id
and pokemon.species_id = pokemon_species_names.pokemon_species_id
and encounters.location_area_id = location_areas.id
and location_areas.location_id = locations.id
and locations.region_id = regions.id
and regions.id = region_names.region_id
and pokemon_species_names.local_language_id = 5
group by pokemon_species_names.name, region_names.name
order by region_names.name, pokemon_species_names.name;

-- Le temps d'exécution a baissé (534 ms), mais les résultats n'ont pas changé
```

4.3. Écrivez une requête qui liste les Pokémons dont l'évolution commence par les mêmes lettres (1, 2 et 3 lettres). Utilisez l'opérateur [`LIKE`] et la fonction [`substr`] dans votre filtre de sélection. Notez son temps d'exécution.

```sql
select pokemon_species_names.name
from pokemon_species
join pokemon_species_names on pokemon_species.id = pokemon_species_names.pokemon_species_id
where substr(pokemon_species_names.name, 1, 1) = substr(pokemon_species_names.name, 2, 1)
   or substr(pokemon_species_names.name, 1, 2) = substr(pokemon_species_names.name, 3, 2)
   or substr(pokemon_species_names.name, 1, 3) = substr(pokemon_species_names.name, 4, 3);
```

4.4. Retirez l'index `ix_pokemon_species_names_name` avec [`DROP INDEX`] et reproduisez la requête précédente. Constatez-vous une différence en termes de temps d'exécution ? Pourquoi ?

```sql
-- Le temps d'exécution a augmenté (18 ms -> 34 ms), parce que la recherche dans la colonne name de la table pokemon_species_names peut être moins efficace sans l'index.
```
