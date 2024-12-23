FROM golang:1.13-alpine

WORKDIR /

COPY Makefile app.go go.mod go.sum LICENSE ./
RUN apk add make
RUN make build

EXPOSE 443
CMD ["/whoami"]
