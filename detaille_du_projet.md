# Analyse  taxi newyork GCP

## Contexte et objectifs

Le projet construit un pipeline ELT de bout en bout sur Google Cloud Platform : les données des taxis jaunes de New York sont extraites, chargées dans BigQuery, transformées avec Dataform, puis exploitées en tableaux de bord Looker Studio et par un modèle de Machine Learning.

contributeurs:@LOIC754 ; @beatricefopoussi <br>
[suivi des tâches dans Jira](./taches.txt)<br>

Objectifs :<br>

- Automatiser l'ingestion des fichiers mensuels NYC TLC, sans intervention manuelle.<br>
- Produire des données fiables, testées et documentées, organisées en couches bronze, silver et gold.<br>
- Exposer des datamarts orientés métier, prêts pour la visualisation.<br>
- Répondre à des questions métier : demande par zone et par heure, revenus, pourboires, durée des courses.<br>
- Entraîner un modèle prédictif à partir de la couche gold.<br>
- Reproduire les standards d'une équipe data en entreprise : tests, revue de code, CI/CD, moindre privilège, maîtrise des coûts.

## Source de données

La source est le jeu public de la NYC Taxi & Limousine Commission (TLC) : un fichier Parquet par mois pour les taxis jaunes, plus une table de référence des zones.

- **Trajets** : `yellow_tripdata_AAAA-MM.parquet`, un fichier par mois.
- **URL type** : `https://d37ci6vzurychx.cloudfront.net/trip-data/<fichier>`
- **Zones** : fichier *taxi zone lookup* (identifiant de zone, arrondissement, nom de zone).
- **Colonnes clés** : dates de prise en charge et de dépose, zones de départ et d'arrivée, distance, nombre de passagers, montant, pourboire, type de paiement.
- **Périmètre retenu** : à fixer en début de projet (par exemple 2023-2024) pour limiter les volumes et les coûts.

Les données brutes contiennent des anomalies connues (distances ou montants négatifs, dates hors période, doublons) : leur nettoyage est une partie centrale du projet.

## Architecture cible

Le projet est basé sur l'architechture Medaillon , suivante le principe ETL orchestré <br>

Source NYC TLC → Cloud Run Job (Python + Cloud Scheduler) → GCS zone raw → BigQuery bronze → Dataform silver → Dataform gold (étoile + datamarts) → Looker Studio et BigQuery ML

- **E + L** (extraction et chargement) : de la source NYC TLC jusqu'à BigQuery bronze.
- **T** (transformations SQL) : de bronze à silver puis gold, avec Dataform.

Les données ne sont transformées qu'une fois dans BigQuery (principe ELT) : bronze garde le brut, silver le nettoyé, gold les tables métier qui alimentent Looker Studio et le modèle ML.

## Stack technique et choix

Chaque outil couvre une étape précise du pipeline ; tous sont natifs GCP sauf GitHub et Jira.

- **Extraction** – Python + Cloud Run Job : télécharger les fichiers TLC et les écrire dans GCS.
- **Planification** – Cloud Scheduler : lancer l'extraction automatiquement (par exemple tous les 2 jours).
- **Stockage brut** – Cloud Storage (GCS) : conserver les fichiers Parquet d'origine (zone raw).
- **Entrepôt** – BigQuery : stocker les couches bronze, silver et gold.
- **Transformation** – Dataform : nettoyer, modéliser, tester et documenter en SQL.
- **Visualisation** – Looker Studio : tableaux de bord sur les datamarts gold.
- **Machine Learning** – BigQuery ML (Vertex AI en option) : entraîner et servir le modèle prédictif.
- **Conteneurs** – Docker + Artifact Registry + Cloud Build : empaqueter et stocker l'image d'extraction.
- **Code et CI/CD** – GitHub + GitHub Actions : versionner, relire, tester et déployer.
- **Suivi** – Jira (gratuit) : epics, stories, sprints, répartition des tâches.

**Cloud Run est-il nécessaire ?** Pas strictement : Storage Transfer Service peut copier des fichiers depuis une liste d'URL sans code. Cloud Run est retenu car il permet une logique incrémentale (ne récupérer que les nouveaux mois), une gestion fine des erreurs, du code testé et versionné, et c'est l'outil utilisé en entreprise sur GCP.

**Dataform est-il nécessaire ?** Des requêtes planifiées BigQuery suffiraient techniquement. Dataform est retenu car il apporte le versionnement Git du SQL, la gestion automatique des dépendances entre tables, les tests de qualité (assertions), la documentation et le lignage : c'est le cœur d'un ELT propre sur GCP.

**Looker ou Looker Studio ?** Looker est payant ; le projet utilise Looker Studio, gratuit et connecté nativement à BigQuery.

## Déroulé du projet

Le projet se découpe en 9 epics, à reprendre tels quels dans Jira.

1. **Cadrage et organisation** – définir les questions métier, la période de données, le board Jira et les conventions Git.
    - Utilité : sans questions métier, les datamarts n'ont pas de cible ; les conventions évitent d'écraser le travail de l'autre.
2. **Socle GCP et droits d'accès** – créer le projet, lier la facturation, poser une alerte de budget, ajouter le partenaire dans IAM avec des rôles ciblés, activer les API, créer un compte de service dédié, fixer la région `europe-west9`.
    - Utilité : sécurité (moindre privilège), coûts sous contrôle, pas d'erreur de chargement entre régions.
3. **Extraction (E + L vers raw)** – script Python découpé en fonctions et incrémental, tests pytest, Dockerfile, image dans Artifact Registry, Cloud Run Job, planification Cloud Scheduler.
    - Utilité : ingestion automatique, sans retélécharger les mois déjà présents.
4. **Chargement BigQuery (bronze)** – créer les datasets `bronze`, `silver` et `gold`, exposer les Parquet en table externe ou native, charger la table des zones.
    - Utilité : les données brutes deviennent requêtables en SQL, sans transformation.
5. **Transformations Dataform (silver)** – connecter Dataform au dépôt GitHub, typer, renommer, dédoublonner, filtrer les anomalies, écrire les assertions, partitionner par date et clusteriser par zone.
    - Utilité : données fiables, qualité testée automatiquement, coûts de requête réduits.
6. **Couche gold et datamarts** – modèle en étoile (`fct_trips`, `dim_zone`, `dim_date`, `dim_payment_type`), puis datamarts : demande horaire par zone, revenus et pourboires, performance des courses.
    - Utilité : tables pré-agrégées par besoin métier, rapides et peu coûteuses à interroger.
7. **Visualisation Looker Studio** – 2 à 3 tableaux de bord branchés uniquement sur la couche gold.
    - Utilité : rendre les résultats lisibles pour un décideur, avec des chiffres cohérents.
8. **Machine Learning** – choisir un cas d'usage (montant de la course, pourboire ou demande par zone et par heure), construire la table de features dans Dataform, entraîner et évaluer avec BigQuery ML.
    - Utilité : passer de l'analyse descriptive à la prédiction, sans sortir les données de BigQuery.
9. **Orchestration, CI/CD et documentation** – enchaîner extraction, Dataform et ré-entraînement ; GitHub Actions pour les tests et le déploiement ; README avec schéma d'architecture.
    - Utilité : un pipeline qui tourne seul et un projet compréhensible par un jury ou un recruteur.

## Organisation de l'équipe

Le binôme travaille sur un dépôt GitHub unique, avec une revue croisée de chaque modification et un suivi dans Jira.

**Git**

- Branche `main` protégée : aucune modification directe.
- Une branche par tâche (`feature/<ticket-jira>-<description>`).
- Pull request obligatoire, relue et validée par le partenaire avant fusion.
- Messages de commit clairs, reprenant la clé du ticket Jira.

**Jira (gratuit)**

- Un epic par étape du déroulé, découpé en stories et tâches.
- Sprints de 1 à 2 semaines, avec un point d'avancement court à chaque début de sprint.
- Chaque tâche a un responsable unique et un critère de fin explicite.

**Accès GCP du partenaire**

- Rôles ciblés plutôt que *Propriétaire* : BigQuery Data Editor et Job User, Storage Object Admin, Dataform Editor, Cloud Run Developer.
- Le pipeline s'exécute avec un compte de service dédié, pas avec vos comptes personnels.

La répartition des epics entre les deux membres reste à décider.

## Livrables

En fin de projet, le binôme remet sept livrables vérifiables.

- **Code d'extraction** : script Python, tests pytest, Dockerfile, Cloud Run Job déployé et planifié.
- **Entrepôt BigQuery** : datasets `bronze`, `silver` et `gold` alimentés automatiquement.
- **Projet Dataform** : transformations SQL, assertions de qualité, documentation des tables.
- **Datamarts gold** : modèle en étoile et tables par usage métier.
- **Tableaux de bord** : 2 à 3 rapports Looker Studio partagés.
- **Modèle ML** : modèle BigQuery ML entraîné, évalué, avec ses métriques.
- **Documentation** : README (architecture, lancement, choix techniques), board Jira à jour.

## Coûts et risques

Le principal risque est financier : chaque fichier mensuel pèse des dizaines de Mo et BigQuery facture le volume scanné.

- **Facture GCP imprévue** : alerte de budget dès la création du projet, période de données limitée.
- **Requêtes BigQuery coûteuses** : tables partitionnées et clusterisées, pas de `SELECT *`, dashboards branchés sur gold.
- **Retéléchargements inutiles** : extraction incrémentale (seuls les mois absents du bucket).
- **Données aberrantes** : filtres en silver et assertions Dataform.
- **Conflits de code dans le binôme** : branches par tâche et pull requests relues.
- **Droits trop larges** : rôles IAM ciblés et compte de service dédié.
- **Erreurs entre régions** : toutes les ressources en `europe-west9`.