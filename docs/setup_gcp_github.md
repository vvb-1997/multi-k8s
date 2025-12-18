Before beginning, make sure you have created a GKE Autopilot cluster as directed here: \
[Create GCP K8S Cluster](setup_gcp_k8s_cluster.md)


Install the Google Cloud CLI tools for your host operating system:

https://cloud.google.com/sdk/docs/install-sdk


Run the following command:

`gcloud auth login`

Log in with the Google Account that is associated with your Google Cloud project.

Then, set the GCP project using the CLI (Replace PROJECT_ID with your actual GCP project ID)

`gcloud config set project PROJECT_ID`

Locally using your computer's terminal and the Google Cloud CLI, follow the (Preferred) Direct Workload Identity Federation steps 1-4 exactly (remembering to replace your ${PROJECT_ID} with your full project ID (not the shorter project name).

https://github.com/google-github-actions/auth?tab=readme-ov-file#preferred-direct-workload-identity-federation

1.  Create a Workload Identity Pool:

    ```sh
    # TODO: replace ${PROJECT_ID} with your value below.

    gcloud iam workload-identity-pools create "github" \
      --project="${PROJECT_ID}" \
      --location="global" \
      --display-name="GitHub Actions Pool"
    ```

1.  Get the full ID of the Workload Identity **Pool**:

    ```sh
    # TODO: replace ${PROJECT_ID} with your value below.

    gcloud iam workload-identity-pools describe "github" \
      --project="${PROJECT_ID}" \
      --location="global" \
      --format="value(name)"
    ```

    This value should be of the format:

    ```text
    projects/123456789/locations/global/workloadIdentityPools/github
    ```

1.  Create a Workload Identity **Provider** in that pool:

    **🛑 CAUTION!** Always add an Attribute Condition to restrict entry into the
    Workload Identity Pool. You can further restrict access in IAM Bindings, but
    always add a basic condition that restricts admission into the pool. A good
    default option is to restrict admission based on your GitHub organization as
    demonstrated below. Please see the [security
    considerations][security-considerations] for more details.

    ```sh
    # TODO: replace ${PROJECT_ID} and ${GITHUB_ORG} with your values below.

    gcloud iam workload-identity-pools providers create-oidc "my-repo" \
      --project="${PROJECT_ID}" \
      --location="global" \
      --workload-identity-pool="github" \
      --display-name="My GitHub repo Provider" \
      --attribute-mapping="google.subject=assertion.sub,attribute.actor=assertion.actor,attribute.repository=assertion.repository,attribute.repository_owner=assertion.repository_owner" \
      --attribute-condition="assertion.repository_owner == '${GITHUB_ORG}'" \
      --issuer-uri="https://token.actions.githubusercontent.com"
    ```

    > **❗️ IMPORTANT** You must map any claims in the incoming token to
    > attributes before you can assert on those attributes in a CEL expression
    > or IAM policy!

1.  Extract the Workload Identity **Provider** resource name:

    ```sh
    # TODO: replace ${PROJECT_ID} with your value below.

    gcloud iam workload-identity-pools providers describe "my-repo" \
      --project="${PROJECT_ID}" \
      --location="global" \
      --workload-identity-pool="github" \
      --format="value(name)"
    ```

    Use this value as the `workload_identity_provider` value in the GitHub
    Actions YAML:

    ```yaml
    - uses: 'google-github-actions/auth@v3'
      with:
        project_id: 'my-project'
        workload_identity_provider: '...' # "projects/123456789/locations/global/workloadIdentityPools/github/providers/my-repo"
    ```

    > **❗️ IMPORTANT** The `project_id` input is optional, but may be required
    > by downstream authentication systems such as the `gcloud` CLI.
    > Unfortunately we cannot extract the project ID from the Workload Identity
    > Provider, since it requires the project _number_.
    >
    > It is technically possible to convert a project _number_ into a project
    > _ID_, but it requires permissions to call Cloud Resource Manager, and we
    > cannot guarantee that the Workload Identity Pool has those permissions.

1.  As needed, allow authentications from the Workload Identity Pool to Google
    Cloud resources. These can be any Google Cloud resources that support
    federated ID tokens, and it can be done after the GitHub Action is
    configured.

    The following example shows granting access from a GitHub Action in a
    specific repository a secret in Google Secret Manager.

    Then, perform step #5 from the Direct Provider docs with a slight adjustment:

    First, get your workload Pool ID:

    `gcloud iam workload-identity-pools describe "github" \ --project="YOUR_GCP_PROJECT_ID" \ --location="global" \ --format="value(name)"`

    This will return something like:

    `projects/32953953504/locations/global/workloadIdentityPools/github`

    The number `32953953504` in the above example is the Pool ID. Take that number and use it below:

    Make sure to also replace YOUR_GCP_PROJECT_ID and YOUR_GITHUB_USERNAME/YOUR_GH_REPO_NAME

    ```sh
    gcloud projects add-iam-policy-binding "YOUR_GCP_PROJECT_ID" \
    --role="roles/container.developer" \
    --member="principalSet://iam.googleapis.com/projects/POOL_ID/locations/global/workloadIdentityPools/github/attribute.repository/YOUR_GITHUB_USERNAME/YOUR_GH_REPO_NAME"
    ```

    So, for me, this looked like:

    ```sh
    gcloud projects add-iam-policy-binding "multi-k8s-autopilot-471618" \
    --role="roles/container.developer" \
    --member="principalSet://iam.googleapis.com/projects/32953953504/locations/global/workloadIdentityPools/github/attribute.repository/0x-cygnet/multi-k8s-4-14"
    ```

Next, we can move on to the creation of the GitHub Workflow.

Navigate to your Github repository.

Click Settings

Click Secrets

Click New repository secret

Create key/value pair secrets for DOCKER_USERNAME, DOCKER_PASSWORD

In your local development environment, create a .github directory at the root of your project

Create a workflows directory inside the new .github directory

In the workflows directory, create a deploy.yaml file which should contain the following code (name does not matter):

Note 1 - remember to change your workload_identity_provider, service account, project_id, cluster_name, and location to the values used by your GKE environment and change my DockerHub username and images to your own.
Note 2 _- Pass the value returned from step 4 (Extract the Workload Identity Provider resource name) of the [instructions here](https://github.com/google-github-actions/auth?tab=readme-ov-file#preferred-direct-workload-identity-federation) as the workload_identity_provider in the workflow below.


```yaml
name: Deploy MultiK8s
on:
  push:
    branches:
      - main
 
env:
  SHA: $(git rev-parse HEAD)
 
jobs:
  build:
    runs-on: ubuntu-latest
    permissions:
      contents: "read"
      id-token: "write"
    steps:
      - uses: actions/checkout@v3
 
      - name: Test
        run: |-
          docker login -u ${{ secrets.DOCKER_USERNAME }} -p ${{ secrets.DOCKER_PASSWORD }}
          docker build -t cygnetops/react-test -f ./client/Dockerfile.dev ./client
          docker run -e CI=true cygnetops/react-test npm test
 
      - name: Set Identity Provider
        uses: "google-github-actions/auth@v2"
        with:
          workload_identity_provider: "projects/32953953504/locations/global/workloadIdentityPools/github/providers/my-repo"
 
      - name: Set Project
        uses: google-github-actions/setup-gcloud@v2
        with:
          project_id: multi-k8s-autopilot-471618
 
      - name: Auth
        run: |-
          gcloud --quiet auth configure-docker
 
      - name: Get Credentials
        uses: google-github-actions/get-gke-credentials@v2
        with:
          cluster_name: multi-k8s
          location: us-central1
          project_id: multi-k8s-autopilot-471618
 
      - name: Build
        run: |-
          docker build -t cygnetops/multi-client:latest -t cygnetops/multi-client:${{ env.SHA }} -f ./client/Dockerfile ./client
          docker build -t cygnetops/multi-server:latest -t cygnetops/multi-server:${{ env.SHA }} -f ./server/Dockerfile ./server
          docker build -t cygnetops/multi-worker:latest -t cygnetops/multi-worker:${{ env.SHA }} -f ./worker/Dockerfile ./worker
 
      - name: Push
        run: |-
          docker push cygnetops/multi-client:latest
          docker push cygnetops/multi-server:latest
          docker push cygnetops/multi-worker:latest
 
          docker push cygnetops/multi-client:${{ env.SHA }}
          docker push cygnetops/multi-server:${{ env.SHA }}
          docker push cygnetops/multi-worker:${{ env.SHA }}
 
      - name: Apply
        run: |-
          kubectl apply -f k8s
          kubectl set image deployments/server-deployment server=cygnetops/multi-server:${{ env.SHA }}
          kubectl set image deployments/client-deployment client=cygnetops/multi-client:${{ env.SHA }}
          kubectl set image deployments/worker-deployment worker=cygnetops/multi-worker:${{ env.SHA }}
```