FROM golang:1.25-alpine3.22 AS build

WORKDIR /src

COPY go.mod go.sum ./
RUN go mod download

COPY . .
RUN CGO_ENABLED=0 GOOS=linux go build \
    -trimpath \
    -ldflags="-s -w" \
    -o /out/momo-api \
    ./cmd/api

FROM golang:1.25-alpine3.22 AS dev

WORKDIR /src

ENV CGO_ENABLED=0 \
    GOOS=linux \
    GOCACHE=/tmp/go-build \
    GOPATH=/tmp/go

RUN addgroup -S app -g 10001 \
    && adduser -S -D -H -u 10001 -G app app

COPY go.mod go.sum ./
RUN go mod download

COPY . .
RUN chown -R app:app /src /tmp

USER app

EXPOSE 8081

CMD ["go", "run", "./cmd/api"]

FROM alpine:3.22

RUN addgroup -S app -g 10001 \
    && adduser -S -D -H -u 10001 -G app app \
    && apk add --no-cache ca-certificates

WORKDIR /app

COPY --from=build /out/momo-api /app/momo-api

USER app

EXPOSE 8081

HEALTHCHECK --interval=30s --timeout=3s --start-period=5s --retries=3 \
    CMD ["/bin/busybox", "wget", "-qO-", "http://127.0.0.1:8081/health"]

ENTRYPOINT ["/app/momo-api"]
