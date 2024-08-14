# Go Poc for HTTP1.1 and HTTP/2

HTTP/2 multiplexes the tcp connection to reduce the connection establishment efforts.
However, you cannot think the truth that HTTP/2 is **always** better than HTTP1.1.
This is a **forked version** of original repo as a POC for [golang/go#47840](https://github.com/golang/go/issues/47840) 

## Single Request with Big Body Benchmark

This benchmark tries to send one request with big size body to test the throughput speed.
The conclusion here is **http/2 throughput speed is much slower than http/1** under single request scenarios.

Run `go run main.go` on Intel(R) Core(TM) i7-8559U CPU @ 2.70GHz gives the result below,
which is roughly x5 slower.

```
    server created and listening at 8765 (http1.1)
    server created and listening at 9876 (http/2 cleartext)
    downloading http://localhost:8765/10000000000
    receiving data with HTTP/1.1
    server sent 10000000000 bytes in 1.448894821s = 55.2 Gbps (262144 chunks)
    client received 10000000000 bytes in 1.448774771s = 55.2 Gbps, 189969 write ops, 2360568 buff 
    downloading http://localhost:9876/10000000000
    receiving data with HTTP/2.0
    server sent 10000000000 bytes in 8.313647415s = 9.6 Gbps (262144 chunks)
    client received 10000000000 bytes in 8.313532464s = 9.6 Gbps, 429444 write ops, 966656 buff 
```

## Multiple Requests Benchmark

The benchmark above focuses on the single request, here I try to send multiple request simultaneously.
The more requests, the faster HTTP/2 performs. Multiplexing has advantages when sending multiple requests. 

#### 10 Requests to Download 10GB  Simultaneously

> HTTP1.1: client totally received 100000000000 bytes in 4m24.693728209s = 3.0 Gbps
> 
> HTTP/2: client totally received 100000000000 bytes in 2m47.446945749s = 4.8 Gbps

#### 100 Requests to Download 1GB Simultaneously

> HTTP1.1: client totally received 100000000000 bytes in 53m39.600091582s = 248.5 Mbps
>
> HTTP/2: client totally received 100000000000 bytes in 14m0.016660672s = 952.4 Mbps

#### 1000 Requests to Download 100MB Simultaneously
When we send the requests simultaneously, the http1.1 starts to have a bottleneck 
while receives some errors such as `connection reset by peer`.

> HTTP1.1: client totally received 100000000 bytes in 30.864083ms = 25.9 Mbps
> 
> HTTP/2: client totally received 100000000000 bytes in 3h1m30.24906591s = 232.5 M

#### 1000 Requests to Download 1MB Simultaneously
When we send the requests simultaneously, the http1.1 starts to have a bottleneck
while receives some errors such as `connection reset by peer`.

> HTTP1.1: client totally received 133000000 bytes in 3.18824188s = 61.4 M
>
> HTTP/2: client totally received 1000000000 bytes in 1m17.611413593s = 655.3 M

## Single Request Benchmark via cURL or Caddy

### Benchmark via Curl


with curl: launch `go run main.go -s` and in a separate shell:

    curl -o /dev/null  http://localhost:8765/10000000000
    % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                 Dload  Upload   Total   Spent    Left  Speed
    100 9536M  100 9536M    0     0  5397M      0  0:00:01  0:00:01 --:--:-- 5394M    

    curl -o /dev/null --http2-prior-knowledge http://localhost:9876/10000000000
    % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                    Dload  Upload   Total   Spent    Left  Speed
    100 9536M  100 9536M    0     0   958M      0  0:00:09  0:00:09 --:--:--  974M

With nspeed: download nspeed at http://nspeed.app/
or execute [nspeed-batch.sh](nspeed-batch.sh)

http/1.1 vs http/2 no encryption:

    # http/1.1
    ./nspeed_linux_amd64 server -n 1 get -w 1 http://localhost:7333/10g
    # http/2 clear text
    ./nspeed_linux_amd64 server -n 1 get -h2c -w 1 http://localhost:7333/10g

http/1.1 vs http/2 with encryption:

    # http/1.1
    ./nspeed_linux_amd64 server -self -n 1 get -self -http11 -w 1 https://localhost:7333/10g
    # http/2
    ./nspeed_linux_amd64 server -self -n 1 get -self -w 1 https://localhost:7333/10g

example results: see [nspeed.results.txt](nspeed.results.txt)

### Caddy

    #https/2
    curl -o /dev/null https://localhost:8082/10G.iso
    % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                    Dload  Upload   Total   Spent    Left  Speed
    100 9536M  100 9536M    0     0   484M      0  0:00:19  0:00:19 --:--:--  483M

    #https/1.1
    curl -o /dev/null --http1.1 https://localhost:8082/10G.iso
    % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                    Dload  Upload   Total   Spent    Left  Speed
    100 9536M  100 9536M    0     0   929M      0  0:00:10  0:00:10 --:--:--  935M

    #http/1.1 (no encryption - max throughput reference)
    curl -o /dev/null http://localhost:8081/10G.iso
    % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                    Dload  Upload   Total   Spent    Left  Speed
    100 9536M  100 9536M    0     0  1672M      0  0:00:05  0:00:05 --:--:-- 1687M

