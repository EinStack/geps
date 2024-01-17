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

Pure binaries are convenient to run Glide on local machines or in bare metal installations.
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

Other considerations:
- Image tags are in format `{glide-version}-{distro}`. For example, `0.0.2-rc.1-alpine`.

### Get Started via Docker Compose

EinStack will provide a demo repo that could be cloned and started as easy as `docker-compose up`.

### Image & Package Registry

There are tree types of artifacts and here are the corresponding registries to store them:


| Artifact   | Registries                                                                                                                                     |
|------------|------------------------------------------------------------------------------------------------------------------------------------------------|
| Binary     | [Github Release](https://docs.github.com/en/repositories/releasing-projects-on-github/managing-releases-in-a-repository), OS package providers |
| Images     | [Github Container Packages](https://docs.github.com/en/packages/working-with-a-github-packages-registry/working-with-the-container-registry )  |
| Helm Chart | [ArtifactHub](https://artifacthub.io/)                                                                                                         |

## Release

The release process step by step:

- Merge `develop` into `main`
- Tag the merge commit with a new version
- Create a Github Release for the tag with release notes
- Wait until Github Action jobs build and push artifacts

### Versioning

Glide uses [semantic versioning](https://semver.org/). 

Before going with exact version (e.g. `0.0.2`), we should one or more release candidates (e.g. `0.0.2-rc.1`)
This should allow people to test the new changes early and give us early feedback. 
Also, it's normally a case that you may forget to add something to the release, so it's useful to have release candidates and wait a few days before promoting that build to the "stable" version. 

## References

- https://github.com/TykTechnologies/tyk?tab=readme-ov-file#get-started
- https://github.com/traefik/traefik/blob/master/.goreleaser.yml

## Alternatives

- Docker, however, a super popular registry, requires a paid plan for organization to have image repositories

## Future Work

[TBU, what we put outside of the equation now, but may want to consider in the future]
