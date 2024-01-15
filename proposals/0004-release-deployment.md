---
MEP: 4
Title: Release & Deployment
Discussion: N/A
Implementation: Link
---

# Release & Deployment

## Abstract

This GEP discusses Glides release workflow and deployment targets.

## Motivation

Glide needs to define:

- the release process & versioning
- deployment targets where we want the gateway to run 

### Requirements

- R1: It must be possible to deploy Glide in popular production environments
- R2: It should be possible to quickly install and start Glide locally for development and prototyping

## Build & Deployment

We will distribute Glide in three ways:
- as binaries for different OS/architectures
- as Docker images with different distros
- as a Helm chart to install in Kubernetes clusters

### Binaries

Pure binaries are convent to run Glide on local machines or in bare metal installations.
They could be attached to Github releases to be able to download them directly or via popular installers (like Homebrew).

Here is the build matrix:

| Platform                 | Architectures                               | Release Channel                               |
|--------------------------|---------------------------------------------|-----------------------------------------------|
| MacOS                    | amd64, arm64                                | [Homebrew](https://brew.sh/)                  |
| Linux, freebsd, openbsd  | amd64, arm64, 386, ppc64le, s390x, riscv64  | [Snapcraft](https://snapcraft.io/)            |
| Windows                  | amd64, 386                                  | [Chocolatey](https://chocolatey.org/)         |

### Docker Images

Docker images could be run both:
- locally via docker and docker-compose
- in production

| Distro     | Description                                                         |
|------------|---------------------------------------------------------------------|
| Alpine     | A popular lightweight disto popular to run both locally and in prod |
| Ubuntu LTS | A popular distro                                                    |
| distroless | An extra small special type of base used in prod                    |
| RedHat UBI | A popular disto in the enterprise world                             |


### Get Started via Docker Compose

EinStack will provide a demo repo that could be cloned and started as easy as `docker-compose up`.

### Image & Package Registry

TBU

## Release

TBU

## References

- https://github.com/TykTechnologies/tyk?tab=readme-ov-file#get-started
- 

## Future Work

[TBU, what we put outside of the equation now, but may want to consider in the future]
