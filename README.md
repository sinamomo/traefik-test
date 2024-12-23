# Doc of everything that was done 
 
Started installing GO based on: https://go.dev/doc/install

## Application

The GO version installed is `1.23.4` and is installed in my WSL environment.

Just ran `go run app.go` and it's requiring a certificate. Based on the message received. It is necessary a certificate:

```
# go run app.go
go: downloading github.com/gorilla/websocket v1.4.2
Starting up on port 80
2024/12/16 12:48:22 You need to provide a certificate
exit status 1
```

Inside the code it can be seen that the application requires the certificate's PEM files at `/cert/` as below:

```
func main() {
	flag.Parse()

	http.HandleFunc("/data", dataHandler)
	http.HandleFunc("/echo", echoHandler)
	http.HandleFunc("/bench", benchHandler)
	http.HandleFunc("/", whoamiHandler)
	http.HandleFunc("/api", apiHandler)
	http.HandleFunc("/health", healthHandler)

	fmt.Println("Starting up on port " + port)

	server := &http.Server{
		Addr: ":" + port,
	}

	_, err := os.Stat("/cert/cert.pem")
	if err != nil {
		log.Fatal("You need to provide a certificate")
	}

	_, err = os.Stat("/cert/key.pem")
	if err != nil {
		log.Fatal("You need to provide a certificate")
	}
	log.Fatal(server.ListenAndServeTLS("/cert/cert.pem", "/cert/key.pem"))
}
```

Moreover, I just noticed that in `go.mod` it uses GO 1.13. When shipping the Docker container I'll try to use a GO 1.13 image.


I created an AWS EC2 instance in Ohio(US-EAST-2). It's a T3.Micro with 2GB of RAM and 2vCPUs.

I don't intend to setup a cluster with multiple nodes as, for now, this doesn't sound necessary for the application. Gérald also mentioned that the certificate should be in a PVC.

I haven't tackled the issue yet. I decided to test it locally. I generated local certificates using the `mkcert` tool to make sure no additional steps would be required for the environment.

It sounds it's getting all the IPs of my environment:

```
# curl https://localhost:80
Hostname: DESKTOP-PGR3UH5
IP: 127.0.0.1
IP: 10.255.255.254
IP: ::1
IP: 172.20.22.76
IP: fe80::215:5dff:fe87:1071
IP: 192.168.49.1
IP: 172.17.0.1
IP: fe80::42:ebff:fefd:6636
IP: 172.22.0.1
IP: 172.18.0.1
IP: fe80::42:83ff:fe52:ee1a
IP: fe80::50d3:12ff:fe67:60f4
IP: fe80::482c:4eff:fe79:d0e1
RemoteAddr: 127.0.0.1:53750
GET / HTTP/1.1
Host: localhost:80
User-Agent: curl/7.81.0
Accept: */*
```

I also decided to check all the routes in the code:

- `/data` - It queries the URL request and returns a file of the size of the requested size in alphabetical order:
```
# curl 'https://localhost:80/data?size=1&unit=kb'
|ABCDEFGHIJKLMNOPQRSTUVWXYZ-ABCDEFGHIJKLMNOPQRSTUVWXYZ-ABCDEFGHIJKLMNOPQRSTUVWXYZ-ABCDEFGHIJKLMNOPQRSTUVWXYZ-ABCDEFGHIJKLMNOPQRSTUVWXYZ-ABCDEFGHIJKLMNOPQRSTUVWXYZ-ABCDEFGHIJKLMNOPQRSTUVWXYZ-ABCDEFGHIJKLMNOPQRSTUVWXYZ-ABCDEFGHIJKLMNOPQRSTUVWXYZ-ABCDEFGHIJKLMNOPQRSTUVWXYZ-ABCDEFGHIJKLMNOPQRSTUVWXYZ-ABCDEFGHIJKLMNOPQRSTUVWXYZ-ABCDEFGHIJKLMNOPQRSTUVWXYZ-ABCDEFGHIJKLMNOPQRSTUVWXYZ-ABCDEFGHIJKLMNOPQRSTUVWXYZ-ABCDEFGHIJKLMNOPQRSTUVWXYZ-ABCDEFGHIJKLMNOPQRSTUVWXYZ-ABCDEFGHIJKLMNOPQRSTUVWXYZ-ABCDEFGHIJKLMNOPQRSTUVWXYZ-ABCDEFGHIJKLMNOPQRSTUVWXYZ-ABCDEFGHIJKLMNOPQRSTUVWXYZ-ABCDEFGHIJKLMNOPQRSTUVWXYZ-ABCDEFGHIJKLMNOPQRSTUVWXYZ-ABCDEFGHIJKLMNOPQRSTUVWXYZ-ABCDEFGHIJKLMNOPQRSTUVWXYZ-ABCDEFGHIJKLMNOPQRSTUVWXYZ-ABCDEFGHIJKLMNOPQRSTUVWXYZ-ABCDEFGHIJKLMNOPQRSTUVWXYZ-ABCDEFGHIJKLMNOPQRSTUVWXYZ-ABCDEFGHIJKLMNOPQRSTUVWXYZ-ABCDEFGHIJKLMNOPQRSTUVWXYZ-ABCDEFGHIJKLMNOPQRSTUVWXYZ-ABCDEFGHIJKLMNOPQRSTUVWXYZ-ABCDEFGHIJKLMNOPQRSTUVWXYZ-ABCDEFGHIJKLMNOPQRSTUVWXYZ-ABCDEFGHIJKLMNOPQRSTUVWXYZ-ABCDEFGHIJKLMNOPQRSTUVWXYZ-ABCDEFGHIJKLMNOPQRSTUVW|
```

- `/echo` - It sounds like it upgrades the connection. Based on what I [searched](https://stackoverflow.com/questions/50204967/what-is-websocket-upgrader-exactly): `A summary of the process is this: The client sends an HTTP request requesting that the server upgrade the connection used for the HTTP request to the WebSocket protocol. The server inspects the request and if all is good the server sends an HTTP response agreeing to upgrade the connection. From that point on, the client and server use the WebSocket protocol over the network connection.`:
```
# curl https://localhost:80/echo
Bad Request
```

- `/bench` - I was not aware of the KeepAlive message. It returns a `1`:

```
# curl https://localhost:80/bench
1
```

- `/api` - Returns a JSON of all IPs and other information:

```
# curl https://localhost:80/api | jq .
{
  "hostname": "DESKTOP-PGR3UH5",
  "ip": [
    "127.0.0.1",
    "10.255.255.254",
    "::1",
    "172.20.22.76",
    "fe80::215:5dff:fe87:1071",
    "192.168.49.1",
    "172.17.0.1",
    "fe80::42:ebff:fefd:6636",
    "172.22.0.1",
    "172.18.0.1",
    "fe80::42:83ff:fe52:ee1a",
    "fe80::50d3:12ff:fe67:60f4",
    "fe80::482c:4eff:fe79:d0e1"
  ],
  "headers": {
    "Accept": [
      "*/*"
    ],
    "User-Agent": [
      "curl/7.81.0"
    ]
  },
  "url": "/api",
  "host": "localhost:80",
  "method": "GET"
}
```

- `/health` was the most challeging one and I had to ask for ChatGPT's help. I was trying to pass `statusCode` as a JSON ```-d '{statusCode":"200"}'``` rather than the integer itself ```-d '200'```. I also referenced this [page](https://www.alexedwards.net/blog/how-to-properly-parse-a-json-request-body):

```
# curl -vvv -X POST https://localhost:80/health -H "Content-Type: application/json" -d '200'
# go run app.go 
Starting up on port 80
Update health check status code [200]
```

I also modified the Port of the application from `80` to `443` as this is a HTTPS application and I'm afraid this might cause some problems with browsers.

## Docker 

I built the application following the [Docker docs](https://docs.docker.com/guides/golang/build-images/).

For now, my Dockerfile looks like this:

```
FROM golang:1.13-alpine

WORKDIR /

COPY Makefile app.go go.mod go.sum LICENSE ./
RUN apk add make
RUN make build

EXPOSE 443

CMD ["/whoami"]
```

Initially I used the `golang:1.13`, however, the image was too big:
```
REPOSITORY                                                          TAG                  IMAGE ID       CREATED         SIZE
traefik                                                             latest               56cdca51611c   7 seconds ago   837MB
```

So I decided to the switch for the Alpine one so it could be a little bit smaller:
```
REPOSITORY                                                          TAG                  IMAGE ID       CREATED              SIZE
traefik                                                             latest               1fda0a6e3366   5 seconds ago        396MB
```

I decided to check if the application was fine by running the image. However, I was receiveing the error below:

```
docker: Error response from daemon: failed to create task for container: failed to create shim task: OCI runtime create failed: runc create failed: unable to start container process: exec: "-d": executable file not found in $PATH: unknown.
```

It sounded that the compiled program was not being sent to the `/` folder. So I modified it a little bit:

```
build:
        CGO_ENABLED=0 go build -a --trimpath --installsuffix cgo --ldflags="-s" -o /whoami
```

So once compiled it could be found at `/whoami`. I received the error message requesting for a certificate, which was already expected once the certificate should be in a PVC:

```
# docker run traefik:latest
Starting up on port 443
2024/12/16 19:01:00 You need to provide a certificate
```

## AWS

I had to increase the size of the EC2 instance to T2.large. It looks like the resources weren't enough.

I had a domain to spare: `achapetcanoas.com.br`. So I decided to use it to issue my certificate:

```
# certbot --apache -d achapetcanoas.com.br
Saving debug log to /var/log/letsencrypt/letsencrypt.log
Requesting a certificate for achapetcanoas.com.br

Successfully received certificate.
Certificate is saved at: /etc/letsencrypt/live/achapetcanoas.com.br/fullchain.pem
Key is saved at:         /etc/letsencrypt/live/achapetcanoas.com.br/privkey.pem
This certificate expires on 2025-03-16.
These files will be updated when the certificate renews.
Certbot has set up a scheduled task to automatically renew this certificate in the background.

Deploying certificate
Successfully deployed certificate for achapetcanoas.com.br to /etc/apache2/sites-available/000-default-le-ssl.conf
Congratulations! You have successfully enabled HTTPS on https://achapetcanoas.com.br
```

As for the cluster deploy I opted out for Minikube as the intention is to run a single master node. The steps taken may be found [here](https://crishantha.medium.com/running-minikube-on-aws-ec2-e845337a956).

I decided to deploy a simple pod with a PV and PVC but Kubernetes is not able to consume local images directly so I decided to use an ECR that I had to upload the image and pull from there. To do this, I had to first install the [AWS CLI](https://docs.aws.amazon.com/cli/latest/userguide/getting-started-install.html) and configure it.

Afterwards, I had some issues on how to mount the certs in the PVC. To resolve this I opted out to not to go to the safest route and used the `hostPath` mount at `/cert/` so I could mount the certificates inside the pod. For reference, I used [Kubernete's docs](https://kubernetes.io/docs/tasks/configure-pod-container/configure-persistent-volume-storage/) to resolve this issue.

I also did some modifications to the application to make sure the certs were being properly mounted. The piece of code was added below:

```
files, err := filepath.Glob("/cert/*")
if err != nil {
  log.Fatal(err)
}
log.Fatal(files) // contains a list of all files in the current directory

_, err = os.Stat("/cert/cert.pem")
if err != nil {
  log.Fatal("You need to provide a certificate - cert.pem not found")
}

_, err = os.Stat("/cert/key.pem")
if err != nil {
  log.Fatal("You need to provide a certificate - key.pem not found")
}
``` 

Funny enough, when looking for the ingress configuration in Traefik I found the [complete whoami application](https://github.com/traefik/whoami).

It was requested that the certificates should be in a PVC, which was strange to me since I could add a secret to run the application. However, I did not notice that I could deploy Traefik to manage this to me. I was avoiding using traefik rather than using it. I just realized this after asking a question to Gérald.

I also had some problems with Minikube, due to using node values with Minikube as it requires to mount the volume before using it and this causes some problems, so I decided to change and installed `Kubeadm` instead.

For now, I have this architecture:

- EC2 application running in a K8s cluster.
- The application is exposed through a service without an ingress and ingress controller.
- An application load balancer that targets the EC2 instance.
- I had a domain of my own that I used for the test so I used it in Route53.

I plan to change the architecture to use Traefik so it can manage this issue as well it already provides metrics using Prometheus.

The new architecture looks like this:

- 2 EC2 instances.
- Route 53 routes and load balances the two instances by geoproximity.
- Both server are running Kubeadm kubernetes cluster with Traefik and the test application.

**"Nice to have"** that were requested and I was able to conclude:

- **Load balance between two clusters**: Via Route 53
- **Liveness and Readiness probes**: Added in the deployment of the application.
- **Service monitoring**: Unfortunately, I did not find the time to configure them in a satisfatory way as the Prometheus operator is installed and aparentely picking up Traefik's metrics but there is no observability of the data.

## Points to improve:

- I was able to template the application in a helm repository and it can be found at the `traefik-test` folder. However, it is also included in Traefik's main template, which may cause some confusion. It was let the to facilitate the deployment of the environment.
- There is no wildcard certificate, instead I deployed them individually for `achapetcanoas.com.br` and `dashboard.achapetcanoas.com.br`. Therefore, issuing a wildcard certifica using cert-manager would be an better option here.
- Since there are 2 `A` records in my DNS, sometimes ACME may route to a different instance, and due to this, can't issue/renew the certificate 100% of the time.
- AWS ECR registry authorization only lasts for 12 hours, so a new secret have to be created everytime a new image is created. I'm aware of plugins that can resolve this issue but since this was not in scope and to keep the simplicity of this test I decided to not add them.
- Inside the cluster, [SSL verification is disabled](https://doc.traefik.io/traefik/routing/overview/#insecureskipverify) as the certificates that I issued were not trusted by the Kubernetes cluster. I intended to issue new certificates and add the rootCA as described [here](https://doc.traefik.io/traefik/routing/overview/#rootcas), however, due to time, I was not able to.
- Everything was deploy in the default namespace. I could have separated them.

## Problems that I faced throughout this challenge

- Faced the Let's Encrypt [#396](https://github.com/traefik/traefik-helm-chart/issues/396) PVC problem, when issuing the certificates and adding the initContainer script I was able to resolve it.
- Kubernetes is not able to use Docker's local images, so I had to create an ECR registry in AWS so I could consume the images. The authorization only lasts for 12 hours so I have to recreate the secret so the images can be pulled again if necessary.
- The certificates inside the application were not trusted by the Kubernetes cluster, this caused a `bad certificate` error as a form of resolving them I skipped SSL verification inside the cluster.

# **Hope you like it!**
