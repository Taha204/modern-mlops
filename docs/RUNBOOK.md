# House Price API — Runbook

## Objectifs de service

Objectifs à vérifier sur une période de 7 jours :

| Indicateur | Objectif |
|---|---|
| Disponibilité | Au moins 99 % des requêtes /predict sans erreur 5xx |
| Latence | p95 de /predict inférieur à 200 ms |
| Monitoring | Dernier monitoring_job réussi depuis moins de 26 heures |
| Qualité | RMSE réelle inférieure ou égale à 1,5 fois la RMSE de test |

## Démarrage et arrêt

```bash
docker compose up -d
docker compose ps
docker compose down
```

`docker compose down` conserve les volumes.
`docker compose down -v` supprime les données persistantes : ne pas l'utiliser
pour un arrêt normal.

## Vérifier le service

- API : http://localhost:8000/docs
- Modèle chargé : http://localhost:8000/ready
- MLflow : http://localhost:5000
- Dagster : http://localhost:3000
- Prometheus : http://localhost:9090
- Grafana : http://localhost:3001

```bash
docker compose exec api mlops-model status
docker compose logs --tail 100 api
```

Chaque requête possède un request_id pour retrouver ses logs.

## Publier un modèle

1. Lancer training_job dans Dagster.
2. Vérifier les métriques, la model card et gate_passed=true dans MLflow.
3. Promouvoir le challenger et redémarrer l'API :

```bash
docker compose exec api mlops-model promote
docker compose restart api
```

4. Vérifier la version sur /ready.
5. Lancer le smoke test :

```bash
uv run python scripts/smoke_test.py --api-key "<cle>"
```

6. Surveiller les métriques dans Grafana pendant 15 minutes.

## Revenir au modèle précédent

```bash
docker compose exec api mlops-model rollback
docker compose restart api
```

Vérifier /ready et relancer le smoke test.

## Réagir aux incidents

| Signal | Vérification et action |
|---|---|
| ApiDown | Consulter docker compose ps et les logs API ; redémarrer le service si nécessaire |
| ModelNotLoaded | Vérifier MLflow, l'existence du champion et les logs API |
| HighErrorRate | Rechercher les exceptions ; revenir au modèle précédent si la publication est en cause |
| HighLatency | Vérifier docker stats et les entraînements en cours |
| PredictionLogFailures | Vérifier Postgres, ses logs et l'espace disque |
| input_drift en échec | Consulter le rapport HTML dans artifacts/reports et analyser les features concernées |
| live_rmse en échec | Vérifier les feedbacks, le marché et la version ; envisager un réentraînement |
| Validation en échec | Lire artifacts/reports/validation.json et corriger ou rejeter les données sources |

## Politique de réentraînement

- Entraînement planifié chaque lundi à 06 h UTC.
- Monitoring quotidien à 07 h UTC.
- Entraînement manuel après analyse d'une dérive ou d'une baisse de qualité.
- Promotion manuelle uniquement après validation du gate et lecture de la model card.

## Sauvegarde et restauration

Sauvegarder les bases Postgres et le volume mlflow-artifacts.
Conserver également le code, uv.lock, les données sources et la configuration
nécessaire à la reproduction. Stocker les secrets séparément.

Tester la restauration dans une base temporaire et vérifier les données.
Une sauvegarde créée sans test de restauration ne constitue pas une
restauration validée.

## Rotation d'une clé API

1. Ajouter la nouvelle clé dans API_KEYS, séparée par une virgule.
2. Appliquer avec docker compose up -d api.
3. Configurer les clients avec la nouvelle clé.
4. Retirer l'ancienne clé et appliquer à nouveau la configuration.

## État de la sécurité

Consulter le dernier résultat GitHub Actions.
Un audit rendu non bloquant reste un audit en échec si des vulnérabilités
sont détectées. Documenter les exceptions et prévoir les mises à jour.