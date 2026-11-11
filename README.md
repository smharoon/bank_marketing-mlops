# Bank Marketing MLOps Service

## Overview

The Bank Marketing MLOps Service is part of a microservices-based banking application developed for the **SEZG583 – Scalable Services** assignment.

The service is responsible for training, evaluating, and managing machine learning models that predict whether a bank customer is likely to subscribe to a term deposit.

The application uses **Python, Scikit-learn, Prefect, PostgreSQL, MLflow, Docker, and Kubernetes**.

## Service Functionality

The MLOps service performs the following operations:

1. Reads and validates Bank Marketing CSV datasets.
2. Trains Logistic Regression and Random Forest models.
3. Evaluates the models using classification metrics.
4. Selects the best-performing model using positive-class F1-score.
5. Registers the selected model in MLflow using the `champion` alias.
6. Evaluates new datasets using the registered champion model.
7. Generates row-level predictions and saves them as CSV files.
8. Stores training and evaluation information in PostgreSQL.

The service contains two independent Prefect workflows:

- **Training:** Triggered manually through Prefect.
- **Evaluation:** Executed automatically every 120 seconds.

## Technologies Used

- **Python:** Machine learning and data processing.
- **Scikit-learn:** Model training, preprocessing, and evaluation.
- **Prefect 3:** Workflow orchestration and monitoring.
- **PostgreSQL:** Training records, evaluation data, and prediction results.
- **MLflow:** Model tracking, registration, and version management.
- **Docker:** Application containerization.
- **Kubernetes:** Container deployment and management.
- **Minikube:** Local Kubernetes cluster.

## Architecture

```text
       Bank Marketing CSV Datasets
                    |
           +--------+--------+
           |                 |
           v                 v
     Training Data      Evaluation Data
           |                 |
           v                 v
     Training Flow      Evaluation Flow
           |                 |
           v                 |
    Model Training           |
    - Logistic Regression    |
    - Random Forest          |
           |                 |
           v                 |
     Model Selection         |
           |                 |
           v                 |
    MLflow Model Registry    |
       (Champion)            |
           |                 |
           +---------------->+
                             |
                             v
                      Model Evaluation
                             |
                    +--------+--------+
                    |                 |
                    v                 v
               PostgreSQL      Prediction CSV
```

Training and evaluation are executed as separate Kubernetes Deployments using the same MLOps Docker image.

The service uses PostgreSQL, Prefect, and MLflow infrastructure running within the Kubernetes cluster.

## Project Structure

```text
bank_mlops/
├── app/
│   ├── common.py
│   ├── database.py
│   ├── modeling.py
│   ├── train_flow.py
│   ├── evaluate_flow.py
│   ├── predict.py
│   └── registry.py
├── train_model/
│   ├── incoming/
│   ├── processed/
│   └── failed/
├── evaluate_model/
│   ├── incoming/
│   ├── processed/
│   └── failed/
├── outputs/
├── tests/
├── Dockerfile
├── compose.yaml
├── requirements.txt
└── README.md
```

## Kubernetes Deployment

The MLOps application is containerized using Docker and deployed to the local Minikube Kubernetes cluster.

### Build Docker Image

From the parent project directory:

```bash
docker build -t bank-mlops:1.0.2 ./bank_mlops
```

### Load Image into Minikube

```bash
minikube image load --daemon --overwrite bank-mlops:1.0.2
```

### Deploy MLOps Services

After deploying the shared Kubernetes infrastructure:

```bash
kubectl apply -f kubernetes/07-mlops-train.yaml
kubectl apply -f kubernetes/08-mlops-evaluate.yaml
```

### Verify Deployments

```bash
kubectl get deployments -n bank-marketing
```

```bash
kubectl get pods -n bank-marketing
```

### View Logs

Training:

```bash
kubectl logs deployment/bank-mlops-train -n bank-marketing
```

Evaluation:

```bash
kubectl logs deployment/bank-mlops-evaluate -n bank-marketing
```

Both workflows run as long-lived Kubernetes Deployments managed by Prefect.

**Note:** The Kubernetes manifests are maintained in the separate infrastructure repository. The shared PostgreSQL, Prefect, and MLflow services, database Secret, and local Minikube folder mounts must be configured before deploying MLOps.

## Running Model Training

Place a labeled Bank Marketing CSV dataset in:

```text
bank_mlops/train_model/incoming/
```

The CSV must use a semicolon (`;`) delimiter and contain the expected 21 columns, including the target column `y`.

Open the Prefect dashboard:

```bash
kubectl -n bank-marketing port-forward service/bank-prefect 4200:4200
```

Access http://localhost:4200.

Navigate to:

`Deployments → bank-marketing-train-model/manual-training`

Trigger the training workflow manually.

The training process:

1. Combines newly received training files.
2. Splits the dataset into 70% training and 30% testing data.
3. Trains Logistic Regression and Random Forest models.
4. Evaluates and selects the best model.
5. Registers the champion model in MLflow.
6. Moves successfully processed files to `train_model/processed/`.

Previously processed file contents are detected using SHA-256 checksums to avoid duplicate training.

## Running Model Evaluation

Place a labeled CSV containing evaluation observations in:

```text
bank_mlops/evaluate_model/incoming/
```

The evaluation workflow automatically checks for new files every 120 seconds.

It loads the champion model from MLflow, generates predictions, calculates evaluation metrics, and stores the results in PostgreSQL.

Generated prediction files are saved to:

```text
bank_mlops/outputs/
```

Successfully evaluated input files are moved to:

```text
bank_mlops/evaluate_model/processed/
```

Evaluation uses the existing champion model without retraining it. For meaningful performance measurement, evaluation data should be separate from training data.

## Monitoring

### Prefect Dashboard

```bash
kubectl -n bank-marketing port-forward service/bank-prefect 4200:4200
```

Open: http://localhost:4200

Prefect displays training and evaluation flow executions, schedules, task statuses, and logs.

### MLflow Dashboard

```bash
kubectl -n bank-marketing port-forward service/bank-mlflow 5000:5000
```

Open: http://localhost:5000

MLflow displays experiment results, model metrics, registered models, and champion versions.

### Kubernetes Dashboard

```bash
minikube dashboard
```

The Kubernetes Dashboard displays the deployed MLOps workloads and their running Pods.

## Integration and Data Storage

The MLOps service uses PostgreSQL for storing training records, evaluation records, metrics, and predictions.

The following tables are maintained within the `mlops` database schema:

- `mlops.files`
- `mlops.training_records`
- `mlops.training_runs`
- `mlops.evaluation_records`
- `mlops.evaluation_metrics`
- `mlops.predictions`

Although DataOps and MLOps use the same PostgreSQL server, MLOps maintains its own database tables and does not directly modify or depend on the DataOps tables.

Prefect and MLflow are shared infrastructure components accessible through Kubernetes Services.

Model preprocessing is fitted using training data only, and the `duration` feature is excluded to prevent prediction-time data leakage.

## Dataset Reference

https://archive.ics.uci.edu/dataset/222/bank+marketing
