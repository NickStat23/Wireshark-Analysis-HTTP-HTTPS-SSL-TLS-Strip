# Wireshark Network Forensics: Plaintext Vulnerabilities & TLS Decryption

## Project Overview
This project uses Wireshark to dig into two real network security scenarios that come up constantly in blue team and forensics work. Phase 1 is pretty eye-opening; it shows how easy it is to pull credentials straight out of unencrypted HTTP traffic. Phase 2 goes deeper into TLS decryption, walking through how an analyst can use a leaked server private key to completely unwrap an "encrypted" session after the fact.

Both phases used publicly available packet captures from a DEF CON packet hunting workshop, so everything here is legal and educational.

* **Source Repository:** [packetrat/packethunting](https://github.com/packetrat/packethunting)
* **Files Used:** `HTTP-password.pcap`, `SSL-decryption.pcap`, `server.pem`

---

## Phase 1: Pulling Credentials Out of Plain HTTP Traffic

### Downloading the Capture
First step was grabbing the targeted pcap from the workshop repo and saving it locally.

![Navigating to Workshop Capture List](images/Choosing_PCAP_Packethunting.png)

![Downloading HTTP Password Capture](images/Downloading_HTTP_Password_PCAP.png)

### Opening the Capture in Wireshark
When I first opened `HTTP-password.pcap`, the packet list was full of TCP handshakes and background noise. Nothing unusual at first glance.

![Initial Import of Plaintext HTTP Capture](images/HTTP_Password_PCAP_Open.png)

Filtering by `http` cleaned things up fast. Packet 70 jumped out right away, showing an outbound HTTP POST request hitting an administrative login page.

![Isolating Packet 70 POST Request](images/Password_Packet.png)

### Following the Stream
To get a clean view of what was actually sent, I right clicked on Packet 70 and selected **Follow > HTTP Stream**. This brings up the whole conversation in a readable window rather than looking at raw individual packet data.

![Executing Follow HTTP Stream](images/HTTP_Follow.png)

### Extracting the Credentials
Because the site used plain HTTP without transport layer encryption, everything inside that stream window was readable.

* **Client IP:** 192.168.56.1
* **Server IP:** 192.168.56.101
* **Target Path:** `/netgear/login/base/cheetah_login.html`
* **Exfiltrated Password:** `pwd=GoodLuckTryingToCrackThisPassword1928364132874234916592364861329`

![Extracted Cleartext Administrative Password](images/HTTP_PasswordPacket_FollowHTTP.png)

> **Takeaway:** This really drives home the point that password complexity does not save you if your transport layer is insecure. The user set a massive 63 character password, but it took two clicks to grab it in cleartext because the connection was unencrypted.

---

## Phase 2: Post Incident Forensic TLS Decryption

### The Encryption Wall
For the next phase, I grabbed `SSL-decryption.pcap` from the workshop files to look at encrypted traffic.

![Downloading Secure TLS Capture File](images/SSL-TLS_STRIP_PCAP_Download.png)

When I opened this file, the application layer data was completely unreadable. Because a secure TLS 1.2 session was established, Wireshark just showed generic application data records. If you try to follow this stream before adding a key, you get nothing but blank or encrypted blocks.

![Encrypted TLS Records Prior to Key Ingestion](images/SSL_STRIP_Before_Applying_Key.png)

### Adding the Server Private Key
To inspect the underlying traffic, I needed to grab the pre-shared server private key file (`server.pem`) hosted in the same repo.

![Reviewing Server Private Key Block](images/SSL-Server_PEM_Key.png)

I went into Wireshark preferences by navigating to **Edit > Preferences > Protocols > TLS**. In the **RSA keys list** menu, I added a new entry, mapped it to port 443, and browsed to where I saved the local copy of `server.pem`.

![Configuring RSA Keys List Parameters](images/SSL_STRIP_KEY_ADD.png)

Once I hit apply, Wireshark updated its rules to use this key block to decrypt matching sessions on the fly.

![Verifying Successful Ingestion of server.pem](images/SSL_STRIP_KEY_ADDED.png)

### Reading the Decrypted Streams
The second the key was applied, Wireshark re-computed the capture file and successfully unwrapped the TLS layer. A whole row of plaintext HTTP packets appeared where the generic TLS records used to be.

![Post Decryption Packet List Exposure](images/SSL_Strip_Open_PCAP.png)

Filtering for `http` brought up Packet 29. Right clicking that packet and choosing **Follow > HTTP Stream** opened up the plaintext conversation that was previously hidden.

![Launching Decrypted Stream Viewer](images/SSL_STRIP_Success.png)

### Analyzing the OpenSSL Diagnostics
Looking at the decrypted stream showed that this wasn't a standard web server conversation. It was actually a diagnostic dump from an OpenSSL testing daemon (`s_server`). The output exposed deep session details:

* **Target Daemon Command:** `s_server -www -cipher AES256-SHA -key server.pem -cert server.crt -accept 443`
* **Negotiated Protocol State:** TLS v1.2
* **Cipher Suite Enforced:** AES256-SHA
* **Derived Session Master Key:** `A0EFF65B633854E386FADA958A8B14B5353E06C3EC6BD55345B363DC
