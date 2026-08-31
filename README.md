# Backend de réservation de salles

## 1. Contexte

Une université souhaite disposer d'une API REST permettant de gérer des salles et leurs réservations.

Un utilisateur peut consulter les salles disponibles pour une période, réserver une salle précise ou laisser l'application sélectionner automatiquement la salle la plus adaptée à son besoin.

Le contrat détaillé de l'API, incluant les endpoints, les types attendus et les erreurs possibles, est défini dans [`openapi.yml`](openapi.yml).

## 2. Fonctionnalités attendues

L'application doit permettre :

- de créer, consulter et modifier des bâtiments ;
- de créer, consulter et modifier des salles ;
- de définir les équipements disponibles dans chaque salle ;
- de créer et consulter des organisateurs ;
- de rechercher les salles disponibles sur une période ;
- de réserver une salle précise ;
- de demander l'attribution automatique d'une salle ;
- d'annuler une réservation ;
- de consulter les réservations, avec des filtres par salle, organisateur ou période ;
- de placer temporairement une salle en maintenance.

## 3. Données manipulées

Les structures JSON échangées avec le front (bâtiments, salles, équipements,
organisateurs, réservations et erreurs) sont définies dans la section
`components.schemas` du fichier [`openapi.yml`](openapi.yml). Le README ne fixe pas
la forme des objets internes qui ne font pas partie du contrat HTTP.

Les relations et contraintes utiles au métier sont les suivantes :

- une salle et un organisateur sont localisés dans un bâtiment et à un étage ;
- le rez-de-chaussée porte le numéro `0` et, pour un bâtiment de `n` étages, un
  étage valide est compris entre `0` et `n - 1` ;
- la localisation de l'organisateur sert à calculer la proximité des salles lors
  d'une attribution automatique ;
- une salle en maintenance ne peut ni être proposée ni être réservée ;
- une réservation annulée reste consultable, mais ne bloque plus la salle.

## 4. Règles métier

### Période de réservation

- Le début doit être strictement antérieur à la fin.
- Le début ne doit pas être dans le passé.
- La durée ne peut pas dépasser huit heures.
- Les dates et heures sont transmises au format ISO 8601 avec un fuseau ou un décalage UTC, par exemple `2026-10-15T14:00:00+02:00`.

### Capacité et équipements

Le nombre de participants doit être strictement positif et inférieur ou égal à la capacité de la salle. Une salle de 30 places peut donc accueillir exactement 30 personnes.

La salle doit posséder tous les équipements demandés. La présence d'équipements supplémentaires est autorisée. Une liste absente ou vide signifie qu'aucun équipement particulier n'est exigé.

### Chevauchement

Deux réservations confirmées ne peuvent pas se chevaucher dans la même salle. Il y a chevauchement lorsque :

```text
reservationExistante.start < nouvelleReservation.end
ET
reservationExistante.end > nouvelleReservation.start
```

Deux réservations consécutives, par exemple `10:00–11:00` et `11:00–12:00`, sont autorisées. Les réservations annulées sont ignorées lors de cette vérification.

## 5. Choix d'une salle

### Salle choisie par l'utilisateur

Une salle explicitement choisie est retenue uniquement si elle existe, si son statut est `AVAILABLE`, si sa capacité est suffisante, si elle possède tous les équipements demandés et si aucune réservation confirmée ne chevauche la période demandée.

La demande est refusée dès qu'une condition n'est pas satisfaite. Le système ne remplace jamais silencieusement la salle choisie par une autre.

### Attribution automatique

Le système commence par ne conserver que les salles qui respectent toutes les conditions suivantes :

- statut `AVAILABLE` ;
- capacité suffisante ;
- présence de tous les équipements demandés ;
- absence de réservation confirmée en conflit avec la période.

Un score est calculé pour chaque salle compatible :

- `placesInutilisées = capacité - nombre de participants` ;
- lorsque les identifiants des bâtiments sont identiques, `distance = abs(étageSalle - étageOrganisateur)` ;
- dans des bâtiments différents, `distance = 10 + abs(étageSalle - étageOrganisateur)` ;
- `score = distance × 10 + placesInutilisées`.

Le score le plus faible est le meilleur. Cette pondération fait équivaloir un étage de distance à dix places inutilisées et ajoute une pénalité de 100 points lors d'un changement de bâtiment.

En cas d'égalité, les salles sont départagées par leur nom dans l'ordre alphabétique, sans tenir compte de la casse, puis par leur identifiant. Le résultat est ainsi déterministe.

Si aucune salle n'est compatible, aucune réservation n'est créée.

## 6. Tests obligatoires

### Tests unitaires

Le classement et l'attribution automatique doivent être testés indépendamment de la persistance :

- sélection d'une salle dont la capacité est exactement suffisante ;
- rejet d'une salle trop petite, en maintenance, déjà réservée ou sans un équipement demandé ;
- acceptation de deux réservations consécutives ;
- calcul de la distance dans un même bâtiment et entre deux bâtiments ;
- calcul du score combinant la distance et les places inutilisées ;
- sélection de la salle ayant le score le plus faible ;
- départage alphabétique, puis par identifiant ;
- absence de salle compatible.

### Tests d'intégration

- création et consultation d'une salle ;
- création d'une réservation et attribution automatique ;
- recherche de disponibilité et vérification de son ordre ;
- refus d'une réservation conflictuelle ;
- annulation puis nouvelle réservation sur la même période ;
- validation du format des réponses d'erreur.


### 7. Pour commencer

Vous pouvez utiliser https://start.spring.io/ pour génerer la structure du projet

### 8. Rappel de la grille de notation

| La Code                                                         | /14 |
|-----------------------------------------------------------------|----:|
| Chaque classe est au bon endroid                                |  /4 |
| Les endpoints demandés existent tous                            |  /4 |
| Le backend renvoie ce qu’on lui demande                         |  /2 |
| Les tests sont exhaustifs, et bien formulés (GIVEN, WHEN, THEN) |  /4 |
| Les erreurs sont gérées correctement                            |  /2 |
| Présence de commentaires (si pertinent)                         |  /1 |
| présence de javadoc                                             | /1 |
| Qualité globale du code (ex : noms des variables)               | /1 |

| La CI | /3 |
| --- |---:|
| job pour linter | /1 |
| job pour les tests | /1 |
| la ci tourne automatiquement sur main et est valide | /1 |

| La GitHub          | /3 |
|--------------------|---:|
| Les commits sont bien équilibrés et ont des noms explicites    | /2 |
| Un readme explique comment utiliser le projet | /1 |

