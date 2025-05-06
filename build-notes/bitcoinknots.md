# Bitcoin Knots Reproducibility Testing Process

**WIP:** for bitcoinknots 28.1.knots20250305

Tested as of 2025-05-06 16:10 UTC+8

This document outlines the reproducible build process for Bitcoin Knots using the Guix build system directly on the host. This approach provides the highest level of reproducibility assurance by using a fully deterministic build environment without the additional layer of Docker containerization.

## Prerequisites

To perform this build, you'll need:
- A Debian/Ubuntu-based Linux system (other distributions may work with modifications)
- At least 32GB of free disk space for Linux and Windows builds
  - The build process requires significant space for dependencies, intermediate files, and final binaries
  - Monitor disk usage during the build to prevent interruptions
- At least 8GB of RAM (16GB recommended for faster builds)
- Internet connection to download dependencies
- Root/sudo access for installing packages and setting up Guix

## 1. Guix Installation and Setup

First, install the necessary dependencies on your system:

```bash
# Update package lists
sudo apt update

# Install essential packages
sudo apt install -y wget curl git build-essential gnupg ca-certificates
```

### 1.1 Install Guix

You can install Guix directly from your distribution's package repository:

```bash
# Install Guix from Debian/Ubuntu repositories
sudo apt install -y guix

# Verify Guix installation
which guix
guix --version
```

If Guix is not available in your repository or you need a specific version, you can install it manually:

```bash
# Download the Guix binary installer
wget https://ftp.gnu.org/gnu/guix/guix-binary-1.4.0.x86_64-linux.tar.xz
wget https://ftp.gnu.org/gnu/guix/guix-binary-1.4.0.x86_64-linux.tar.xz.sig

# Import the signing key
gpg --keyserver hkps://keys.openpgp.org --recv-keys 3CE464558A84FDC69DB40CFB090B11993D9AEBB5

# Verify the signature
gpg --verify guix-binary-1.4.0.x86_64-linux.tar.xz.sig guix-binary-1.4.0.x86_64-linux.tar.xz

# Extract the tarball
tar -xf guix-binary-1.4.0.x86_64-linux.tar.xz

# Move files to their proper locations (requires root)
sudo mv var/guix /var/
sudo mv gnu /

# Copy binaries to a location in your PATH
sudo mkdir -p /usr/local/bin
sudo cp -a /var/guix/profiles/per-user/root/current-guix/bin/guix /usr/local/bin/
sudo cp -a /var/guix/profiles/per-user/root/current-guix/bin/guix-daemon /usr/local/bin/
sudo chmod +x /usr/local/bin/guix-daemon

# Clean up
rm -rf guix-binary-1.4.0.x86_64-linux.tar.xz*
```

### 1.2 Set Up Guix Build Users

Guix requires a group of build users to perform builds in isolation:

```bash
# Create guixbuild group
sudo groupadd --system guixbuild

# Create build users
for i in $(seq -w 1 10); do
    sudo useradd -g guixbuild -G guixbuild -d /var/empty -s /usr/sbin/nologin \
        -c "Guix build user $i" --system "guixbuilder$i"
done

# Set correct permissions
sudo mkdir -p /var/guix /gnu/store
sudo chmod 1775 /gnu/store
```

### 1.3 Start the Guix Daemon

The Guix daemon needs to be running for builds to work:

```bash
# Create log directory
sudo mkdir -p /var/log/guix

# Start the daemon
sudo guix-daemon --build-users-group=guixbuild &

# Alternatively, if you have systemd:
# sudo systemctl enable guix-daemon
# sudo systemctl start guix-daemon
```

### 1.4 Set Up Guix Channels for Reproducibility

Create a channels configuration file to ensure reproducibility:

```bash
# Create directory for Guix configuration
mkdir -p ~/.config/guix

# Create channels.scm file
cat > ~/.config/guix/channels.scm << 'EOF'
(list (channel
        (name 'guix)
        (url "https://git.savannah.gnu.org/git/guix.git")
        (branch "master")
        (commit "e1c81df2cfb0d6f1e490083ad1b2b7f8df313bfd")
        (introduction
          (make-channel-introduction
            "9edb3f66fd807b096b48283debdcddccfea34bad"
            (openpgp-fingerprint
              "BBB0 2DDF 2CEA F6A8 0D1D  E643 A2A0 6DF2 A33A 54FA")))))
EOF

# Pull the specified Guix version
guix pull
```

> **Note:** If you encounter build errors with the specified commit, try using a more recent commit from the Guix repository. You can find recent commits at https://git.savannah.gnu.org/cgit/guix.git/

### 1.5 Create Cache Directories

Setting up cache directories will make future builds faster:

```bash
# Create cache directories
mkdir -p ~/work/builds/cache/sources
mkdir -p ~/work/builds/cache/builds
mkdir -p ~/work/builds/desktop/bitcoinknots/logs
```

## 2. Repository Setup

Create a directory for the Bitcoin Knots source code and clone the repository:

```bash
# Create directory if it doesn't exist
mkdir -p ~/work/builds/desktop/bitcoinknots
cd ~/work/builds/desktop/bitcoinknots

# Clone the repository
git clone https://github.com/bitcoinknots/bitcoin.git
cd bitcoin

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
export SOURCES_PATH="/home/dannybuntu/work/builds/cache/sources"
export BASE_CACHE="/home/dannybuntu/work/builds/cache/builds"

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
./contrib/guix/guix-build 2>&1 | tee ~/work/builds/desktop/bitcoinknots/logs/build_$(date +%Y%m%d_%H%M%S).log
```

This will take several hours to complete, especially for the first run. The Guix build system will:
1. Create a deterministic build environment
2. Download and build all dependencies
3. Compile Bitcoin Knots for the specified platforms
4. Output the binaries in a directory named `guix-build-<timestamp>/outdir/`

The build log will be saved to the logs directory for easy access and archiving.

> **Important:** During the build process, you will see many repetitive messages like "SWH Vault Processing...". These are **completely normal** and expected. Guix is retrieving source code from the Software Heritage archive as part of its reproducibility guarantees. This ensures that the exact same source code is used even if original download locations become unavailable or change over time.

## 6. Verification

After the build completes, verify the binaries:

```bash
# Find the most recent build output directory
BUILD_DIR=$(ls -td guix-build-* | head -1)

# Create a detailed manifest of the build outputs
find $BUILD_DIR/outdir -type f -exec sha256sum {} \; > ~/work/builds/desktop/bitcoinknots/logs/build_manifest.txt

# Create an output directory for the binaries
mkdir -p ~/work/builds/desktop/bitcoinknots/output

# Copy the binaries to the output directory
cp -r $BUILD_DIR/outdir/* ~/work/builds/desktop/bitcoinknots/output/

# Generate hashes for verification
cd ~/work/builds/desktop/bitcoinknots/output
sha256sum * > SHA256SUMS.txt
```

## 7. Comparing with Official Releases

Download the official Bitcoin Knots binaries and verify their authenticity:

```bash
# Create a directory for official releases
mkdir -p ~/work/builds/desktop/bitcoinknots/official
cd ~/work/builds/desktop/bitcoinknots/official

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
cd ~/work/builds/desktop/bitcoinknots
DIFF_LOG=~/work/builds/desktop/bitcoinknots/logs/binary_diff_$(date +%Y%m%d_%H%M%S).log
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
mkdir -p ~/work/builds/desktop/bitcoinknots/output/report

# Collect system information
uname -a > ~/work/builds/desktop/bitcoinknots/output/report/system_info.txt
guix --version >> ~/work/builds/desktop/bitcoinknots/output/report/system_info.txt

# Copy build logs
cp ~/work/builds/desktop/bitcoinknots/logs/* ~/work/builds/desktop/bitcoinknots/output/report/

# Document build environment
env | grep -E 'GUIX|HOSTS|SOURCES|BASE_CACHE|JOBS' > ~/work/builds/desktop/bitcoinknots/output/report/build_env.txt

# Create a summary report
cat > ~/work/builds/desktop/bitcoinknots/output/report/summary.md << EOF
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

## 9. Troubleshooting

### Common Guix Build Errors

#### Dependency Build Failures

If you encounter errors like these during the build process:

```
cannot build derivation `/gnu/store/6qa55c1s262gqh7l28i8z8hxzik2n527-boost-1.80.0.drv': 1 dependencies couldn't be built
cannot build derivation `/gnu/store/hq19fqz4q1vdivm8fdlkgcnrig0af376-swig-4.0.2.drv': 1 dependencies couldn't be built
```

Try these solutions:

1. **Update your Guix channels.scm file** to use a different commit:
   ```bash
   # Edit your channels.scm file with a newer commit
   nano ~/.config/guix/channels.scm
   
   # After editing, pull the new version
   guix pull
   ```

2. **Allow substitutes** by removing or modifying the `ADDITIONAL_GUIX_COMMON_FLAGS` variable:
   ```bash
   # Remove the --no-substitutes flag to allow using pre-built binaries
   unset ADDITIONAL_GUIX_COMMON_FLAGS
   # Or set it to allow authenticated substitutes
   export ADDITIONAL_GUIX_COMMON_FLAGS='--allow-authenticated-substitutes'
   ```

3. **Clear Guix store garbage** to free up space and remove corrupted builds:
   ```bash
   guix gc
   ```

4. **Increase available disk space** if you're running out of space during builds.

#### Guix Daemon Issues

If you encounter errors related to the Guix daemon:

```
ERROR: build of `/gnu/store/7hnqzmgpvkrp2l8blxa28hbzx5ddw2k8-guix-daemon-1.4.0-24.9a2ddcc.drv' failed
```

Try these solutions:

1. **Restart the Guix daemon**:
   ```bash
   sudo systemctl restart guix-daemon
   # Or if not using systemd
   sudo killall guix-daemon
   sudo guix-daemon --build-users-group=guixbuild &
   ```

2. **Check daemon logs** for specific errors:
   ```bash
   sudo journalctl -u guix-daemon
   # Or check the log files directly
   cat /var/log/guix/daemon.log
   ```

3. **Verify permissions** on the Guix store:
   ```bash
   sudo chown -R root:root /gnu/store
   sudo chmod 1775 /gnu/store
   ```

#### Network-Related Issues

If you encounter network-related errors during `guix pull` or downloads:

1. **Check your internet connection** and ensure you can access Guix servers:
   ```bash
   # Test connectivity to Guix servers
   ping -c 4 git.savannah.gnu.org
   ping -c 4 ci.guix.gnu.org
   ```

2. **Try a different mirror** by setting the `GUIX_SUBSTITUTE_URL` environment variable:
   ```bash
   export GUIX_SUBSTITUTE_URL="https://ci.guix.gnu.org https://bordeaux.guix.gnu.org"
   ```

3. **Use a different network** if available, such as a different WiFi network or a mobile hotspot.

4. **Increase download timeouts**:
   ```bash
   export GUIX_DOWNLOAD_TIMEOUT=3600
   ```

5. **Try using a proxy or VPN** if your network might be blocking Git or Guix connections.

6. **For "Connection reset by peer" errors** specifically:
   ```bash
   # Try pulling with fallback options
   guix pull --fallback
   
   # Or try pulling with a specific substitute URL
   guix pull --substitute-urls="https://ci.guix.gnu.org https://bordeaux.guix.gnu.org"
   
   # If all else fails, try with no network authentication
   guix pull --no-authenticate
   ```
   > Note: Using `--no-authenticate` reduces security and should only be used temporarily when troubleshooting.

#### Git-Related Errors

If you encounter Git-specific errors during `guix pull`, such as "Git error: object not found - no match for id":

1. **Clean the local Git cache** to force Guix to clone repositories anew:
   ```bash
   # Remove cached Git checkouts
   rm -rf ~/.cache/guix/checkouts
   ```

2. **Specify a known valid commit** in your channels.scm file:
   ```bash
   # Edit your channels.scm with a known good commit
   cat > ~/.config/guix/channels.scm << 'EOF'
   (list (channel
           (name 'guix)
           (url "https://git.savannah.gnu.org/git/guix.git")
           (commit "e1c81df2cfb0d6f1e490083ad1b2b7f8df313bfd")))
   EOF
   
   # Then pull
   guix pull
   ```
   > Note: The commit `e1c81df2cfb0d6f1e490083ad1b2b7f8df313bfd` is a known good commit from March 2025. If this commit becomes unavailable, check the Guix repository for a more recent valid commit.

7. **Try pulling without specifying a commit** in your channels.scm file, which might be more reliable:
   ```bash
   # Edit your channels.scm to remove the commit specification
   cat > ~/.config/guix/channels.scm << 'EOF'
   (list (channel
           (name 'guix)
           (url "https://git.savannah.gnu.org/git/guix.git")))
   EOF
   
   # Then pull
   guix pull
   ```