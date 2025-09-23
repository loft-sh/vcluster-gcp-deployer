# vcluster-gcp-deployer

## Build & Push
```
export REGISTRY=gcr.io/$(gcloud config get-value project | tr ':' '/')
export APP_NAME=vcluster-platform

docker build --tag $REGISTRY/$APP_NAME/deployer:4.4 .

docker push $REGISTRY/$APP_NAME/deployer:4.4
```