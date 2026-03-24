# Changelog

All notable changes to this project will be documented in this file.

The format is based on [Keep a Changelog](https://keepachangelog.com/en/1.0.0/),
and this project adheres to [Semantic Versioning](https://semver.org/spec/v2.0.0.html).

## [0.1.0] - 2026-03-24

### Added
- Initial release of Valkey Sentinel HA Helm chart
- Valkey StatefulSet with 3 replicas (1 master + 2 replicas)
- Sentinel Deployment with 3 instances for automatic failover
- Password authentication enabled by default
  - Auto-generated secure passwords
  - Support for custom passwords
  - Support for existing secrets
- Persistent storage with configurable PVCs
- Security features:
  - PodSecurity compliance (restricted)
  - Non-root containers
  - Dropped capabilities
  - Seccomp profiles
- OpenShift compatibility via `values-openshift.yaml`
- Configurable resources, probes, and affinity
- Comprehensive documentation in README.md
- Post-installation NOTES with usage instructions

### Features
- **High Availability**: Automatic failover with Sentinel quorum
- **Data Persistence**: Configurable persistent volumes (default 2Gi)
- **Secure by Default**: Password authentication enabled
- **Production Ready**: Health checks, resource limits, security contexts
- **Flexible Configuration**: Extensive values.yaml options

### Configuration
- Default configuration:
  - 3 Valkey replicas
  - 3 Sentinel instances
  - Quorum: 2
  - Authentication: enabled
  - Persistence: 2Gi PVC per pod
  - Failover timeout: 10 seconds

### Documentation
- Comprehensive README with examples
- Python and Node.js client code samples
- Troubleshooting guide
- Security best practices

[0.1.0]: https://github.com/yourusername/valkey-sentinel-ha-helm-chart/releases/tag/v0.1.0
