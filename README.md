# AWS Contextual Chatbot with Amazon Bedrock

This repository contains a production-ready, fully serverless contextual chatbot powered by Amazon Bedrock Knowledge Bases. It features a robust backend deployed with the AWS CDK and a modern React frontend.

The solution allows users to upload their own documents and ask questions, receiving answers generated from the context of those documents, complete with source citations.

---

⚠️ **CRITICAL PRE-DEPLOYMENT STEP: ENABLE BEDROCK MODEL ACCESS**
--------------------------------------------------------------------

Before you deploy this application, you **MUST** enable access to the required foundation models in your AWS account. **Failure to do so will cause the deployment to fail with a clear error message.**

1.  Navigate to the **[Amazon Bedrock console](https://console.aws.amazon.com/bedrock/home)** in your AWS account.
2.  In the bottom-left corner, click on **Model access**.
3.  Click **Manage model access** in the top-right.
4.  Enable access for the following two models:
    *   ✅ **Titan Embeddings G1 - Text:** `amazon.titan-embed-text-v1` (Used for the Knowledge Base)
    *   ✅ **Anthropic Claude 3 Sonnet:** `anthropic.claude-3-sonnet-20240229-v1:0` (Used for generating answers)

Click "Save changes" and wait for access to be granted before proceeding with deployment.

![Model Access Screenshot](images/bedrock-model-access.png) 
*Note: This image should be added to the images folder.*

---

## Table of Contents
- [Architecture](#architecture)
- [Features](#features)
- [Deployment Guide](#deployment-guide)
- [Usage Guide](#usage-guide)
- [API Usage Examples](#api-usage-examples)
- [Security Considerations](#security-considerations)
- [Performance Tuning](#performance-tuning)
- [Cost Estimation](#cost-estimation)
- [Troubleshooting Guide](#troubleshooting-guide)
- [Alternatives Considered](#alternatives-considered)
- [Cleanup](#cleanup)

---

## Architecture

The architecture is fully serverless, event-driven, and designed for scalability and security.

*High-level diagram to be added here*

### Core Components:

*   **Frontend (React on AWS Amplify/S3+CloudFront):** A modern, responsive web interface for user interaction. Hosted on S3 and served globally via CloudFront for low latency.
*   **API Layer (Amazon API Gateway):** A secure, scalable entry point for all frontend requests, with throttling enabled to prevent abuse.
*   **Backend Compute (AWS Lambda):**
    *   **Query Lambda:** Handles user questions, performs the two-step RAG process (Retrieve then Invoke), and applies Guardrails.
    *   **Upload Lambda:** Generates secure, pre-signed S3 URLs for direct file uploads from the browser.
    *   **Ingestion Lambda:** Triggered by S3 file uploads, it starts the Bedrock Knowledge Base ingestion job.
    *   **Status Lambdas:** Provide real-time status updates for document ingestion.
*   **AI Services (Amazon Bedrock):**
    *   **Knowledge Bases:** Ingests documents into a secure, managed vector store (Amazon OpenSearch Serverless) using the Titan G1 Embeddings model.
    *   **Foundation Model:** Uses Anthropic Claude 3 Sonnet to generate answers based on the retrieved context.
    *   **Guardrails:** Provides content filtering for both user inputs and model outputs to ensure responsible AI behavior.
*   **Storage (Amazon S3):** A versioned, encrypted S3 bucket stores all source documents.
*   **Monitoring & Observability (AWS CloudWatch, X-Ray, SNS, SQS):**
    *   **CloudWatch:** Provides logs, metrics, a monitoring dashboard, and alarms.
    *   **X-Ray:** Enables end-to-end tracing for debugging and performance analysis.
    *   **SNS:** Sends real-time alerts on critical errors.
    *   **SQS:** A Dead Letter Queue (DLQ) captures failed ingestion events for analysis.

---

## Features

This solution is more than a simple demo; it's a production-ready application with a focus on security, observability, and user experience.

*   **📖 Contextual Q&A:** Get answers from your own documents, not just the base model's knowledge.
*   **⬆️ Simple File Upload:** Modern drag-and-drop UI for uploading documents directly to the Knowledge Base.
*   **🤖 Responsible AI:** Content filtering for both user questions and AI answers is enforced by **Amazon Bedrock Guardrails**.
*   **⚡ Real-Time Feedback:** The UI provides live status updates on document ingestion, so you know exactly when your new context is ready.
*   **🔒 Secure by Design:** Follows AWS best practices with IAM least privilege, encrypted S3 buckets, and CloudFront OAC.
*   **📊 Comprehensive Observability:**
    *   **CloudWatch Dashboard:** A pre-configured dashboard visualizes API performance, Lambda errors, and ingestion queue depth.
    *   **CloudWatch Alarms:** Automatic SNS alerts for high error rates or ingestion failures.
    *   **X-Ray Tracing:** End-to-end tracing enabled on all critical API paths for easy debugging.
*   **🏗️ Production-Ready Backend:**
    *   **Dead Letter Queue:** Failed ingestion jobs are captured for analysis, ensuring no data is lost.
    *   **Automatic Retries:** The ingestion Lambda automatically retries on transient errors.
    *   **Foolproof Deployment:** A pre-deployment check automatically verifies that the required Bedrock models are enabled, failing fast with a clear error message if they are not.

---

## Deployment Guide

### Prerequisites

1.  **AWS Account & IAM:** An AWS account with administrator-level access.
2.  **AWS CLI:** Installed and configured. Verify by running `aws sts get-caller-identity`.
3.  **Node.js:** Version 20.x or later.
4.  **AWS CDK:** The AWS Cloud Development Kit CLI. Install it with `npm install -g aws-cdk`.
5.  **Docker Desktop:** Must be installed and running. CDK uses Docker to bundle Lambda function assets.

### Deployment Steps

1.  **Clone the Repository:**
    ```bash
    git clone https://github.com/your-repo/your-chatbot.git
    cd your-chatbot
    ```

2.  **Enable Bedrock Models (CRITICAL):**
    *   Follow the instructions in the "CRITICAL PRE-DEPLOYMENT STEP" section at the top of this README.

3.  **Install Dependencies:**
    *   Navigate to the backend directory and install npm packages.
    ```bash
    cd backend
    npm install
    ```

4.  **Bootstrap CDK (If First Time):**
    *   If you have never used the CDK in your AWS account/region before, you need to bootstrap it.
    ```bash
    cdk bootstrap
    ```

5.  **Deploy the Stack:**
    *   This command will synthesize the CloudFormation template and deploy all the resources to your account. The deployment will take approximately 10-15 minutes.
    ```bash
    cdk deploy
    ```

Upon successful deployment, the CDK will output the `CloudFrontURL`, which is the link to your web application.

---

## Usage Guide

1.  **Access the Application:** Open the `CloudFrontURL` from the deployment outputs in your web browser.
2.  **Upload Documents:**
    *   Drag and drop one or more documents (PDF, DOCX, TXT, MD) into the upload area, or click "Select Files".
    *   The files will begin uploading automatically.
3.  **Monitor Ingestion:**
    *   After the upload finishes, a status message will appear: `⏳ Ingesting new document...`
    *   Wait for the status to change to `✅ Ingestion complete!`. This process can take a few minutes depending on the size and number of documents.
    *   The success message acknowledges a small propagation delay. It's best to wait another minute after seeing the success message before asking a question.
4.  **Ask a Question:**
    *   Type a question related to the content of your uploaded documents into the chat input.
    *   Press Enter or click the send icon.
    *   The chatbot will generate an answer based on the retrieved context and provide the source document as a citation.

---

## API Usage Examples

You can also interact with the chatbot programmatically by making POST requests to the API Gateway endpoint.

### Prerequisites

1.  Get your `APIGatewayUrl` from the CDK deployment outputs.
2.  The endpoint for asking questions is `{APIGatewayUrl}docs`.

### Example Request with `curl`

This example sends a question to the chatbot. Replace `<Your-API-Gateway-URL>` with the actual URL from your deployment.

```bash
curl -X POST <Your-API-Gateway-URL>/docs \
-H "Content-Type: application/json" \
-d '{
    "question": "What is the process for reviewing performance?"
}'
```

### Example Response

The API will return a JSON object containing the answer and the source citation.

```json
{
    "response": "The performance review process involves a self-evaluation, a manager assessment, and a final review meeting.",
    "citation": "s3://your-docs-bucket-name/HR-Performance-Review-Guide.pdf",
    "sessionId": null
}
```

You can use this API endpoint to integrate the chatbot into other applications, scripts, or services.

---

## Security Considerations

This solution is designed with security in mind, adhering to the principles of the **[AWS Well-Architected Framework's Security Pillar](https://aws.amazon.com/architecture/well-architected/?wa-it-neg-wa-pillar-security-rem.cta-well-architected-whitepaper)**.

*   **IAM Least Privilege:**
    *   All IAM roles for Lambda functions are scoped with the minimum necessary permissions.
    *   For example, the Query Lambda is only granted `bedrock:InvokeModel` permission for the specific Claude 3 Sonnet model ARN, not a wildcard (`*`).
    *   The `bedrock:Retrieve` action is restricted to the specific Knowledge Base ARN.

*   **Infrastructure Protection:**
    *   **S3:** The document bucket is configured to **Block All Public Access**. All data is encrypted at rest (`s3.BucketEncryption.S3_MANAGED`).
    *   **CloudFront:** A CloudFront Origin Access Control (OAC) is used, ensuring the S3 bucket is only accessible via CloudFront, not directly. Traffic between viewers and CloudFront is encrypted (HTTPS).
    *   **API Gateway:** The API has no public authentication and relies on throttling for abuse protection. For production use, it is **highly recommended** to add an authorizer (e.g., AWS IAM, Lambda Authorizer, or Amazon Cognito).

*   **Responsible AI:**
    *   **Amazon Bedrock Guardrails:** A Guardrail is configured to filter both user inputs and model outputs for harmful content across four categories (Hate, Insults, Sexual, Violence). This helps ensure the chatbot behaves responsibly.

*   **Data Protection:**
    *   The S3 bucket for documents is **versioned**, providing protection against accidental deletions or overwrites.

---

## Performance Tuning

While the serverless architecture scales automatically, you can tune several aspects for performance and cost.

*   **Lambda Memory:**
    *   All Lambda functions are currently configured with the default memory size (128 MB).
    *   **Recommendation:** For production workloads, analyze the CloudWatch Logs for "Max Memory Used" for each function. Adjust the `memorySize` property in `backend/lib/backend-stack.ts` to be slightly higher than the max used. This can significantly improve performance, especially for the `query` Lambda. You can also use the **[AWS Lambda Power Tuning](https://github.com/alexcasalboni/aws-lambda-power-tuning)** tool to find the optimal memory setting.

*   **Knowledge Base Chunking:**
    *   The chunking strategy is set to 500 tokens with a 20% overlap in `backend/lib/backend-stack.ts`.
    *   **Recommendation:** If you are getting incomplete or fragmented answers, you may need to adjust this. A larger `maxTokens` value provides more context to the model but can be less precise. Experiment with these values based on your document structure.

*   **API Gateway Throttling:**
    *   The API Usage Plan is currently set to a generous 100 requests/second with a burst of 200.
    *   **Recommendation:** Adjust the `rateLimit` and `burstLimit` in `backend/lib/backend-stack.ts` to match your expected user load and to control costs.

---

## Cost Estimation

The cost of this serverless solution depends entirely on usage. For low to moderate traffic, the costs are minimal and often fall within the **[AWS Free Tier](https://aws.amazon.com/free/)**.

### Cost Breakdown by Service:

*   **Amazon Bedrock:**
    *   **Pricing Model:** Pay-per-tokens processed (both input and output).
    *   **Knowledge Base:** You pay for the vector storage in Amazon OpenSearch Serverless and for the data ingestion.
    *   **Reference:** [Amazon Bedrock Pricing](https://aws.amazon.com/bedrock/pricing/)

*   **AWS Lambda:**
    *   **Pricing Model:** Pay-per-request and per-GB-second of compute time.
    *   **Free Tier:** 1 million requests and 400,000 GB-seconds per month.
    *   **Reference:** [AWS Lambda Pricing](https://aws.amazon.com/lambda/pricing/)

*   **Amazon S3:**
    *   **Pricing Model:** Pay-per-GB for storage and per-request for data transfer.
    *   **Free Tier:** 5 GB of standard storage and 20,000 Get/2,000 Put requests per month.
    *   **Reference:** [Amazon S3 Pricing](https://aws.amazon.com/s3/pricing/)

*   **Amazon API Gateway:**
    *   **Pricing Model:** Pay-per-million API calls.
    *   **Free Tier:** 1 million REST API calls per month.
    *   **Reference:** [Amazon API Gateway Pricing](https://aws.amazon.com/api-gateway/pricing/)

*   **Amazon CloudFront:**
    *   **Pricing Model:** Pay-per-GB of data transfer out and per-10,000 requests.
    *   **Free Tier:** 1 TB of data transfer out and 10 million requests per month.
    *   **Reference:** [Amazon CloudFront Pricing](https://aws.amazon.com/cloudfront/pricing/)

*   **Other Services (CloudWatch, SQS, SNS):** These services have generous free tiers, and costs are typically negligible for this application's scale.

### Example Scenario:

*Assumptions: 100 documents (avg. 5 pages each), 500 user queries per month.*
*   **Bedrock Ingestion:** A one-time cost to ingest the 100 documents.
*   **Bedrock Queries:** Cost for 500 queries (input tokens from context + output tokens from answer).
*   **Lambda, API Gateway, S3, CloudFront:** Likely to fall entirely within the Free Tier.
*   **Estimated Monthly Cost:** Likely **under $5**, with the majority of the cost coming from Bedrock inference.

**Note:** This is a rough estimate. Use the **[AWS Pricing Calculator](https://calculator.aws/)** to create a more detailed estimate based on your specific workload.

---

## Troubleshooting Guide

This guide covers common issues that may arise during deployment or runtime.

### Deployment Failures

*   **Error: `Bedrock model access denied...` (from our custom resource)**
    *   **Cause:** You have not enabled access to the required Bedrock models.
    *   **Solution:** Follow the **CRITICAL PRE-DEPLOYMENT STEP** at the top of this README to enable `amazon.titan-embed-text-v1` and `anthropic.claude-3-sonnet-20240229-v1:0`.

*   **Error: `Cannot connect to the Docker daemon`**
    *   **Cause:** The CDK requires Docker to bundle Lambda function assets, and the Docker daemon is not running.
    *   **Solution:** Start Docker Desktop and ensure it is operational. You can test this by running `docker ps` in your terminal.

*   **Error: `DELETE_FAILED` on `BucketNotificationsHandler` or `AccessDenied` for `PutBucketNotificationConfiguration`**
    *   **Cause:** The CloudFormation stack is in a broken state from a previous failed deployment. This was a bug in earlier versions of this repository.
    *   **Solution:** You must completely destroy the stack to clear the broken resource, then redeploy.
        ```bash
        cdk destroy
        cdk deploy
        ```

### Runtime Errors

*   **Chatbot returns "Server side error: please check function logs"**
    *   **Cause 1: Bedrock Model Access (Post-Deployment):** You may have had access when you deployed, but it was later revoked.
    *   **Cause 2: Invalid File Type:** You may have uploaded a corrupted or unsupported file, causing the ingestion to fail.
    *   **Solution:**
        1.  Check the CloudWatch Logs for the `query-bedrock-llm` Lambda function for the specific error message.
            ```bash
            aws logs tail /aws/lambda/query-bedrock-llm --since 10m
            ```
        2.  If the error mentions `AccessDeniedException` for `bedrock:InvokeModel`, re-verify your model access in the Bedrock console.

*   **Ingestion Status shows "FAILED"**
    *   **Cause:** The document you uploaded may be corrupted, password-protected, or in an unsupported format.
    *   **Solution:**
        1.  Check the CloudWatch Logs for the `start-ingestion-trigger` Lambda function.
        2.  If the ingestion repeatedly fails, try re-saving the document or testing with a different file.
        3.  If failures are persistent, a message will be sent to the Dead Letter Queue (DLQ). You can view the failed event message in the SQS console in the `ingestion-failures-dlq` queue.

*   **Guardrail blocks a seemingly normal question**
    *   **Cause:** The Guardrail's sensitivity might be too high for your use case, or the question may contain subtle language that triggers a filter.
    *   **Solution:** You can adjust the `inputStrength` and `outputStrength` for the filters in `backend/lib/backend-stack.ts` from `HIGH` to `MEDIUM` or `LOW`.

---

## Alternatives Considered

Several architectural decisions were made during the development of this solution. Here are some alternatives and the reasons for the final choices.

*   **`RetrieveAndGenerate` API vs. Manual "Retrieve then Invoke"**
    *   **Alternative:** The `RetrieveAndGenerate` API is a single, convenient call that performs both retrieval and answer generation.
    *   **Reason for Choice:** The `RetrieveAndGenerate` API **does not support Bedrock Guardrails**. To implement the critical Responsible AI feature, we had to switch to the manual two-step process. This gives us more control and allows Guardrail integration, but comes at the cost of losing automatic citation generation, which we had to re-implement manually.

*   **Amazon OpenSearch Serverless vs. Other Vector Stores**
    *   **Alternative:** We could have used other vector stores like Pinecone, Redis, or a self-managed OpenSearch cluster.
    *   **Reason for Choice:** Amazon OpenSearch Serverless is the managed vector store used by Bedrock Knowledge Bases. It is fully integrated, requires no management overhead, and scales automatically. For a production-ready, low-maintenance solution, it is the ideal choice.

*   **Different Foundation Models**
    *   **Alternative:** We could have used other models like Claude 3 Haiku or Amazon Titan.
    *   **Reason for Choice:** Claude 3 Sonnet provides an excellent balance of performance, intelligence, and cost. Hardcoding to a single model simplifies the architecture, as different model providers (e.g., Anthropic vs. Amazon) require different API request/response structures.

---

## Cleanup

To remove all the AWS resources created by this project, you can destroy the CDK stack.

1.  Navigate to the `backend` directory.
2.  Run the destroy command:
    ```bash
    cdk destroy
    ```

This will delete the S3 buckets, Lambda functions, API Gateway, and all other associated resources.



