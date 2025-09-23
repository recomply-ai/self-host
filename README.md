# recomply.ai Self-Hosted Deployment

This document outlines the deployment and operation of the recomply.ai compliance platform on your premises using Docker Compose.

## Deployment Requirements

### Prerequisites
- Docker and Docker Compose installed on your system
- Network access to Google Cloud's container registry (us-central1-docker.pkg.dev)
- The `artifact-repository-key.json` file provided by recomply.ai team (necessary for pulling our private docker images)
- GCP account for AI API access and screenshot storage for the web scraping agent

## Setting up the GCP credentials for AI API access & storage

The recomply.ai system uses Google Cloud's Vertex AI to power its compliance screening and classification features. 
In addition we use Google Cloud's Cloud Storage to store screenshots for the web scraping agent.
You'll need to set up a Google Cloud service account with appropriate permissions to access these AI services and
storage.

#### Step 1: Create a Google Cloud Project (if you don't have one)

1. Go to the [Google Cloud Console](https://console.cloud.google.com/)
2. Sign in with your Google account
3. Click "Select a project" at the top of the page
4. Click "New Project" and give it a meaningful name (e.g., "recomply-ai-production")
5. Note down your **Project ID** (you'll need this later)

#### Step 2: Enable Required APIs

1. In the Google Cloud Console, make sure your project is selected
2. Go to "APIs & Services" > "Library" in the left sidebar
3. Search for **and enable** the following APIs:
   - **Vertex AI API** (this is the main one we need)
   - **Cloud Storage** (this is needed for the web scraping agent to store screenshots)
   
#### Step 3a: Create a Service Account

1. Go to "IAM & Admin" > "Service Accounts" in the left sidebar
2. Click "Create Service Account"
3. Fill in the details:
   - **Service account name**: `recomply-ai-vertex`
   - **Description**: `Service account for recomply.ai to access Vertex AI`
4. Click "Create and Continue"

#### Step 3b: Create a storage bucket for screenshots

1. Go to "Cloud Storage" > "Buckets" in the left sidebar
2. Click "Create" at the top
3. Enter the bucket name such as `recomply-data`
4. Click through until the end, until the bucket is created

#### Step 4: Assign Permissions

1. In the "Grant this service account access to project" section, add these roles:
   - **Vertex AI User** (allows making AI API calls)
   - **AI Platform Developer** (additional permissions for Vertex AI)
   - **Storage Admin** (allows the web scraping agent to store screenshots and create access links)
2. Click "Continue" and then "Done"

#### Step 5: Create and Download the Service Account Key

1. Find your newly created service account in the list
2. Click on the service account name to open its details
3. Go to the "Keys" tab
4. Click "Add Key" > "Create new key"
5. Select "JSON" format and click "Create"
6. A JSON file will be downloaded to your computer - **keep this file secure!**
7. Rename the downloaded file to `gcp-vertex-key.json` for easier reference

#### Step 6: Convert the Key to Base64

You need to convert the JSON key file to a base64 string. Open a terminal and run:

**On macOS/Linux:**
```bash
base64 -i gcp-vertex-key.json
```

**On Windows (PowerShell):**
```powershell
[Convert]::ToBase64String([IO.File]::ReadAllBytes("gcp-vertex-key.json"))
```

Copy the entire base64 output (it will be a long string).

#### Step 7a: Update the .env file (for Docker Compose configuration)

- In the root directory of the project, open the existing `.env` file and set `GOOGLE_SERVICE_ACCOUNT_CREDS_BASE64`
  to the base64 string you copied in the previous step:
```env
GOOGLE_SERVICE_ACCOUNT_CREDS_BASE64=<your-base64-encoded-key-here>
GCP_STORAGE_SERVICE_ACCOUNT_CREDS_BASE64=<the same base64 string as above>
GCP_PROJECT_ID=<GCP project name that you have created this key for (where the storage bucket is created)>
GCP_STORAGE_BUCKET_NAME=<name of the storage bucket that you have created>
OPENSANCTIONS_API_KEY=<provided by recomply.ai team>
BRIGHT_DATA_CDP_WS_URL=<provided by recomply.ai team>
```

#### Step 7b: Update the `docker-swarm.yaml` file (for Docker Swarm configuration)

Docker Swarm does not allow injecting environment variables via `.env` files,
so you need to update the `docker-swarm.yaml` file directly.

- Under the `x-common-environment: &common-environment` section in the `docker-swarm.yaml` file,
  update the `GOOGLE_SERVICE_ACCOUNT_CREDS_BASE64` variable with the base64 string you copied:
```yaml
x-common-environment: &common-environment
  ...
  GOOGLE_SERVICE_ACCOUNT_CREDS_BASE64=<your-base64-encoded-key-here>
  GCP_STORAGE_SERVICE_ACCOUNT_CREDS_BASE64=<the same base64 string as above>
  GCP_PROJECT_ID=<GCP project name that you have created this key for (where the storage bucket is created)>
  GCP_STORAGE_BUCKET_NAME=<name of the storage bucket that you have created>
  OPENSANCTIONS_API_KEY=<provided by recomply.ai team>
  BRIGHT_DATA_CDP_WS_URL=<provided by recomply.ai team>
```

## Running the system

recomply.ai stores their system docker images in Google Cloud's Artifact Registry. To pull these images, you need to first
authenticate with Google Cloud Artifact Registry.

### Prerequisites
1. **Ensure the artifact-repository-key.json is copied**
   - The `artifact-repository-key.json` file is provided by recomply.ai team. It contains a GCP key required to
     pull our private docker container images. This key ensures secure access to the latest verified
     builds of our software.
   - Copy the `artifact-repository-key.json` file to the root directory of the project.

2. **Install Google Cloud SDK (gcloud)**
   - Download and install the [Google Cloud SDK](https://cloud.google.com/sdk/docs/install) for your platform

### Authentication

Authenticate with Google Cloud Artifact Registry using the service account:

```bash
# Authenticate with the service account
gcloud auth activate-service-account --key-file=artifact-repository-key.json

# Configure Docker to use gcloud credentials for the registry
gcloud auth configure-docker us-central1-docker.pkg.dev
```

**For Docker Swarm**: Run these authentication commands on each swarm node before deploying the stack.

### Starting the Platform

For **Docker Compose**:
```bash
docker compose up --pull always
```
(Optionally in background mode if that's preferred)
```bash
docker compose up --pull always -d
```

For **Docker Swarm**:
```bash
docker stack deploy -c docker-swarm.yaml --with-registry-auth recomply
```
to stop, run:
```bash
docker stack rm recomply
```

The system will automatically:
- Pull the latest verified images from our repositories
- Initialize the database with the latest schema
- Start all services in the correct dependency order
- Make the entire system available at http://localhost (both frontend and API)

## Using the system

Once the service is running (see the previous step):

1) Go to <BASE_URL> (where you are hosting the system e.g. http://localhost or http://custom.domain.name)
2) Use the following credentials on the login page:
   - **Username**: `admin`
   - **Password**: `secret123`
3) You may view the API docs at <BASE_URL>/api/redoc (make sure the system is running before accessing the docs)
4) You will need to configure your system to feed your sanction alerts to our system via the
   POST /api/v1/screening-cases (POST <BASE_URL>/api/v1/screening-cases) endpoint.
   For more information about the endpoint, please see
   <BASE_URL>/api/redoc#tag/Screening-Cases-(v1)/operation/create_screening_case_api_v1_screening_cases_post

## Webhook Notifications

The recomply.ai system can send webhook notifications when screening cases are processed. This allows you to integrate with external systems and receive real-time updates about screening results.

### Configuration

Webhook notifications are configured via environment variables:

#### For Docker Compose (.env file):
```env
# Optional: URL where webhook notifications will be sent
NOTIFICATION_WEBHOOK_URL=http://your-webhook-endpoint.com/webhook

# Optional: JSON object containing custom headers for webhook requests
NOTIFICATION_WEBHOOK_HEADERS_JSON={"Authorization": "Bearer your-token", "X-Custom-Header": "value"}
```

#### For Docker Swarm (docker-swarm.yaml):
```yaml
x-common-environment: &common-environment
  ...
  # Optional: URL where webhook notifications will be sent
  NOTIFICATION_WEBHOOK_URL: http://your-webhook-endpoint.com/webhook
  
  # Optional: JSON object containing custom headers for webhook requests
  NOTIFICATION_WEBHOOK_HEADERS_JSON: '{"Authorization": "Bearer your-token", "X-Custom-Header": "value"}'
```

### Webhook Payload

When a screening case is processed, the system will send a POST request to the configured webhook URL with a JSON payload. The payload schema is equivalent to the response from the `GET /api/v1/screening-cases/{id}` endpoint.

For detailed information about the webhook payload structure, see the API documentation at:
`<BASE_URL>/api/redoc#tag/Screening-Cases-(v1)/operation/get_screening_case_api_v1_screening_cases__case_id__get`

### Example Configuration

**Basic webhook setup:**
```env
NOTIFICATION_WEBHOOK_URL=https://your-integration.com/api/webhooks/recomply
```

**Webhook with authentication:**
```env
NOTIFICATION_WEBHOOK_URL=https://your-integration.com/api/webhooks/recomply
NOTIFICATION_WEBHOOK_HEADERS_JSON={"Authorization": "Bearer your-api-token"}
```

### Notes

- Both environment variables are optional. If `NOTIFICATION_WEBHOOK_URL` is not set, no webhook notifications will be sent.
- The webhook request is sent asynchronously after the screening case processing is complete.
- If the webhook request fails (e.g., network issues, invalid URL), the system will retry sending the webhook for a day with backoff. After a day of retrying, the system will stop retrying and log the error.
- Ensure your webhook endpoint can handle POST requests with JSON payloads and responds with appropriate HTTP status code (2xx).

## Custom Domain

To use a custom domain to access the system, let's assume you are running the system on a server with IP `a.b.c.d`,
then you need to create a DNS A record that maps your chosen hostname (e.g., `custom.domain.name`) to `a.b.c.d`.
This will allow you to access the system at `http://custom.domain.name`.

## Important Considerations

Please do not modify `docker-compose.yaml` - this may introduce unexpected behaviour, and will make it harder for you
to consume system updates due to potential merge conflicts. Any configuration on your end should be done via
the `.env` file, custom domain mappings or reverse proxy configurations. If you want certain parts of the
system to be configurable, please speak to the recomply.ai team.
