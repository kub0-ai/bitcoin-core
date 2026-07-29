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

# KEYS is required: fingerprints of trusted Bitcoin Core signers (no TOFU).
# Verify path: import only allowlisted keys → require GOODSIG (not EXPKEYSIG) →
# every VALIDSIG primary fingerprint must be in the KEYS allowlist (case-insensitive hex).
RUN set -ex \
  && case "${TARGETPLATFORM}" in \
       linux/amd64) ARCH=x86_64-linux-gnu ;; \
       linux/arm64) ARCH=aarch64-linux-gnu ;; \
       *) echo "Unsupported platform: ${TARGETPLATFORM}" && exit 1 ;; \
     esac \
  && if [ -z "$(echo "${KEYS}" | tr -d '[:space:]')" ]; then \
       echo "ERROR: KEYS build-arg is empty — refusing TOFU auto-import from the artifact under verification"; \
       exit 1; \
     fi \
  && curl -SLO https://bitcoincore.org/bin/bitcoin-core-${BITCOIN_VERSION}/SHA256SUMS \
  && curl -SLO https://bitcoincore.org/bin/bitcoin-core-${BITCOIN_VERSION}/SHA256SUMS.asc \
  && echo "Importing allowlisted KEYS only..." \
  && for key in ${KEYS}; do \
       gpg --batch --keyserver keyserver.ubuntu.com --recv-keys "$key" || \
       gpg --batch --keyserver keys.openpgp.org --recv-keys "$key" || \
       gpg --batch --keyserver pgp.mit.edu --recv-keys "$key" ; \
     done \
  && GPG_STATUS=$(gpg --verify --status-fd=1 SHA256SUMS.asc SHA256SUMS 2>/dev/null || true) \
  && echo "$GPG_STATUS" \
  && if ! echo "$GPG_STATUS" | grep -q "GOODSIG"; then \
       echo "ERROR: No GOODSIG on SHA256SUMS.asc (EXPKEYSIG alone is not accepted)"; \
       exit 1; \
     fi \
  && ALLOW_HEX=$(echo "${KEYS}" | tr -d '[:space:]' | tr 'a-f' 'A-F') \
  && VALID_LINES=$(echo "$GPG_STATUS" | awk '/\[GNUPG:\] VALIDSIG / {print toupper($3), toupper($NF)}') \
  && if [ -z "$VALID_LINES" ]; then \
       echo "ERROR: No VALIDSIG status lines from gpg --verify"; \
       exit 1; \
     fi \
  && printf '%s\n' "$VALID_LINES" > /tmp/validsig.fps \
  && while read -r sign_fp primary_fp; do \
       matched=0; \
       for fp in "$sign_fp" "$primary_fp"; do \
         [ -z "$fp" ] && continue; \
         case "$ALLOW_HEX" in \
           *"$fp"*) matched=1; echo "VALIDSIG $fp is in KEYS allowlist"; break ;; \
         esac; \
       done; \
       if [ "$matched" -ne 1 ]; then \
         echo "ERROR: VALIDSIG fingerprint(s) $sign_fp / $primary_fp not in KEYS allowlist"; \
         exit 1; \
       fi; \
     done < /tmp/validsig.fps \
  && rm -f /tmp/validsig.fps \
  && curl -SLO https://bitcoincore.org/bin/bitcoin-core-${BITCOIN_VERSION}/bitcoin-${BITCOIN_VERSION}-${ARCH}.tar.gz \
  && grep " bitcoin-${BITCOIN_VERSION}-${ARCH}.tar.gz" SHA256SUMS | sha256sum -c - \
  && tar -xzf *.tar.gz -C /opt \
  && rm *.tar.gz *.asc SHA256SUMS \
  && rm -rf /opt/bitcoin-${BITCOIN_VERSION}/bin/bitcoin-qt

COPY docker-entrypoint.sh /entrypoint.sh
RUN chmod +x /entrypoint.sh

VOLUME ["/home/bitcoin/.bitcoin"]
EXPOSE 8332 8333 18332 18333 18443 18444 38333 38332

ENTRYPOINT ["/entrypoint.sh"]
RUN bitcoind -version | grep -E "Bitcoin Core.* v${BITCOIN_VERSION}"
CMD ["bitcoind"]
