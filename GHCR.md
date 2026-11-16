name: Build and push image

on:
  push:
    tags:
      - '[0-9]+.[0-9]+'   # matches 1.1, 1.2, 1.35, ...

permissions:
  contents: read     # check out the code
  packages: write    # push to ghcr.io (required)

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      # QEMU emulates arm64 on the amd64 runner; Buildx builds multiple platforms at once
      - uses: docker/setup-qemu-action@v3
      - uses: docker/setup-buildx-action@v3

      # Log in to ghcr.io with the built-in token (no secret to create)
      - uses: docker/login-action@v3
        with:
          registry: ghcr.io
          username: ${{ github.actor }}
          password: ${{ secrets.GITHUB_TOKEN }}

      # Produce a lowercase image name and two tags: the git tag (e.g. 1.2) and latest
      - id: meta
        uses: docker/metadata-action@v5
        with:
          images: ghcr.io/${{ github.repository }}
          tags: |
            type=ref,event=tag
            type=raw,value=latest

      # Build both architectures and push them as one multi-arch image
      - uses: docker/build-push-action@v6
        with:
          context: .
          platforms: linux/amd64,linux/arm64
          push: true
          tags: ${{ steps.meta.outputs.tags }}
          labels: ${{ steps.meta.outputs.labels }}
