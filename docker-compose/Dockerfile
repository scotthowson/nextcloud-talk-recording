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

# Keep original as template
COPY server.conf.in /etc/nextcloud-talk-recording/server.conf.template

# Create simple entrypoint
RUN cat > /entrypoint.sh << 'EOF'
#!/bin/bash
set -e

echo "Starting Nextcloud Talk Recording Server..."

# Copy template to working config
cp /etc/nextcloud-talk-recording/server.conf.template /etc/nextcloud-talk-recording/server.conf

# Basic configuration
sed -i 's/#listen = 127.0.0.1:8000/listen = 0.0.0.0:8000/' /etc/nextcloud-talk-recording/server.conf

# Add environment-based configuration if variables are set
if [ -n "$NC_DOMAIN" ] && [ -n "$RECORDING_SECRET" ]; then
    echo "Configuring with environment variables..."
    
    # Set backend configuration
    sed -i 's/#allowall = false/allowall = false/' /etc/nextcloud-talk-recording/server.conf
    sed -i 's/#backends = backend-id, another-backend/backends = docker-backend/' /etc/nextcloud-talk-recording/server.conf
    
    # Add backend section at end of file
    cat >> /etc/nextcloud-talk-recording/server.conf << BACKEND_END

[docker-backend]
url = https://$NC_DOMAIN
secret = $RECORDING_SECRET
BACKEND_END

    # Add signaling configuration if provided
    if [ -n "$SIGNALING_URL" ] && [ -n "$SIGNALING_INTERNAL_SECRET" ]; then
        echo "Adding signaling server configuration..."
        sed -i 's/#signalings = signaling-id, another-signaling/signalings = docker-signaling/' /etc/nextcloud-talk-recording/server.conf
        
        cat >> /etc/nextcloud-talk-recording/server.conf << SIGNALING_END

[docker-signaling]
url = $SIGNALING_URL
internalsecret = $SIGNALING_INTERNAL_SECRET
SIGNALING_END
    fi
else
    echo "No environment variables set, using basic configuration"
fi

# Set proper ownership
chown recording:recording /etc/nextcloud-talk-recording/server.conf

echo "Configuration complete, starting recording server..."
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