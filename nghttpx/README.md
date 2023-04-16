## Purpose 
This directory contains a Dockerfile that builds the [nghttpx](https://nghttp2.org/documentation/nghttpx-howto.html), HTTP/2 proxy and load balancer, using [OpenSSL v3](https://github.com/openssl/openssl) and [OQS provider](https://github.com/open-quantum-safe/oqs-provider) which allows nghttpx to negotiate quantum-safe keys in TLS 1.3.

## Getting started

### Building
Assuming Docker is [installed](https://docs.docker.com/install), the following command

```
docker network create nghttpx-test
docker build -t nghttpx .
docker run --name nghttpx --network nghttpx-test -it nghttpx
```

will run the container for the PQ-enabled nghttpx on the docker network called nghttpx-test

### Testing
### Reverse Proxy server
After running the nghttpx container, verify that nghttpx performs quantum-safe key exchange by running test servers in separate containers using the following command

```bash
docker run --rm --name oqs-nginx1 --network nghttpx-test openquantumsafe/nginx
```

Then on the command prompt in the nghttpx docker container, set up the proxy server by running 
```bash
nghttpx /certs/server.key /certs/server.crt -f "*,4433;" -b "oqs-nginx1,8080" --ecdh-curves kyber512:p256_bikel1  --no-ocsp
```
This command starts the nghttpx server using the specified key and certificate, listening on all IP addresses on port 4433. It forwards incoming requests to a backend server listening on port 8080. Additionally, it specifies Kyber512 and P-256/BiKeL1 for key exchange.
Note that the front end of nghttpx is secured by TLS by default and that the backend is unencrypted.

To interact with the proxy server, start another container and run the following commands:
```bash
# start curl container
docker run --rm -it --network nghttpx-test openquantumsafe/curl
curl https://nghttpx:4433  --insecure --curves kyber512
```

 This command will send a request to the nghttpx server, which will forward it to the OQS-enabled nginx server. The `--curves` option specifies the curve for the key exchange with the proxy server.

[See list of supported key exchange algorithms here](https://github.com/open-quantum-safe/oqs-provider#algorithms
).

### Load balancing
For load balancing, you can modify the nghttpx command to include multiple backend servers.

```bash
# First run a second oqs-nginx instance
docker run --rm --name oqs-nginx2 --network nghttpx-test openquantumsafe/nginx
```

On the nghttpx container, run
```bash
nghttpx /certs/server.key /certs/server.crt -f "*,4433;" -b "oqs-nginx1,8080;" --ecdh-curves kyber512:p256_bikel1  -b "oqs-nginx2,8080;" --no-ocsp
```

In this example, the nghttpx server will distribute incoming requests to two backend servers, each running on the same port 8080, but with different hostnames. You can add as many backend servers as needed.

Interact with the load balancer using the following commands:

```
curl https://nghttpx:4433  --insecure --curves kyber512
```
Running this command multiple times will demonstrate how nghttpx distributes incoming requests among the backend servers. By default, nghttpx uses a round-robin load balancing algorithm.


### Usage
Documentation for using the nghttpx docker image is contained in the separate [USAGE.md](./USAGE.md) file.


## Disclaimer

[THIS IS NOT FIT FOR PRODUCTION USE](https://github.com/open-quantum-safe/openssl#limitations-and-security).
