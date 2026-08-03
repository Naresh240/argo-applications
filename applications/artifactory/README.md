# Generate Credentials before going to setup Artifactory

```bash
export MASTER_KEY=$(openssl rand -hex 32)
export JOIN_KEY=$(openssl rand -hex 32)

echo $MASTER_KEY
echo $JOIN_KEY

kubectl create namespace artifactory

kubectl create secret generic artifactory-mandatory-keys \
  -n artifactory \
  --from-literal=master-key=${MASTER_KEY} \
  --from-literal=join-key=${JOIN_KEY}
```