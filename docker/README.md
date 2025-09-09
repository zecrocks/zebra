# Zebra Docker Multi-Architecture Support

This directory contains Docker configurations for building and running Zebra with multi-architecture support.

## Supported Architectures

- `linux/amd64` (x86_64)
- `linux/arm64` (ARM64/AArch64)

## Docker Images

### Production Images
- `zfnd/zebra:latest` - Latest release (multi-arch)
- `zfnd/zebra:v1.x.y` - Specific version (multi-arch)
- `zfnd/zebra:latest-experimental` - Latest with experimental features (multi-arch)

### Development Images
- `zfnd/zebra:main-<sha>` - Built from main branch (multi-arch)
- `zfnd/zebra:edge` - Latest main branch build (multi-arch)

## Building Locally

### Single Architecture (current platform)
```bash
docker build -f docker/Dockerfile -t zebra:local .
```

### Multi-Architecture
```bash
# Set up buildx for multi-arch
docker buildx create --use --name multiarch

# Build for multiple architectures
docker buildx build \
  --platform linux/amd64,linux/arm64 \
  -f docker/Dockerfile \
  -t zebra:multiarch \
  --push .
```

## Running

### Using Docker Compose
```bash
cd docker
docker-compose up -d
```

### Direct Docker Run
```bash
# The image will automatically use the correct architecture
docker run -d \
  --name zebra \
  -p 8233:8233 \
  -v zebra-data:/var/cache/zebrad-cache \
  zfnd/zebra:latest
```

## CI/CD Workflows

### Automatic Multi-Architecture Builds

1. **Main Branch Pushes**: Triggers multi-arch builds tagged as `edge`
2. **Releases**: Triggers multi-arch builds with version tags and `latest`
3. **Manual Triggers**: Can be triggered manually via GitHub Actions

### GitHub Actions Workflows

- `build-multiarch-docker.yml` - Dedicated multi-architecture build workflow
- `sub-build-docker-image.yml` - Updated to support multi-arch via `multiarch` parameter
- `release-binaries.yml` - Updated to build multi-arch on releases

## Configuration

### Environment Variables
- `FEATURES` - Rust features to enable (default: `default-release-binaries`)
- `RUST_LOG` - Logging level (default: `info`)
- `ZEBRA_CONF_DIR` - Config directory (default: `/etc/zebrad`)
- `ZEBRA_CONF_FILE` - Config file name (default: `zebrad.toml`)

### Volumes
- `/var/cache/zebrad-cache` - Blockchain data and cache
- `/etc/zebrad` - Configuration files

### Ports
- `8233` - Mainnet P2P
- `18233` - Testnet P2P
- `8232` - RPC (if enabled)

## Architecture Detection

The multi-architecture images automatically detect and run on the appropriate architecture:

```bash
# Check the architecture of a running container
docker exec <container> uname -m
# x86_64 for AMD64
# aarch64 for ARM64
```

## Troubleshooting

### Build Issues
- Ensure Docker Buildx is installed and configured
- For local multi-arch builds, ensure QEMU is available
- ARM64 builds may take significantly longer than AMD64

### Runtime Issues
- Check that the correct architecture image is being pulled
- Verify platform-specific dependencies are available
- Monitor resource usage (ARM64 may have different performance characteristics)

## Development

When developing with multi-architecture support:

1. Test builds on both architectures when possible
2. Be aware of architecture-specific performance differences
3. Use `docker buildx imagetools inspect <image>` to verify multi-arch manifests
4. Consider using emulation for local cross-architecture testing
