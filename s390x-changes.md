# s390x Porting Changes — acmeair-bookingservice-java

## Summary

Ported the Open Liberty-based booking service to run on s390x (IBM Z) by creating a multi-stage `Dockerfile.s390x`. The application code (pure Java WAR) requires no changes; all modifications are at the container build level.

## Dockerfile.s390x — What Changed and Why

### Multi-stage build (new)

The original `Dockerfile` assumes a pre-built WAR on the host. The s390x Dockerfile uses a two-stage build so the WAR is compiled inside a container matching the target architecture.

| Stage | Image | Purpose |
|-------|-------|---------|
| builder | `eclipse-temurin:11-jdk` | Compile the WAR with Java 11 |
| runtime | `icr.io/appcafe/open-liberty:full-java17-openj9-ubi` | Run with Java 17 |

### Why Java 11 for the build stage

Maven's default `maven-war-plugin` version (2.2, inherited from the super POM) is broken on Java 16+. The project's `pom.xml` defines a `version.maven-war-plugin` property (3.2.2) but never applies it to a plugin declaration, so the default 2.2 is used. Building with Java 11 avoids this breakage.

### Why Java 17 for the runtime stage

`server.xml` enables `microProfile-7.1`, which requires Java 17+ at runtime. The `icr.io/appcafe/open-liberty:full-java17-openj9-ubi` image provides this and publishes multi-arch manifests including s390x.

### Base image selection

| Original image | s390x image | Reason |
|----------------|-------------|--------|
| `open-liberty:full-java17-openj9` | `icr.io/appcafe/open-liberty:full-java17-openj9-ubi` | ICR image has verified s390x multi-arch support |
| (host build) | `eclipse-temurin:11-jdk` | Multi-arch including s390x; available on Docker Hub |

### `--platform=linux/s390x`

Both `FROM` lines include `--platform=linux/s390x` to ensure the correct architecture is pulled, even when building on a multi-arch host or through `docker buildx`.

### Wildcard COPY for the WAR

```dockerfile
# Original
COPY --chown=1001:0 target/acmeair-bookingservice-java-7.1.war /config/apps/

# s390x
COPY --chown=1001:0 --from=builder /build/target/*.war /config/apps/
```

Using `*.war` avoids breakage if the artifact name or version changes.

### Tests skipped during build

```dockerfile
RUN ./mvnw package -DskipTests -B
```

The `de.flapdoodle.embed.mongo` test dependency bundles platform-specific MongoDB binaries and does not ship s390x binaries. Tests that depend on it will fail at build time. Skip tests in the container build; run integration tests separately against a real MongoDB instance.

## Network / DNS Constraints

- `public.dhe.ibm.com` is unreachable from the target OpenShift cluster. The `pom.xml` contains a commented-out `<runtimeUrl>` referencing that host — it remains commented out and is not used.
- Maven Central (`repo.maven.apache.org`), Docker Hub, and `icr.io` are reachable.

## Build Command

```bash
docker buildx build --platform linux/s390x -f Dockerfile.s390x -t acmeair-bookingservice:s390x .
```

## Files Created

| File | Description |
|------|-------------|
| `Dockerfile.s390x` | Multi-stage Dockerfile targeting linux/s390x |
| `s390x-changes.md` | This document |

## Files Not Modified

No existing files were changed. The original `Dockerfile`, `Dockerfile-slim`, `pom.xml`, `server.xml`, and application source remain untouched.
