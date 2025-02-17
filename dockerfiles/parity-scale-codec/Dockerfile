ARG VCS_REF=master
ARG BUILD_DATE
ARG REGISTRY_PATH=docker.io/paritytech

FROM ${REGISTRY_PATH}/base-ci-linux:latest

# metadata
LABEL io.parity.image.authors="devops-team@parity.io" \
	io.parity.image.vendor="Parity Technologies" \
	io.parity.image.title="${REGISTRY_PATH}/parity-scale-codec" \
	io.parity.image.description="Inherits from docker.io/paritytech/base-ci-linux:latest \
	rust nightly, clippy, rustfmt, miri, rust-src grcov, rust-covfix, llvm-tools-preview, \
	cargo-contract, xargo, binaryen, parallel, codecov, gnupg" \
	io.parity.image.source="https://github.com/paritytech/scripts/blob/${VCS_REF}/\
	dockerfiles/parity-scale-codec/Dockerfile" \
	io.parity.image.documentation="https://github.com/paritytech/scripts/blob/${VCS_REF}/\
	dockerfiles/parity-scale-codec/README.md" \
	io.parity.image.revision="${VCS_REF}" \
	io.parity.image.created="${BUILD_DATE}"

WORKDIR /builds

ENV SHELL /bin/bash
ENV CXX=/usr/bin/clang++-13
ENV DEBIAN_FRONTEND=noninteractive

RUN	set -eux; \
	# Needed for running commands in parallel, without overlapping output.
	# gnupg is only needed to verify the signature of the codecov uploader.
	apt-get -y update && \
	apt-get install -y --no-install-recommends parallel gnupg && \
	# codecov uploader
	curl --remote-name --silent https://uploader.codecov.io/latest/linux/codecov && \
	curl --remote-name --silent https://uploader.codecov.io/latest/linux/codecov.SHA256SUM && \
	curl --remote-name --silent https://uploader.codecov.io/latest/linux/codecov.SHA256SUM.sig && \
	curl https://keybase.io/codecovsecurity/pgp_keys.asc | gpg --import && \
	gpg --verify codecov.SHA256SUM.sig codecov.SHA256SUM && \
	shasum --algorithm 256 --check codecov.SHA256SUM && \
	chmod +x codecov && \
	mv codecov /usr/local/bin/codecov && \
	rm -f codecov.SHA256SUM codecov.SHA256SUM.sig && \
	# `binaryen` is needed by `cargo-contract` for optimizing Wasm files.
	# We fetch the latest release which contains a Linux binary.
	curl --silent https://api.github.com/repos/WebAssembly/binaryen/releases | \
	egrep --only-matching 'https://github.com/WebAssembly/binaryen/releases/download/version_[0-9]+/binaryen-version_[0-9]+-x86_64-linux.tar.gz' | \
	head -n1 | \
	xargs curl -L -O && \
	tar -xvzf binaryen-*-x86_64-linux.tar.gz  && \
	rm binaryen-*-x86_64-linux.tar.gz && \
	chmod +x binaryen-*/bin/wasm-opt && \
	mv binaryen-*/bin/wasm-opt /usr/local/bin/ && \
	rm -rf binaryen-*/ && \
	# The supported Rust nightly version must support the following components
	# to allow for a functioning CI pipeline:
	#
	#  - cargo: General build tool.
	#  - rustfmt: Rust formatting tool.
	#  - clippy: Rust linter.
	#  - miri: Rust interpreter with additional safety checks.
	#
	# We also need to install the wasm32-unknown-unknown target to test
	# ink! smart contracts compilation for this target architecture.
	#
	# Only Rust nightly builds supporting all of the above mentioned components
	# and targets can be used for this docker image.
	# Installs the latest common nightly for the listed components,
	# adds those components, wasm target and sets the profile to minimal
	rustup toolchain install nightly --target wasm32-unknown-unknown \
	--profile minimal --component rustfmt clippy miri rust-src llvm-tools-preview && \
	rustup default nightly && \
	# We require `xargo` so that `miri` runs properly
	# We require `grcov` for coverage reporting and `rust-covfix` to improve it.
	# We require `cargo-spellcheck` for automated spell-checking
	cargo install grcov rust-covfix xargo dylint-link && \
	cargo install cargo-contract && \

	# Stuff that compiles on stable.
	cargo +stable install cargo-spellcheck && \

	# versions
	rustup show && \
	cargo --version && \
	cargo-contract --version && \
	wasm-opt --version && \
	# Clean up and remove compilation artifacts that a cargo install creates (>250M).
	rm -rf "${CARGO_HOME}/registry" "${CARGO_HOME}/git" /root/.cache/sccache && \
	# apt clean up
	apt-get remove -y gnupg && \
	apt-get autoremove -y && \
	apt-get clean && \
	rm -rf /var/lib/apt/lists/*

# TODO: switch to non-root here when ink-waterfall-ci won't be the dependency of this img
# USER nonroot:nonroot
