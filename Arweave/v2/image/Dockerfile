# =============================================================================
# Arweave 2.9.5.1 -- runtime image.
#
# WHY THIS FILE EXISTS
#   There is NO official Arweave Docker image. Verified exhaustively:
#     - Docker Hub org `arweave`      -> count: 0  (namespace has zero repos)
#     - Docker Hub org `arweaveteam`  -> count: 0
#     - hub.docker.com/r/arweave/arweave -> HTTP 404
#     - github.com/orgs/ArweaveTeam/packages -> 1 package, and it is npm
#       (`wasm-secp256k1`). Zero container packages.
#     - .github/workflows/*.yml -> zero matches for docker|ghcr|registry|
#       buildx|push. The release pipeline publishes ONLY .tar.gz assets.
#     - ArweaveTeam's own `testweave-docker` repo (archived 2022) pulls a
#       THIRD PARTY image, `lucaarweave/arweave-node:0.0.4` (last pushed
#       2021-04-09), because there was no ArweaveTeam image to use.
#     - Issue #523 "How to setup an arweave mainnet node using docker"
#       (Feb 2024) is still open, never answered by a maintainer.
#     - PR #284 "Docker Integration" -- open since 2020-12-31, unmerged.
#   Every community image on Docker Hub was last pushed 2021-2023 and
#   predates 2.9 replica packing. None are usable.
#
#   The Dockerfiles that DO exist in the arweave repo (deploy/Dockerfile.*)
#   are BUILD tooling -- their CMD clones the repo, runs `rebar3 as prod
#   tar`, and copies a tarball to /output. They never run a node.
#
#   So: we build our own from the official release tarball. Same conclusion
#   the community reached in #523.
#
# BASE IMAGE -- ubuntu:22.04 is NOT arbitrary.
#   The release tarball bundles ERTS 14.2.5.11 (OTP 26) plus precompiled
#   NIFs (RandomX, VDF, secp256k1, RocksDB). lib/crypto-5.4.2.3/priv/lib/
#   crypto.so has a DT_NEEDED on libcrypto.so.3, and the whole tree is
#   built against glibc 2.35. That means:
#     ubuntu:22.04 / 24.04  -> OK   (libcrypto.so.3, glibc >= 2.35)
#     ubuntu:20.04, debian:bullseye -> BROKEN (libcrypto.so.1.1)
#     alpine                -> BROKEN (musl, not glibc)
#   Match the base to the tarball you download. We use the ubuntu22 asset.
#
# RUNTIME PACKAGES: the tarball needs ZERO apt packages on ubuntu:22.04.
#   DT_NEEDED across every ELF in the release resolves to exactly:
#     ld-linux-x86-64.so.2  libc.so.6   libcrypto.so.3  libgcc_s.so.1
#     libm.so.6  libstdc++.so.6  libsystemd.so.0  libtinfo.so.6  libz.so.1
#   All of those ship in the ubuntu:22.04 base already (libc6, libgcc-s1,
#   libstdc++6, libssl3, libtinfo6, zlib1g, libsystemd0). RandomX, VDF and
#   RocksDB statically link GMP/snappy/lz4/zstd -- no extra packages.
#
#   We install ca-certificates ONLY, and only because config.json sets
#   transaction_blacklist_urls to an https:// endpoint. Drop that config
#   key and you can drop this apt layer entirely.
#
# BUILD:  see RUNBOOK.md section 3 (az acr build -- no local Docker needed)
# =============================================================================

FROM ubuntu:22.04

ARG ARWEAVE_VERSION=2.9.5.1
ARG ARWEAVE_TAG=N.2.9.5.1
ARG ARWEAVE_ASSET=arweave-2.9.5.1.ubuntu22.x86_64.tar.gz
# From the release's checksums.txt. Verified against the downloaded asset.
ARG ARWEAVE_SHA256=dcacd52be21cb6b00da438a762fc5b352d1afa410608039d5255901916bc968f

ENV DEBIAN_FRONTEND=noninteractive

# curl+ca-certificates are needed to fetch the tarball; only ca-certificates
# is kept for runtime (https transaction_blacklist_urls). curl is left in
# because it costs ~2 MB and makes `kubectl exec ... curl localhost:1984/info`
# work for debugging -- remove both lines below if you want a minimal image.
RUN apt-get update \
 && apt-get install -y --no-install-recommends ca-certificates curl \
 && rm -rf /var/lib/apt/lists/*

# -- Fetch, verify, extract ---------------------------------------------------
# The tarball extracts FLAT (bin/, lib/, releases/, erts-*/, genesis_data/)
# with no top-level directory, so we must mkdir and extract INTO it.
#
# It also contains two absolute-path members left over from a packaging bug
# on Arweave's CI:
#   /home/runner/actions-runner/_work/arweave/arweave/_build/prod/rel/arweave/
#     2.9.5.1/{bin/arweave,vm.args}
# tar strips the leading "/" and creates a junk ./home/runner/... tree.
# We delete it -- it is not used at runtime.
RUN mkdir -p /opt/arweave \
 && curl -fsSL -o /tmp/arweave.tar.gz \
      "https://github.com/ArweaveTeam/arweave/releases/download/${ARWEAVE_TAG}/${ARWEAVE_ASSET}" \
 && echo "${ARWEAVE_SHA256}  /tmp/arweave.tar.gz" | sha256sum -c - \
 && tar -xzf /tmp/arweave.tar.gz -C /opt/arweave \
 && rm -f /tmp/arweave.tar.gz \
 && rm -rf /opt/arweave/home

# -- Container-tuned vm.args.src ---------------------------------------------
# Upstream ships +sbwt very_long, which busy-waits and burns CPU at idle in a
# cgroup. Our copy sets +sbwt/+sbwtdcpu/+sbwtdio to none -- which is upstream's
# OWN commented-out recommendation ("Uncomment the following line if running in
# a container"). See image/vm.args.src for the full diff rationale.
#
# NOTE: bin/arweave regenerates releases/<vsn>/vm.args from this .src on every
# single invocation. That write is redirected by RELX_OUT_FILE_PATH at runtime.
COPY vm.args.src /opt/arweave/releases/${ARWEAVE_VERSION}/vm.args.src

# -- Directories + permissions ------------------------------------------------
#   /run/arweave -- RELX_OUT_FILE_PATH target. bin/arweave only honours this if
#                   the directory ALREADY EXISTS ( [ -d ... ] test ); if the
#                   test fails it SILENTLY falls back to writing into the
#                   release dir, which then fails read-only and leaves an
#                   unexpanded "${ARNAME:-...}" literal in vm.args -> the VM
#                   fails to start with a confusing name error. It is mounted
#                   as an emptyDir at runtime, but create it here so plain
#                   `docker run` works too.
#   /var/log/arweave -- log_dir. Arweave's default log_dir is the RELATIVE path
#                   "logs", resolved against the release dir after the script
#                   does `cd $ROOTDIR`. On a read-only rootfs that fails, so we
#                   always pass an absolute log_dir in config.json.
#   /data        -- data_dir mountpoint. Arweave's default data_dir is "." --
#                   i.e. it would write RocksDB into the install directory.
#                   config.json always sets data_dir explicitly.
#
# a+rX (capital X) sets +x on directories only, not on data files.
RUN mkdir -p /run/arweave /var/log/arweave /data \
 && chmod -R a+rX /opt/arweave \
 && chmod 0777 /run/arweave /var/log/arweave /data

# UID 1025 matches the convention already used by thornode and mantra on this
# cluster, and satisfies the Gatekeeper / Azure Policy "no root containers"
# rule. Nothing in the release needs root.
USER 1025:1025

# Redirect the vm.args regeneration out of the read-only release directory.
ENV RELX_OUT_FILE_PATH=/run/arweave

# Single port: HTTP API *and* peer gossip. Arweave has no separate P2P port.
EXPOSE 1984

WORKDIR /opt/arweave

# `foreground` -- NOT `bin/start`.
#
#   bin/start is a bash `while true` loop that calls `bin/arweave foreground`
#   WITHOUT exec. In a container that leaves bash as PID 1, so it swallows
#   SIGTERM (Kubernetes graceful shutdown never reaches the VM, and the pod is
#   SIGKILLed at the grace deadline -> RocksDB corruption), and its internal
#   15-second restart loop hides crashes from Kubernetes so the pod stays
#   "Running" while the node is broken.
#
#   `bin/arweave foreground` ends in `exec "$BINDIR/erlexec" ...`, and erlexec
#   execv()s beam.smp -- so the Erlang VM becomes PID 1 and receives SIGTERM
#   directly. OTP's erl_signal_handler turns SIGTERM into init:stop(), which
#   runs ar:prep_stop/1 (drains the HTTP listener) and then tears down
#   ar_sup, closing RocksDB cleanly. That is exactly what `bin/arweave stop`
#   does -- same code path.
ENTRYPOINT ["/opt/arweave/bin/arweave", "foreground"]

# Overridden by the StatefulSet. Kept so `docker run <img>` is not silently
# wrong (data_dir would otherwise default to ".").
CMD ["config_file", "/etc/arweave/config.json"]
