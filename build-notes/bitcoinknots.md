# Bitcoin Knots Reproducibility Testing Process

Here's a comprehensive summary of the steps we've undertaken so far in our Action Plan 3 for Bitcoin Knots reproducibility testing:

## 1\. Initial Setup and Guix Installation

-   Attempted to install Guix using the official shell installer script:

    `wget https://git.savannah.gnu.org/cgit/guix.git/plain/etc/guix-install.sh less guix-install.sh  # Reviewed the script for security sudo bash guix-install.sh`

-   Found that Guix was already detected on the system, but the command wasn't available
-   Verified Guix installation with

    ```
    which guix
    ```

    , confirming it's at `/usr/bin/guix`

## 2\. Repository and Cache Setup

-   Created cache directories for more efficient builds:

    `mkdir -p ~/work/builds/cache/sources ~/work/builds/cache/builds`

-   Navigated to the Bitcoin Knots repository:

    `cd ~/work/builds/desktop/bitcoinknots/bitcoin`

-   Updated the repository and checked for available tags:

    `git fetch --all git tag | grep knots-28`

-   Identified the target version: v28.1.knots20250305
-   Checked out the specific version:

    `git checkout v28.1.knots20250305`

## 3\. Build Environment Configuration

-   Set up environment variables for the build:

    `# Set cache directories export SOURCES_PATH="$HOME/work/builds/cache/sources" export BASE_CACHE="$HOME/work/builds/cache/builds" # Set target platforms - Linux and Windows only export HOSTS="x86_64-linux-gnu x86_64-w64-mingw32" # Set number of parallel jobs for faster building export JOBS=$(nproc)`

-   Decided to focus only on Linux and Windows builds, excluding macOS to avoid SDK requirements

## 4\. Build Process Initiation

-   Started the Guix build process:

    `./contrib/guix/guix-build`
