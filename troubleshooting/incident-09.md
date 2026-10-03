# Incident 09 — Web service stopped (port/service unreachable)

## Category
Application layer

## Scenario

Deliberately staged incident: stopped the Nginx service on `SRV-Ubuntu1`.
```
sudo systemctl stop nginx
```

## Symptom

From `PC-user1`:
```
ping 10.10.20.10
```
→ 0% loss, normal success (network/IP layer intact, the server still replies to ICMP).

```
curl http://10.10.20.10
```
→ immediate failure:
```
curl: (7) Failed to connect to 10.10.20.10 port 80 after 2062 ms: Could not connect to server
```

## Analysis

The difference between the two results is the key point of this incident: a successful ping proves the machine is powered on, routed correctly, and that the firewall allows at least ICMP traffic — but it says **nothing** about the state of a specific application service. The network layer (3) and the application layer (7) need to be tested separately.

## Diagnosis (server side)

```
sudo systemctl status nginx
```
→ status `inactive (dead)`, confirms the service is stopped.

## Fix

```
sudo systemctl start nginx
```

## Verification

```
curl http://10.10.20.10
```
→ full "Welcome to nginx!" HTML page correctly returned.

## Lesson learned

Always separate layers during network diagnosis: connectivity (ping, layer 3) ≠ application service availability (layer 7). A successful ping never guarantees a specific service is actually running on the target machine; the relevant port/protocol must be tested directly (`curl`, `telnet host port`, or equivalent) to confirm the service's actual state.
