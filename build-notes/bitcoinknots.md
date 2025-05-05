# Bitcoin Knots Reproducibility Testing Process

**WIP:** for bitcoinknots 28.1.knots20250305

Tested as of 2025-05-05 18:44 UTC+8

This document outlines the reproducible build process for Bitcoin Knots using the Guix build system in a containerized environment. This approach provides the highest level of reproducibility assurance by using a fully deterministic build environment.

## Prerequisites

To perform this build, you'll need:
- Docker installed on your system
- At least 32GB of free disk space for Linux and Windows builds
  - The build process requires significant space for dependencies, intermediate files, and final binaries
  - Monitor disk usage during the build to prevent interruptions
- At least 8GB of RAM (16GB recommended for faster builds)
- Internet connection to download dependencies

## 1. Container Setup

Create a Dockerfile for the build environment:

```dockerfile
FROM debian:bullseye

# Install dependencies
RUN apt-get update && apt-get install -y \
    wget curl git sudo \
    build-essential libtool autotools-dev automake pkg-config \
    bsdmainutils python3 \
    gnupg ca-certificates locales daemonize \
    && rm -rf /var/lib/apt/lists/*

# Set up locale
RUN sed -i -e 's/# en_US.UTF-8 UTF-8/en_US.UTF-8 UTF-8/' /etc/locale.gen && \
    locale-gen
ENV LANG en_US.UTF-8
ENV LANGUAGE en_US:en
ENV LC_ALL en_US.UTF-8

# Create a non-root user
RUN useradd -m -s /bin/bash builder
RUN echo "builder ALL=(ALL) NOPASSWD:ALL" > /etc/sudoers.d/builder

# Set up Guix build users
RUN sudo groupadd --system guixbuild && \
    for i in $(seq -w 1 10); do \
        sudo useradd -g guixbuild -G guixbuild -d /var/empty -s /usr/sbin/nologin -c "Guix build user $i" --system "guixbuilder$i"; \
    done

# Switch to non-root user
USER builder
WORKDIR /home/builder

# Download and verify Guix binary
RUN wget https://ftp.gnu.org/gnu/guix/guix-binary-1.4.0.x86_64-linux.tar.xz -O guix-binary.tar.xz && \
    wget https://ftp.gnu.org/gnu/guix/guix-binary-1.4.0.x86_64-linux.tar.xz.sig -O guix-binary.tar.xz.sig && \
    # Import Guix release signing key from keyserver
    gpg --keyserver hkps://keys.openpgp.org --recv-keys 3CE464558A84FDC69DB40CFB090B11993D9AEBB5 && \
    gpg --verify guix-binary.tar.xz.sig guix-binary.tar.xz && \
    tar -xf guix-binary.tar.xz && \
    sudo mv var/guix /var/ && \
    sudo mv gnu /gnu && \
    sudo mkdir -p /usr/local/bin && \
    sudo cp -a /var/guix/profiles/per-user/root/current-guix/bin/guix /usr/local/bin/guix && \
    sudo cp -a /var/guix/profiles/per-user/root/current-guix/bin/guix-daemon /usr/local/bin/guix-daemon && \
    sudo chmod +x /usr/local/bin/guix-daemon && \
    rm -rf guix-binary.tar.xz

# Set correct permissions
RUN sudo mkdir -p /var/guix /gnu/store && \
    sudo chown -R builder:builder /var/guix && \
    sudo chmod 1775 /gnu/store

# Create channels.scm file for reproducibility
RUN mkdir -p ~/.config/guix && \
    echo '(list (channel \
        (name '\''guix) \
        (url "https://git.savannah.gnu.org/git/guix.git") \
        (branch "master") \
        (commit "3b74a2b2fef5e5e2b545e0f1d6b3fe557a7ef08c") \
        (introduction \
          (make-channel-introduction \
            "9edb3f66fd807b096b48283debdcddccfea34bad" \
            (openpgp-fingerprint \
              "BBB0 2DDF 2CEA F6A8 0D1D  E643 A2A0 6DF2 A33A 54FA")))))' \
    > ~/.config/guix/channels.scm

# Start Guix daemon
RUN sudo mkdir -p /var/log/guix && \
    sudo daemonize -o /var/log/guix/daemon.log -e /var/log/guix/daemon.err \
    /usr/local/bin/guix-daemon --build-users-group=guixbuild

# Initialize Guix
RUN guix pull

# Add Guix to PATH
ENV PATH="/usr/local/bin:/home/builder/.config/guix/current/bin:${PATH}"

# Verify Guix is working
RUN guix --version

# Create cache directories
RUN mkdir -p /home/builder/cache/sources /home/builder/cache/builds /home/builder/logs

# Set working directory
WORKDIR /home/builder/bitcoin-knots

```

The `channels.scm` file is now created directly in the Dockerfile for better reproducibility. This eliminates the need to create a separate file in the build context.

Build and run the container:

```bash
# Build the Docker image
docker build -t bitcoin-knots-builder .

# Run the container with mounted volumes for output and logs
docker run -it --name bitcoin-knots-build \
  -v $(pwd)/output:/home/builder/output \
  -v $(pwd)/logs:/home/builder/logs \
  bitcoin-knots-builder
```

## 2. Repository Setup

Inside the container, clone the Bitcoin Knots repository:

```bash
# Clone the repository
git clone https://github.com/bitcoinknots/bitcoin.git .

# Fetch all tags and branches
git fetch --all

# List available tags to find the desired version
git tag | grep knots

# Check out the specific version (example: v28.1.knots20250305)
git checkout v28.1.knots20250305
```

## 3. Dependencies

Bitcoin Knots requires several dependencies which will be handled by the Guix build system. The main dependencies include:

- **Required**:
  - Boost (1.73.0 or newer)
  - libevent (2.1.8 or newer)
  - GCC (11.1 or newer) or Clang (16.0 or newer)
  - Python 3.9+

- **Optional** (handled automatically by Guix):
  - SQLite (for descriptor wallet)
  - Berkeley DB 4.8 (for legacy wallet compatibility)
  - Qt 5.11.3+ (for GUI)
  - libnatpmp and MiniUPnPc (for port mapping)
  - ZeroMQ (for notifications)

## 4. Build Environment Configuration

Set up environment variables for the build:

```bash
# Set cache directories - these are recognized by the depends system
export SOURCES_PATH="/home/builder/cache/sources"
export BASE_CACHE="/home/builder/cache/builds"

# Set target platforms - Linux and Windows only (excluding macOS to reduce complexity and build time)
export HOSTS="x86_64-linux-gnu x86_64-w64-mingw32"

# Note: Building for Windows requires cross-compilation tools which Guix will handle automatically
# This still requires significant disk space even without macOS builds

# Set number of parallel jobs for faster building
export JOBS=$(nproc)

# Set a fixed timestamp for reproducibility
export SOURCE_DATE_EPOCH=$(git log -1 --format=%ct)

# Disable substitutes for maximum reproducibility assurance
# This ensures everything is built from source
# Note: This significantly increases build time but provides the highest reproducibility assurance
# For faster builds with a trade-off in reproducibility, you can remove this flag or use trusted substitutes
export ADDITIONAL_GUIX_COMMON_FLAGS='--no-substitutes'

# Enable verbose output for better logging
export V=1
```

## 5. Build Process Initiation

Start the Guix build process with logging enabled:

```bash
# Run the build with logging
./contrib/guix/guix-build 2>&1 | tee /home/builder/logs/build_$(date +%Y%m%d_%H%M%S).log
```

This will take several hours to complete, especially for the first run. The Guix build system will:
1. Create a deterministic build environment
2. Download and build all dependencies
3. Compile Bitcoin Knots for the specified platforms
4. Output the binaries in a directory named `guix-build-<timestamp>/outdir/`

The build log will be saved to the logs directory, which is mounted to the host system for easy access and archiving.

## 6. Verification

After the build completes, verify the binaries:

```bash
# Find the most recent build output directory
BUILD_DIR=$(ls -td guix-build-* | head -1)

# Create a detailed manifest of the build outputs
find $BUILD_DIR/outdir -type f -exec sha256sum {} \; > /home/builder/logs/build_manifest.txt

# Copy the binaries to the mounted output volume
cp -r $BUILD_DIR/outdir/* /home/builder/output/

# Generate hashes for verification
cd /home/builder/output
sha256sum * > SHA256SUMS.txt
```

## 7. Comparing with Official Releases

Download the official Bitcoin Knots binaries and verify their authenticity:

```bash
# Create a directory for official releases
mkdir -p /home/builder/official
cd /home/builder/official

# Download official release and signature
wget https://bitcoinknots.org/files/28.1.knots20250305/bitcoin-28.1.knots20250305-x86_64-linux-gnu.tar.gz
wget https://bitcoinknots.org/files/28.1.knots20250305/SHA256SUMS
wget https://bitcoinknots.org/files/28.1.knots20250305/SHA256SUMS.asc

# Import Luke Dash Jr's GPG key
gpg --keyserver hkps://keyserver.ubuntu.com --recv-keys 01EA5486DE18A882D4C2684590C8019E36C2E964

# Verify the fingerprint of the imported key
gpg --fingerprint 01EA5486DE18A882D4C2684590C8019E36C2E964
# Ensure the displayed fingerprint matches: 01EA 5486 DE18 A882 D4C2 6845 90C8 019E 36C2 E964

# Verify the signature
gpg --verify SHA256SUMS.asc SHA256SUMS

# Verify the downloaded binary against the signed checksums
grep "bitcoin-28.1.knots20250305-x86_64-linux-gnu.tar.gz" SHA256SUMS | sha256sum -c

# Extract the official release
tar -xzf bitcoin-28.1.knots20250305-x86_64-linux-gnu.tar.gz

# Compare our built binaries with the official ones
cd /home/builder
DIFF_LOG=/home/builder/logs/binary_diff_$(date +%Y%m%d_%H%M%S).log
echo "Binary comparison results" > $DIFF_LOG

# Compare bitcoin-qt binary
diff -q output/bin/bitcoin-qt official/bitcoin-28.1.knots20250305/bin/bitcoin-qt >> $DIFF_LOG 2>&1
if [ $? -eq 0 ]; then
  echo "bitcoin-qt: MATCH" >> $DIFF_LOG
else
  echo "bitcoin-qt: DIFFERENT" >> $DIFF_LOG
  # For detailed binary analysis
  cmp -l output/bin/bitcoin-qt official/bitcoin-28.1.knots20250305/bin/bitcoin-qt | \
    gawk '{printf "%08X %02X %02X\n", $1, strtonum(0$2), strtonum(0$3)}' | head -n 20 >> $DIFF_LOG
fi

# Repeat for other important binaries
# ...
```

## 8. Documentation and Reporting

Prepare a comprehensive report for WalletScrutiny.com:

```bash
# Create a report directory
mkdir -p /home/builder/output/report

# Collect system information
uname -a > /home/builder/output/report/system_info.txt
guix --version >> /home/builder/output/report/system_info.txt

# Copy build logs
cp /home/builder/logs/* /home/builder/output/report/

# Document build environment
env | grep -E 'GUIX|HOSTS|SOURCES|BASE_CACHE|JOBS' > /home/builder/output/report/build_env.txt

# Create a summary report
cat > /home/builder/output/report/summary.md << EOF
# Bitcoin Knots v28.1.knots20250305 Reproducibility Report

## Build Information
- Date: $(date)
- Version: v28.1.knots20250305
- Target Platforms: $HOSTS
- Build System: Guix ($(guix --version | head -n1))

## Reproducibility Results
$(cat $DIFF_LOG)

## Analysis of Differences
[Provide detailed analysis of any differences found]

## Conclusion
[State whether the build is reproducible or not, and any caveats]

## Recommendations
[Provide recommendations for improving reproducibility if issues were found]
EOF
```

This comprehensive report will document the entire build process, verification results, and any discrepancies found.