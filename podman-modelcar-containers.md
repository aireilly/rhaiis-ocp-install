# Deploying modelcar containers with Podman

https://developers.redhat.com/articles/2025/01/30/build-and-deploy-modelcar-container-openshift-ai#how_to_build_a_modelcar_container

```cmd
cd /workspace/modelcar-proc/
```

Create a venv

```cmd
python3 -m venv venv
source venv/bin/activate
pip install --upgrade pip
pip install huggingface_hub
```

Create the model downloader python script:

```cmd
vi download_model.py
```

```py
from huggingface_hub import snapshot_download

# Specify the Hugging Face repository containing the model
model_repo = "ibm-granite/granite-3.1-2b-instruct"
snapshot_download(
    repo_id=model_repo,
    local_dir="/models",
    allow_patterns=["*.safetensors", "*.json", "*.txt"],
)
```

Create a modelcar Dockerfile:

```docker
FROM registry.access.redhat.com/ubi9/python-311:latest as base

USER root

RUN pip install huggingface-hub

# Download the model file from hugging face
COPY download_model.py .

RUN python download_model.py 

# Final image containing only the essential model files
FROM registry.access.redhat.com/ubi9/ubi-micro:9.4

# Copy the model files from the base container
COPY --from=base /models /models

USER 1001
```

## Side quest - configuring Podman storage

```cmd
~/.config/containers/storage.conf
```

Add or edit:

```cmd
[storage]
driver = "overlay"
rootless_storage_path = "/workspace/containers/storage"
graphroot = "/workspace/containers/storage"
runroot = "/run/user/$UID"  # or default
```

Optionally delete or move existing storage (~/.local/share/containers/storage) if you want to free that space.

```cmd
podman system migrate
```

```cmd
podman info | grep graphRoot
```

```cmd
podman info | grep imageCopyTmpDir
```

```cmd
export TMPDIR=/workspace/tmp
```

<END INTERLUDE>

## Build the Dockerfile

```cmd
podman build . -t modelcar-example:latest --platform linux/amd64
```