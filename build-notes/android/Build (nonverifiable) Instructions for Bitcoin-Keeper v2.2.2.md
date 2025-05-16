# Instructions to Build the Bitcoin Keeper (io.hexawallet.bitcoinkeeper) v2.2.2

1.  **Create your work directory**

    ```bash
    cd ~/work/builds/android
    mkdir io.hexawallet.bitcoinkeeper_2.2.2
    cd io.hexawallet.bitcoinkeeper_2.2.2
    ```

2.  **(Optional) Record your session**

    ```bash
    asciinema rec 2025-05-16.1648.bitcoin-keeper_v2.2.2.cast
    ```

3.  **Clone the repo**

    ```bash
    git clone https://github.com/bithyve/bitcoin-keeper
    cd bitcoin-keeper
    ```

4.  **Attempt to install dependencies (failed on Node 14)**

    ```bash
    yarn install
    # → error: “Expected version >=18. Got 14.21.3”
    ```

5.  **Install Node 18 via NVM**

    ```bash
    curl -o- https://raw.githubusercontent.com/nvm-sh/nvm/v0.39.3/install.sh | bash
    export NVM_DIR="$HOME/.nvm" && \. "$NVM_DIR/nvm.sh"
    nvm install 18
    nvm use 18
    node -v   # should report v18.x.x
    ```

6.  **Re-run Yarn and apply patches**

    ```bash
    yarn install                  # now succeeds
    patch-package && node patches/patch-sifir_android.mjs
    ```

7.  **Set up shims, git hooks, and skip CocoaPods on Linux**

    ```bash
    husky install
    chmod +x setup.sh
    sh ./setup.sh   # “pod: not found” is fine on Linux
    ```

8.  **(Optional dev step) Start Metro**

    ```bash
    yarn start     # launches the JS bundler on localhost:8081
    ```

9.  **Generate the AAB**

    ```bash
    cd android
    ./gradlew bundleProductionRelease --no-daemon
    # → produces app-production-release.aab under
    #   android/app/build/outputs/bundle/productionRelease/
    ```

10.  **Build device-specific split APKs**

    ```bash
    bundletool build-apks \
      --bundle=android/app/build/outputs/bundle/productionRelease/app-production-release.aab \
      --output=app.apks \
      --device-spec=/var/shared/device-spec/11/device-spec.json
    ```

11.  **Unzip the APK Set**

    ```bash
    unzip -o app.apks -d extracted_apks
    # → you now have extracted_apks/splits/{base-master.apk, base-armeabi_v7a.apk, base-en.apk, base-xhdpi.apk}
    ```

12.  **Prepare directories and extract both built & official APKs**

    ```bash
    mkdir -p built/{base,armeabi_v7a,en,xhdpi} \
             official/{base,armeabi_v7a,en,xhdpi}

    # Built
    unzip -o extracted_apks/splits/base-master.apk       -d built/base
    unzip -o extracted_apks/splits/base-armeabi_v7a.apk  -d built/armeabi_v7a
    unzip -o extracted_apks/splits/base-en.apk           -d built/en
    unzip -o extracted_apks/splits/base-xhdpi.apk        -d built/xhdpi

    # Official (from Google Play)
    unzip -o /var/shared/apk/io.hexawallet.bitcoinkeeper/2.2.2/base.apk                     -d official/base
    unzip -o /var/shared/apk/io.hexawallet.bitcoinkeeper/2.2.2/split_config.armeabi_v7a.apk -d official/armeabi_v7a
    unzip -o /var/shared/apk/io.hexawallet.bitcoinkeeper/2.2.2/split_config.en.apk          -d official/en
    unzip -o /var/shared/apk/io.hexawallet.bitcoinkeeper/2.2.2/split_config.xhdpi.apk       -d official/xhdpi
    ```

13.  **Compare built vs official**

    ```bash
    diff -r built/base        official/base
    diff -r built/armeabi_v7a official/armeabi_v7a
    diff -r built/en          official/en
    diff -r built/xhdpi       official/xhdpi
    ```
