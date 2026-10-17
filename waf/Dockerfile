# AlgoCrack API WAF. ModSecurity v3 + OWASP CRS on the official nginx image.
# Config under /etc/nginx/templates is rendered at start. MODSEC_RULE_ENGINE
# defaults to DetectionOnly; set it to On to block.
FROM owasp/modsecurity-crs:4.25.2-nginx-202610051210-lts

USER root

COPY nginx.conf /etc/nginx/templates/nginx.conf.template
COPY conf.d/default.conf /etc/nginx/templates/conf.d/default.conf.template
COPY modsecurity.conf /etc/nginx/templates/modsecurity.d/modsecurity.conf.template
COPY rules/REQUEST-900-EXCLUSION-RULES-BEFORE-CRS.conf /opt/owasp-crs/rules/REQUEST-900-EXCLUSION-RULES-BEFORE-CRS.conf

# Local/demo defaults. Compose and the cluster override BACKEND and the engine mode.
ENV MODSEC_RULE_ENGINE=DetectionOnly \
    BLOCKING_PARANOIA=1 \
    DETECTION_PARANOIA=1 \
    ANOMALY_INBOUND=5 \
    ANOMALY_OUTBOUND=4 \
    MODSEC_REQ_BODY_ACCESS=On \
    MODSEC_REQ_BODY_LIMIT=1048576 \
    MODSEC_REQ_BODY_NOFILES_LIMIT=1048576 \
    MODSEC_REQ_BODY_LIMIT_ACTION=Reject \
    MODSEC_AUDIT_ENGINE=RelevantOnly \
    MODSEC_AUDIT_LOG=/dev/stdout \
    MODSEC_AUDIT_LOG_FORMAT=JSON \
    MODSEC_AUDIT_LOG_PARTS=AHKZ \
    REAL_IP_HEADER=X-Real-IP \
    REAL_IP_RECURSIVE=off \
    ALLOWED_METHODS="GET HEAD POST PUT PATCH DELETE OPTIONS" \
    BACKEND=http://api-gateway:9090 \
    PORT=8080 \
    SERVER_NAME=localhost \
    SERVER_TOKENS=off \
    LOGLEVEL=warn \
    ERRORLOG=/dev/stderr \
    ACCESSLOG=/dev/stdout

USER nginx
