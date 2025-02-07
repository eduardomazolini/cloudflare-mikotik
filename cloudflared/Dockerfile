FROM arm32v7/alpine:latest

RUN apk update \
    && apk upgrade

RUN wget -O cloudflared https://github.com/cloudflare/cloudflared/releases/latest/download/cloudflared-linux-arm

RUN chmod 755 cloudflared

COPY --chmod=755 <<EOT /entrypoint.sh
#!/usr/bin/env sh
set -e
echo TUNNEL_EDGE_BIND_ADDRESS "\$TUNNEL_EDGE_BIND_ADDRESS"
echo TUNNEL_GRACE_PERIOD "\$TUNNEL_GRACE_PERIOD"
echo TUNNEL_LOGFILE "\$TUNNEL_LOGFILE"
echo TUNNEL_LOGLEVEL "\$TUNNEL_LOGLEVEL"
echo TUNNEL_METRICS "\$TUNNEL_METRICS"
echo NO_AUTOUPDATE "\$NO_AUTOUPDATE"
echo TUNNEL_ORIGIN_CERT "\$TUNNEL_ORIGIN_CERT"
echo TUNNEL_PIDFILE "\$TUNNEL_PIDFILE"
echo TUNNEL_TRANSPORT_PROTOCOL "\$TUNNEL_TRANSPORT_PROTOCOL"
echo TUNNEL_REGION "\$TUNNEL_REGION"
echo TUNNEL_RETRIES "\$TUNNEL_RETRIES"
echo TUNNEL_TAG "\$TUNNEL_TAG"
echo TUNNEL_TOKEN "\$TUNNEL_TOKEN"
/cloudflared \$*
EOT

ENTRYPOINT ["/entrypoint.sh"]

CMD ["tunnel", "run"]
