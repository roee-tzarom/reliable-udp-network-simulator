# Network Initialization and Reliable UDP Simulator

A local Python networking system that ties together address discovery, name lookup and file retrieval. It offers two delivery paths for the same source file: a length-framed TCP connection and a custom reliable-UDP protocol. Packet captures let you inspect the behavior in Wireshark as well as in the code.

## End-to-end design

```text
client
  ├─ address offer → local UDP service, port 6767
  ├─ name lookup   → local UDP service, port 5353
  └─ file request  → TCP server, port 2121
                  or reliable-UDP server, port 2122
                         ↓
                 local HTTP source, port 8080
```

The address and name services imitate the shape of DHCP and DNS discovery for a loopback-only demonstration. They do not reconfigure the operating system or implement interoperable DHCP/DNS servers.

### TCP path

`client.py` and `app_server.py` exchange framed messages. A 10-character ASCII length prefix tells the receiver how many bytes belong to the following payload, solving the message-boundary problem that appears when reading from a TCP stream.

### Reliable-UDP path

`client_rudp.py` and `app_server_rudp.py` use an 11-byte network-order header (`!IIcH`): sequence number, acknowledgement number, flag and payload length. The peers establish a simple session, deliver chunks in order, send cumulative acknowledgements and retransmit outstanding data after a timeout. A small sending window grows when progress is acknowledged and shrinks after a timeout. Client-side switches can introduce loss and latency for observation.

## Run the local stack

Use Python 3. From the repository root, start these services in separate terminals:

```bash
python -m http.server 8080
python dhcp_server.py
python dns_server.py
```

Then start one transfer server and its matching client:

```bash
# TCP
python app_server.py
python client.py

# Or reliable UDP
python app_server_rudp.py
python client_rudp.py
```

Both clients request `test_file.txt` through the local HTTP source and write a downloaded result. The scripts use fixed loopback addresses, including `127.0.0.3` for the application server. Check the address constants if your environment handles loopback aliases differently.

## Explore the repository

| Files | What to inspect |
| --- | --- |
| `dhcp_server.py`, `dns_server.py` | Address offer and name lookup |
| `client.py`, `app_server.py` | TCP framing and file transfer |
| `client_rudp.py`, `app_server_rudp.py` | Datagram header, ACKs, window and retransmission |
| `captures/` | Packet captures of clean, loss and delayed flows |

A useful review path is to run the TCP version, compare its downloaded bytes with `test_file.txt`, then run the UDP version and inspect a capture. The UDP client has loss and latency simulation switches near the top of the file; set them as desired for a clean or impaired run.

The custom transport is a focused local simulator. It does not provide authentication, production congestion control or compatibility with standard reliable-transport protocols.
