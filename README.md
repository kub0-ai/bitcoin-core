# bitcoin-core

Parameterized Bitcoin Core Docker image and Helm chart for Kubernetes.

## Docker

Single parameterized Dockerfile supporting any Bitcoin Core version. Multi-arch: `linux/amd64`, `linux/arm64`.

```bash
# Build locally
docker buildx build \
  --build-arg VERSION=$(cat VERSION) \
  --build-arg "KEYS=$(tr '\n' ' ' < KEYS)" \
  --platform linux/amd64 \
  -t bitcoin-core:$(cat VERSION) \
  docker/
```

Image: `ghcr.io/kub0-ai/bitcoin-core`

## Helm Chart

StatefulSet-based deployment with persistent storage for blockchain data.

```bash
helm install bitcoin ./chart \
  --set persistence.storageClass=block-sata \
  --set persistence.size=700Gi
```

## Version Management

- `VERSION` — current Bitcoin Core version (single source of truth)
- `KEYS` — GPG signing key fingerprints for binary verification
- `watch-release.yml` — daily cron checks upstream for new releases, opens PR
- `backfill.yml` — manual dispatch to build a specific historical version

## License

MIT
