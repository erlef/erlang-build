# Erlang Builds

> **Status: Proposal/RFC** - This document outlines a proposal by the
[Erlang Ecosystem Foundation (Erlef)](https://erlef.org/) to create a unified
Erlang/OTP binary distribution infrastructure.

## Goals

* Provide Central & Official Erlang Binary Distribution for the Ecosystem
  - Endorsed by Erlang, downloadable on erlang.org
* Reduce Fragmentation of the Space through many providers doing a part of the problem
* Provide Build SBoM with builds
* Provide SLSA Provenance with Builds
* Wide Range of supported Arch / OS

## Current Providers

| Provider | Linux x64 | Linux ARM64 | macOS x64 | macOS ARM64 | Windows x64 | Binary | Docker | Notes |
|----------|-----------|-------------|-----------|-------------|-------------|--------|--------|-------|
| [Erlang OTP](https://github.com/erlang/otp) | | | | | ✓ | ✓ | | Official builds |
| [Erlef OTP Builds](https://github.com/erlef/otp_builds) | | | ✓ | ✓ | | ✓ | | |
| [BEAM Machine](https://github.com/doawoo/beam-machine) | ✓ | ✓ | ✓ | ✓ | | ✓ | | MUSL Linux, Fat macOS builds |
| [Erlang Solutions](https://www.erlang-solutions.com/downloads-2/) | ✓ | | ✓ | | ✓ | ✓ | | Ubuntu, Debian, Fedora, CentOS, Raspbian |
| [Hex.pm Bob](https://github.com/hexpm/bob) | ✓ | ✓ | ✓ | ✓ | | ✓ | ✓ | Ubuntu 20.04/22.04/24.04 |
| [Burrito Erlang Builder](https://github.com/burrito-elixir/erlang-builder) | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ | | OTP 25.3+, for Burrito |
| [BEAM runtime](https://meta-erlang.github.io/beamruntime/intro/) | ✓ | ✓ | | | | ✓ | ✓ | runtime relocated based on libc or musl |
| [Alpine Linux](https://pkgs.alpinelinux.org/package/v3.21/community/x86/erlang) | ✓ | ✓ | | | | ✓ | | musl-based, also armhf/armv7/ppc64le/s390x |
| [Docker Erlang OTP](https://github.com/erlang/docker-erlang-otp) | ✓ | ✓ | | | | | ✓ | Official Docker library image |

## Requirements

Not all of those requirements need to be fulfilled to get started.

### Platform Support
* Needs to be able to build at least Windows, Linux, macOS
  - Needs to be able to partially support other OSes, partial support for SBoM /
    Build Attestations are acceptable
* Needs to be able to build common architectures on the OS

### Build Artifacts
* Creates Build SBoMs for Builds
  - [CycloneDX](https://cyclonedx.org/) & [SPDX](https://spdx.dev/) Formats
* Creates verifiable [SLSA](https://slsa.dev/) Provenance for Builds
* Windows builds signed via Azure Trusted Signing
* macOS builds notarized (infrastructure TBD)
* Can build multiple flavors, Examples:
  - Dynamically Linked vs Statically Linked
  - Different Library Support like OpenSSL v1 vs v3
  - FIPS Enabled
  - MUSL / Libc portable (for burrito)

### Lifecycle Management
* Builds are created for each non EOL Erlang Version
* New Build Flavors can be added retroactively for all non EOL versions
* Append Only (old artifacts stay, but tags can be pointed to a newer revision)
* Can rebuild for library updates / security patches

### Infrastructure
* Tracks all Build in a SBoM Management Tool like [Dependency Track](https://dependencytrack.org/)
* Reproducible Builds - Provide a reproducible build platform for the builds (lib versions etc.)
* Offers a Discovery API to locate the right image for the user (OS / Arch, OTP Version, Flavor, Library versions etc.)
* Can profit from free OSS Infrastructure like CI, Storage & CDN

## Design Proposal

* Using ORAS (OCI Registry As Storage) for Binary Storage / Exchange - https://oras.land/
* Using OCI for Docker Builds
* Using GitHub for
  - Repository containing API, Config, CI Workflows
  - GitHub OCI Registry
* Pluggable Build Infrastructure
  - Using GitHub Actions for initial OS support
  - Other Build Runners can be added using separate Infrastructure, for example if we ever wanted to provide BSD Builds
* Reproducible Build Infra
  - [Linux Build SDK](linux-build-sdk.md) using [Yocto Project](https://www.yoctoproject.org/) and QEMU
  - [Mac Build SDK](mac-build-sdk.md) using [Nix](https://nixos.org/)
  - [Windows Build SDK](windows-build-sdk.md): TBD
* Dependency Track to store all Build SBoM / Track Vulnerabilities
* Support for building intermediate artifacts. For example not all OTP Apps need a recompile for a different infrastructure
* Observer CLI for Build SBoM? - https://docs.sbom.observer/getting-started/observer-cli
* OCI Registry is primary keeper of State, not a database

### API

The project will run an orchestrator application. Its job is twofold:
* Build Orchestrator
  - Check Build Configuration & OTP Versions against built artifacts
    * Schedules jobs to create missing builds
  - Check Library Versions (manual change / security updates) and re-schedules
    builds where necessary
* Public API
  - Build Status (Done / Planned / Errors)
  - Discovery
    * Filter builds by criteria to locate the correct build for a user
    * Provide Redirect to OCI Artifact without needing to understand ORAS on the client
    * Provide metadata for build like vulnerable status

### ORAS Structure

Artifacts are stored in GitHub Container Registry using the
[OCI Distribution Spec](https://github.com/opencontainers/distribution-spec)
(compatible with any OCI registry).

#### Repository Naming
* Binary builds: `ghcr.io/erlef/erlang-builds/erlang-[FLAVOR]` (e.g., `erlang-musl`, `erlang-openssl3`)
* Docker images: `ghcr.io/erlef/erlang-builds/erlang-[FLAVOR]-[BASE]` (e.g., `erlang-musl-alpine`)

#### Structure per Repository
```
ghcr.io/erlef/erlang-builds/erlang-musl:27.0
├── Image Index (multi-arch manifest)
│   ├── linux/amd64 → Image Manifest
│   │   └── Layers (one per OTP app: erts, kernel, stdlib, ssl, ...)
│   └── linux/arm64 → Image Manifest
│       └── Layers (one per OTP app)
└── Attached Artifacts (via OCI referrers API)
    ├── SBOM (CycloneDX)
    └── SLSA Provenance

ghcr.io/erlef/erlang-builds/erlang-musl-alpine:27.0
├── Image Index (multi-arch manifest)
│   ├── linux/amd64 → Docker Image (all apps included)
│   └── linux/arm64 → Docker Image (all apps included)
└── Attached Artifacts
    ├── SBOM (CycloneDX)
    └── SLSA Provenance
```

#### Key Concepts
* **Tag** = OTP version (e.g., `27.0`, `26.2.5`)
* **Image Index** = multi-arch entry point, clients select platform via `Accept` header or registry API
* **Layers** = one layer per OTP application, enabling partial installation for binary builds
* **Attached Artifacts** = SBOMs and attestations linked via `subject` field, discoverable via `/v2/.../referrers/<digest>` API
* **Annotations** = build metadata (workflow, library versions, build timestamp)
