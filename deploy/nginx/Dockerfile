FROM nginxinc/nginx-unprivileged:1.29-alpine3.22

USER root
RUN apk upgrade --no-cache
USER 101

COPY backend-lb.conf /etc/nginx/conf.d/default.conf

EXPOSE 8080

HEALTHCHECK --interval=30s --timeout=3s --start-period=5s --retries=3 \
    CMD ["/bin/busybox", "wget", "-qO-", "http://127.0.0.1:8080/health"]
