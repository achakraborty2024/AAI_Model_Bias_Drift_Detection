# Churn Model Quality Monitoring on AWS SageMaker

 **Automated Model Monitoring for XGBoost Churn Prediction Model**
 
 **Scheduled Model Quality Evaluation (every 2 hour) using SageMaker Model Monitor**
 
 **Reports written to S3 → analyzed to detect model quality drift**
 
 **Test rows captured via endpoint for monitoring**

---

## Project Overview

This project demonstrates how to set up an **automated model quality monitoring pipeline** on AWS SageMaker for a production XGBoost Churn Prediction Model.

### Goals:

*  Deploy trained XGBoost model to a real-time endpoint
*  Configure Model Monitor to capture inference requests and predictions
*  Automate scheduled Model Quality jobs (ModelQualityMonitor)
*  Generate hourly model quality reports
*  Analyze reports to detect **quality drift, threshold violations, and metrics trends**

---

## Architecture

```plaintext
SageMaker Endpoint (XGBoost Model) ← EndpointInput → Model Monitor → Reports → S3
                         ↑                              ↓
                     Inference Capture (requests + responses) → Hourly scheduled Monitoring jobs
```

---

## Components

### Model:

* `xgb-churn-prediction-model.tar.gz` — XGBoost model trained for customer churn prediction

### Data:

* Test data: `test-dataset-input-cols.csv` (69 columns, fully numeric)
* Ground Truth Labels: `ground_truth_data/` in S3

### Monitoring:

* `ModelQualityJobDefinition` created via `boto3` API
* `MonitoringSchedule` created via `boto3` API with:

```cron
cron(0 * * * ? *)  → runs hourly at minute 0
```

* Reports written to:

```text
s3://sagemaker-us-east-1-672518276407/sagemaker/Churn-ModelQualityMonitor-20201201/reports/
```

---

## Main Scripts

### 1️⃣ Create Model Quality Job Definition

```python
sm_client.create_model_quality_job_definition(...)
```

### 2️⃣ Create Monitoring Schedule

```python
sm_client.create_monitoring_schedule(
    MonitoringScheduleName=...,
    MonitoringScheduleConfig={
        'ScheduleExpression': 'cron(0 * * * ? *)',
        ...
    }
)
```

### 3️⃣ Test Payload Generation

* Test rows sent to endpoint using `text/csv` payloads (69 columns).
* Capture S3 path automatically populated.

### 4️⃣ Report Reader Script

* Reads:

```text
/reports/YYYYMMDDHHMM/statistics.json
/reports/YYYYMMDDHHMM/constraint_violations.json
```

* Extracts **Model Quality Metrics**:

  * Precision
  * Recall
  * F1 score
  * AUC
  * Accuracy

---

## Monitoring Reports Structure

```text
/reports/YYYYMMDDHHMM/
    constraint_violations.json  → any violations vs baseline
    statistics.json             → full model quality metrics
```

Example:

```json
"binary_classification_metrics": {
    "precision": 0.85,
    "recall": 0.90,
    "f1": 0.87,
    "auc": 0.92,
    "accuracy": 0.88
}
```

---

## Usage

 Send valid test rows to endpoint → captured in S3
 MonitoringSchedule runs hourly → generates reports
 Run report reader script → analyze model quality drift

---

## Learnings / Notes

*  SageMaker MonitoringSchedule **only supports certain cron formats** → `cron(0 * * * ? *)`, `cron(0 */2 * * ? *)`, etc.
*  For sub-hourly monitoring (e.g. every 10 min), need to use **EventBridge triggers + StartModelQualityJobDefinition** (advanced).
*  Sending valid **69-feature test rows** is critical → else Monitoring job fails.
*  Correct Ground Truth data required for accurate metrics.

---

## Next Improvements

* [ ] Add EventBridge rule to run every 10 min
* [ ] Automate Monitoring job status checks
* [ ] Implement metrics trend visualization dashboard

---

## References

* [AWS SageMaker Model Monitor](https://docs.aws.amazon.com/sagemaker/latest/dg/model-monitor.html)
* [AWS SageMaker ModelQualityMonitor API](https://boto3.amazonaws.com/v1/documentation/api/latest/reference/services/sagemaker.html#SageMaker.Client.create_model_quality_job_definition)
* [AWS Cron Expressions](https://docs.aws.amazon.com/AmazonCloudWatch/latest/events/ScheduledEvents.html)

---

## Credits

* AWS SageMaker
* Project developed and tested on: `sagemaker-us-east-1-672518276407`

---

## License

MIT License

---

## Final Status (current):

- ModelQualityJobDefinition → Created
- MonitoringSchedule → Created → Hourly
- Test rows sent → YES
- First timestamped report → pending / waiting for first scheduled run
