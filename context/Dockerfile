ARG EE_BASE_IMAGE="rockylinux:9.3"
ARG PYCMD="/usr/bin/python3"
ARG PKGMGR_PRESERVE_CACHE=""
ARG ANSIBLE_GALAXY_CLI_COLLECTION_OPTS=""
ARG ANSIBLE_GALAXY_CLI_ROLE_OPTS=""
ARG ANSIBLE_INSTALL_REFS="ansible-core==2.15.8 ansible-runner==2.3.1"
ARG PKGMGR="/usr/bin/dnf"

# Base build stage
FROM $EE_BASE_IMAGE as base
USER root
ARG EE_BASE_IMAGE
ARG PYCMD
ARG PKGMGR_PRESERVE_CACHE
ARG ANSIBLE_GALAXY_CLI_COLLECTION_OPTS
ARG ANSIBLE_GALAXY_CLI_ROLE_OPTS
ARG ANSIBLE_INSTALL_REFS
ARG PKGMGR

RUN $PYCMD -m ensurepip
RUN $PYCMD -m pip install --no-cache-dir $ANSIBLE_INSTALL_REFS
COPY _build/scripts/ /output/scripts/
COPY _build/scripts/entrypoint /opt/builder/bin/entrypoint

# Galaxy build stage
FROM base as galaxy
ARG EE_BASE_IMAGE
ARG PYCMD
ARG PKGMGR_PRESERVE_CACHE
ARG ANSIBLE_GALAXY_CLI_COLLECTION_OPTS
ARG ANSIBLE_GALAXY_CLI_ROLE_OPTS
ARG ANSIBLE_INSTALL_REFS
ARG PKGMGR

RUN /output/scripts/check_galaxy
COPY _build /build
WORKDIR /build

RUN ansible-galaxy role install $ANSIBLE_GALAXY_CLI_ROLE_OPTS -r requirements.yml --roles-path "/usr/share/ansible/roles"
RUN ANSIBLE_GALAXY_DISABLE_GPG_VERIFY=1 ansible-galaxy collection install $ANSIBLE_GALAXY_CLI_COLLECTION_OPTS -r requirements.yml --collections-path "/usr/share/ansible/collections"

# Builder build stage
FROM base as builder
WORKDIR /build
ARG EE_BASE_IMAGE
ARG PYCMD
ARG PKGMGR_PRESERVE_CACHE
ARG ANSIBLE_GALAXY_CLI_COLLECTION_OPTS
ARG ANSIBLE_GALAXY_CLI_ROLE_OPTS
ARG ANSIBLE_INSTALL_REFS
ARG PKGMGR

RUN $PYCMD -m pip install --no-cache-dir bindep pyyaml requirements-parser

COPY --from=galaxy /usr/share/ansible /usr/share/ansible

COPY _build/requirements.txt requirements.txt
COPY _build/bindep.txt bindep.txt
RUN $PYCMD /output/scripts/introspect.py introspect --sanitize --user-pip=requirements.txt --user-bindep=bindep.txt --write-bindep=/tmp/src/bindep.txt --write-pip=/tmp/src/requirements.txt
RUN /output/scripts/assemble

# Final build stage
FROM base as final
ARG EE_BASE_IMAGE
ARG PYCMD
ARG PKGMGR_PRESERVE_CACHE
ARG ANSIBLE_GALAXY_CLI_COLLECTION_OPTS
ARG ANSIBLE_GALAXY_CLI_ROLE_OPTS
ARG ANSIBLE_INSTALL_REFS
ARG PKGMGR

RUN /output/scripts/check_ansible $PYCMD

COPY --from=galaxy /usr/share/ansible /usr/share/ansible

COPY --from=builder /output/ /output/
RUN /output/scripts/install-from-bindep && rm -rf /output/wheels
RUN chmod ug+rw /etc/passwd
RUN mkdir -p /runner && chgrp 0 /runner && chmod -R ug+rwx /runner
WORKDIR /runner
RUN $PYCMD -m pip install --no-cache-dir 'dumb-init==1.2.5'
RUN wget https://get.helm.sh/helm-v3.14.0-linux-amd64.tar.gz -O - | tar -xz && mv linux-amd64/helm /usr/bin/helm && chmod +x /usr/bin/helm && rm -rf linux-amd64
RUN wget https://github.com/kubernetes-sigs/kustomize/releases/download/kustomize/v5.3.0/kustomize_v5.3.0_linux_amd64.tar.gz -O - | tar -xz && mv kustomize /usr/bin/kustomize && chmod +x /usr/bin/kustomize
RUN rm -rf /output
LABEL ansible-execution-environment=true
USER 1000
ENTRYPOINT ["/opt/builder/bin/entrypoint", "dumb-init"]
CMD ["bash"]
