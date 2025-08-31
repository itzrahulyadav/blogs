# Advance Networking concepts

1. Software define routing

Software-defined routing (SDR) is an approach to routing where the control of routing decisions (traditionally handled by distributed routing protocols running on routers) is centralized and programmable through software.

2. IP in IP

IP-in-IP (IPIP) is a tunneling technique where an entire IP packet is wrapped inside another IP packet.

That means:
- The original IP packet (with its own source and destination IPs) becomes the payload.

- A new outer IP header is added on top.

- This creates an encapsulated IP packet.


3. GRE (Generic routing encapsulation)

GRE (Generic Routing Encapsulation) is a tunneling protocol (RFC 2784) that allows you to encapsulate a wide variety of network layer protocols inside IP packets.

- Unlike IP-in-IP, which can only carry another IP packet inside, GRE is generic → it can carry IPv4, IPv6, multicast, broadcast, or even non-IP protocols (like IPX).

- GRE adds an extra GRE header between the outer IP header and the encapsulated packet.

4. 
