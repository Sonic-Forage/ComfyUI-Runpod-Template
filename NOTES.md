
# MindBotz ComfyUI — Docker Build Cloud Notes

## Configuration

| Setting | Value |
|---|---|
| Docker Cloud account | `mindbotz` |
| Cloud builder | `mindbotz-builder` |
| Buildx builder | `cloud-mindbotz-mindbotz-builder` |
| Docker Hub image | `m1ndb0tz/comfyui` |
| ComfyUI version | `v0.42.5` |
| CUDA target | `cuda13` |
| Image tag | `v0.42.5-cuda13.0` |

## 1. Install Docker CLI + Buildx (Ubuntu 24.04)

Install Docker's official APT repository first:
https://docs.docker.com/engine/install/ubuntu/

Then:

```bash
apt-get update
apt-get install -y docker-ce-cli docker-buildx-plugin
```

Verify:

```bash
docker --version
docker buildx version
```

Docker Build Cloud requires Buildx 0.37.0 or newer.

A local Docker daemon is not required.

## 2. Log into Docker Hub

```bash
docker login -u mindbotz
```

Use a Docker Hub access token with the appropriate permissions.

Important: The cloud builder account is `mindbotz`, while the image repository is `m1ndb0tz/comfyui`.

Your Docker Hub credentials must have permission to push to that image repository.

Never commit authentication tokens.

## 3. Connect to the Cloud Builder

The builder must already exist at:
https://app.docker.com/

```bash
docker buildx create --driver cloud mindbotz/mindbotz-builder
```

This registers the builder:

```text
cloud-mindbotz-mindbotz-builder
```

Verify:

```bash
docker buildx ls
```

If the builder is already registered, skip the create command.

## 4. Open the Correct Project

```bash
cd /workspace/ComfyUI-Runpod-Template
```

Check the files:

```bash
ls -la
```

Validate the CUDA 13 build definition:

```bash
docker buildx bake cuda13 --print
```

Make sure the project contains `docker-bake.hcl` or another supported Bake file.

## 5. Build, Tag, and Push CUDA 13

```bash
docker buildx bake cuda13 \
  --builder cloud-mindbotz-mindbotz-builder \
  --set "cuda13.tags=m1ndb0tz/comfyui:v0.42.5-cuda13.0" \
  --push \
  --progress=plain
```

Expected image:

```text
m1ndb0tz/comfyui:v0.42.5-cuda13.0
```

This builds the image remotely and pushes it directly to Docker Hub.

## 6. Build a New Version

When updating ComfyUI:

1. Change `COMFYUI_VERSION` in `docker-bake.hcl`.
2. Confirm the ComfyUI release exists.
3. Change the Docker image tag to match.
4. Run the same build command.

Example:

```bash
docker buildx bake cuda13 \
  --builder cloud-mindbotz-mindbotz-builder \
  --set "cuda13.tags=m1ndb0tz/comfyui:v0.43.0-cuda13.0" \
  --push \
  --progress=plain
```

## 7. Docker Build Cloud Disk Settings

Open:
https://app.docker.com/

Go to Docker Build Cloud → Builder settings → Disk allocation.

For large CUDA/PyTorch builds:

- Previous cache allocation: 80%
- Suggested starting allocation: 30%
- Lower cache allocation can leave more storage for active builds.
- Less cache space may increase future build times.

Inspect cache usage:

```bash
docker buildx du --builder cloud-mindbotz-mindbotz-builder
```

## Troubleshooting

### Error: Cannot connect to Docker daemon

```text
Cannot connect to unix:///var/run/docker.sock
```

Cause: Docker is trying to use the local default builder.

Fix: Specify:

```bash
--builder cloud-mindbotz-mindbotz-builder
```

### Error: 404 builder instances not found

Cause: Incorrect builder namespace, nonexistent builder, or missing access.

Correct identifier:

```text
mindbotz/mindbotz-builder
```

Not:

```text
m1ndb0tz/mindbotz-builder
```

### Error: No builder found

Connect it:

```bash
docker buildx create --driver cloud mindbotz/mindbotz-builder
```

### Error: Couldn't find a bake definition

Find the project:

```bash
find /workspace -name docker-bake.hcl
```

Run the build from the directory containing that file.

### Error: Custom node archive returns 404

Check the version pins in `docker-bake.hcl`.

Use valid repository branches or commit hashes.

For the custom-node repositories previously configured, `main` works where `latest` did not.

### Error: No space left on device

The remote build worker ran out of storage while resolving and hashing dependencies.

Check:

```bash
docker buildx du --builder cloud-mindbotz-mindbotz-builder
```

Possible solutions:

1. Reduce cache allocation in Docker Build Cloud settings.
2. Optimize `pip-compile --generate-hashes`.
3. Pre-generate dependency lockfiles.
4. Use a builder with more available storage.

Increasing your RunPod Pod's disk does not increase Docker Build Cloud's storage.

### Docker credential warning

If authentication succeeds but Docker warns about unencrypted credentials, configure a credential helper:

https://docs.docker.com/go/credential-store/

## Quick Commands

Reconnect if necessary:

```bash
docker login -u mindbotz
docker buildx create --driver cloud mindbotz/mindbotz-builder
```

Build and push:

```bash
cd /workspace/ComfyUI-Runpod-Template

docker buildx bake cuda13 \
  --builder cloud-mindbotz-mindbotz-builder \
  --set "cuda13.tags=m1ndb0tz/comfyui:v0.42.5-cuda13.0" \
  --push \
  --progress=plain
```

## Links

- Docker Cloud: https://app.docker.com/
- GitHub: https://github.com/Sonic-Forage/ComfyUI-Runpod-Template
- Docker Hub: https://hub.docker.com/r/m1ndb0tz/comfyui
- Build Cloud documentation: https://docs.docker.com/build-cloud/

## Current Status

- Cloud builder: Created and connected.
- Local Docker daemon: Not required.
- Build reached: Python dependency resolution.
- Last failure: No space left on device during `pip-compile`.
- Successful push: Not yet confirmed.
