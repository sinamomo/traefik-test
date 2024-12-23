FROM golang:1.13-alpine

WORKDIR /

COPY Makefile app.go go.mod go.sum LICENSE ./
RUN apk add make
RUN apk add openssl
RUN openssl req -x509 -newkey rsa:4096 -keyout /key.pem -out /cert.pem -sha256 -days 3650 -nodes -subj "/C=XX/ST=StateName/L=CityName/O=CompanyName/OU=CompanySectionName/CN=CommonNameOrHostname"
RUN make build

EXPOSE 443
CMD ["/whoami"]
