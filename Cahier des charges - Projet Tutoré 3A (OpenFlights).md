# **CAHIER DES CHARGES — PROJET TUTORÉ 3A INFO APP**

**Thème :** Étude, Analyse Algorithmique et Visualisation du Trafic Aérien Mondial (OpenFlights)  
Année universitaire : 2026–2027 \| Formation : Polytech Lyon — 3A Informatique Apprentissage  
**Encadrants :** H. Elghazel & M. Haddad

---

## **Informations de l'équipe**

| Rôle | Identité / Lien |
| :---- | :---- |
| **Membres du groupe** | Josua Kebaili, Aurelien Sintes, Leo-Paul Jay, Adrien Dumas Philippe et Gabriel Salas |
| **Chef d'équipe coordinateur** | Josua Kebaili |

---

# **1\. Présentation et Contexte du Projet**

## **1.1 Contexte**

Dans le cadre de la formation 3A Informatique en Apprentissage, ce projet tutoré a pour objectif de concevoir et développer une application web adossée à une base de données relationnelle permettant de manipuler, d'interroger et d'analyser un très grand volume de données relatives au transport aérien mondial (basé sur le jeu de données OpenFlights enrichi).

Contrairement à une application de gestion classique, l'enjeu central de ce projet réside dans la performance des traitements : il s'agit d'évaluer l'efficacité algorithmique, de comparer les temps d'exécution entre le moteur de base de données (SQL) et le code applicatif, et de préparer la transition de ces données relationnelles vers une modélisation orientée graphes prévue en 4e année (4A).

## **1.2 Objectifs du Projet**

* **Conception et gestion de base de données :** Modéliser un schéma relationnel robuste (MCD/MLD), importer et nettoyer un jeu de données massif (plusieurs centaines de milliers de lignes), et exploiter la programmation en base de données (vues, procédures stockées, triggers, indexation).  
* **Étude algorithmique et optimisation :** Identifier des traitements coûteux (recherches d'itinéraires, calculs géodésiques, agrégations complexes, tris), implémenter des versions naïves puis optimisées, et mesurer l'évolution du temps d'exécution en fonction du volume de données.  
* **Développement d'une application web analytique :** Offrir une interface interactive permettant d'explorer la flotte d'avions, les compagnies, les aéroports et les routes, tout en visualisant les résultats statistiques et les benchmarks de performance.  
* **Préparation au projet 4A :** Poser les bases théoriques et structurelles permettant de transformer ce réseau aérien relationnel en un graphe exploitable par des algorithmes dédiés.

## **1.3 Périmètre du Projet**

* **Inclus dans le périmètre :**  
  * Scripts ETL (extraction, nettoyage et chargement des données).  
  * Base de données relationnelle optimisée.  
  * Module de benchmark automatisé.  
  * Interface web de visualisation et d'interrogation.  
* **Exclus du périmètre :**  
  * Réservation de billets en temps réel.  
  * Suivi télémétrique radar en direct (ADS-B temps réel).

---

# **2\. Jeux de Données et Modélisation (Volet Base de Données)**

## **2.1 Sources et Volumétrie des Données**

Le socle de données repose sur la base ouverte **OpenFlights**, qui recense les infrastructures et liaisons aériennes mondiales :

* **Modèles d'avions (Planes) :** \~250 modèles d'aéronefs (codes IATA/ICAO, désignation complète).  
* **Aéroports (Airports) :** \~14 000 aéroports (identifiants, nom, ville, pays, coordonnées GPS latitude/longitude, altitude, fuseau horaire).  
* **Compagnies aériennes (Airlines) :** \~6 000 compagnies (identifiants, nom, pays d'origine, statut actif/inactif).  
* **Routes aériennes (Routes) :** \~67 000 liaisons régulières entre un aéroport source et un aéroport de destination, associées à une compagnie et aux types d'avions utilisés.

**Justification du passage à l'échelle (Volumétrie \> 500 000 lignes) :**  
Afin de répondre à l'exigence d'un volume de données suffisant pour que l'efficacité des traitements soit déterminante, le jeu de données OpenFlights est croisé et enrichi avec un historique de vols réalisés (Instances de vols / Flight Logs) (issu de jeux de données publics type BTS / Kaggle ou généré de manière réaliste à partir des routes et flottes existantes). Cela permet d'obtenir une table centrale contenant entre 500 000 et 1 000 000 de lignes (dates de départ/arrivée, retards, appareil spécifique mobilisé, distance parcourue).

## **2.2 Modèle Logique de Données (MLD prévisionnel)**

Le schéma relationnel s'articule autour des tables suivantes :

1. **AVION** (`id_avion` \[PK\], `code_iata`, `code_icao`, `nom_modele`)  
2. **COMPAGNIE** (`id_compagnie` \[PK\], `nom`, `alias`, `code_iata`, `code_icao`, `pays`, `actif`)  
3. **AEROPORT** (`id_aeroport` \[PK\], `nom`, `ville`, `pays`, `code_iata`, `code_icao`, `latitude`, `longitude`, `altitude`, `fuseau_horaire`)  
4. **ROUTE** (`id_route` \[PK\], `id_compagnie` \[FK\], `id_aeroport_depart` \[FK\], `id_aeroport_arrivee` \[FK\], `escales`, `distance_km`)  
5. **ROUTE\_AVION** (`id_route` \[PK, FK\], `id_avion` \[PK, FK\]) — *Table de jointure associant les modèles d'avions opérant sur chaque route.*  
6. **VOL\_HISTORIQUE** (`id_vol` \[PK\], `id_route` \[FK\], `id_avion` \[FK\], `date_depart`, `duree_minutes`, `retard_minutes`, `statut`) — *Table volumineuse (\> 500 000 entrées).*

---

# **3\. Spécifications Fonctionnelles (Application Web)**

## **3.1 Profils d'utilisateurs**

L'application s'adresse à un utilisateur de type **Analyste / Explorateur**, souhaitant interroger les données aériennes, visualiser le réseau et exécuter des tests de performance.

## **3.2 Fonctionnalités principales**

| Code | Module | Description détaillée |
| :---- | :---- | :---- |
| **F1** | **Exploration & Recherche** | Recherche et filtrage d'avions, compagnies et aéroports (pays, IATA, modèle). Fiche détaillée par appareil : utilisation par compagnie, typologie de route (court/moyen/long-courrier), distance moyenne. |
| **F2** | **Calculateur d'Itinéraires** | Recherche de trajets entre un aéroport A et B (vol direct, 1 à 2 escales) avec options de tri (distance, durée estimée, type d'appareil). |
| **F3** | **Tableaux de Bord (Dashboard)** | Statistiques globales : top aéroports connectés, modèles d'avions les plus déployés, analyse des retards moyens par zone géographique. |
| **F4** | **Laboratoire de Benchmark** | Test de performance paramétrable (choix du traitement, taille d'échantillon de 10k à 500k+ lignes, méthode d'exécution) avec restitution graphique. |

---

# **4\. Volet Algorithmique et Performances**

Cette section constitue le cœur scientifique du projet. Quatre axes d'expérimentation seront menés et documentés :

## **4.1 Identification des traitements coûteux programmés**

Nous implémenterons nous-mêmes dans le code applicatif au moins deux traitements majeurs :

1. **Recherche d'itinéraires avec escales et filtrage d'appareils :** Croisement en mémoire entre les tables des routes, des horaires de vols et des modèles d'avions pour trouver tous les chemins valides de A vers B avec 1 ou 2 escales.  
2. **Agrégation géodésique et tri massif :** Calcul de la distance exacte (Haversine) sur des centaines de milliers de vols, regroupement par modèle d'avion pour déterminer le rayon d'action moyen, et tri personnalisé des résultats.

## **4.2 Étude d'efficacité et passage à l'échelle**

Pour vérifier expérimentalement la complexité théorique de nos algorithmes :

* Les traitements seront exécutés sur des sous-ensembles de données de taille doublée à chaque étape : **N \= 25 000, 50 000, 100 000, 200 000, 400 000 et 800 000 lignes**.  
* Les temps d'exécution CPU et l'empreinte mémoire seront relevés afin de tracer les courbes d'évolution et de confronter les mesures empiriques aux complexités théoriques (ex: O(n), O(n log n), O(n²)).

## **4.3 Comparaison : Version Naïve vs Version Optimisée**

| Axe | Approche Naïve | Approche Optimisée |
| :---- | :---- | :---- |
| **Code Applicatif** | Boucles imbriquées pour les jointures (complexité O(n²)), tri basique à bulles ou par sélection. | Tables de hachage (HashMaps) pour jointures en O(n) et recherche en O(1) moyen. Tris rapides (Quicksort / Mergesort en O(n log n)). |
| **Base de Données** | Sous-requêtes corrélées sans indexation sur clés étrangères / filtres (parcours séquentiel / Full Table Scan). | Index B-Tree composites, jointures explicites, optimisation via analyse du plan d'exécution (`EXPLAIN ANALYZE`). |

## **4.4 Confrontation : SQL vs Code Applicatif**

Un même traitement (ex: le classement des modèles d'avions les plus utilisés par pays avec calcul de retard moyen sur 500 000 vols) sera exécuté :

1. **100 % en SQL** (traité par l'optimiseur du SGBD).  
2. **100 % dans le langage serveur** (après récupération des données brutes en mémoire).

L'étude analysera le coût du transfert réseau/mémoire (sérialisation des données entre le SGBD et l'application) face à la puissance des index et des algorithmes de jointure natifs du SGBD (Hash Join, Merge Join).

---

# **5\. Perspectives 4A : Modélisation sous forme de Graphes**

Ce projet est conçu dès cette année pour servir de socle au projet de 4A orienté théorie des graphes.

## **5.1 Transformation du modèle relationnel en Graphe**

Le réseau aérien OpenFlights se prête naturellement à une représentation sous forme de **graphe orienté et pondéré (Multi-graphe attribué) G \= (V, E)** :

* **Les Sommets (V \- Vertices) :** Représentent les **Aéroports** (propriétés : code IATA, pays, coordonnées GPS, capacité).  
* **Les Arêtes / Arcs (E \- Edges) :** Représentent les **Routes aériennes ou les Vols** reliant un aéroport source u à un aéroport destination v. Les arcs sont orientés et porteurs de poids (distance en km, durée moyenne du vol, fréquence hebdomadaire, type d'avion).  
* **Structures de stockage en 4A :** Passage d'une table relationnelle de liens à une **liste d'adjacence** en mémoire ou à un SGBD orienté graphe.

## **5.2 Questions et Algorithmes de Graphes envisageables en 4A**

\[Aéroport A\] \---\> (Route 1: Distance / Temps) \---\> \[Aéroport B\]

             \---\> (Route 2: Compagnie Y)       \---\> \[Aéroport C\]

* **Recherche de plus courts chemins (Itinéraires optimaux) :**  
  * *Question :* Quel est le trajet le plus court (en distance ou en temps cumulé) entre deux aéroports quelconques du monde avec un nombre quelconque d'escales ?  
  * *Algorithmes :* **Parcours en largeur (BFS)** pour minimiser le nombre d'escales ; **Algorithme de Dijkstra** ou **A**\* (avec la distance à vol d'oiseau comme heuristique) pour minimiser la distance ou le temps de vol.  
* **Analyse de centralité et de vulnérabilité du réseau :**  
  * *Question :* Quels sont les aéroports critiques ("Hubs") dont la fermeture (météo, panne) paralyserait le plus le trafic mondial ?  
  * *Algorithmes :* **Centralité d'intermédiarité (Betweenness Centrality)**, **PageRank** appliqué aux aéroports, et étude de la **connexité** (composantes fortement connexes).  
* **Détection de communautés :**  
  * *Question :* Comment le trafic aérien mondial se découpe-t-il en sous-réseaux régionaux denses indépendamment des frontières politiques ?  
  * *Algorithmes :* **Algorithme de Louvain** ou **Label Propagation** (maximisation de la modularité).

---

# **6\. Cadre Technique et Organisation**

## **6.1 Stack Technologique (Prévisionnelle)**

* **SGBD :** PostgreSQL (choisi pour la richesse de son optimiseur de requêtes, le support avancé du PL/pgSQL et la précision de son outil `EXPLAIN ANALYZE`).  
* **Backend & Moteur Algorithmique :** Python (FastAPI / Flask) ou Node.js / Java.  
* **Frontend & Visualisation :** HTML5 / CSS3 / JavaScript (React ou [Vue.js](http://Vue.js)) et **Chart.js** (graphiques statistiques et courbes de benchmark).  
* **Gestion de version et Collaboration :** Git (GitHub) et diagramme de Gantt pour le suivi des tâches.

## **6.2 Calendrier et Jalons (S5)**

| Date | Échéance / Jalon | Livrables attendus |
| :---- | :---- | :---- |
| **Ven. 16/10/2026** | Lancement du projet | Constitution du groupe et choix du sujet |
| **Lun. 19/10/2026** | Déclaration officielle | Envoi par mail du groupe, du chef d'équipe et du thème |
| **Mer. 21/10/2026** | Cahier des charges (V1) | Premier jet du cahier des charges |
| **Ven. 23/10/2026** | Validation du cadrage | Retour des encadrants, cahier des charges définitif et répartition des tâches |
| **Sem. du 07/12/2026** | Point d'étape technique | Schéma BD validé, données massives importées, 1er traitement codé et mesuré |
| **Ven. 22/01/2027** | Rendu final & Soutenance | Dépôt Git complet, rapport final (avec section 4A) et soutenance de 30 min |

## **6.3 Répartition Initiale des Rôles**

*  **— Josua Kebaili (Chef d'équipe coordinateur) :** Coordination générale, suivi du dépôt Git, mise en place de l'architecture web et intégration.  
*  **— Aurelien Sintes (Responsable Base de Données & ETL) :** Conception du MCD/MLD, scripts de nettoyage et d'importation massive des données OpenFlights, indexation, vues et triggers SQL.+   
*  **—** Gabriel Salas et Adrien Dumas Philippe **(Responsable Algorithmique & Benchmarks) :** Implémentation des algorithmes en code (naïf vs optimisé), protocole de mesure de complexité et comparatif SQL vs Code.  
* **— Leo-Paul Jay (Responsable Frontend & DataViz) :** Développement de l'interface d'exploration et des graphiques de performance.

