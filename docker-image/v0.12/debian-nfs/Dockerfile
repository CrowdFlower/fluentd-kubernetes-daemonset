FROM 411719562396.dkr.ecr.us-east-1.amazonaws.com/crowdflower/ruby:2.3.5-v1.0.0

MAINTAINER Keith Woody <keith.woody@crowdflower.com>

###############################################################################
# extracted from https://raw.githubusercontent.com/fluent/fluentd-docker-image/master/v0.12/debian/Dockerfile
ENV DUMB_INIT_VERSION=1.2.0

ENV GOSU_VERSION=1.10

ARG DEBIAN_FRONTEND=noninteractive

# Do not split this into multiple RUN!
# Docker creates a layer for every RUN-Statement
# therefore an 'apt-get purge' has no effect
RUN apt-get update \
      && apt-get upgrade -y \
      && apt-get install -y --no-install-recommends \
                 ca-certificates \
                 ruby \
      && buildDeps=" \
           make gcc g++ libc-dev \
           ruby-dev \
           wget bzip2 gnupg dirmngr \
         " \
      && apt-get install -y --no-install-recommends $buildDeps \
      && update-ca-certificates \
      && echo 'gem: --no-document' >> /etc/gemrc \
      && gem install oj -v 2.18.3 \
      && gem install json -v 2.1.0 \
      && gem install fluentd -v 0.12.40 \
      && gem install fluent-plugin-record-reformer \
      && gem install fluent-plugin-kubernetes_metadata_filter \
      && gem install ffi \
      && gem install fluent-plugin-systemd -v 0.0.8 \
      && dpkgArch="$(dpkg --print-architecture | awk -F- '{ print $NF }')" \
      && wget -O /usr/bin/dumb-init https://github.com/Yelp/dumb-init/releases/download/v${DUMB_INIT_VERSION}/dumb-init_${DUMB_INIT_VERSION}_$dpkgArch \
      && chmod +x /usr/bin/dumb-init \
      && wget -O /usr/local/bin/gosu "https://github.com/tianon/gosu/releases/download/$GOSU_VERSION/gosu-$dpkgArch" \
      && wget -O /usr/local/bin/gosu.asc "https://github.com/tianon/gosu/releases/download/$GOSU_VERSION/gosu-$dpkgArch.asc" \
      && export GNUPGHOME="$(mktemp -d)" \
      && gpg --keyserver hkp://p80.pool.sks-keyservers.net:80 --recv-keys B42F6819007F00F88E364FD4036A9C25BF357DD4 \
      && gpg --batch --verify /usr/local/bin/gosu.asc /usr/local/bin/gosu \
      && rm -rf "$GNUPGHOME" /usr/local/bin/gosu.asc \
      && chmod +x /usr/local/bin/gosu \
      && gosu nobody true \
      && SUDO_FORCE_REMOVE=yes \
         apt-get purge -y --auto-remove \
                       -o APT::AutoRemove::RecommendsImportant=false \
                       $buildDeps \
      && rm -rf /var/lib/apt/lists/* \
      && rm -rf /tmp/* /var/tmp/* /usr/lib/ruby/gems/*/cache/*.gem

# for log storage (maybe shared with host)
RUN mkdir -p /fluentd/log
# configuration/plugins path (default: copied from .)
RUN mkdir -p /fluentd/etc /fluentd/plugins

COPY entrypoint.sh /bin/
RUN chmod +x /bin/entrypoint.sh

EXPOSE 24224 5140

ENTRYPOINT ["/bin/entrypoint.sh"]
# end extracted
################################################################################

USER root
WORKDIR /home/fluent
ENV PATH /home/fluent/.gem/ruby/2.3.0/bin:$PATH

# Copy configuration files
COPY ./conf/fluent.conf /fluentd/etc/
COPY ./conf/systemd.conf /fluentd/etc/
COPY ./conf/kubernetes.conf /fluentd/etc/

# Copy plugins
COPY plugins /fluentd/plugins/

# Environment variables
ENV FLUENTD_OPT=""
ENV FLUENTD_CONF="fluent.conf"

# Run Fluentd
CMD exec fluentd -c /fluentd/etc/$FLUENTD_CONF -p /fluentd/plugins $FLUENTD_OPT
