# CI/CD demo: Jenkins -> Docker Hub -> Kubernetes

## Steps
1. Build and test locally:
   - `cd app && docker build -t <DOCKERHUB_USER>/k8s-cicd-demo:local .`

2. Apply k8s manifests:
   - `kubectl apply -f k8s/deployment.yaml`
   - `kubectl apply -f k8s/service.yaml`

3. Configure Jenkins:
   - Create Pipeline job pointing to this repo
   - Add credential id `dockerhub-creds` for Docker Hub
   - Run the pipeline

4. Verify:
   - `kubectl get pods -o wide`
   - `kubectl get svc k8s-cicd-demo-svc -o wide`
