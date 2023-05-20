#build stage
FROM golang:1.20.4-alpine3.18 AS builder
RUN apk add git
WORKDIR /go/src/app
RUN git clone https://github.com/GhazouaniHichem/microservices-weather-app-code
WORKDIR /go/src/app/microservices-weather-app-code/auth/src/main
RUN go build -o /go/bin/app -v ./...

#final stage
FROM alpine:3.18.0
RUN apk --no-cache add ca-certificates
COPY --from=builder /go/bin/app /app
ENTRYPOINT /app
LABEL Name=auth Version=0.0.1
RUN addgroup --system admin
RUN adduser --system ghazouanihm --ingroup admin
USER ghazouanihm:admin
EXPOSE 8080
HEALTHCHECK --interval=1m --timeout=30s --retries=3 CMD curl --fail http://localhost:8080 || exit 1
