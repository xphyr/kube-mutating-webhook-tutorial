FROM gcr.io/distroless/base-debian10

# set environment variables
ENV SIDECAR_INJECTOR=/usr/local/bin/sidecar-injector \
  USER_UID=1001 \
  USER_NAME=sidecar-injector

WORKDIR /

COPY build/_output/linux/bin/mwh-tutorial /mwh-tutorial

EXPOSE 8080

# switch to non-root user
# USER ${USER_UID}
USER nonroot:nonroot

ENTRYPOINT ["/mwh-tutorial"]