# Rate limiting

Rate limiting allows to control the number of requests going to the SFTPGo services.

SFTPGo implements a [token bucket](https://en.wikipedia.org/wiki/Token_bucket){:target="_blank"} initially full and refilled at the configured rate. The `burst` configuration parameter defines the size of the bucket. The rate is defined by dividing `average` by `period`, so for a rate below 1 req/s, one needs to define a period larger than a second.

Requests that exceed the configured limit will be delayed or denied if they exceed the maximum delay time.

SFTPGo allows to define per-protocol rate limiters so you can have different configurations for different protocols.

The supported protocols are:

- `SSH`, includes SFTP and SSH commands
- `FTP`, includes FTP, FTPES, FTPS
- `DAV`, WebDAV
- `HTTP`, REST API and web admin

You can also define two types of rate limiters:

- global, it is independent from the source host and therefore define an aggregate limit for the configured protocol/s
- per-host, this type of rate limiter can be connected to the built-in [defender](./defender.md) and generate `score_limit_exceeded` events and thus hosts that repeatedly exceed the configured limit can be automatically blocked

If you configure a per-host rate limiter, SFTPGo will keep a rate limiter in memory for each host that connects to the service, you can limit the memory usage using the `entries_soft_limit` and `entries_hard_limit` configuration keys.

You can exclude a list of IP addresses and IP ranges from rate limiters by adding them to rate limites allow list using the WebAdmin UI or the REST API. In multi-nodes setups, the list entries propagation between nodes may take some minutes.

You can define as many rate limiters as you want, but keep in mind that if you define multiple rate limiters each request will be checked against all the configured limiters and so it can potentially be delayed multiple times. Let's clarify with an example, here is a configuration that defines a global rate limiter and a per-host rate limiter for the SSH and FTP protocols:

```json
"rate_limiters": [
    {
      "average": 100,
      "period": 1000,
      "burst": 1,
      "type": 1,
      "protocols": [
        "SSH",
        "FTP",
        "DAV",
        "HTTP"
      ],
      "generate_defender_events": false,
      "entries_soft_limit": 100,
      "entries_hard_limit": 150
    },
    {
      "average": 10,
      "period": 1000,
      "burst": 1,
      "type": 2,
      "protocols": [
        "SSH",
        "FTP"
      ],
      "generate_defender_events": true,
      "entries_soft_limit": 100,
      "entries_hard_limit": 150
    }
]
```

Alternatively (recommended), you can use environment variables by creating the file `/etc/sftpgo/env.d/rate-limiting.env` with the following content.

```shell
SFTPGO_COMMON__RATE_LIMITERS__0__AVERAGE=100
SFTPGO_COMMON__RATE_LIMITERS__0__PERIOD=1000
SFTPGO_COMMON__RATE_LIMITERS__0__BURST=1
SFTPGO_COMMON__RATE_LIMITERS__0__TYPE=1
SFTPGO_COMMON__RATE_LIMITERS__0__PROTOCOLS=SSH,FTP,DAV,HTTP
SFTPGO_COMMON__RATE_LIMITERS__0__GENERATE_DEFENDER_EVENTS=0
SFTPGO_COMMON__RATE_LIMITERS__0__ENTRIES_SOFT_LIMIT=100
SFTPGO_COMMON__RATE_LIMITERS__0__ENTRIES_HARD_LIMIT=150
SFTPGO_COMMON__RATE_LIMITERS__1__AVERAGE=10
SFTPGO_COMMON__RATE_LIMITERS__1__PERIOD=1000
SFTPGO_COMMON__RATE_LIMITERS__1__BURST=1
SFTPGO_COMMON__RATE_LIMITERS__1__TYPE=2
SFTPGO_COMMON__RATE_LIMITERS__1__PROTOCOLS=SSH,FTP
SFTPGO_COMMON__RATE_LIMITERS__1__GENERATE_DEFENDER_EVENTS=1
SFTPGO_COMMON__RATE_LIMITERS__1__ENTRIES_SOFT_LIMIT=100
SFTPGO_COMMON__RATE_LIMITERS__1__ENTRIES_HARD_LIMIT=150
```

We have a global rate limiter that limit the aggregate rate for the all the services to 100 req/s and an additional rate limiter that limits `SSH` and `FTP` protocols to 10 req/s per host.
With this configuration, when a client connects via SSH and FTP will be limited first by the global rate limiter and then by the per host rate limiter.
Clients connecting via WebDAV or HTTP will be checked only against the global rate limiter.
