# Enterprise Generative AI — Deployment Guide

**Project:** Cloudage Enterprise Generative AI Application
**Stack:** Amazon SageMaker · AWS Lambda · API Gateway · S3 · CloudFront
**Model:** Falcon-7B-Instruct (HuggingFace, deployed via SageMaker JumpStart)

---

## Architecture Overview

```
User Browser
    │
    ▼
CloudFront (CDN)
    │
    ▼
S3 Bucket  ──────────►  index.html + cloudage_logo.jpeg
    │
    (static frontend calls)
    ▼
API Gateway (REST) — /summarize  POST
    │
    ▼
Lambda Function (Python 3.11)  ──►  SageMaker Endpoint (Falcon-7B)
```

---

## Phase 1 — Deploy the AI Model on SageMaker

### Step 1 — Set up SageMaker Studio

1. Log in to the **AWS Console** and navigate to **Amazon SageMaker AI**.
2. In the left panel go to **Studio → Users** and create a new user with **default settings**.
3. Click **Launch Studio** to open SageMaker Studio.

### Step 2 — Create a JupyterLab Space

1. Inside Studio, navigate to **JupyterLab Spaces** and create a new space with these recommended settings:
   - **Instance:** `ml.t3.medium`
   - **Storage:** `5 GB`
2. Wait for the space to start, then click **Open JupyterLab**.

### Step 3 — Run the Notebook to Deploy the Model

1. Upload `jupyter_notebook.ipynb` from your local machine into JupyterLab.
2. Open the notebook and run **all cells from Step 2 through Step 3 first** (these cells deploy the SageMaker endpoint).
3. Continue running all remaining cells in sequence.
4. Once the endpoint is created, **copy the endpoint name** — you will need it in Phase 2.

   > Example endpoint name: `hf-llm-falcon-7b-instruct-bf16-2025-10-14-07-29-29-666`
   >
   > Best practice: always copy the name directly from **SageMaker → Inference → Endpoints**.

---

## Phase 2 — Configure the Lambda Function

### Step 4 — Create the Lambda Function

1. In the AWS Console navigate to **AWS Lambda → Functions → Create function**.
2. Choose **Author from scratch** and configure:
   - **Runtime:** Python 3.11
   - **Architecture:** x86_64
3. Under **Permissions**, assign (or create) an IAM execution role with the **cloudage-lambda-roleee`AmazonSageMakerFullAccess`** policy attached.

   > If your organisation uses a custom policy, use that instead of the AWS-managed one.

### Step 5 — Add the Function Code

1. In the Lambda Code Editor, delete the default placeholder code.
2. Paste the entire contents of `lambda_function.py`.
3. Confirm the handler is set to: `lambda_function.lambda_handler`

   Key code behaviour:
   - Reads `ENDPOINT_NAME` from an environment variable (never hard-coded).
   - Calls `sagemaker-runtime` → `invoke_endpoint` with the request body.
   - Returns CORS headers so the frontend can call it from the browser.

### Step 6 — Set Environment Variables & Timeout

1. Go to **Configuration → Environment variables → Edit → Add environment variable**:
   | Key | Value |
   |---|---|
   | `ENDPOINT_NAME` | *(paste your SageMaker endpoint name here)* |

2. Go to **Configuration → General configuration → Edit**:
   - Set **Timeout** to **5 minutes** (300 seconds).

3. Click **Deploy** to save and publish the function.

---

## Phase 3 — Configure API Gateway

### Step 7 — Import the REST API

1. In the AWS Console navigate to **API Gateway → REST API → Import**.
2. Upload `generative-ai-api-prod-swagger-apigateway.json`.
3. Review the imported resources — you should see:
   - `OPTIONS /` (CORS preflight)
   - `POST /summarize` (main inference endpoint)
   - `OPTIONS /summarize` (CORS preflight)

### Step 8 — Wire the Lambda Function

1. In the **Resources** panel, click `POST` under `/summarize`.
2. Click **Integration Request** and update the **Lambda Function** ARN to point to your newly created Lambda function.
3. Confirm CORS headers are configured (they are pre-set in the imported Swagger).

### Step 9 — Deploy the API

1. Click **Actions → Deploy API**.
2. Create a **New Stage** — name it according to your organisation's naming convention (e.g. `prod`, `dev`, `staging`).
3. Copy the **Invoke URL** for the `/summarize` POST resource. It will look like:
   ```
   https://<api-id>.execute-api.<region>.amazonaws.com/<stage>/summarize
   https://7q2epnpf77.execute-api.us-east-1.amazonaws.com/proddd/summarize
   ```

> **The backend is now fully operational.**

---

## Phase 4 — Prepare the Frontend

### Step 10 — Update `index.html` with the API URL

1. Open `index.html` in a text editor.
2. Find **line ~245** and update the API Gateway URL:
   ```javascript
   // Before:
   var apiGatewayUrl = "https://xgx15g51t7.execute-api.us-east-1.amazonaws.com/prodd/summarize";

   // After (replace with your own):
   var apiGatewayUrl = "https://<your-api-id>.execute-api.<your-region>.amazonaws.com/<your-stage>/summarize";
   ```
3. Save the file.

### Step 11 — Upload Assets to S3

1. Navigate to **AWS S3** and create (or select) the bucket you will use as the CloudFront origin (e.g. `cloudage-gen-ai-webapp` `raw-data-2-visualizations`).
2. Upload both files:
   - `index.html`
   - `cloudage_logo.jpeg`
3. Do **not** enable public access — CloudFront will handle access control.

---

## Phase 5 — Set Up CloudFront

### Step 12 — Create a CloudFront Distribution

1. Go to **AWS CloudFront → Distributions → Create distribution**.
2. Set **Origin domain** to your S3 bucket.
3. Under **Origin access**, select **Origin access control (OAC)** and create a new OAC if prompted.
4. Leave other settings as default and click **Create distribution**.

### Step 13 — Set the Default Root Object

1. Open the newly created distribution.
2. Go to **General → Edit**.
3. Set **Default root object** to `index.html`.
4. Save changes.

### Step 14 — Update the S3 Bucket Policy

1. From the distribution, go to **Origins → select your origin → Edit**.
2. Click **Copy policy** to copy the auto-generated S3 bucket policy.
3. Navigate to your S3 bucket → **Permissions → Bucket policy → Edit**.
4. Paste the copied policy. It will look like this (from `Bucket_Policy_S3.json`):

   ```json
   {
     "Version": "2012-10-17",
     "Statement": [
       {
         "Sid": "AllowCloudFrontServicePrincipalReadOnly",
         "Effect": "Allow",
         "Principal": { "Service": "cloudfront.amazonaws.com" },
         "Action": "s3:GetObject",
         "Resource": "arn:aws:s3:::YOUR-BUCKET-NAME/*",
         "Condition": {
           "StringEquals": {
             "AWS:SourceArn": "arn:aws:cloudfront::YOUR-ACCOUNT-ID:distribution/YOUR-DISTRIBUTION-ID"
           }
         }
       }
     ]
   }
   ```

   > Replace `YOUR-BUCKET-NAME`, `YOUR-ACCOUNT-ID`, and `YOUR-DISTRIBUTION-ID` with your actual values.

5. Save the policy.

### Step 15 — Wait for Deployment & Invalidate Cache

1. Wait for the CloudFront distribution status to show **Deployed** (typically 5–10 minutes).
2. To force-refresh edge caches after any file update, go to:
   **Distributions → Invalidations → Create Invalidation**
   Enter `/*` and click **Create**.
3. Once deployed, copy the **Distribution Domain Name** (e.g. `d1abc123xyz.cloudfront.net`) and open it in your browser.

---

## Verification Checklist

| # | Check | Expected Result |
|---|-------|-----------------|
| 1 | SageMaker endpoint status | `InService` |
| 2 | Lambda test invocation | Status 200, JSON response |
| 3 | API Gateway test (POST `/summarize`) | 200 with `generated_text` in body |
| 4 | CloudFront distribution status | `Deployed` |
| 5 | Open CloudFront URL in browser | `index.html` loads with logo |
| 6 | Enter a prompt and click Generate | AI response appears in output box |

---

## Keyboard Shortcut

Inside the web app, you can also press **Ctrl + Enter** inside the prompt textarea to trigger generation without clicking the button.

---

## Important Notes

- **Cost awareness:** The Falcon-7B-Instruct endpoint runs on a dedicated SageMaker instance. Remember to **delete or stop the endpoint** when not in use to avoid ongoing charges.
- **Model flexibility:** You can deploy any other model from the SageMaker JumpStart catalogue. Update the `ENDPOINT_NAME` environment variable in Lambda accordingly.
- **CORS:** CORS is pre-configured in both the Lambda response headers and the API Gateway Swagger definition — no additional setup is needed.
- **Security:** The S3 bucket is private; only CloudFront can read from it via the Origin Access Control policy. Never make the bucket public.
- **AI output disclaimer:** Review all AI-generated outputs for accuracy before acting on them.

---

> **Project complete.** Your enterprise GenAI application is live on CloudFront, backed by Falcon-7B-Instruct on SageMaker.
