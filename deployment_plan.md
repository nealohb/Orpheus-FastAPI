# Orpheus FastAPI Deployment Plan (Cloud Run)

**Goal:** Deploy a two-component TTS service (FastAPI frontend + Inference backend) on Google Cloud Run, using a GPU for the backend, scaling to zero, loading the model from GCS, and requiring an API key for access.

**Project ID:** `gen-lang-client-0479297407`
**Region:** `us-central1`
**Base Service Name:** `orpheus`

**Architecture:**

1.  **`orpheus-backend` (Cloud Run Service):** Runs the TTS inference engine (e.g., llama.cpp server) using an `nvidia-t4` GPU. Loads the `.gguf` model from a GCS bucket mounted as a volume. Scales to zero. Allows internal traffic only.
2.  **`orpheus-frontend` (Cloud Run Service):** Runs the FastAPI application (`app.py`) using a CPU instance. Receives external requests, checks for a valid API key (passed via `X-API-Key` header from Supabase Edge Function), and forwards valid requests to the `orpheus-backend` service. Scales to zero. Allows external traffic but requires the API key check within the app.
3.  **Google Cloud Storage (GCS):** Stores the large `.gguf` model file in bucket `orpheus-models-gen-lang-client-0479297407`.
4.  **Google Secret Manager:** Securely stores the API key (`orpheus-api-key`) used by the frontend.
5.  **Google Artifact Registry:** Stores the Docker images (`orpheus-repo`) for both services.
6.  **Google Cloud Build:** Builds the Docker images automatically.

**Detailed Plan:**

**Phase 1: Prerequisites & Setup**

1.  **Install/Configure Tools:** Ensure `gcloud` CLI and Docker are installed and configured. Log in (`gcloud auth login`) and set project (`gcloud config set project gen-lang-client-0479297407`).
2.  **Enable APIs:**
    ```bash
    gcloud services enable \
      run.googleapis.com \
      artifactregistry.googleapis.com \
      cloudbuild.googleapis.com \
      secretmanager.googleapis.com \
      storage.googleapis.com \
      compute.googleapis.com
    ```
3.  **Create GCS Bucket:**
    ```bash
    export BUCKET_NAME=orpheus-models-gen-lang-client-0479297407
    gsutil mb -p gen-lang-client-0479297407 -l us-central1 gs://${BUCKET_NAME}
    ```
4.  **Upload Model:** (Replace placeholders)
    ```bash
    gsutil cp path/to/your-model.gguf gs://${BUCKET_NAME}/your-model.gguf
    ```
5.  **Create API Key Secret:** (Replace `YOUR_STRONG_API_KEY`)
    ```bash
    export API_KEY="YOUR_STRONG_API_KEY"
    echo -n "${API_KEY}" | gcloud secrets create orpheus-api-key --data-file=- --replication-policy=automatic --project=gen-lang-client-0479297407
    ```
6.  **Create Artifact Registry Repo:**
    ```bash
    gcloud artifacts repositories create orpheus-repo \
      --repository-format=docker \
      --location=us-central1 \
      --description="Orpheus Docker repository" \
      --project=gen-lang-client-0479297407
    ```

**Phase 2: Backend Inference Service (`orpheus-backend`)**

7.  **Create `Dockerfile.backend`:**
    *   Create a new Dockerfile for the inference server (e.g., using `llama.cpp`).
    *   Base on `nvidia/cuda:12.x.x-base-ubuntu22.04` or similar.
    *   Install dependencies (Python, pip, `llama-cpp-python[server]`, etc.).
    *   Define `CMD` to start the server, loading model from `/models/your-model.gguf` (GCS mount), listening on port `1234`, using GPU.
    *   *(Requires specific implementation based on chosen inference server)*.
8.  **Build & Push Backend Image (Locally):**
    *   Configure Docker credential helper for Artifact Registry:
        ```bash
        gcloud auth configure-docker us-central1-docker.pkg.dev
        ```
    *   Build the image locally:
        ```bash
        # Use buildx for --gpus support. --load makes the image available locally.
        docker buildx build --load --progress=plain --gpus all -t us-central1-docker.pkg.dev/gen-lang-client-0479297407/orpheus-repo/orpheus-backend:latest -f Dockerfile.backend .
        ```
    *   Push the image to Artifact Registry:
        ```bash
        docker push us-central1-docker.pkg.dev/gen-lang-client-0479297407/orpheus-repo/orpheus-backend:latest
        ```
9.  **Deploy Backend Service:**
    ```bash
    # Note: Command below uses bash syntax (\). For PowerShell, use backticks (`)
    gcloud run deploy orpheus-backend \
      --image=us-central1-docker.pkg.dev/gen-lang-client-0479297407/orpheus-repo/orpheus-backend:latest \
      --region=us-central1 \
      --port=1234 \
      --execution-environment=gen2 \
      --ingress=internal \
      --no-allow-unauthenticated \
      --gpu 1 \
      --gpu-type nvidia-l4 \
      --no-gpu-zonal-redundancy \
      --memory=16Gi \
      --cpu=4 \
      --add-volume="name=model-storage,type=cloud-storage,bucket=$BUCKET_NAME" \
      --add-volume-mount="volume=model-storage,mount-path=/models" \
      --max-instances=1 \
      --no-cpu-throttling \
      --timeout=600 \
      --project=gen-lang-client-0479297407
      # Note: May need to grant service account GCS read access.
      # Note: Ensure llama-cpp-python env vars (e.g., N_GPU_LAYERS) are set in Dockerfile.backend.
      # Note: Default concurrency (80) might be too high. Consider adjusting --concurrency based on performance.
    ```
10. **Get Backend URL:** Note the internal URL (`https://orpheus-backend-...a.run.app`).

**Phase 3: Frontend FastAPI Service (`orpheus-frontend`)**

11. **Modify `app.py` / `tts_engine/inference.py` for Auth:**
    *   Implement FastAPI dependency in `app.py` to read API key from `API_KEY_SECRET` env var (populated by Secret Manager).
    *   Check `X-API-Key` header against the secret key. Reject requests with 401 if key is missing/invalid.
    *   Modify `tts_engine/inference.py` to fetch OIDC token for backend IAP authentication.
12. **Build & Push Frontend Image (Locally):** (Using `Dockerfile.cpu`)
    *   Ensure Docker credential helper for Artifact Registry is configured (see Phase 2, Step 8).
    *   Build the image locally:
        ```bash
        docker build -t us-central1-docker.pkg.dev/gen-lang-client-0479297407/orpheus-repo/orpheus-frontend:latest -f Dockerfile.cpu .
        ```
    *   Push the image to Artifact Registry:
        ```bash
        docker push us-central1-docker.pkg.dev/gen-lang-client-0479297407/orpheus-repo/orpheus-frontend:latest
        ```
13. **Deploy Frontend Service:**
    ```bash
    # Set BACKEND_URL from step 10
    export BACKEND_URL="https://orpheus-backend-...a.run.app"

    gcloud run deploy orpheus-frontend \
      --image=us-central1-docker.pkg.dev/gen-lang-client-0479297407/orpheus-repo/orpheus-frontend:latest \
      --region=us-central1 \
      --port=5005 \
      --execution-environment=gen2 \
      --allow-unauthenticated \
      --memory=1Gi \
      --cpu=1 \
      --update-secrets=API_KEY_SECRET=orpheus-api-key:latest \
      --set-env-vars=ORPHEUS_API_URL=${BACKEND_URL}/v1/completions \
      # Add other necessary env vars from .env.example
      --project=gen-lang-client-0479297407
      # Note: May need to grant service account Secret Manager & Run Invoker roles.
    ```
14. **Get Frontend URL:** Note the public URL.

**Phase 4: Testing**

15. **Test Invocation:** Use `curl` or similar to call the public `orpheus-frontend` URL via your Supabase Edge Function, ensuring the `X-API-Key` header is correctly set. Verify success and test invalid key scenarios (expect 401).