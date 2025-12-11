# AGENTS.md

This file provides guidance for AI agents working with the catalogd codebase.

## Critical Information

**REPOSITORY STATUS**: This repository is in **maintenance mode**.
- The `main` branch is **NO LONGER UPDATED** as of January 2025
- All changes (CVEs, bug fixes) must be made against the `release-4.18` branch
- Catalogd's functionality has been integrated into [operator-controller](https://github.com/openshift/operator-framework-operator-controller)
- Support continues for OLM v1 GA in OpenShift 4.18

## Project Overview

**catalogd** is a Kubernetes extension that unpacks file-based catalog (FBC) content for on-cluster clients. It serves as the catalog metadata service for OLM v1.

### Core Functionality
- Unpacks FBC content packaged as container images
- Hosts catalog metadata for Kubernetes extensions (Operators, controllers)
- Provides HTTP API for querying catalog content
- Manages ClusterCatalog custom resources

### Key Components
- **ClusterCatalog CRD**: Defines catalog sources (typically container images)
- **catalogd-controller**: Reconciles ClusterCatalog resources
- **HTTP Server**: Serves unpacked catalog content via REST API
- **Storage**: In-memory caching of unpacked FBC content

## Architecture Patterns

### API Structure
```
/catalogs/{catalog-name}/api/v1/all
```
Returns all catalog content as newline-delimited JSON with schemas:
- `olm.package`: Available packages
- `olm.channel`: Update channels for packages
- `olm.bundle`: Specific operator versions

### Resource Lifecycle
1. User creates ClusterCatalog pointing to an image
2. catalogd pulls and unpacks the image
3. Content is stored and served via HTTP
4. Status conditions track progression: Progressing → Serving

### Controller Pattern
Uses standard Kubernetes controller-runtime patterns:
- Reconciliation loops
- Status conditions
- Finalizers for cleanup
- Leader election for HA

## Development Workflow

### Essential Commands
```bash
# Build the project
make build

# Generate manifests and code
make generate
make manifests

# Run tests
make test           # All tests
make test-unit      # Unit tests only
make test-e2e       # End-to-end tests

# Local development
make run            # Build image, create KIND cluster, deploy

# Code quality
make lint
make verify         # Pre-commit checks
```

### Local Testing
1. `make run` creates a complete local environment:
   - Builds controller image
   - Creates KIND cluster
   - Deploys catalogd

2. Test with a ClusterCatalog:
   ```bash
   kubectl apply -f - <<EOF
   apiVersion: olm.operatorframework.io/v1
   kind: ClusterCatalog
   metadata:
     name: operatorhubio
   spec:
     source:
       type: Image
       image:
         ref: quay.io/operatorhubio/catalog:latest
   EOF
   ```

3. Port-forward and query:
   ```bash
   kubectl -n olmv1-system port-forward svc/catalogd-service 8080:443
   curl https://localhost:8080/catalogs/operatorhubio/api/v1/all
   ```

## Code Organization

### Key Directories
- `api/` - API definitions (ClusterCatalog CRD)
- `cmd/` - Main entry points
- `internal/` - Core implementation
  - `controllers/` - Reconciliation logic
  - `storage/` - Content storage
  - `server/` - HTTP API server
- `config/` - Kubernetes manifests
- `test/` - E2E tests
- `openshift/` - OpenShift-specific configurations

### Generated Code
**DO NOT edit generated files directly**. Regenerate using:
- `make generate` - deepcopy, client code
- `make manifests` - CRDs, RBAC

Markers in Go files (e.g., `//+kubebuilder:`) control generation.

## Important Development Notes

### Code Style
- Follow standard Go conventions
- Use controller-runtime patterns consistently
- Keep reconciliation logic idempotent
- Proper error handling and status updates

### Testing Requirements
- Unit tests for business logic
- E2E tests for integration scenarios
- Always run `make verify` before submitting changes
- Test against actual catalog images

### Security Considerations
- Validate image references
- Secure TLS for HTTP server
- RBAC for cluster resources
- Container image scanning

### Common Tasks

#### Adding a New API Field
1. Update types in `api/`
2. Run `make generate manifests`
3. Update controller logic
4. Add tests
5. Update documentation

#### Modifying Controller Logic
1. Read existing reconciliation code first
2. Maintain idempotency
3. Update status conditions appropriately
4. Handle errors gracefully
5. Add unit tests

#### Updating Dependencies
1. Modify `go.mod`
2. Run `go mod tidy`
3. Update vendored deps if used
4. Test thoroughly
5. Check for breaking changes

## Troubleshooting

### Common Issues

**ClusterCatalog stuck in Progressing**
- Check controller logs: `kubectl logs -n olmv1-system deployment/catalogd-controller`
- Verify image accessibility
- Check status conditions

**HTTP endpoint not accessible**
- Verify service: `kubectl get svc -n olmv1-system catalogd-service`
- Check pod status: `kubectl get pods -n olmv1-system`
- Review server logs

**Build failures**
- Run `make generate manifests` to update generated code
- Check Go version compatibility
- Verify all dependencies available

### Debugging Tips
- Enable verbose logging in controller
- Use `kubectl describe` for resource details
- Check finalizers if resources won't delete
- Review E2E test logs for integration issues

## Integration with OLM v1

catalogd is part of the OLM v1 microservices architecture:
- **operator-controller**: Installs and manages operators
- **catalogd**: Provides catalog metadata
- **dependency resolution**: Handled by operator-controller

For broader OLM v1 context, see [operator-controller docs](https://github.com/operator-framework/operator-controller/tree/main/docs).

## Contributing Guidelines

See [CONTRIBUTING.md](./CONTRIBUTING.md) for detailed contribution guidelines.

### Quick Checklist
- [ ] Changes target `release-4.18` branch (not main)
- [ ] Code follows Go best practices
- [ ] Generated code updated (`make generate manifests`)
- [ ] Tests added/updated
- [ ] `make verify` passes
- [ ] Commit messages follow conventions
- [ ] Documentation updated if needed

## Resources

- **Slack**: [#olm-dev](https://kubernetes.slack.com/archives/C0181L6JYQ2)
- **Issues**: [GitHub Issues](https://github.com/operator-framework/catalogd/issues)
- **Releases**: [GitHub Releases](https://github.com/operator-framework/catalogd/releases)
- **OLM v1 Docs**: [operator-controller](https://github.com/operator-framework/operator-controller)

## Agent-Specific Tips

### When Reading Code
1. Start with API definitions in `api/`
2. Understand the ClusterCatalog lifecycle
3. Review controller reconciliation logic
4. Check HTTP server implementation

### When Making Changes
1. Always read existing code first - never propose changes to unread files
2. Use `make generate manifests` after API changes
3. Maintain backward compatibility for 4.18 support
4. Target `release-4.18` branch for all changes
5. Test locally with `make run` before submitting

### When Debugging
1. Check status conditions on ClusterCatalog resources
2. Review controller logs for reconciliation errors
3. Verify HTTP endpoints return expected data
4. Use E2E tests to reproduce issues

### Code Quality Standards
- **Avoid over-engineering**: Only change what's necessary
- **No premature abstraction**: Don't create helpers for one-time use
- **Security first**: Validate at system boundaries
- **Delete unused code**: No backwards-compatibility hacks
- **Simple solutions**: Minimum complexity for the task