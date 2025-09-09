FROM ubuntu:20.04

# SPDX-FileCopyrightText: 2023 Nextcloud GmbH and Nextcloud contributors
# SPDX-License-Identifier: AGPL-3.0-or-later

ENV DEBIAN_FRONTEND=noninteractive
ARG TARGETARCH

RUN apt-get --assume-yes update
RUN apt-get --assume-yes upgrade

# Common dependencies
RUN apt-get --assume-yes install software-properties-common

# nextcloud-talk-recording dependencies
RUN apt-get --assume-yes install ffmpeg pulseaudio python3-pip xvfb
RUN pip3 install --upgrade requests

# firefox
RUN apt-get --assume-yes install firefox firefox-geckodriver

# chromium - architecture specific installation
RUN if [ "$TARGETARCH" = "amd64" ]; then \
        echo "deb https://freeshell.de/phd/chromium/focal /" > /etc/apt/sources.list.d/phd-chromium.list && \
        apt-key adv --keyserver keyserver.ubuntu.com --recv-keys 869689FE09306074 && \
        apt-get update && \
        apt-get --assume-yes install chromium; \
    else \
        apt-get --assume-yes install chromium-browser && \
        ln -sf /usr/bin/chromium-browser /usr/bin/chromium; \
    fi

COPY ./docker-compose/wrap_chromium_binary /opt/bin/wrap_chromium_binary
RUN /opt/bin/wrap_chromium_binary

# nextcloud-talk-recording config
RUN useradd --create-home recording

# Create template and working config at build time
COPY server.conf.in /etc/nextcloud-talk-recording/server.conf.template
COPY server.conf.in /etc/nextcloud-talk-recording/server.conf
RUN sed -i 's/#listen = 127.0.0.1:8000/listen = 0.0.0.0:8000/' /etc/nextcloud-talk-recording/server.conf
RUN chown recording:recording /etc/nextcloud-talk-recording/server.conf

# Create entrypoint that stays as root longer
RUN printf '#!/bin/bash\nset -e\n\n# Configure with environment variables if provided\nif [ -n "$NC_DOMAIN" ] && [ -n "$RECORDING_SECRET" ]; then\n  echo "Configuring with environment variables..."\n  cp /etc/nextcloud-talk-recording/server.conf.template /etc/nextcloud-talk-recording/server.conf\n  sed -i "s/#listen = 127.0.0.1:8000/listen = 0.0.0.0:8000/" /etc/nextcloud-talk-recording/server.conf\n  \n  # Add backend configuration\n  sed -i "s/#allowall = false/allowall = false/" /etc/nextcloud-talk-recording/server.conf\n  sed -i "s/#backends = backend-id, another-backend/backends = docker-backend/" /etc/nextcloud-talk-recording/server.conf\n  \n  echo "" >> /etc/nextcloud-talk-recording/server.conf\n  echo "[docker-backend]" >> /etc/nextcloud-talk-recording/server.conf\n  echo "url = https://$NC_DOMAIN" >> /etc/nextcloud-talk-recording/server.conf\n  echo "secret = $RECORDING_SECRET" >> /etc/nextcloud-talk-recording/server.conf\n  \n  if [ -n "$SIGNALING_URL" ] && [ -n "$SIGNALING_INTERNAL_SECRET" ]; then\n    sed -i "s/#signalings = signaling-id, another-signaling/signalings = docker-signaling/" /etc/nextcloud-talk-recording/server.conf\n    echo "" >> /etc/nextcloud-talk-recording/server.conf\n    echo "[docker-signaling]" >> /etc/nextcloud-talk-recording/server.conf\n    echo "url = $SIGNALING_URL" >> /etc/nextcloud-talk-recording/server.conf\n    echo "internalsecret = $SIGNALING_INTERNAL_SECRET" >> /etc/nextcloud-talk-recording/server.conf\n  fi\n  \n  chown recording:recording /etc/nextcloud-talk-recording/server.conf\nfi\n\n# Switch to recording user and run command\ncd /home/recording\nexec su recording -c "$*"\n' > /entrypoint.sh && chmod +x /entrypoint.sh

# Deploy recording server
RUN mkdir --parents /tmp/recording
COPY src /tmp/recording/
COPY pyproject.toml /tmp/recording/
RUN python3 -m pip install file:///tmp/recording/

# Cleanup
RUN apt-get clean && rm --recursive --force /var/lib/apt/lists/* /tmp/* /var/tmp/*
RUN rm --recursive --force /tmp/recording

# Start as root, then switch to recording user
WORKDIR "/home/recording/"

ENTRYPOINT ["/entrypoint.sh"]
CMD ["python3", "-m", "nextcloud.talk.recording", "--config", "/etc/nextcloud-talk-recording/server.conf"]