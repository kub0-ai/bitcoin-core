FROM debian:bookworm-slim

ARG UID=101
ARG GID=101
ARG VERSION
ARG KEYS

LABEL org.opencontainers.image.source="https://github.com/kub0-ai/bitcoin-core"
LABEL org.opencontainers.image.description="Bitcoin Core v${VERSION}"
LABEL org.opencontainers.image.licenses="MIT"

RUN groupadd --gid ${GID} bitcoin \
  && useradd --create-home --no-log-init -u ${UID} -g ${GID} bitcoin \
  && apt-get update -y \
  && apt-get install -y curl gnupg gosu \
  && apt-get clean \
  && rm -rf /var/lib/apt/lists/* /tmp/* /var/tmp/*

ARG TARGETPLATFORM
ENV BITCOIN_VERSION=${VERSION}
ENV BITCOIN_DATA=/home/bitcoin/.bitcoin
ENV PATH=/opt/bitcoin-${BITCOIN_VERSION}/bin:$PATH

RUN set -ex \
  && case "${TARGETPLATFORM}" in \
       linux/amd64) ARCH=x86_64-linux-gnu ;; \
       linux/arm64) ARCH=aarch64-linux-gnu ;; \
       *) echo "Unsupported platform: ${TARGETPLATFORM}" && exit 1 ;; \
     esac \
  && for key in ${KEYS}; do \
       gpg --batch --keyserver keyserver.ubuntu.com --recv-keys "$key" || \
       gpg --batch --keyserver keys.openpgp.org --recv-keys "$key" || \
       gpg --batch --keyserver pgp.mit.edu --recv-keys "$key" ; \
     done \
  && curl -SLO https://bitcoincore.org/bin/bitcoin-core-${BITCOIN_VERSION}/bitcoin-${BITCOIN_VERSION}-${ARCH}.tar.gz \
  && curl -SLO https://bitcoincore.org/bin/bitcoin-core-${BITCOIN_VERSION}/SHA256SUMS \
  && curl -SLO https://bitcoincore.org/bin/bitcoin-core-${BITCOIN_VERSION}/SHA256SUMS.asc \
  && gpg --verify SHA256SUMS.asc SHA256SUMS \
  && grep " bitcoin-${BITCOIN_VERSION}-${ARCH}.tar.gz" SHA256SUMS | sha256sum -c - \
  && tar -xzf *.tar.gz -C /opt \
  && rm *.tar.gz *.asc SHA256SUMS \
  && rm -rf /opt/bitcoin-${BITCOIN_VERSION}/bin/bitcoin-qt

COPY docker-entrypoint.sh /entrypoint.sh
RUN chmod +x /entrypoint.sh

VOLUME ["/home/bitcoin/.bitcoin"]
EXPOSE 8332 8333 18332 18333 18443 18444 38333 38332

ENTRYPOINT ["/entrypoint.sh"]
RUN bitcoind -version | grep "Bitcoin Core version v${BITCOIN_VERSION}"
CMD ["bitcoind"]
