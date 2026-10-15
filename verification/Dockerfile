FROM node:22-bookworm-slim

ARG DEBIAN_FRONTEND=noninteractive
ARG HERMES_REPOSITORY_URL=https://github.com/nousresearch/hermes-agent.git
ARG HERMES_COMMIT=ea74f61d983ebdfd6a863c45761d1b38081f1d08

# tirith ships in the official GitHub release; pin it so the image does not
# download it at runtime via the Hermes resolver.
ARG TIRITH_VERSION=0.4.2
ARG TIRITH_BASE_URL=https://github.com/sheeki03/tirith/releases/download
ARG TIRITH_ASSET=tirith-x86_64-unknown-linux-gnu.tar.gz
ARG TIRITH_SHA256=efa6bf414a83dba385d4f13137e8677f850ced9102fe74ebb14c72f31df0dc77

ENV PIP_DISABLE_PIP_VERSION_CHECK=1
ENV PYTHONDONTWRITEBYTECODE=1
ENV PYTHONUNBUFFERED=1
ENV PATH=/opt/hermes-venv/bin:/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin
ENV AGENT_BROWSER_EXECUTABLE_PATH=/usr/bin/chromium

RUN apt-get update \
  && apt-get install -y --no-install-recommends ca-certificates git python3 python3-pip python3-venv build-essential chromium \
  && rm -rf /var/lib/apt/lists/*

RUN useradd --create-home --uid 10001 verifier \
  && mkdir -p /opt/verification /opt/hermes-agent /tmp/hermesagent20-runs \
  && chown -R verifier:verifier /opt/verification /tmp/hermesagent20-runs

RUN git clone "${HERMES_REPOSITORY_URL}" /opt/hermes-agent \
  && cd /opt/hermes-agent \
  && git checkout --force "${HERMES_COMMIT}" \
  && python3 -m venv /opt/hermes-venv \
  && /opt/hermes-venv/bin/pip install --upgrade pip setuptools wheel \
  && /opt/hermes-venv/bin/pip install -e /opt/hermes-agent \
  && /opt/hermes-venv/bin/pip install pytest croniter aiohttp \
  && npm install -g agent-browser \
  && chown -R verifier:verifier /opt/hermes-agent /opt/hermes-venv

# Install tirith v0.4.2 at build time from the official release and verify its
# checksum, so the Hermes resolver finds it on PATH (no runtime download, no
# 600s-timeout stall on slow links). The base image has no curl/wget, so the
# download uses python3's urllib. The tarball is removed afterwards so no
# /tmp/tirith-install-* leftovers ship in the image.
RUN mkdir -p /tmp/tirith-dl \
  && cd /tmp/tirith-dl \
  && python3 -c "import urllib.request,sys; urllib.request.urlretrieve(sys.argv[1], sys.argv[2])" \
       "${TIRITH_BASE_URL}/v${TIRITH_VERSION}/${TIRITH_ASSET}" "${TIRITH_ASSET}" \
  && echo "${TIRITH_SHA256}  ${TIRITH_ASSET}" | sha256sum -c - \
  && tar -xzf "${TIRITH_ASSET}" \
  && install -m 0755 "$(find /tmp/tirith-dl -name tirith -type f | head -n 1)" /usr/local/bin/tirith \
  && /usr/local/bin/tirith --version \
  && cd / \
  && rm -rf /tmp/tirith-dl

WORKDIR /opt/verification

COPY manifest.mjs hermes-runtime.mjs core.mjs server.mjs agent-runner.py ./

USER verifier

ENTRYPOINT ["node", "/opt/verification/server.mjs"]
