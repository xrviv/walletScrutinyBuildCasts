# Reproducible Builds for Sparrow Wallet on Windows

This guide provides step-by-step instructions to build Sparrow Wallet on Windows, making it easy to produce reproducible builds that match the official releases. 

**Note:** I have tried to keep the instructions here sequentially correct, but in reality, during the [actual build](https://github.com/xrviv/walletScrutinyBuildCasts/blob/main/2025/2025-05-12.1531.sparrowdesktop_v2.1.3-windows.log), I experienced a lot of issues that made the sequential ordering not follow the guide. 

## Build Process

Since you're starting with a barebones Windows 10 VM, we'll need to install all required dependencies directly on the system:

### Step 1: Install Prerequisites

1. **Install Chocolatey Package Manager**:
   Open PowerShell as Administrator (right-click on Start menu and select "Windows PowerShell (Admin)") and run:

   ```powershell
   # Install Chocolatey
   Set-ExecutionPolicy Bypass -Scope Process -Force
   [System.Net.ServicePointManager]::SecurityProtocol = [System.Net.ServicePointManager]::SecurityProtocol -bor 3072
   iex ((New-Object System.Net.WebClient).DownloadString('https://community.chocolatey.org/install.ps1'))
   ```

2. **Install Git and WiX Toolset**:
   ```powershell
   # Install Git and WiX Toolset using Chocolatey
   choco install git wixtoolset -y
   ```

3. **Refresh environment variables**:
   After installing with Chocolatey, you need to refresh your environment variables. You have three options:

   **Option A: Close and reopen PowerShell as Administrator** (recommended)
   
   **Option B: Use the refreshenv command**:
   ```powershell
   refreshenv
   ```
   
   **Option C: Manually update PATH for the current session**:
   ```powershell
   # Find the WiX Toolset installation directory
   dir "C:\Program Files (x86)" | findstr -i wix
   
   # Add the correct path to your PATH (update the path based on your actual installation)
   $env:PATH = $env:PATH + ";C:\Program Files (x86)\WiX Toolset v3.14.1\bin"
   ```

4. **Verify Git installation**:
   ```powershell
   git --version
   ```

   If Git is not recognized, manually add it to your PATH:
   ```powershell
   $env:PATH = $env:PATH + ";C:\Program Files\Git\cmd"
   ```

5. **Verify WiX Toolset installation**:
   ```powershell
   # Check if WiX tools are in PATH
   where.exe candle
   where.exe light
   ```

   If these commands show the path to the executables, WiX Toolset is correctly configured.

### Step 2: Install Java

1. **Create directory for Java**:
   ```powershell
   mkdir -Force C:\Users\danny\work\builds\desktop\java
   cd C:\Users\danny\work\builds\desktop
   ```

2. **Download Eclipse Temurin JDK**:
   ```powershell
   Invoke-WebRequest -Uri https://github.com/adoptium/temurin22-binaries/releases/download/jdk-22.0.2%2B9/OpenJDK22U-jdk_x64_windows_hotspot_22.0.2_9.zip -OutFile jdk.zip
   ```

3. **Extract the JDK**:
   ```powershell
   Expand-Archive jdk.zip -DestinationPath java
   ```

4. **Set up Java environment variables**:
   ```powershell
   $javaDir = (Get-ChildItem java)[0].FullName
   [Environment]::SetEnvironmentVariable('JAVA_HOME', $javaDir, 'User')
   [Environment]::SetEnvironmentVariable('PATH', $env:PATH + ';' + $javaDir + '\bin', 'User')
   
   # Update variables in current session
   $env:JAVA_HOME = $javaDir
   $env:PATH = $env:PATH + ';' + $javaDir + '\bin'
   ```

5. **Verify Java installation**:
   ```powershell
   java -version
   javac -version
   ```

### Step 3: Clone and Build Sparrow

1. **Clone the repository**:
   ```powershell
   cd C:\Users\danny\work\builds\desktop
   git clone --recursive --branch 2.1.3 https://github.com/sparrowwallet/sparrow.git
   ```

2. **Build Sparrow**:
   ```powershell
   cd sparrow
   ./gradlew.bat jpackage
   ```

### Step 4: Verify Your Build

1. **Download the official release**:
   ```powershell
   cd C:\Users\danny\work\builds\desktop
   Invoke-WebRequest -Uri https://github.com/sparrowwallet/sparrow/releases/download/2.1.3/Sparrow-2.1.3.zip -OutFile sparrow-official.zip -UseBasicParsing
   mkdir -Force official
   Expand-Archive sparrow-official.zip -DestinationPath official
   ```

2. **Compare your build with the official release**:
   ```powershell
   # Navigate to the sparrow directory
   cd C:\Users\danny\work\builds\desktop\sparrow
   
   # Generate hash for your build
   certutil -hashfile build\jpackage\Sparrow\Sparrow.exe SHA256
   
   # Generate hash for the official release
   certutil -hashfile ..\official\Sparrow\Sparrow.exe SHA256
   ```

## Successful Verification

When verification is successful, you should see matching SHA256 hashes for both your build and the official release. For example:

```
SHA256 hash of build\jpackage\Sparrow\Sparrow.exe:
72f63f7e5d7c903e1647329533829771dbc4c9b128cfa16589b895cdeb6ff00

SHA256 hash of ..\official\Sparrow\Sparrow.exe:
72f63f7e5d7c903e1647329533829771dbc4c9b128cfa16589b895cdeb6ff00
```

Matching hashes confirm that your build is reproducible and identical to the official release.

## Troubleshooting Common Issues

### PowerShell Download Issues

If you encounter this error when using Invoke-WebRequest:

```
Invoke-WebRequest: The response content cannot be parsed because the Internet Explorer engine is not available, or Internet Explorer's first-launch configuration is not complete. Specify the UseBasicParsing parameter and try again.
```

Use one of these solutions:

1. **Add the UseBasicParsing parameter**:
   ```powershell
   Invoke-WebRequest -Uri "https://github.com/sparrowwallet/sparrow/releases/download/2.1.3/Sparrow-2.1.3.zip" -OutFile sparrow-official.zip -UseBasicParsing
   ```

2. **Use alternative download methods**:
   ```powershell
   # Using System.Net.WebClient
   (New-Object System.Net.WebClient).DownloadFile("https://github.com/sparrowwallet/sparrow/releases/download/2.1.3/Sparrow-2.1.3.zip", "sparrow-official.zip")
   ```

### WiX Tools Not Found Error

If you encounter this error during the build:

```
FAILURE: Build failure with an exception

What went wrong:
Execution failed for task ':jpackage'

Can not find WiX tools (light.exe, candle.exe)
Download WiX 3.0 or later from https://wixtoolset.org and add it to the PATH.
Error: Invalid or unsupported type: [msi]
```

Follow these steps to resolve it:

1. **Install WiX Toolset if not already installed**:
   ```powershell
   choco install wixtoolset -y
   ```

2. **Add WiX Toolset to PATH**:
   ```powershell
   # For current session
   $env:PATH = $env:PATH + ";C:\Program Files (x86)\WiX Toolset v3.11\bin"
   
   # For permanent addition
   [Environment]::SetEnvironmentVariable('PATH', $env:PATH + ';C:\Program Files (x86)\WiX Toolset v3.11\bin', 'User')
   ```

3. **Verify WiX tools are in PATH**:
   ```powershell
   where.exe candle
   where.exe light
   ```

4. **Restart the build**:
   ```powershell
   ./gradlew.bat clean
   ./gradlew.bat jpackage
   ```

### Git Not Recognized

If Git is not recognized after installation:

1. **Close and reopen PowerShell** to refresh environment variables

2. **If still not recognized, manually add Git to PATH**:
   ```powershell
   $env:PATH = $env:PATH + ";C:\Program Files\Git\cmd"
   ```

### Java Not Recognized

If Java commands are not recognized:

1. **Verify Java installation directory**:
   ```powershell
   ls C:\Users\danny\work\builds\desktop\java
   ```

2. **Set JAVA_HOME and PATH manually**:
   ```powershell
   $javaDir = (Get-ChildItem C:\Users\danny\work\builds\desktop\java)[0].FullName
   $env:JAVA_HOME = $javaDir
   $env:PATH = $env:PATH + ";$javaDir\bin"
   ```

