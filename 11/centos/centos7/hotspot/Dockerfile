FROM eclipse-temurin:11.0.19_7-jdk-centos7 as jre-build

# Generate smaller java runtime without unneeded files
# for now we include the full module path to maintain compatibility
# while still saving space (approx 200mb from the full distribution)
RUN jlink \
         --add-modules ALL-MODULE-PATH \
         --no-man-pages \
         --compress=2 \
         --output /javaruntime

FROM centos:centos7.9.2009

RUN yum install -y \
    curl \
    fontconfig \
    freetype \
    git \
    unzip \
    which \
    tzdata \
  && yum clean all

RUN curl -s https://packagecloud.io/install/repositories/github/git-lfs/script.rpm.sh -o /tmp/script.rpm.sh \
  && bash /tmp/script.rpm.sh \
  && rm -f /tmp/script.rpm.sh \
  && yum install -y \
    git-lfs \
  && yum clean all \
  && git lfs install

ENV LANG='en_US.UTF-8' LANGUAGE='en_US:en' LC_ALL='en_US.UTF-8'

ARG TARGETARCH
ARG COMMIT_SHA

ARG user=jenkins
ARG group=jenkins
ARG uid=1000
ARG gid=1000
ARG http_port=8080
ARG agent_port=50000
ARG JENKINS_HOME=/var/jenkins_home
ARG REF=/usr/share/jenkins/ref

ENV JENKINS_HOME $JENKINS_HOME
ENV JENKINS_SLAVE_AGENT_PORT ${agent_port}
ENV REF $REF

# Jenkins is run with user `jenkins`, uid = 1000
# If you bind mount a volume from the host or a data container,
# ensure you use the same uid
RUN mkdir -p $JENKINS_HOME \
  && chown ${uid}:${gid} $JENKINS_HOME \
  && groupadd -g ${gid} ${group} \
  && useradd -N -d "$JENKINS_HOME" -u ${uid} -g ${gid} -l -m -s /bin/bash ${user}

# Jenkins home directory is a volume, so configuration and build history
# can be persisted and survive image upgrades
VOLUME $JENKINS_HOME

# $REF (defaults to `/usr/share/jenkins/ref/`) contains all reference configuration we want
# to set on a fresh new installation. Use it to bundle additional plugins
# or config file with your custom jenkins Docker image.
RUN mkdir -p ${REF}/init.groovy.d

# Use tini as subreaper in Docker container to adopt zombie processes
ARG TINI_VERSION=v0.19.0
COPY tini_pub.gpg "${JENKINS_HOME}/tini_pub.gpg"
RUN curl -fsSL "https://github.com/krallin/tini/releases/download/${TINI_VERSION}/tini-static-${TARGETARCH}" -o /sbin/tini \
  && curl -fsSL "https://github.com/krallin/tini/releases/download/${TINI_VERSION}/tini-static-${TARGETARCH}.asc" -o /sbin/tini.asc \
  && gpg --no-tty --import "${JENKINS_HOME}/tini_pub.gpg" \
  && gpg --verify /sbin/tini.asc \
  && rm -rf /sbin/tini.asc /root/.gnupg \
  && chmod +x /sbin/tini

# jenkins version being bundled in this docker image
ARG JENKINS_VERSION
ENV JENKINS_VERSION ${JENKINS_VERSION:-2.410}

# jenkins.war checksum, download will be validated using it
ARG JENKINS_SHA=20e3436e1c05f1fa8c441d7fb41f2a797604194fd9f8e774acb74d47b6187e45

# Download directory, either `war` for weekly or `war-stable` for LTS
ARG RELEASE_LINE

# Can be used to customize jenkins.war download location
ARG JENKINS_URL=https://get.jenkins.io/${RELEASE_LINE}/${JENKINS_VERSION}/jenkins.war

# could use ADD but this one does not check Last-Modified header neither does it allow to control checksum
# see https://github.com/docker/docker/issues/8331
RUN curl -fsSL ${JENKINS_URL} -o /usr/share/jenkins/jenkins.war \
  && echo "${JENKINS_SHA}  /usr/share/jenkins/jenkins.war" >/tmp/jenkins_sha \
  && sha256sum -c --strict /tmp/jenkins_sha \
  && rm -f /tmp/jenkins_sha

ENV JENKINS_UC https://updates.jenkins.io
ENV JENKINS_UC_EXPERIMENTAL=https://updates.jenkins.io/experimental
ENV JENKINS_INCREMENTALS_REPO_MIRROR=https://repo.jenkins-ci.org/incrementals
RUN chown -R ${user} "$JENKINS_HOME" "$REF"

ARG PLUGIN_CLI_VERSION=2.12.13
ARG PLUGIN_CLI_URL=https://github.com/jenkinsci/plugin-installation-manager-tool/releases/download/${PLUGIN_CLI_VERSION}/jenkins-plugin-manager-${PLUGIN_CLI_VERSION}.jar
RUN curl -fsSL ${PLUGIN_CLI_URL} -o /opt/jenkins-plugin-manager.jar

# for main web interface:
EXPOSE ${http_port}

# will be used by attached agents:
EXPOSE ${agent_port}

ENV COPY_REFERENCE_FILE_LOG $JENKINS_HOME/copy_reference_file.log

ENV JAVA_HOME=/opt/java/openjdk
ENV PATH "${JAVA_HOME}/bin:${PATH}"
COPY --from=jre-build /javaruntime $JAVA_HOME

USER ${user}

COPY jenkins-support /usr/local/bin/jenkins-support
COPY jenkins.sh /usr/local/bin/jenkins.sh
COPY jenkins-plugin-cli.sh /bin/jenkins-plugin-cli

ENTRYPOINT ["/sbin/tini", "--", "/usr/local/bin/jenkins.sh"]

# from a derived Dockerfile, can use `RUN install-plugins.sh active.txt` to setup $REF/plugins from a support bundle
COPY install-plugins.sh /usr/local/bin/install-plugins.sh

# metadata labels
LABEL \
    org.opencontainers.image.vendor="Jenkins project" \
    org.opencontainers.image.title="Official Jenkins Docker image" \
    org.opencontainers.image.description="The Jenkins Continuous Integration and Delivery server" \
    org.opencontainers.image.version="${JENKINS_VERSION}" \
    org.opencontainers.image.url="https://www.jenkins.io/" \
    org.opencontainers.image.source="https://github.com/jenkinsci/docker" \
    org.opencontainers.image.revision="${COMMIT_SHA}" \
    org.opencontainers.image.licenses="MIT"
