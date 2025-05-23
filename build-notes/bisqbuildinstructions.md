# Bisq 1.9.19 Reproducibility Verification Guide

## Prerequisites

1. You need Docker installed on your system
2. You need approximately 4GB of free disk space
3. You need an internet connection to download the official Bisq release

## Step 1: Create a Working Directory

```bash
# Create a dedicated directory for this verification
mkdir -p ~/bisq-verification
cd ~/bisq-verification
```

## Step 2: Download the Official Debian Package

```bash
# Download the official Bisq Debian package
wget https://github.com/bisq-network/bisq/releases/download/v1.9.19/Bisq-64bit-1.9.19.deb

# Extract the Debian package
mkdir -p extracted-deb
dpkg-deb -x Bisq-64bit-1.9.19.deb extracted-deb/

# Find the main JAR files in the extracted package
find extracted-deb/ -name "*.jar" | grep -v runtime > official-jars.txt
```

## Step 3: Build Your Own Version Using Docker

```bash
# Navigate to your Dockerfile location
cd ~/work/builds/desktop/bisq/1.9.19

# Clean up any previous Docker images (if necessary)
docker rmi bisq-builder:1.9.19 || true

# Build the Docker image (this will take some time)
docker build -t bisq-builder:1.9.19 .

# Create a container that stays running for inspection
docker run -d --name bisq-inspect bisq-builder:1.9.19 sleep 3600

# Verify the JAR files exist in the container
docker exec bisq-inspect ls -la /app/jars/

# Navigate back to your verification directory
cd ~/bisq-verification

# Create a directory for our built JARs
mkdir -p built-jars

# Copy the JAR files from the container to your verification directory
docker cp bisq-inspect:/app/jars/ ~/bisq-verification/built-jars/

# Clean up the container when done
# docker stop bisq-inspect
# docker rm bisq-inspect
```

## Step 4: Compare the Key JAR Files

```bash
# Make sure you're in the verification directory
cd ~/bisq-verification

# Compare the main JAR files (desktop.jar, core.jar, etc.)
echo "Comparing desktop.jar:"
sha256sum extracted-deb/opt/bisq/lib/app/desktop.jar built-jars/jars/desktop.jar

echo "Comparing core.jar:"
sha256sum extracted-deb/opt/bisq/lib/app/core.jar built-jars/jars/core.jar

echo "Comparing p2p.jar:"
sha256sum extracted-deb/opt/bisq/lib/app/p2p.jar built-jars/jars/p2p.jar
```

## Step 5: Detailed JAR Comparison (if hashes differ)

If the JAR file hashes differ, you can perform a more detailed analysis:

```bash
# Create directories for JAR extraction
mkdir -p jar-compare/official jar-compare/built

# Extract the official desktop.jar
unzip -q extracted-deb/opt/bisq/lib/app/desktop.jar -d jar-compare/official

# Extract our built desktop.jar
unzip -q built-jars/jars/desktop.jar -d jar-compare/built

# Compare the extracted contents
diff -r jar-compare/official jar-compare/built > jar-diff.txt

# View the differences
head -20 jar-diff.txt
```

## Step 6: Analyze the Differences (if any)

Common reasons for differences in Java JAR files:
1. Timestamps in the manifest files (META-INF/MANIFEST.MF)
2. Build paths embedded in class files
3. Different compiler versions
4. Build timestamps in resources
5. Order of entries in the JAR file

For a truly reproducible build, these differences should be minimal and explainable.

## Additional Notes

- The build process may take 15-30 minutes depending on your system
- If you encounter "Permission denied" errors, prefix commands with `sudo`
- Some differences in file timestamps and manifest data are expected and don't affect functionality
- Class files may contain different debug information but identical bytecode
- You can use tools like `jarsigner -verify` to check JAR file integrity

This verification process confirms whether the official Bisq release can be reproduced from source code, which is an important security practice for cryptocurrency applications.
