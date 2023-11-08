# The Sonorous protocol reference

## What is this?

This is an informal documentation for the Sonorous protocol. It describes what is being sent through the network and how. 
This documentation is not comprehensive and might not be complete, or it might include outdated information. Please look at the implementation code at the client and server repos for more depth.

## Network Channels

The Sonorous protocol uses 10 [ENet](http://enet.bespin.org/) channels, each configured for a specific type of network traffic. 

With the exception of Channel 2, which is exclusively dedicated to the encryption handshake process, all communications across the other channels must be secured using symmetric encryption. Each channel inherently presupposes that the data being transmitted has already been encrypted with the agreed-upon symmetric key derived from the initial encryption handshake.

When referring to the content and structure of data for each channel as described in this documentation, it is implied that the raw data being transmitted—whether it's audio frames on Channels 0 and 1, control messages, or any other types of data on additional channels—needs to be pre-processed through the agreed-upon encryption system before sending.

The content of all packets (except channel2) should be encrypted using the Advanced Encryption Standard (AES) in Galois/Counter Mode (GCM). Encrypted packet is prefixed with a nonce (a 12-byte value) required by AES-GCM. This nonce should be unique for each packet to ensure the security of the encryption.

Upon reception of a packet on any channel besides Channel 2, the nonce (found in the initial 12 bytes of the packet) is used along with the shared symmetric key to decrypt the contents. Post decryption, the protocol rules for the respective channel are followed to process the packet's content. Importantly, the nonce must not be incorporated in parsing the decrypted data of a packet.

### Channels 0 and 1: Bidirectional Audio Communication

We establish two separate channels for bidirectional audio communication, each with a unique configuration tailored to its direction of transmission.

#### Channel 0: Audio Out (Server to Clients)

- **Enet Configuration**: Unreliable, Unsequenced
- **Usage**: Channel 0 is designated for the server to send audio frames to clients.
- **Frame Packet Format**: Audio frames dispatched by the server on Channel 0 include a 4-byte header, with each component in little-endian format:

    1. **User ID** `(unsigned 16-bit integer)`: Denotes the frames owner. A frame referencing an unrecognized User ID should be ignored.
    2. **Sequence Number** `(unsigned 16-bit integer)`: Ensures independent sequencing of audio frames per user. Clients are expected to handle frames arriving with a sequence number lower than anticipated.

- **Payload**: Following the header, the packet encases the raw audio frame data.

Channel 0's lack of Enet sequencing and reliability is suitable for real-time audio out to clients, since maintaining the audio stream's continuity with minimal latency is paramount, even if some packets are lost or arrive out of order. We disable Enet's sequencing for this channel so that we can handle out of order packets our selves, which is needed to handle multiple users on a single channel as well as things like packet reordering if that's possible (but that's client-specific.)

#### Channel 1: Audio In (Client to Server)

- **Enet Configuration**: Unreliable, Sequenced
- **Usage**: Channel 1 is used by clients to send audio frames to the server.
  
When clients send audio frames on Channel 1, the packet content should be the audio frame itself without any additional headers or footers. This approach is possible because ENet's allocates a distinct Peer ID to each client, allowing the server to attribute incoming packets to their source accurately.

Channel 1 adopts Enet's sequencing to ensure that while frames may still be discarded due to network issues (as the channel remains unreliable), the server can get the packets in order. This is critical to maintain the coherence of the audio stream going from the client to the server.

### Channel 2: The encryption handshake.

- **Enet Configuration**: Reliable, Sequenced
- **Usage**: Channel 2 is used To initiate a session, by securely sharing a semetric key.

Note that while all the other channels require everything sent through them be encrypted, this channel does not, as it deals with establishing the encryption itself. Packets described here must be in the order they are described here

First of all, the server must have an RSA key pare of size 2048

1. **Public Key Transfer**
   The server sends its public key to the client in plain text, using the Privacy-Enhanced Mail (PEM) format.

2. **Public Key Verification**
   The client verifies the server's public key. This is client-specific. If the client decides that it does not trust the key, it should disconnect immediately.

3. **Session Key Generation and Encryption**
   Assuming the key is trusted, the client generates a new 256-bit AES key, unique for the session. The client then encrypts the AES key using the server's public key and sends it back to the server.

4. **Nonce Creation and Packet Encryption**
   After step 3, The client generates a random 32-byte packet. The packet is encrypted with the previously generated AES session key, and the 12-byte nonce is prepended to this encrypted data. The final packet is then sent to the server.

5. **Server Response and Verification**
   The server receives the encrypted AES key, decrypts it using its private key, and then uses the decrypted AES key and nonce to decrypt the 32-byte packet. The server sends back this decrypted packet in its unencrypted form to the client.

6. **Handshake Completion**
   If the packet from the server matches the one the client sent, the client verifies that the server has indeed decrypted the AES key securely, thus owning the corresponding private key. If it matches, the handshake is successful, and secure communication can proceed through other channels using the AES session key. If the response is incorrect or there is a timeout, the client should disconnect immediately.
   