ARG PYTHON_VERSION=3.11.9
ARG DEBIAN_BASE=bookworm
FROM python:${PYTHON_VERSION}-slim-${DEBIAN_BASE} AS base

ARG WKHTMLTOPDF_VERSION=0.12.6.1-3
ARG WKHTMLTOPDF_DISTRO=bookworm
ARG NODE_VERSION=18.20.2
ENV NVM_DIR=/home/frappe/.nvm
ENV PATH=${NVM_DIR}/versions/node/v${NODE_VERSION}/bin/:${PATH}
## trigger: website-settings fixture + logo assets 2026-04-23
RUN useradd -ms /bin/bash frappe \
  && apt-get update \
  && apt-get install --no-install-recommends -y \
  curl git vim nginx gettext-base file \
  libpango-1.0-0 libharfbuzz0b libpangoft2-1.0-0 libpangocairo-1.0-0 \
  restic gpg \
  mariadb-client less \
  libpq-dev postgresql-client \
  wait-for-it jq media-types \
  && mkdir -p ${NVM_DIR} \
  && curl -o- https://raw.githubusercontent.com/nvm-sh/nvm/v0.39.5/install.sh | bash \
  && . ${NVM_DIR}/nvm.sh \
  && nvm install ${NODE_VERSION} \
  && nvm use v${NODE_VERSION} \
  && npm install -g yarn \
  && nvm alias default v${NODE_VERSION} \
  && rm -rf ${NVM_DIR}/.cache \
  && if [ "$(uname -m)" = "aarch64" ]; then export ARCH=arm64; fi \
  && if [ "$(uname -m)" = "x86_64" ]; then export ARCH=amd64; fi \
  && downloaded_file=wkhtmltox_${WKHTMLTOPDF_VERSION}.${WKHTMLTOPDF_DISTRO}_${ARCH}.deb \
  && curl -sLO https://github.com/wkhtmltopdf/packaging/releases/download/$WKHTMLTOPDF_VERSION/$downloaded_file \
  && apt-get install -y ./$downloaded_file \
  && rm $downloaded_file \
  && rm -rf /var/lib/apt/lists/* \
  && pip3 install frappe-bench

FROM base AS builder

RUN apt-get update \
  && DEBIAN_FRONTEND=noninteractive apt-get install --no-install-recommends -y \
  wget libcairo2-dev libpango1.0-dev libjpeg-dev libgif-dev librsvg2-dev \
  libpq-dev libffi-dev liblcms2-dev libldap2-dev libmariadb-dev \
  libsasl2-dev libtiff5-dev libwebp-dev pkg-config redis-tools \
  rlwrap tk8.6-dev cron gcc build-essential libbz2-dev rsync \
  && rm -rf /var/lib/apt/lists/*

ARG APPS_JSON_BASE64
RUN if [ -n "${APPS_JSON_BASE64}" ]; then \
  mkdir -p /opt/frappe && echo "${APPS_JSON_BASE64}" | base64 -d > /opt/frappe/apps.json; \
  fi

ARG FRAPPE_BRANCH=version-14
ARG FRAPPE_PATH=https://github.com/frappe/frappe

# CACHE_BUSTER is now keyed to the SHA-256 hash of apps.json (computed by CI).
# It only changes when apps.json changes (new apps, version bumps) — NOT on every
# prime push. This keeps the expensive bench init + pip + full asset build cached
# across routine code pushes to prime.
ARG CACHE_BUSTER=stable

ENV BENCH_DISABLE_UV=1

# Step 1: copy GitHub token as root (BuildKit secret perms)
USER root
RUN --mount=type=secret,id=github_token,required=false \
  set -e; \
  if [ -f /run/secrets/github_token ] && [ -s /run/secrets/github_token ]; then \
  cat /run/secrets/github_token > /tmp/github_token; \
  chmod 0644 /tmp/github_token; \
  echo "GitHub token secret copied to /tmp/github_token"; \
  else \
  echo "No github_token secret provided"; \
  fi

# Step 2: bench init — CACHED until CACHE_BUSTER (apps.json hash) changes
USER frappe
RUN set -e && \
  echo "Cache buster (apps.json hash): ${CACHE_BUSTER}" && \
  export APP_INSTALL_ARGS="" && \
  if [ -n "${APPS_JSON_BASE64}" ]; then \
  export APP_INSTALL_ARGS="--apps_path=/opt/frappe/apps.json"; \
  if [ -f /opt/frappe/apps.json ]; then echo "--- apps.json ---"; cat /opt/frappe/apps.json; echo; fi; \
  fi && \
  if [ -f /tmp/github_token ] && [ -s /tmp/github_token ]; then \
  GIT_TOKEN="$(cat /tmp/github_token)" && \
  git config --global url."https://x-access-token:${GIT_TOKEN}@github.com/".insteadOf "https://github.com/" && \
  echo "Git credential configured for github.com"; \
  else \
  echo "No readable GitHub token; cloning public repos only"; \
  fi && \
  bench init ${APP_INSTALL_ARGS} \
  --frappe-branch=${FRAPPE_BRANCH} \
  --frappe-path=${FRAPPE_PATH} \
  --no-procfile \
  --no-backups \
  --skip-redis-config-generation \
  --verbose \
  /home/frappe/frappe-bench || (echo "--- BENCH INIT FAILED ---"; exit 1) && \
  rm -f /tmp/github_token || true

RUN cd /home/frappe/frappe-bench && \
  echo "{}" > sites/common_site_config.json && \
  find apps -mindepth 1 -path "*/.git" -type d -exec rm -fr {} + 2>/dev/null || true

# Fix: prevent PyPI frappe (0.0.1) from shadowing the local framework
RUN set -e \
  && /home/frappe/frappe-bench/env/bin/pip uninstall -y frappe || true \
  && /home/frappe/frappe-bench/env/bin/pip install -e /home/frappe/frappe-bench/apps/frappe

# Fix: bench init sometimes clones an app into apps/<name> without pip-installing
# it into the venv (seen with the "chat" app), which surfaces as a hard-to-debug
# ModuleNotFoundError at runtime rather than a build failure. Sweep every cloned
# app and install any that are missing from the venv.
RUN cd /home/frappe/frappe-bench && \
  for app_dir in apps/*/; do \
  app=$(basename "$app_dir"); \
  [ "$app" = "frappe" ] && continue; \
  if [ -f "${app_dir}setup.py" ] || [ -f "${app_dir}pyproject.toml" ]; then \
  if ! env/bin/pip show "$app" > /dev/null 2>&1; then \
  echo "Installing missed app: $app"; \
  env/bin/pip install -e "$app_dir" --no-build-isolation --quiet || echo "WARN: pip install failed for $app"; \
  fi; \
  fi; \
  done

# Patch Frappe jinja.py: some apps may register "jinja" hook as list; Frappe expects dict
RUN jinja_py="/home/frappe/frappe-bench/apps/frappe/frappe/utils/jinja.py" \
  && if grep -q 'values = frappe.get_hooks("jinja")' "$jinja_py" \
  && ! grep -q 'if not isinstance(values, dict)' "$jinja_py"; then \
  sed -i '/values = frappe.get_hooks("jinja")/a\\tif not isinstance(values, dict):\n\t\tvalues = {}' "$jinja_py" \
  && echo "Patched frappe utils/jinja.py"; \
  else \
  echo "jinja.py already patched or pattern not found — skipping"; \
  fi

# Initial full asset build — CACHED alongside bench init.
# Builds frappe, erpnext, healthcare, hrms, mma_design, and the bench-init
# snapshot of prime. Result is stored in sites/assets and .built-assets.
RUN <<'EOF'
set -e
cd /home/frappe/frappe-bench
mkdir -p sites/assets
ln -sfn /home/frappe/frappe-bench/sites/assets /home/frappe/frappe-bench/assets

export NVM_DIR=/home/frappe/.nvm
for node_bin in /home/frappe/.nvm/versions/node/*/bin; do
  [ -d "$node_bin" ] && export PATH="$node_bin:$PATH"
done
export NODE_OPTIONS="${NODE_OPTIONS:-} --max-old-space-size=4096"

bench build --production || echo "WARN: initial bench build failed; prime-rebuild step will retry"

[ -s sites/assets/assets.json ] || echo "{}" > sites/assets/assets.json
echo "Initial asset build complete ($(du -sh sites/assets | cut -f1))"
EOF

# ── FAST-UPDATE ZONE ─────────────────────────────────────────────────────────
# Everything above is cached. Only the layers below re-run on each app push.
# PRIME_SHA and MMA_DESIGN_SHA are the exact commit SHAs dispatched from each repo.
# Changing either invalidates only from that ARG onwards (~2-4 min).
ARG PRIME_SHA=head

# Download the exact prime commit via GitHub API tarball (handles auth redirects for private repos).
# The API endpoint redirects to a pre-signed CDN URL — no header stripping on redirect.
RUN --mount=type=secret,id=github_token,required=false,mode=0444 \
  set -e && \
  BENCH="/home/frappe/frappe-bench" && \
  echo "Fast-updating prime → ${PRIME_SHA}" && \
  mkdir -p /tmp/prime-extract && \
  GITHUB_TOKEN="" && \
  if [ -f /run/secrets/github_token ] && [ -s /run/secrets/github_token ]; then \
    GITHUB_TOKEN=$(cat /run/secrets/github_token); \
  fi && \
  if [ -n "$GITHUB_TOKEN" ]; then \
    curl -fsSL \
      -H "Authorization: token ${GITHUB_TOKEN}" \
      -H "Accept: application/vnd.github.v3+json" \
      "https://api.github.com/repos/AyoubDahir/prime/tarball/${PRIME_SHA}" \
      -o /tmp/prime.tar.gz; \
  else \
    curl -fsSL \
      "https://github.com/AyoubDahir/prime/archive/${PRIME_SHA}.tar.gz" \
      -o /tmp/prime.tar.gz; \
  fi && \
  tar xz -C /tmp/prime-extract -f /tmp/prime.tar.gz && \
  EXTRACTED=$(ls /tmp/prime-extract/ | head -1) && \
  rsync -a --delete "/tmp/prime-extract/${EXTRACTED}/" "${BENCH}/apps/prime/" && \
  rm -rf /tmp/prime-extract /tmp/prime.tar.gz && \
  "${BENCH}/env/bin/pip" install -e "${BENCH}/apps/prime" --quiet && \
  echo "Prime updated: $(ls ${BENCH}/apps/prime/)"

ARG MMA_DESIGN_SHA=head

# Download the exact mma_design commit via GitHub API tarball.
RUN --mount=type=secret,id=github_token,required=false,mode=0444 \
  set -e && \
  BENCH="/home/frappe/frappe-bench" && \
  echo "Fast-updating mma_design → ${MMA_DESIGN_SHA}" && \
  mkdir -p /tmp/mma-extract && \
  GITHUB_TOKEN="" && \
  if [ -f /run/secrets/github_token ] && [ -s /run/secrets/github_token ]; then \
    GITHUB_TOKEN=$(cat /run/secrets/github_token); \
  fi && \
  if [ -n "$GITHUB_TOKEN" ]; then \
    curl -fsSL \
      -H "Authorization: token ${GITHUB_TOKEN}" \
      -H "Accept: application/vnd.github.v3+json" \
      "https://api.github.com/repos/AyoubDahir/mma_design/tarball/${MMA_DESIGN_SHA}" \
      -o /tmp/mma.tar.gz; \
  else \
    curl -fsSL \
      "https://github.com/AyoubDahir/mma_design/archive/${MMA_DESIGN_SHA}.tar.gz" \
      -o /tmp/mma.tar.gz; \
  fi && \
  tar xz -C /tmp/mma-extract -f /tmp/mma.tar.gz && \
  EXTRACTED=$(ls /tmp/mma-extract/ | head -1) && \
  rsync -a --delete "/tmp/mma-extract/${EXTRACTED}/" "${BENCH}/apps/mma_design/" && \
  rm -rf /tmp/mma-extract /tmp/mma.tar.gz && \
  "${BENCH}/env/bin/pip" install -e "${BENCH}/apps/mma_design" --quiet && \
  echo "mma_design updated: $(ls ${BENCH}/apps/mma_design/)"

# Rebuild prime and mma_design frontend assets, merge with cached full build.
RUN <<'EOF'
set -e
cd /home/frappe/frappe-bench

export NVM_DIR=/home/frappe/.nvm
for node_bin in /home/frappe/.nvm/versions/node/*/bin; do
  [ -d "$node_bin" ] && export PATH="$node_bin:$PATH"
done
export NODE_OPTIONS="${NODE_OPTIONS:-} --max-old-space-size=4096"

echo "Rebuilding prime and mma_design assets..."
bench build --app prime --app mma_design --production 2>&1 || {
  echo "WARN: bench build --app prime/mma_design failed, falling back to full build"
  bench build --production || echo "WARN: full fallback build also failed"
}

[ -s sites/assets/assets.json ] || echo "{}" > sites/assets/assets.json

mkdir -p .built-assets
rm -rf .built-assets/*
cp -rL sites/assets/. .built-assets/
[ -s .built-assets/assets.json ] || echo "{}" > .built-assets/assets.json
echo "Prime assets rebuilt and backed up ($(du -sh .built-assets | cut -f1))"
EOF

FROM base AS backend

USER frappe

COPY --from=builder --chown=frappe:frappe /home/frappe/frappe-bench /home/frappe/frappe-bench

WORKDIR /home/frappe/frappe-bench

VOLUME [ \
  "/home/frappe/frappe-bench/sites", \
  "/home/frappe/frappe-bench/logs" \
  ]

CMD [ \
  "/home/frappe/frappe-bench/env/bin/gunicorn", \
  "--chdir=/home/frappe/frappe-bench/sites", \
  "--bind=0.0.0.0:8000", \
  "--threads=4", \
  "--workers=2", \
  "--worker-class=gthread", \
  "--worker-tmp-dir=/dev/shm", \
  "--timeout=120", \
  "--preload", \
  "frappe.app:application" \
  ]
