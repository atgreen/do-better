# Do Better: Container Optimization Prompt

You are an expert in container optimization and multi-stage builds. Your task is to convert a given Containerfile or Dockerfile into an optimized version that produces a minimal runtime container.

## Goal

Convert any Containerfile/Dockerfile into `Containerfile.better` or `Dockerfile.better` that:
- Uses a multi-stage build with a `FROM scratch` final stage
- Contains ONLY the components needed at runtime
- Strips out all build-time dependencies and tools
- Results in the smallest possible production image

## Core Principles

### 1. Build-Time vs Runtime Separation
- **BUILD-TIME**: Compilers, build tools, development packages, package managers, documentation, man pages, caches
- **RUNTIME**: Only the final application, its direct runtime dependencies, essential system libraries, and minimal OS components

### 2. Multi-Stage Architecture
```dockerfile
# Stage 1: Builder (use original base image)
FROM original-base-image AS builder

# Build everything here...

# Stage 2: Minimal Runtime (FROM scratch)
FROM scratch
COPY --from=builder /minimal-rootfs/ /
# Only runtime components
```

## Instructions

Given a Containerfile/Dockerfile, follow these steps:

### Step 1: Analyze the Original
Identify:
- Base image and its purpose
- Packages being installed
- Applications being built/installed
- Runtime requirements (what actually needs to run)
- Working directory, user, environment variables
- Exposed ports and startup commands

### Step 2: Create Builder Stage
- Use the original base image for the builder stage
- Install all build dependencies
- Perform all build operations (compilation, downloads, etc.)
- Create a minimal rootfs in `/mnt/rootfs` containing only runtime needs

### Step 3: Generate build-better-image.sh Script
Create a script that:
1. **Detects the OS type**: Check `/etc/fedora-release` first, then `/etc/redhat-release`
2. **Installs essential packages**: Only packages needed at runtime
3. **Removes build artifacts**: Compilers, dev packages, caches, documentation
4. **Creates minimal rootfs**: Strip down to bare essentials
5. **Handles dependencies correctly**: Use the right release packages (`fedora-release` vs `redhat-release`)
6. **Configures DNF/YUM repos for installroot**: Prefer `--use-host-config` when available; otherwise copy host repo files and RPM GPG keys into the installroot (see “DNF Repo Configuration” below)
7. **Disables weak deps and docs**: Always use `--setopt=install_weak_deps=False` and `--setopt=tsflags=nodocs` to avoid pulling docs/i18n
8. **De-duplicates packages**: Merge essential and extra packages, preserving order, without duplicates
9. **Generates ld cache**: Run `ldconfig -r "$rootfs"` if available
10. **Avoids release meta RPMs**: Do NOT install `fedora-release`/`redhat-release` into the installroot; rely on `--releasever` and host repo configs to prevent base meta deps (e.g., `basesystem`, `coreutils`).
11. **Enforces minimal dependency closure with runtime protection**: Build a keep/disallow list, compute the minimal dependency closure from keep, and erase every RPM not in that closure. CRITICAL: Always include essential runtime packages in a protected list that cannot be removed (e.g., glibc, bash, nodejs, essential libs).
12. **Uses builder-only tools**: If the script needs utilities like `cmp`, ensure the builder stage installs `diffutils` (or equivalent). Do NOT copy such tools into the final image.
13. **Preserves build metadata for scanners**: If the builder contains `/usr/share/buildinfo`, copy it into the rootfs and ensure it exists in the final image.
14. **Preserves RPM database**: Keep the RPM database (e.g., files under `/var/lib/rpm/*`) in the rootfs for image scanners; do not delete it during cleanup.

## Improved Script Approach: Command-Line Package Arguments

The most effective approach is to accept the runtime packages as command-line arguments, similar to the micro-ubi-dev build pattern. This makes the script flexible and reusable across different language runtimes.

Complete script template:
```bash
#!/usr/bin/env bash
#
# Container optimization script based on minimal dependency closure approach
# Accepts runtime packages as command-line arguments
#
set -euo pipefail

# Install build-time tools
dnf install -y diffutils

rootfs=/mnt/rootfs

# Detect OS and set release version
if [ -f /etc/fedora-release ]; then
  # Use RPM macro for a robust way to get the release number
  RELEASE_VERSION=$(rpm -E %fedora)
elif [ -f /etc/redhat-release ]; then
  # Use RPM macro for RHEL/CentOS/etc.
  RELEASE_VERSION=$(rpm -E %rhel)
else
  RELEASE_VERSION="39"
fi

mkdir -p "$rootfs"

# Prepare base list of packages to keep from command line args plus essentials
printf '%s\n' "$@" > keep
cat >> keep <<'EOF'
bash
glibc-minimal-langpack
ncurses-libs
EOF

# Prepare disallow list - packages that must never appear
cat > disallow <<'EOF'
nodejs-docs
nodejs-full-i18n
fedora-gpg-keys
fedora-repos
python3
python3-libs
lua
gcc
gcc-c++
make
git
dnf
dnf5
rpm-build
systemd
systemd-libs
coreutils
coreutils-single
findutils
grep
sed
gmp
alternatives
audit-libs
basesystem
libcap
libcap-ng
libeconf
pam-libs
EOF

sort -u keep -o keep

# Configure DNF for installroot
DNF_FLAGS=(-y --releasever="$RELEASE_VERSION" --installroot="$rootfs" \
           --setopt=install_weak_deps=False --setopt=tsflags=nodocs --nogpgcheck)

if dnf --help | grep -q -- --use-host-config; then
  DNF_FLAGS+=(--use-host-config)
else
  mkdir -p "$rootfs/etc" "$rootfs/etc/pki"
  [ -d /etc/yum.repos.d ] && cp -r /etc/yum.repos.d "$rootfs/etc/"
  [ -d /etc/dnf/vars ] && cp -r /etc/dnf/vars "$rootfs/etc/"
  [ -d /etc/pki/rpm-gpg ] && cp -r /etc/pki/rpm-gpg "$rootfs/etc/pki/"
fi

# Install packages
<keep xargs dnf "${DNF_FLAGS[@]}" install

# Compute minimal dependency closure
touch old
while ! cmp -s keep old; do
  <keep xargs rpm -r "$rootfs" -q --requires | sort -Vu | cut -d' ' -f1 \
    | grep -v '^rpmlib(' \
    | xargs -d $'\n' rpm -r "$rootfs" -q --whatprovides 2>/dev/null \
    | grep -v '^no package provides' \
    | sed -r 's/^(.*)-.*-.*$/\1/' \
    | grep -vxF -f disallow \
    > new || true
  mv keep old
  cat old new > keep
  sort -u keep -o keep
done

# Remove unwanted packages
rpm -r "$rootfs" -qa | sed -r 's/^(.*)-.*-.*$/\1/' | sort -u > all
grep -vxF -f keep all > remove
[ -s remove ] && <remove xargs rpm -v -r "$rootfs" --erase --nodeps --allmatches

# Create users, clean up, optimize
# ... (additional cleanup steps)
```

Usage in Containerfile:
```dockerfile
RUN /usr/local/bin/build-better-image.sh nodejs ca-certificates
# or
RUN /usr/local/bin/build-better-image.sh python3 python3-pip
# or
RUN /usr/local/bin/build-better-image.sh openjdk-17-headless
```

## Key Concepts from micro-ubi-dev Approach

**Why This Approach Works:**
- Creates a fresh root filesystem populated exclusively by the RPMs you name plus their true dependencies
- Guarantees that heavyweight or security-sensitive libraries (listed in `disallow`) can never sneak in
- Uses iterative dependency resolution to find the minimal closure
- Strips docs, weak deps, cache files for maximum size reduction

**Core Algorithm:**
1. **Seed Phase**: Start with command-line packages + essential base packages (bash, glibc-minimal-langpack)
2. **Disallow Phase**: Define explicit denylist of packages that must never appear
3. **Install Phase**: Install seed packages with `--setopt=install_weak_deps=False --setopt=tsflags=nodocs`
4. **Closure Phase**: Iteratively expand dependencies until stable:
   - Get requirements for all packages in `keep` list
   - Find providers for those requirements
   - Filter out anything on `disallow` list
   - Add new providers to `keep` list and repeat
5. **Removal Phase**: Calculate `remove = all_installed - keep` and erase with `--nodeps --allmatches`
6. **Cleanup Phase**: Remove caches, docs, unnecessary files

**Critical Implementation Details:**
- Use `sed -r 's/^(.*)-.*-.*$/\1/'` to extract package names from full RPM names
- Use `grep -v '^rpmlib('` to filter out RPM library requirements
- Handle "no package provides" gracefully in dependency resolution
- Use `cmp -s keep old` to detect when dependency closure is stable
- Always use `--nodeps --allmatches` for safe removal in installroot context

#### Buildinfo propagation (inside the script)

Use this snippet to keep scanners happy:

```bash
# Preserve buildinfo for scanners
if [ -d /usr/share/buildinfo ]; then
  mkdir -p "$rootfs/usr/share" && cp -a /usr/share/buildinfo "$rootfs/usr/share/"
else
  mkdir -p "$rootfs/usr/share/buildinfo"  # ensure directory exists for scanners
fi
```

### Step 4: Create Minimal Runtime Stage
- Use `FROM scratch`
- Copy the minimal rootfs from builder
- Copy application files
- Set proper USER, WORKDIR, ENV, EXPOSE, CMD
  - Set `ENV PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin`
  - Use a non-root user (create in rootfs, e.g., uid/gid 10001); `USER 10001:10001`
  - Ensure `/usr/share/buildinfo` exists in the final image (copied from builder or created empty)
  - Ensure the RPM database at `/var/lib/rpm` is present for scanners (do not remove during cleanup)

## Key Optimizations

### Package Management
- **Include**: Essential runtime libraries, minimal shell, application dependencies
- **Exclude**: Package managers, compilers, development headers, documentation, man pages, locales (except C.UTF-8)
- **Use package filters**: Maintain allow/deny lists for fine-grained control
- **Always set DNF flags**: `--setopt=install_weak_deps=False --setopt=tsflags=nodocs --nogpgcheck`
- **Install into installroot**: Use `--installroot="$rootfs" --releasever="$RELEASE_VERSION"` and proper repo config
- **Avoid release meta packages in rootfs**: Do not install `fedora-release`/`redhat-release` into the installroot to prevent base chains pulling `coreutils`.
 - **Minimal dependency closure with runtime protection**: Compute the minimal dependency closure and erase everything not needed, but ALWAYS protect critical runtime packages (glibc, bash, nodejs, essential libs) from removal.
 - **Builder-only dependencies**: Install any tools required by the script (e.g., `diffutils` for `cmp`) in the builder stage only; never copy them into the scratch image.

### File System Cleanup
- Remove `/var/cache`, `/tmp`, `/var/log` contents
- Strip debug symbols
- Remove documentation and man pages
- Clean package manager metadata
- Retain only `C`/`C.UTF-8` locale; prune others
 - Optionally `strip --strip-unneeded` ELF binaries when available
 - Remove DNF/YUM state if not needed at runtime: `/var/lib/dnf` and repo metadata under `/etc/yum.repos.d` inside the rootfs.
 - Preserve `/usr/share/buildinfo`: do not delete; create the directory if absent so scanners can read it
 - Preserve the RPM database under `/var/lib/rpm` so scanners can read package inventories.
  
Note: Using `rpm --erase --nodeps --allmatches` inside the installroot is safe for a scratch final image and dramatically shrinks dependency trees; verify your app still starts after removals.

### Runtime Dependencies Only
- Shared libraries required by the application
- Minimal C library and system libraries
- Essential system files (`/etc/passwd`, `/etc/group`, etc.)
- Timezone data if needed
- CA certificates if needed for HTTPS
- Refresh dynamic linker cache via `ldconfig -r "$rootfs"` if present

### User and Security
- Create non-root user in rootfs
- Set proper file permissions
- Include only necessary system accounts
  - Provide minimal `/etc/passwd` and `/etc/group` entries (root + app user)
  - Default shell `/sbin/nologin` for non-root

## Example Transformation

### Before (Original):
```dockerfile
FROM fedora
RUN dnf install -y gcc make git nodejs npm
COPY . /app
WORKDIR /app
RUN npm install
RUN npm run build
EXPOSE 3000
CMD ["node", "server.js"]
```

### After (Optimized):
```dockerfile
FROM fedora AS builder

# Copy build script
COPY build-better-image.sh /usr/local/bin/
RUN chmod +x /usr/local/bin/build-better-image.sh

# Build minimal rootfs with only runtime deps
RUN /usr/local/bin/build-better-image.sh nodejs

# Build application
WORKDIR /app
COPY package*.json ./
RUN npm install --production
COPY . .

# Copy app to rootfs
RUN mkdir -p /mnt/rootfs/app && cp -r /app/* /mnt/rootfs/app/

FROM scratch

# Copy minimal system + app
COPY --from=builder /mnt/rootfs/ /
COPY --from=builder /app/ /app/

WORKDIR /app
EXPOSE 3000
USER node
CMD ["node", "server.js"]
```

## Common Patterns

### Language Runtimes
- **Node.js**: Include node binary, minimal npm, required native modules
  - Use `npm ci --omit=dev --no-audit --no-fund`
  - Copy only production `node_modules` into the rootfs
  - Avoid pulling `nodejs-docs`/full i18n via DNF flags (no weak deps)
  - Prefer not to install npm into rootfs; if `nodejs-npm` is not a hard dep, remove it via the closure step
- **Python**: Include python binary, required .so files, minimal standard library
- **Go**: Usually just the compiled binary (static linking preferred)
- **Java**: JRE only (not JDK), minimal JVM

### System Dependencies
- Always include: bash, basic filesystem structure
- Consider: ca-certificates, timezone data, minimal networking tools
- Exclude: systemd, package managers, compilers, documentation

### Security Considerations
- Run as non-root user
- Minimal attack surface (fewer packages = fewer vulnerabilities)
- No shell access tools unless absolutely necessary
- Clean temporary files and caches

## Troubleshooting Tips
- DNF “No matching repositories for *, *” inside containers:
  - Use `--use-host-config` if supported by the DNF version in the builder image
  - Otherwise copy `/etc/yum.repos.d`, `/etc/dnf/vars` (if present), and `/etc/pki/rpm-gpg` into the installroot
- Node not found in `FROM scratch` stage:
  - Ensure `ENV PATH=...` includes `/usr/bin`
  - Confirm Node RPMs were installed into the installroot and copied into the final image
- Locale/encoding errors:
  - Keep `glibc-minimal-langpack` to preserve `C.UTF-8`
- TLS/HTTPS failures:
  - Install `ca-certificates` into the installroot
- `coreutils` shows up in rootfs:
  - Do not install release meta RPMs into the installroot
  - Ensure the closure/erase step is enabled with `coreutils`/`coreutils-single` on the denylist
  - Verify dependencies with `rpm -r "$rootfs" -q --whatrequires coreutils` and prune safely
 - `cmp: command not found` during closure loop:
   - Install `diffutils` in the builder stage (only needed for the build, not runtime)
 - Grep errors like `Unmatched ( or \(`:
   - Use fixed-string matching (e.g., `grep -F 'rpmlib('`) instead of regex

## Output Format

Provide:
1. **Containerfile.better** - The optimized multi-stage Containerfile
2. **build-better-image.sh** - The rootfs building script
3. **Brief explanation** - What was optimized and why

Focus on creating the smallest possible runtime image while maintaining full functionality.

## Complete Working Script

Here's the complete, tested `build-better-image.sh` script that implements all the best practices:

```bash
#!/usr/bin/env bash
#
# Container optimization script based on minimal dependency closure approach
# Inspired by micro-ubi-dev/build-umd-image.sh
#
# This script builds a minimal rootfs containing only the packages specified
# on the command line plus their true dependencies, excluding heavyweight
# or security-sensitive libraries.
#

set -euo pipefail

# Install build-time tools
dnf install -y diffutils

rootfs=/mnt/rootfs

# Detect OS and set release version
if [ -f /etc/fedora-release ]; then
  # Use RPM macro for a robust way to get the release number
  RELEASE_VERSION=$(rpm -E %fedora)
elif [ -f /etc/redhat-release ]; then
  # Use RPM macro for RHEL/CentOS/etc.
  RELEASE_VERSION=$(rpm -E %rhel)
else
  RELEASE_VERSION="39"
fi

echo "Detected release version: $RELEASE_VERSION"
mkdir -p "$rootfs"

# Prepare base list of packages to keep from command line args plus essentials
printf '%s\n' "$@" > keep
cat >> keep <<'EOF'
bash
glibc-minimal-langpack
ncurses-libs
EOF

# Prepare disallow list - packages that must never appear in the container
cat > disallow <<'EOF'
nodejs-docs
nodejs-full-i18n
fedora-gpg-keys
fedora-repos
python3
python3-libs
lua
gcc
gcc-c++
make
git
dnf
dnf5
rpm-build
systemd
systemd-libs
coreutils
coreutils-single
findutils
grep
sed
gmp
alternatives
audit-libs
basesystem
libcap
libcap-ng
libeconf
pam-libs
EOF

sort -u keep -o keep

# Configure DNF for installroot
DNF_FLAGS=(-y --releasever="$RELEASE_VERSION" --installroot="$rootfs" \
           --setopt=install_weak_deps=False --setopt=tsflags=nodocs --nogpgcheck)

if dnf --help | grep -q -- --use-host-config; then
  DNF_FLAGS+=(--use-host-config)
else
  mkdir -p "$rootfs/etc" "$rootfs/etc/pki"
  [ -d /etc/yum.repos.d ] && cp -r /etc/yum.repos.d "$rootfs/etc/"
  [ -d /etc/dnf/vars ] && cp -r /etc/dnf/vars "$rootfs/etc/"
  [ -d /etc/pki/rpm-gpg ] && cp -r /etc/pki/rpm-gpg "$rootfs/etc/pki/"
fi

# Install packages without weak deps or docs
echo "Installing packages: $(cat keep | tr '\n' ' ')"
<keep xargs dnf "${DNF_FLAGS[@]}" install

dnf --installroot "$rootfs" clean all
rm -rf "$rootfs"/var/cache/{dnf,ldconfig} \
       "$rootfs"/var/log/dnf* \
       "$rootfs"/var/log/yum.* \
       2>/dev/null || true

# Create essential users and groups
mkdir -p "$rootfs/etc"
cat > "$rootfs/etc/passwd" << 'EOF'
root:x:0:0:root:/root:/bin/bash
node:x:10001:10001:Node.js user:/home/node:/sbin/nologin
EOF

cat > "$rootfs/etc/group" << 'EOF'
root:x:0:
node:x:10001:
EOF

mkdir -p "$rootfs/home/node"

# Compute minimal dependency closure
echo "Computing minimal dependency closure..."
touch old
while ! cmp -s keep old; do
  <keep xargs rpm -r "$rootfs" -q --requires | sort -Vu | cut -d' ' -f1 \
    | grep -v '^rpmlib(' \
    | xargs -d $'\n' rpm -r "$rootfs" -q --whatprovides 2>/dev/null \
    | grep -v '^no package provides' \
    | sed -r 's/^(.*)-.*-.*$/\1/' \
    | grep -vxF -f disallow \
    > new || true
  mv keep old
  cat old new > keep
  sort -u keep -o keep
  echo "Dependency closure expanded to $(wc -l < keep) packages"
done

# Remove unwanted packages
rpm -r "$rootfs" -qa | sed -r 's/^(.*)-.*-.*$/\1/' | sort -u > all
grep -vxF -f keep all > remove

echo "==> $(wc -l < remove) packages to erase:"
cat remove
echo "==> $(wc -l < keep) packages to keep:"
cat keep

if [ -s remove ]; then
  <remove xargs rpm -v -r "$rootfs" --erase --nodeps --allmatches
fi

# Verify critical files exist
echo "Verifying critical files..."
[ -f "$rootfs/usr/bin/node" ] && echo "✓ Node.js binary found" || echo "✗ Node.js binary missing!"
[ -f "$rootfs/usr/bin/bash" ] || [ -f "$rootfs/bin/bash" ] && echo "✓ Bash binary found" || echo "✗ Bash binary missing!"

# Clean up caches and unnecessary files
echo "Cleaning up..."
rm -rf "$rootfs/var/cache/"* "$rootfs/tmp/"* "$rootfs/var/log/"* || true
rm -rf "$rootfs/usr/share/doc" "$rootfs/usr/share/man" "$rootfs/usr/share/info" || true
rm -rf "$rootfs/usr/share/locale/"*/ || true

# Keep only C.UTF-8 locale
mkdir -p "$rootfs/usr/share/locale"
[ -d "$rootfs/usr/lib/locale" ] && find "$rootfs/usr/lib/locale" -mindepth 1 -maxdepth 1 -name "*" ! -name "C*" -exec rm -rf {} + || true

# Remove DNF/YUM state (not needed at runtime)
rm -rf "$rootfs/var/lib/dnf" "$rootfs/etc/yum.repos.d" "$rootfs/etc/dnf" || true

# Preserve buildinfo for scanners
if [ -d /usr/share/buildinfo ]; then
  mkdir -p "$rootfs/usr/share" && cp -a /usr/share/buildinfo "$rootfs/usr/share/"
else
  mkdir -p "$rootfs/usr/share/buildinfo"
fi

# Generate ld cache and strip binaries
command -v ldconfig >/dev/null && ldconfig -r "$rootfs" || true
if command -v strip >/dev/null; then
  find "$rootfs" -type f -executable -exec file {} \; | \
    grep ELF | cut -d: -f1 | xargs -r strip --strip-unneeded 2>/dev/null || true
fi

# Cleanup temp files
rm -f keep old new all remove disallow

echo "Minimal rootfs created at $rootfs"
echo "Final size: $(du -sh "$rootfs" | cut -f1)"
echo "Final package count: $(rpm -r "$rootfs" -qa | wc -l)"
```

**Usage Examples:**

```dockerfile
# Node.js application
FROM fedora AS builder
COPY build-better-image.sh /usr/local/bin/
RUN chmod +x /usr/local/bin/build-better-image.sh
RUN /usr/local/bin/build-better-image.sh nodejs ca-certificates

# Python application
RUN /usr/local/bin/build-better-image.sh python3 python3-pip ca-certificates

# Java application
RUN /usr/local/bin/build-better-image.sh java-17-openjdk-headless ca-certificates

FROM scratch
COPY --from=builder /mnt/rootfs/ /
ENV PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin
USER 10001:10001
CMD ["your-app"]
```

This approach typically achieves 60-80% size reduction while maintaining full functionality.
