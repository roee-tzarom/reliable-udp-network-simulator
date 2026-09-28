# Network Initialization and Reliable UDP Simulator

A local Python networking project that walks through DHCP-like address assignment, DNS-like name resolution and file retrieval. It implements both a length-framed TCP transfer and a custom reliable-UDP path, using only the standard library for the application code.

## End-to-end flow

```text
Client → UDP address offer (127.0.0.1:6767)
       → UDP name lookup (127.0.0.1:5353)
       → TCP server (127.0.0.3:2121) or RUDP server (127.0.0.3:2122)
       → local HTTP file source (127.0.0.1:8080)
```

These are **local educational DHCP/DNS-like services**, not implementations intended to replace system DHCP or DNS. The TCP mode prefixes payloads with a 10-character ASCII length field so the receiver knows exactly how many stream bytes to read. The UDP mode defines an 11-byte `!IIcH` binary header with sequence, acknowledgement, flag and payload length. It uses a SYN-style handshake, ordered receive with cumulative ACKs, Go-Back-N-style retransmission and an additive-increase/multiplicative-decrease sending window capped at five chunks. The client can simulate packet loss and latency.

## Run locally

Use Python 3 and start each service in a separate terminal from the repository root:

```bash
python -m http.server 8080
python dhcp_server.py
python dns_server.py
```

Then choose one transfer mode:

```bash
# TCP: separate server and client terminals
python app_server.py
python client.py

# Or RUDP: separate server and client terminals
python app_server_rudp.py
python client_rudp.py
```

Both clients request `test_file.txt` from the local HTTP server and write a downloaded file. The transfer endpoints and ports are fixed in the scripts. Set `SIMULATE_PACKET_LOSS` or `SIMULATE_LATENCY` near the top of `client_rudp.py` to demonstrate recovery behavior.

## Source and evidence

| File | Responsibility |
| --- | --- |
| `dhcp_server.py`, `dns_server.py` | Local discovery and resolution services |
| `client.py`, `app_server.py` | Length-framed TCP path |
| `client_rudp.py`, `app_server_rudp.py` | Reliable-UDP path |
| `captures/` | Wireshark screenshots and packet captures for clean, loss and latency flows |

The capture files in [`captures/`](captures/) make the protocol behavior inspectable. This is a simulation on loopback interfaces; it does not claim production network security, congestion fairness or real-world interoperability.


## Transport design

The project compares two ways to deliver the same locally sourced file. The TCP path relies on TCP's ordering and reliability, adding only an ASCII length prefix so the application knows where the payload ends. The UDP path must provide its own bookkeeping. Each datagram carries a sequence number, acknowledgement number, one-byte flag and payload length. The client and server establish a simple session, move through ordered chunks, acknowledge contiguous progress and retry after a timeout. The sending window grows while acknowledgements advance and shrinks after a timeout, up to a small configured cap.

The address-offer and name-resolution stages give the client a discovery sequence before transfer. Both are educational services bound to loopback addresses; they do not configure the host operating system. A separate local HTTP server is the origin for `test_file.txt`, which lets the transfer servers fetch and relay known bytes.

## Observe a run

Start the HTTP, address and name services before the chosen application server, then run its client. Compare the downloaded output with the source file. For the UDP path, change the packet-loss or latency toggles in `client_rudp.py` and watch retransmission behavior. The `captures/` directory contains PCAPNG traces and Wireshark screenshots for clean, loss and delayed scenarios, so the protocol can be inspected beyond console output.

## What this proves and what it does not

The code demonstrates framing, cumulative acknowledgements, send-window state and local recovery from dropped packets. It uses fixed loopback addresses and a simple session format. It does not provide authentication, checksums beyond UDP's normal handling, congestion fairness, production DHCP/DNS compatibility or internet deployment configuration.
