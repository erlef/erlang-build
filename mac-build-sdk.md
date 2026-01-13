# macOS Build SDK

This document describes two blocks for creating a macOS Build SDK:

1. build infrastructure as a set of Nix flake configurations for creating a reproducible build environment
   with all compilers, libraries, and tools for building Erlang/OTP binaries from source code.
2. runtime infrastructure which describes where and how the build environment will be executed.
3. code signing and notarization for producing distributable macOS binaries.

## Build Infrastructure

The following describes what is called _build infrastructure_ which is a set of Nix configurations used
for creating all tools needed to build Erlang/OTP binaries for macOS platforms.

We plan to use [Nix](https://nixos.org/) with flakes to create reproducible build environments for each
supported build architecture, with all tools needed to perform native builds.

That means for each build architecture Erlang/OTP source code will be natively built using the build
[runtime infrastructure](#runtime-infrastructure). There is no cross-compilation planned.
This is for minimizing host contamination issues and ensuring build reproducibility.

As a framework Nix gives us some important non-functional requirements:

 - support for both x86_64-darwin (Intel) and aarch64-darwin (Apple Silicon) architectures
 - the capability to pin exact versions of all dependencies
 - quickly upgrade component versions via flake.lock updates
 - SBOM generation tooling available
 - well supported by community and commercial entities
 - hermetic builds isolated from host system pollution

And the following functional requirements:

 - possibility to install specific versions of components without conflicting with system packages
 - produce an Erlang/OTP binary for macOS that runs on macOS 14 (Sonoma) and later
 - have configuration for the following architectures:
   - x86_64-darwin (Intel Macs)
   - aarch64-darwin (Apple Silicon M1/M2/M3/M4)
 - possibility to tune build flags for specific architecture

One of the key points is to select the right nixpkgs version and Apple SDK. For reproducibility and
stability, we will:

 - Use a stable nixpkgs release (e.g., `nixos-24.11`) pinned via `flake.lock`
 - Use Apple SDK 14.4 via nixpkgs (`apple-sdk_14`) to match our minimum macOS 14 target
 - Pin all dependency versions explicitly

The nixpkgs 24.11 release provides:

 - LLVM/Clang toolchain
 - OpenSSL 3.x
 - ncurses 6.x
 - wxWidgets 3.2.x (with Cocoa backend)
 - unixODBC

### Nix Flake Configuration

A `flake.nix` will be defined that provides development shells for building Erlang/OTP. This flake
will have all dependencies needed to perform native builds.

The flake will provide:

 - A development shell with all build dependencies
 - Pinned Apple SDK for reproducible framework access
 - All required libraries for building Erlang/OTP with full features

The following libraries will be provided:

 - OpenSSL 3.x (for `crypto`, `ssl` applications)
 - ncurses (for terminal handling)
 - wxWidgets with Cocoa backend (for `wx` application / Observer)
 - unixODBC (for `odbc` application)

Additionally, the following build tools will be available:

 - autoconf, automake, libtool (OTP build system requirements)
 - GNU make
 - Perl (for OTP build scripts)

### Example flake.nix

```nix
{
  description = "Erlang/OTP macOS Build Environment";

  inputs = {
    nixpkgs.url = "github:NixOS/nixpkgs/nixos-24.11";
    flake-utils.url = "github:numtide/flake-utils";
  };

  outputs = { self, nixpkgs, flake-utils }:
    flake-utils.lib.eachSystem [ "x86_64-darwin" "aarch64-darwin" ] (system:
      let
        pkgs = import nixpkgs {
          inherit system;
          config = { };
        };
      in
      {
        devShells.default = pkgs.mkShell {
          buildInputs = with pkgs; [
            # Apple SDK (explicit dependency per new nixpkgs pattern)
            apple-sdk_14

            # Build tools
            autoconf
            automake
            libtool
            gnumake
            perl

            # Erlang/OTP dependencies
            openssl_3
            ncurses
            wxGTK32  # wxWidgets with Cocoa backend on Darwin
            unixODBC
            libiconv

            # Additional tools
            git
            curl
          ];

          shellHook = ''
            echo "Erlang/OTP macOS Build Environment"
            echo "Architecture: ${system}"
            echo "OpenSSL: $(openssl version)"
          '';

          # Environment variables for OTP configure
          OPENSSL_ROOT_DIR = "${pkgs.openssl_3}";
          NCURSES_ROOT_DIR = "${pkgs.ncurses}";
          WXWIDGETS_ROOT_DIR = "${pkgs.wxGTK32}";
          ODBC_ROOT_DIR = "${pkgs.unixODBC}";
        };
      }
    );
}
```

### Library Linking Strategy

macOS has restrictions on static linking of system libraries. Our strategy is:

 - **Dynamic linking** for system frameworks (CoreFoundation, Cocoa, etc.) - required by macOS
 - **Static linking** where possible for third-party dependencies (OpenSSL, ncurses, etc.) to reduce
   runtime dependencies
 - Dependencies installed in Nix store paths, referenced explicitly during OTP configure

Unlike Linux, macOS does not have a stable syscall interface, so fully static executables are not
possible. The produced binaries will require:

 - macOS 14.0 or later (minimum deployment target)
 - System frameworks (dynamically linked)

### SBOM Generation

For SBOM (Software Bill of Materials) generation, we can use:

 - `nix-sbom` or similar tooling to generate CycloneDX/SPDX from the Nix derivation
 - The flake.lock provides exact input revisions for reproducibility verification
 - Annotations in the build output with dependency versions

## Runtime Infrastructure

Given that the previous section takes care about producing a reproducible build environment for each
target architecture, this section describes how we use the build environment.

Unlike the Linux approach which uses Docker containers and QEMU emulation, macOS builds must run
directly on the host system. Each architecture requires its own physical or virtual machine:

 - **aarch64-darwin**: Requires Apple Silicon Mac (M1/M2/M3/M4)
 - **x86_64-darwin**: Requires Intel Mac or virtualized environment

### GitHub Actions Integration

For CI/CD, we will use GitHub-hosted macOS runners:

 - `macos-14` for aarch64-darwin (Apple Silicon M1 runners)
 - `macos-14` or `macos-15` for x86_64-darwin (Intel runners)

Note: GitHub follows an N-1 support policy for macOS runners. As of late 2025:
 - macOS 13 is deprecated (Nov 2025)
 - macOS 14 and 15 are supported
 - `macos-latest` points to macos-15 as of August 2025

### Example GitHub Actions Workflow

```yaml
name: Build Erlang/OTP for macOS

on:
  workflow_dispatch:
    inputs:
      otp_version:
        description: 'OTP version to build'
        required: true
        type: string

jobs:
  build:
    strategy:
      matrix:
        include:
          - runner: macos-14
            arch: aarch64-darwin
          - runner: macos-14  # or macos-15 for Intel
            arch: x86_64-darwin

    runs-on: ${{ matrix.runner }}

    steps:
      - name: Checkout
        uses: actions/checkout@v4

      - name: Install Nix
        uses: DeterminateSystems/nix-installer-action@main
        with:
          extra-conf: |
            experimental-features = nix-command flakes

      - name: Setup Nix cache
        uses: DeterminateSystems/magic-nix-cache-action@main

      - name: Build Erlang/OTP
        run: |
          nix develop --command bash -c '
            # Download OTP source
            curl -LO https://github.com/erlang/otp/releases/download/OTP-${{ inputs.otp_version }}/otp_src_${{ inputs.otp_version }}.tar.gz
            tar xzf otp_src_${{ inputs.otp_version }}.tar.gz
            cd otp_src_${{ inputs.otp_version }}

            # Configure with explicit library paths
            ./configure \
              --with-ssl=$OPENSSL_ROOT_DIR \
              --with-odbc=$ODBC_ROOT_DIR \
              --enable-darwin-64bit

            # Build
            make -j$(sysctl -n hw.ncpu)

            # Install to staging directory
            make DESTDIR=$PWD/../install install
          '

      - name: Upload artifact
        uses: actions/upload-artifact@v4
        with:
          name: erlang-${{ inputs.otp_version }}-${{ matrix.arch }}
          path: install/
```

### Nix Installation in CI

We use the [Determinate Systems Nix Installer](https://github.com/DeterminateSystems/nix-installer-action)
for CI environments because:

 - Automatic cleanup after workflow
 - Flakes enabled by default
 - Works reliably on GitHub-hosted runners
 - Fast installation

The [Magic Nix Cache](https://github.com/DeterminateSystems/magic-nix-cache-action) provides automatic
caching of Nix store paths, significantly speeding up subsequent builds.

## Code Signing and Notarization

For distributable macOS binaries, Apple requires:

1. **Code Signing**: Binaries must be signed with a Developer ID certificate
2. **Notarization**: Signed binaries must be submitted to Apple for malware scanning
3. **Stapling**: The notarization ticket should be attached to the binary

### Prerequisites

 - Apple Developer Program membership ($99/year)
 - Developer ID Application certificate
 - App Store Connect API key (for `notarytool`)

### Code Signing Process

Sign all Mach-O binaries and dynamic libraries:

```bash
# Find all Mach-O files
find ./install -type f -exec file {} \; | grep "Mach-O" | cut -d: -f1 > binaries.txt

# Sign each binary
while read binary; do
  codesign --force --sign "Developer ID Application: YOUR_TEAM_NAME (TEAM_ID)" \
    --options runtime \
    --timestamp \
    "$binary"
done < binaries.txt

# Verify signatures
codesign --verify --verbose ./install/usr/local/bin/erl
```

Key codesign flags:
 - `--options runtime`: Enable hardened runtime (required for notarization)
 - `--timestamp`: Include secure timestamp (required for notarization)
 - `--force`: Replace any existing signature

### Notarization Process

Submit the signed binaries to Apple for notarization:

```bash
# Create a ZIP archive for notarization
ditto -c -k --keepParent ./install erlang-otp.zip

# Submit for notarization using notarytool (Xcode 13+)
xcrun notarytool submit erlang-otp.zip \
  --apple-id "your-apple-id@example.com" \
  --team-id "TEAM_ID" \
  --password "@keychain:AC_PASSWORD" \
  --wait

# Or using App Store Connect API key (preferred for CI)
xcrun notarytool submit erlang-otp.zip \
  --key ~/.private_keys/AuthKey_KEYID.p8 \
  --key-id "KEYID" \
  --issuer "ISSUER_UUID" \
  --wait
```

### Stapling

After successful notarization, staple the ticket to the archive:

```bash
# For ZIP archives
xcrun stapler staple erlang-otp.zip

# For individual binaries (if distributing loose files)
xcrun stapler staple ./install/usr/local/bin/erl
```

### CI Secrets Configuration

For GitHub Actions, store the following secrets:

 - `APPLE_DEVELOPER_ID_CERT`: Base64-encoded .p12 certificate
 - `APPLE_DEVELOPER_ID_CERT_PASSWORD`: Certificate password
 - `APPLE_NOTARYTOOL_KEY`: Base64-encoded AuthKey .p8 file
 - `APPLE_NOTARYTOOL_KEY_ID`: API key ID
 - `APPLE_NOTARYTOOL_ISSUER`: API key issuer UUID

### Example Signing Workflow Step

```yaml
- name: Import Code Signing Certificate
  env:
    CERTIFICATE: ${{ secrets.APPLE_DEVELOPER_ID_CERT }}
    CERTIFICATE_PASSWORD: ${{ secrets.APPLE_DEVELOPER_ID_CERT_PASSWORD }}
  run: |
    # Create temporary keychain
    KEYCHAIN_PATH=$RUNNER_TEMP/signing.keychain-db
    KEYCHAIN_PASSWORD=$(openssl rand -base64 32)

    security create-keychain -p "$KEYCHAIN_PASSWORD" "$KEYCHAIN_PATH"
    security set-keychain-settings -lut 21600 "$KEYCHAIN_PATH"
    security unlock-keychain -p "$KEYCHAIN_PASSWORD" "$KEYCHAIN_PATH"

    # Import certificate
    echo "$CERTIFICATE" | base64 --decode > $RUNNER_TEMP/cert.p12
    security import $RUNNER_TEMP/cert.p12 -P "$CERTIFICATE_PASSWORD" \
      -A -t cert -f pkcs12 -k "$KEYCHAIN_PATH"
    security list-keychain -d user -s "$KEYCHAIN_PATH"

- name: Sign Binaries
  run: |
    find ./install -type f -exec file {} \; | grep "Mach-O" | cut -d: -f1 | while read binary; do
      codesign --force --sign "Developer ID Application: YOUR_TEAM (TEAM_ID)" \
        --options runtime --timestamp "$binary"
    done

- name: Notarize
  env:
    NOTARYTOOL_KEY: ${{ secrets.APPLE_NOTARYTOOL_KEY }}
    NOTARYTOOL_KEY_ID: ${{ secrets.APPLE_NOTARYTOOL_KEY_ID }}
    NOTARYTOOL_ISSUER: ${{ secrets.APPLE_NOTARYTOOL_ISSUER }}
  run: |
    echo "$NOTARYTOOL_KEY" | base64 --decode > $RUNNER_TEMP/AuthKey.p8

    ditto -c -k --keepParent ./install erlang-otp.zip

    xcrun notarytool submit erlang-otp.zip \
      --key $RUNNER_TEMP/AuthKey.p8 \
      --key-id "$NOTARYTOOL_KEY_ID" \
      --issuer "$NOTARYTOOL_ISSUER" \
      --wait

    xcrun stapler staple erlang-otp.zip
```

## Reproducibility Considerations

### What is Reproducible

With this setup, we achieve _functional reproducibility_:

 - Same nixpkgs revision (via flake.lock) = same toolchain versions
 - Same Apple SDK version = same framework APIs
 - Same OTP source = same build output
 - Pinned dependency versions = consistent library linkage

### Limitations

Full bit-for-bit reproducibility on macOS is challenging due to:

 - Timestamps embedded by Apple toolchain
 - Code signing includes timestamps
 - Some build tools embed host-specific information
 - macOS/Darwin has known reproducibility issues in nixpkgs

For our purposes, functional reproducibility (same inputs produce equivalent, working binaries) is
acceptable and practical.

### Verification

To verify reproducibility:

```bash
# Compare two builds (excluding signatures and timestamps)
diff <(otool -L build1/bin/beam.smp) <(otool -L build2/bin/beam.smp)

# Verify linked libraries match expected versions
otool -L ./install/usr/local/lib/erlang/erts-*/bin/beam.smp
```

## Future Considerations

 - **Universal binaries**: Could produce fat/universal binaries containing both architectures, though
   this increases binary size and complexity
 - **Older macOS support**: If needed, could target older SDK (11.3) for macOS 11+ compatibility,
   though this limits access to newer APIs
 - **Cross-compilation**: Darwin-to-Darwin cross-compilation is supported in recent nixpkgs if
   needed in the future
