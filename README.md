# Set the release version
export CS_VERSION=v1.23.3
# Apply core Config Sync manifests to your cluster
kubectl apply -f "https://github.com/GoogleContainerTools/kpt-config-sync/releases/download/${CS_VERSION}/config-sync-manifest.yaml"