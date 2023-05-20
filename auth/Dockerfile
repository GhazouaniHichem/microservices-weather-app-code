#build stage
FROM golang:1.20.4-alpine3.18 AS builder
RUN apk add git
WORKDIR /go/src/app
RUN git clone https://github.com/GhazouaniHichem/microservices-weather-app-code
WORKDIR /go/src/app/microservices-weather-app-code/auth/src/main
RUN go build -o /go/bin/app -v ./...

#final stage
FROM alpine:latest
RUN apk --no-cache add ca-certificates
COPY --from=builder /go/bin/app /app
ENTRYPOINT /app
LABEL Name=auth Version=0.0.1
EXPOSE 8080
