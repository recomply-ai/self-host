# recomply.ai Self-Hosted Deployment

This document outlines the deployment and operation of the recomply.ai compliance platform on your premises using Docker Compose.

## Deployment Requirements

### Prerequisites
- Docker and Docker Compose installed on your system
- Network access to Google Cloud's container registry (us-central1-docker.pkg.dev)
- The `artifact-repository-key.json` file provided by recomply.ai team (necessary for pulling our private docker images)
- GCP account for AI API access

## Setting up the GCP credentials for AI API access

The recomply.ai system uses Google Cloud's Vertex AI to power its compliance screening and classification features. You'll need to set up a Google Cloud service account with appropriate permissions to access these AI services.

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
   
#### Step 3: Create a Service Account

1. Go to "IAM & Admin" > "Service Accounts" in the left sidebar
2. Click "Create Service Account"
3. Fill in the details:
   - **Service account name**: `recomply-ai-vertex`
   - **Description**: `Service account for recomply.ai to access Vertex AI`
4. Click "Create and Continue"

#### Step 4: Assign Permissions

1. In the "Grant this service account access to project" section, add these roles:
   - **Vertex AI User** (allows making AI API calls)
   - **AI Platform Developer** (additional permissions for Vertex AI)
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

#### Step 7: Update the .env file

- In the root directory of the project, open the existing `.env` file and set `GOOGLE_SERVICE_ACCOUNT_CREDS_BASE64`
  to the base64 string you copied in the previous step:
```env
GOOGLE_SERVICE_ACCOUNT_CREDS_BASE64=<your-base64-encoded-key-here>
```

## Running the system

recomply.ai stores their system docker images in Google Cloud's Artifact Registry. To pull these images, you need to first
authenticate with Google Cloud Artifact Registry.

1. **Ensure the artifact-repository-key.json is copied**
   - The `artifact-repository-key.json` file is provided by recomply.ai team. It contains a GCP key required to
     pull our private docker container images. This key ensures secure access to the latest verified
     builds of our software.
   - Copy the `artifact-repository-key.json` file to the root directory of the project.

2. **Authenticate with Google Cloud Artifact Registry**:
   ```bash
   cat artifact-repository-key.json | docker login -u _json_key --password-stdin https://us-central1-docker.pkg.dev
   ```

3. **Start the entire platform**:
   ```bash
   docker compose up -d --pull always
   ```

The system will automatically:
- Pull the latest verified images from our repositories
- Initialize the database with the latest schema
- Start all services in the correct dependency order
- Make the frontend available at http://localhost:3000
- Make the API available at http://localhost:8000

## Using the system

Once the service is running (see the previous step):

1) Go to http://localhost:3000
2) Use the following credentials on the login page:
   - **Username**: `admin`
   - **Password**: `secret123`
3) You may view the API docs at http://localhost:8000/redoc (make sure the system is running before accessing the docs)
4) You will need to configure your system to feed your sanction alerts to our system via the
   POST /api/v1/screening-cases (POST http://localhost:8000/api/v1/screening-cases) endpoint.
   For more information about the endpoint, please see
   http://localhost:8000/redoc#tag/Screening-Cases/operation/create_screening_case_api_v1_screening_cases_post

## Data Persistence

Application data is persisted under `.data/` directory:
- PostgreSQL data: `.data/postgres/`
- Redis data: `.data/redis/`

These directories will be created automatically and contain all your compliance data and system state.
