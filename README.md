# NovaMind AI — Generative AI Chat Application

A production-grade generative AI web application built on AWS, powered by the **Falcon-7B-Instruct** large language model.

---

## Live Demo

Deployed via **AWS CloudFront** — accessible from any browser, no login required.

---

## Tech Stack

| Layer | Service |
|---|---|
| Frontend | HTML, CSS, JavaScript — hosted on **AWS S3** |
| CDN | **AWS CloudFront** |
| API | **AWS API Gateway** (REST) |
| Backend | **AWS Lambda** (Python 3.11) |
| AI Model | **Falcon-7B-Instruct** on **AWS SageMaker** (JumpStart) |

---

## Architecture

```
Browser → CloudFront → S3 (static frontend)
                ↓
         API Gateway /summarize
                ↓
           Lambda Function
                ↓
        SageMaker Endpoint (Falcon-7B)
```

---

## Features

- Ask any question and get an AI-generated response
- Clean dark UI with auto-expanding output window
- Prompt-only output — user input stripped from model response
- `Ctrl + Enter` keyboard shortcut to generate
- Fully serverless — no servers to manage
- Responsive design — works on desktop and mobile

---

## Project Files

| File | Description |
|---|---|
| `index.html` | Frontend UI |
| `lambda_function.py` | Lambda handler — calls SageMaker endpoint |
| `generative-ai-api-prod-swagger-apigateway.json` | API Gateway Swagger definition |
| `Bucket_Policy_S3.json` | S3 bucket policy for CloudFront OAC |
| `jupyter_notebook.ipynb` | SageMaker notebook — model deployment |
| `steps_to_do.md` | Full step-by-step deployment guide |

---

## Deployment Guide

See [`steps_to_do.md`](./steps_to_do.md) for the complete step-by-step setup.

**Quick summary:**
1. Deploy Falcon-7B via SageMaker JumpStart notebook
2. Create Lambda function with `ENDPOINT_NAME` env variable
3. Import API Gateway from Swagger JSON
4. Upload frontend to S3, serve via CloudFront

---

## Known Limitation

API Gateway has a **hard 29-second timeout**. Very long prompts may be cut off. For a portfolio/demo this is expected behaviour — the fix is an async Lambda + polling pattern or migrating to WebSocket API.

---

## Author

**Aamir Imran**
[linkedin.com/in/aamir-imran](https://www.linkedin.com/in/aamir-imran)

---

## License

This project is for educational and portfolio purposes.
