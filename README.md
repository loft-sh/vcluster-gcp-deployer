## Build & Push
```
export PROJECT_ID=$(gcloud config get-value project | tr ':' '/')
export REGION=us
export REGISTRY=${REGION}-docker.pkg.dev/${PROJECT_ID}
export APP_NAME=vcluster-platform
export IMAGE=${REGISTRY}/${APP_NAME}/deployer
export TAG="4.4"

# Make registry public
gcloud artifacts repositories add-iam-policy-binding $APP_NAME \
  --location=$REGION \
  --member="allUsers" \
  --role="roles/artifactregistry.reader" \
  --project=$PROJECT_ID

# Grant registry access to GCP marketplace service account
gcloud artifacts repositories add-iam-policy-binding $APP_NAME \
  --location=$REGION \
  --member="serviceAccount:cloud-commerce-producer@system.gserviceaccount.com" \
  --role="roles/artifactregistry.reader" \
  --project=$PROJECT_ID

# Build and push image
docker buildx build \
      --platform linux/amd64,linux/arm64,linux/arm/v7 \
      --tag ${IMAGE}:${TAG} \
      --tag "${IMAGE}:${TAG}.0" \
      --push .
```

## Test: Verify
```
docker pull gcr.io/cloud-marketplace-tools/k8s/dev

BIN_FILE="$HOME/bin/mpdev"
docker run \
  gcr.io/cloud-marketplace-tools/k8s/dev \
  cat /scripts/dev > "$BIN_FILE"
chmod +x "$BIN_FILE"

kubectl apply -f "https://raw.githubusercontent.com/GoogleCloudPlatform/marketplace-k8s-app-tools/master/crd/app-crd.yaml"

$BIN_FILE /scripts/doctor

$BIN_FILE verify \
  --deployer=$IMAGE:$TAG
```


## Test: Manual Deployment
```
export NAMESPACE=test-1
kubectl create namespace $NAMESPACE

$BIN_FILE install \
  --deployer=$IMAGE:$TAG \
  --parameters='{"name": "test-deployment", "namespace": "'$NAMESPACE'"}'

kubectl get po --all-namespaces
kubectl logs -n $NAMESPACE ...

kubectl get clusterrolebindings | grep -i deployer
kubectl get clusterrolebindings -o json | jq -r '
  .items[] | select(.subjects[]? |
    .kind=="ServiceAccount" and .name=="test-deployment-deployer-sa" and .namespace=="$NAMESPACE") |
  .metadata.name'
```

## View schema.yaml
```
docker run --rm --entrypoint "" -it $IMAGE:$TAG cat /data/schema.yaml
```
