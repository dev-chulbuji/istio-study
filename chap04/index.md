# 04 The edge: Getting traffic into your cluster

## 4.1 Traffic ingress concepts

### 4.1.1 Virtual IPs: simplifying service access
![chat04_01](images/04_01.png)
- client -> (query) -> `api.istioinaction.io/v1/products`
- client network stack
    - resolve domain name to ip address (DNS server)
- server side
    - map domain name to virtual IP
    - virtual Ip -> (single instance x) -> reverse proxy

### 4.1.2 Virtual Hosting: multiple services from a single access point
![chat04_01](images/04_01.png)
- prd.istioinaction.io & api.istioinaction.io -> same virtual IP
- virtual hosting
    - hosting multiple different services at single entry point
- particular request -> virtual-host group
    - HTTP/1.1: `Host` header
    - HTTP/2: `:authority` header
    - TCP: `SNI(Server Name Indicator`
    

---

## 4.2 Istio Gateway
 