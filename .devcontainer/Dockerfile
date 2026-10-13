FROM ghcr.io/hermetoproject/hermeto:latest

# python3.12-pip was stripped by the multi-stage production build (builder stage only).
# gcc and python3.12-devel are not needed — createrepo-c is already compiled in /venv/.
RUN dnf -y install \
    --setopt install_weak_deps=0 \
    --nodocs \
    python3.12-pip \
    && dnf clean all

# Pre-install all dev/test tools into /devenv/ so container startup is instant.
# Copy the already-compiled createrepo_c .so from the prod /venv/ instead of
# recompiling it (which would require gcc + python3.12-devel).
COPY requirements-extras.txt /tmp/requirements-extras.txt
RUN python3.12 -m venv /devenv && \
    cp -r /venv/lib/python3.12/site-packages/createrepo_c* \
          /devenv/lib/python3.12/site-packages/ && \
    /devenv/bin/pip install --upgrade pip --quiet && \
    /devenv/bin/pip install -r /tmp/requirements-extras.txt --quiet && \
    rm /tmp/requirements-extras.txt
