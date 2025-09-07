# syntax=docker/dockerfile:1
# SPDX-FileCopyrightText: Copyright (C) SchedMD LLC.
# SPDX-License-Identifier: Apache-2.0
################################################################################
ARG SLURM_VERSION=24.05.7
################################################################################
# ARG PARENT_IMAGE=rocm/megatron-lm:v25.3
ARG PARENT_IMAGE=nvcr.io/nvidia/pytorch:24.12-py3
FROM ${PARENT_IMAGE} AS build
SHELL ["bash", "-c"]
ARG DEBIAN_FRONTEND=noninteractive
ARG SLURM_VERSION
ENV SLURM_VERSION=${SLURM_VERSION}
USER root
WORKDIR /tmp/

# Add Ubuntu repositories for NVIDIA base
RUN echo "deb http://archive.ubuntu.com/ubuntu/ jammy main restricted universe multiverse" >> /etc/apt/sources.list && \
    echo "deb http://archive.ubuntu.com/ubuntu/ jammy-updates main restricted universe multiverse" >> /etc/apt/sources.list && \
    apt-get update

# Ref: https://slurm.schedmd.com/quickstart_admin.html#debuild
RUN <<EOR
# Download and Build Slurm
set -xeuo pipefail
apt-get -qq update
apt-get -qq -y install --no-install-recommends \
  build-essential fakeroot devscripts equivs curl
## Download
SLURM_DIR="slurm-${SLURM_VERSION}"
mkdir -p $SLURM_DIR
if [ $(curl -s -Iw '%{http_code}' -o /dev/null https://download.schedmd.com/slurm/${SLURM_DIR}.tar.bz2) -eq 200 ]; then
  curl -s -O https://download.schedmd.com/slurm/${SLURM_DIR}.tar.bz2
  tar --strip-components=1 -jxvf ${SLURM_DIR}.tar.bz2 -C $SLURM_DIR
else
  curl -s -L -H "Accept: application/vnd.github+json" -O https://github.com/SchedMD/slurm/archive/${SLURM_VERSION}.tar.gz
  tar --strip-components=1 -zxvf ${SLURM_VERSION}.tar.gz -C $SLURM_DIR
fi
## Build
mk-build-deps -ir --tool='apt-get -qq -y -o Debug::pkgProblemResolver=yes --no-install-recommends' ${SLURM_DIR}/debian/control
( cd $SLURM_DIR && debuild -b -uc -us >/dev/null )
## Cleanup
apt-get clean && rm -rf /var/lib/apt/lists/*
EOR
################################################################################
FROM ${PARENT_IMAGE} AS base
SHELL ["bash", "-c"]
ARG DEBIAN_FRONTEND=noninteractive
ARG SLURM_VERSION
ENV SLURM_VERSION=${SLURM_VERSION}
# OCI Annotations
# https://github.com/opencontainers/image-spec/blob/v1.0/annotations.md
LABEL \
  org.opencontainers.image.authors="slinky@schedmd.com" \
  org.opencontainers.image.documentation="https://slurm.schedmd.com/documentation.html" \
  org.opencontainers.image.license="GPL-2.0-or-later WITH openssl-exception" \
  org.opencontainers.image.vendor="SchedMD LLC." \
  org.opencontainers.image.version="${SLURM_VERSION}" \
  org.opencontainers.image.source="https://github.com/SlinkyProject/containers"
# HasRequiredLabel requirement from Red Hat OpenShift Software Certification
# https://access.redhat.com/documentation/en-us/red_hat_software_certification/2024/html/red_hat_openshift_software_certification_policy_guide/assembly-requirements-for-container-images_openshift-sw-cert-policy-introduction#con-image-metadata-requirements_openshift-sw-cert-policy-container-images
LABEL \
  vendor="SchedMD LLC." \
  version="${SLURM_VERSION}" \
  release="https://github.com/SlinkyProject/containers"
USER root
WORKDIR /tmp/
# Add Ubuntu repositories manually - ADD HERE
RUN echo "deb http://archive.ubuntu.com/ubuntu/ jammy main restricted universe multiverse" >> /etc/apt/sources.list
RUN echo "deb http://archive.ubuntu.com/ubuntu/ jammy-updates main restricted universe multiverse" >> /etc/apt/sources.list
RUN apt-get update

ARG SLURM_USER=slurm
ARG SLURM_USER_UID=401
ARG SLURM_USER_GID=401
RUN <<EOR
# Create SlurmUser
set -xeuo pipefail
groupadd --system --gid=${SLURM_USER_GID} ${SLURM_USER}
useradd --system --no-log-init --uid=${SLURM_USER_UID} --gid=${SLURM_USER_GID} --shell=/usr/sbin/nologin ${SLURM_USER}
EOR
# COPY --from=build /tmp/*.deb /tmp/
RUN <<EOR
# Install Dependencies
set -xeuo pipefail
apt-get -qq update
# For: Helm Chart
apt-get -qq -y install --no-install-recommends \
  rsync gettext-base iputils-ping
# Init System
apt-get -qq -y install --no-install-recommends tini
apt-get clean && rm -rf /var/lib/apt/lists/*
EOR
ENV KUBECTL_VERSION=1.32
# Ref: https://kubernetes.io/docs/tasks/tools/install-kubectl-linux/#install-using-native-package-management
RUN <<EOR
# Install kubectl
set -xeuo pipefail
apt-get -qq update
apt-get -qq -y install --no-install-recommends \
  apt-transport-https ca-certificates curl gnupg
mkdir -p -m 755 /etc/apt/keyrings
curl -fsSL https://pkgs.k8s.io/core:/stable:/v${KUBECTL_VERSION}/deb/Release.key | gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg
chmod 644 /etc/apt/keyrings/kubernetes-apt-keyring.gpg
echo "deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v${KUBECTL_VERSION}/deb/ /" | tee /etc/apt/sources.list.d/kubernetes.list
chmod 644 /etc/apt/sources.list.d/kubernetes.list
apt-get -qq update
apt-get -qq -y install --no-install-recommends kubectl
apt-get clean && rm -rf /var/lib/apt/lists/*
EOR
## START DEBUG SECTION ###
ARG DEBUG=0
RUN <<EOR
# For: Development and Debugging
set -xeuo pipefail
if [ "$DEBUG" = true ] || [ "$DEBUG" = 1 ]; then
  apt-get -qq update
  apt-get -qq -y install --no-install-recommends sudo
  echo "${SLURM_USER} ALL=(ALL) NOPASSWD: ALL" >/etc/sudoers.d/slurm
  apt-get clean && rm -rf /var/lib/apt/lists/*
fi
EOR
### END DEBUG SECTION ###
################################################################################
# SLURM: slurmd
# BUILD: `docker build --target=slurmd -t [<registry>/]slurmd:<tag> .`
################################################################################
FROM base AS slurmd
SHELL ["bash", "-c"]
ARG DEBIAN_FRONTEND=noninteractive

# Re-declare ARG variables for this stage
ARG SLURM_USER=slurm
ARG SLURM_USER_UID=401
ARG SLURM_USER_GID=401

# Ensure slurm user exists
RUN if ! id "${SLURM_USER}" &>/dev/null; then \
      groupadd --system --gid=${SLURM_USER_GID} ${SLURM_USER} && \
      useradd --system --no-log-init --uid=${SLURM_USER_UID} --gid=${SLURM_USER_GID} --shell=/usr/sbin/nologin ${SLURM_USER}; \
    fi

# Ensure tini is available
RUN apt-get -qq update && \
    apt-get -qq -y install --no-install-recommends tini && \
    apt-get clean && rm -rf /var/lib/apt/lists/*

# OCI Annotations
# https://github.com/opencontainers/image-spec/blob/v1.0/annotations.md
LABEL \
org.opencontainers.image.title="Slurm Worker Agent" \
org.opencontainers.image.description="slurmd - The compute node daemon for Slurm" \
org.opencontainers.image.documentation="https://slurm.schedmd.com/slurmd.html"
# HasRequiredLabel requirement from Red Hat OpenShift Software Certification
# https://access.redhat.com/documentation/en-us/red_hat_software_certification/2024/html/red_hat_openshift_software_certification_policy_guide/assembly-requirements-for-container-images_openshift-sw-cert-policy-introduction#con-image-metadata-requirements_openshift-sw-cert-policy-container-images
LABEL \
name="Slurm Worker Agent" \
summary="slurmd - The compute node daemon for Slurm" \
description="slurmd - The compute node daemon for Slurm"
USER root
WORKDIR /workspace
# Ref: https://slurm.schedmd.com/quickstart_admin.html#pkg_install

RUN cd /tmp && ls -la && echo "=== Looking for .deb files ===" && find /tmp -name "*.deb" -ls

# RUN cd /tmp && \
#     set -xeuo pipefail && \
#     apt-get -qq update && \
#     apt-get -qq -y install --fix-broken \
#     ./slurm-smd-client_[0-9]*.deb \
#     ./slurm-smd-dev_[0-9]*.deb \
#     ./slurm-smd-doc_[0-9]*.deb \
#     ./slurm-smd-libnss-slurm_[0-9]*.deb \
#     ./slurm-smd-libpam-slurm-adopt_[0-9]*.deb \
#     ./slurm-smd-libpmi2-0_[0-9]*.deb \
#     ./slurm-smd-slurmd_[0-9]*.deb \
#     ./slurm-smd_[0-9]*.deb && \
#     rm *.deb && apt-get clean && rm -rf /var/lib/apt/lists/*
# Install Slurm from repositories instead of building
# RUN apt-get -qq update && \
#     apt-get -qq -y install --no-install-recommends \
#     slurm-wlm \
#     slurm-wlm-basic-plugins \
#     slurm-client \
#     slurmd && \
#     apt-get clean && rm -rf /var/lib/apt/lists/*

# Add Nebius public registry
RUN curl -fsSL https://dr.nebius.cloud/public.gpg -o /usr/share/keyrings/nebius.gpg.pub && \
    codename="$(. /etc/os-release && echo $VERSION_CODENAME)" && \
    echo "deb [signed-by=/usr/share/keyrings/nebius.gpg.pub] https://dr.nebius.cloud/ $codename main" > /etc/apt/sources.list.d/nebius.list && \
    echo "deb [signed-by=/usr/share/keyrings/nebius.gpg.pub] https://dr.nebius.cloud/ stable main" >> /etc/apt/sources.list.d/nebius.list

RUN apt-get update && \
    apt -y install \
      slurm-smd-client=${SLURM_VERSION}-1 \
      slurm-smd-dev=${SLURM_VERSION}-1 \
      slurm-smd-libnss-slurm=${SLURM_VERSION}-1 \
      slurm-smd=${SLURM_VERSION}-1 \
      slurm-smd-slurmd=${SLURM_VERSION}-1 && \
    apt-get clean && \
    rm -rf /var/lib/apt/lists/*
    
# 
RUN cp -a /root /workspace/env
# set usable workspacerights
RUN chown -R slurm:slurm ./ && \
    chmod -R g+w ./
# add all users to video/render group
RUN usermod -a -G sudo slurm && \
    # usermod -a -G render slurm  && \
    usermod -a -G video slurm
#create worksape home for test user
RUN mkdir -p /workspace/slurm && \
    chown -R slurm:slurm /workspace
# CLEAN
RUN apt-get clean && \
    rm -rf /var/lib/apt/lists/* /tmp/* /var/tmp/*
ENTRYPOINT ["/usr/bin/tini", "-g", "--", "slurmd", "-D"]
