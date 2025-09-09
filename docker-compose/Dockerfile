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
# The phd/chromium repository for Ubuntu is used because since Ubuntu 20.04
# Chromium is provided as a snap package, and the equivalent PPA has been
# discontinued. However, this PPA only supports AMD64.
RUN if [ "$TARGETARCH" = "amd64" ]; then \
        echo "deb https://freeshell.de/phd/chromium/focal /" > /etc/apt/sources.list.d/phd-chromium.list && \
        apt-key adv --keyserver keyserver.ubuntu.com --recv-keys 869689FE09306074 && \
        apt-get update && \
        apt-get --assume-yes install chromium; \
    else \
        # For ARM64 and other architectures, install chromium-browser from Ubuntu repos
        apt-get --assume-yes install chromium-browser && \
        ln -sf /usr/bin/chromium-browser /usr/bin/chromium; \
    fi

COPY ./docker-compose/wrap_chromium_binary /opt/bin/wrap_chromium_binary
RUN /opt/bin/wrap_chromium_binary

# nextcloud-talk-recording config
RUN useradd --create-home recording

# Keep original as template, create entrypoint for env var support
COPY server.conf.in /etc/nextcloud-talk-recording/server.conf.in
RUN cat > /entrypoint.sh << 'EOF'
#!/bin/bash
set -e

# Copy template to config location
cp /etc/nextcloud-talk-recording/server.conf.in /etc/nextcloud-talk-recording/server.conf

# Basic configuration (always applied)
sed -i 's/#listen = 127.0.0.1:8000/listen = 0.0.0.0:8000/' /etc/nextcloud-talk-recording/server.conf

# Environment variable configuration (optional)
if [ -n "$NC_DOMAIN" ] && [ -n "$RECORDING_SECRET" ]; then
    BACKEND_ID=${BACKEND_ID:-"nextcloud-backend"}
    
    # Configure backend
    sed -i 's/#allowall = false/allowall = false/' /etc/nextcloud-talk-recording/server.conf
    sed -i "s/#backends = backend-id, another-backend/backends = $BACKEND_ID/" /etc/nextcloud-talk-recording/server.conf
    
    # Add backend section
    echo "" >> /etc/nextcloud-talk-recording/server.conf
    echo "[$BACKEND_ID]" >> /etc/nextcloud-talk-recording/server.conf
    echo "url = https://$NC_DOMAIN" >> /etc/nextcloud-talk-recording/server.conf
    echo "secret = $RECORDING_SECRET" >> /etc/nextcloud-talk-recording/server.conf
    
    # Add signaling if provided
    if [ -n "$SIGNALING_URL" ] && [ -n "$SIGNALING_INTERNAL_SECRET" ]; then
        SIGNALING_ID=${SIGNALING_ID:-"signaling-server"}
        sed -i "s/#signalings = signaling-id, another-signaling/signalings = $SIGNALING_ID/" /etc/nextcloud-talk-recording/server.conf
        
        echo "" >> /etc/nextcloud-talk-recording/server.conf
        echo "[$SIGNALING_ID]" >> /etc/nextcloud-talk-recording/server.conf
        echo "url = $SIGNALING_URL" >> /etc/nextcloud-talk-recording/server.conf
        echo "internalsecret = $SIGNALING_INTERNAL_SECRET" >> /etc/nextcloud-talk-recording/server.conf
    fi
fi

# Set ownership
chown recording:recording /etc/nextcloud-talk-recording/server.conf

exec "$@"
EOF

RUN chmod +x /entrypoint.sh

# Deploy recording server
RUN mkdir --parents /tmp/recording
COPY src /tmp/recording/
COPY pyproject.toml /tmp/recording/
RUN python3 -m pip install file:///tmp/recording/

# Cleanup
RUN apt-get clean && rm --recursive --force /var/lib/apt/lists/* /tmp/* /var/tmp/*
RUN rm --recursive --force /tmp/recording

# Switch user and start the recording server
WORKDIR "/home/recording/"
USER "recording"

ENTRYPOINT ["/entrypoint.sh"]
CMD ["python3", "-m", "nextcloud.talk.recording", "--config", "/etc/nextcloud-talk-recording/server.conf"]