# Install RHAIIS on OCP

GPU Operators already installed.

```cmd
HF_TOKEN=blah

echo $NAMESPACE
aireilly-rhaiis

oc create secret generic hf-secret --from-literal=HF_TOKEN=$HF_TOKEN -n $NAMESPACE
```