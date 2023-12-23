FROM cgr.dev/chainguard/wolfi-base
LABEL authors="Selenium <selenium-developers@googlegroups.com>"

# Arguments to define the version of dependencies to download
ARG VERSION=4.16.0
ARG RELEASE=selenium-${VERSION}
# Default value should be aligned with upstream Selenium (https://github.com/SeleniumHQ/selenium/blob/trunk/java/maven_deps.bzl)
ARG OPENTELEMETRY_VERSION=1.28.0
ARG GRPC_VERSION=1.57.1

#Arguments to define the user running Selenium
ARG SEL_USER=seluser
ARG SEL_GROUP=${SEL_USER}
ARG SEL_PASSWD=secret
ARG UID=1200
ARG GID=1201

USER root

#========================
# Miscellaneous packages
# Includes minimal runtime used for executing non GUI Java programs
#========================
RUN apk update && apk add \
    acl \
    bash \
    bzip2 \
    ca-certificates \
    openjdk-11-jre \
    openjdk-11-default-jvm \
    tzdata \
    sudo-rs \
    unzip \
    wget \
    jq \
    curl \
    supervisor \
    gnupg \
    libnss-tools \
    yq

RUN sed -i 's/securerandom\.source=file:\/dev\/random/securerandom\.source=file:\/dev\/urandom/' ./usr/lib/jvm/java-*-openjdk/conf/security/java.security

#===================
# Timezone settings
# Possible alternative: https://github.com/docker/docker/issues/3359#issuecomment-32150214
#===================
ENV TZ=UTC
RUN apk add --no-cache tzdata \
    && cp /usr/share/zoneinfo/${TZ} /etc/localtime \
    && echo "${TZ}" >  /etc/timezone \
    && apk del tzdata

#======================================
# Configure environement
#======================================
ENV SEL_USER=${SEL_USER}
ENV SEL_UID=${UID}
ENV SEL_GID=${GID}
ENV HOME=/home/${SEL_USER}
ENV SEL_DOWNLOAD_DIR=${HOME}/Downloads

#========================================
# Add normal user and group with passwordless sudo
#========================================
RUN mkdir -p /etc/sudoers.d \
  && addgroup -g ${SEL_GID} ${SEL_GROUP} \
  && adduser -D -u ${SEL_UID} -G ${SEL_GROUP} -s /bin/sh -h /home/${SEL_USER} ${SEL_USER} \
  && echo "${SEL_USER}:${SEL_PASSWD}" | chpasswd \
  && echo "${SEL_USER} ALL=(ALL) NOPASSWD: ALL" > /etc/sudoers.d/${SEL_USER} \
  && chmod 0440 /etc/sudoers.d/${SEL_USER}


#======================================
# Add Grid check script
#======================================
COPY --chown="${SEL_UID}:${SEL_GID}" check-grid.sh entry_point.sh /opt/bin/

#======================================
# Add Supervisor configuration file
#======================================
COPY supervisord.conf /etc

#==========
# Selenium & relaxing permissions for OpenShift and other non-sudo environments
#==========
RUN mkdir -p /opt/selenium /opt/selenium/assets /var/run/supervisor /var/log/supervisor ${SEL_DOWNLOAD_DIR} ${HOME}/.mozilla ${HOME}/.vnc \
  && touch /opt/selenium/config.toml \
  && chown -R ${SEL_USER}:${SEL_GROUP} /opt/selenium /var/run/supervisor /var/log/supervisor /etc/passwd ${HOME} \
  && chmod -R 775 /opt/selenium /var/run/supervisor /var/log/supervisor /etc/passwd ${HOME} \
  && wget --no-verbose https://github.com/SeleniumHQ/selenium/releases/download/${RELEASE}/selenium-server-${VERSION}.jar \
    -O /opt/selenium/selenium-server.jar \
  && echo "${SEL_PASSWD}" > /opt/selenium/initialPasswd \
  && chgrp -R 0 /opt/selenium ${HOME} /opt/selenium/assets /var/run/supervisor /var/log/supervisor \
  && chmod -R g=u /opt/selenium ${HOME} /opt/selenium/assets /var/run/supervisor /var/log/supervisor \
  && setfacl -Rm u:${SEL_USER}:rwx /opt /opt/selenium ${HOME} /opt/selenium/assets /var/run/supervisor /var/log/supervisor \
  && setfacl -Rm g:${SEL_GROUP}:rwx /opt /opt/selenium ${HOME} /opt/selenium/assets /var/run/supervisor /var/log/supervisor

#=====
# Download observability related jaegar jars and make them available in a separate directory
# so that the container can skip downloading them everytime it comes up
#=====
RUN curl -fLo /tmp/cs https://github.com/coursier/launchers/raw/master/coursier \
  && chmod +x /tmp/cs \
  && mkdir -p /external_jars \
  && chmod -R 775 /external_jars

RUN /tmp/cs fetch --classpath --cache /external_jars \
    io.opentelemetry:opentelemetry-exporter-otlp:${OPENTELEMETRY_VERSION} \
    io.opentelemetry:opentelemetry-exporter-jaeger:${OPENTELEMETRY_VERSION} \
    io.grpc:grpc-netty:${GRPC_VERSION} > /external_jars/.classpath.txt

RUN chmod 664 /external_jars/.classpath.txt
RUN rm -fr /root/.cache/*

#===================================================
# Run the following commands as non-privileged user
#===================================================
USER ${SEL_UID}:${SEL_GID}
VOLUME ${SEL_DOWNLOAD_DIR}

# Boolean value, maps "--bind-host"
ENV SE_BIND_HOST false

# A too high maximum number of file descriptors (with the default value
# inherited from the docker host) can cause issues with some of our tools:
#  - sanitizers hanging: https://github.com/google/sanitizers/issues/1662
#  - valgrind crashing: https://stackoverflow.com/a/75293014
# This is not be a problem on our CI hosts, but developers who run the image
# on their machines may run into this (e.g., on Arch Linux), so warn them.
# (Note that .bashrc is only executed in interactive bash shells.)
RUN echo 'if [[ $(ulimit -n) -gt 200000 ]]; then echo "WARNING: Very high value reported by \"ulimit -n\". Consider passing \"--ulimit nofile=32768\" to \"docker run\"."; fi' >> ${HOME}/.bashrc

CMD ["/opt/bin/entry_point.sh"]
