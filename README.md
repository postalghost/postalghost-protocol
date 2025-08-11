## PostalGhost: A Secret Inheritance Protocol

A robust protocol for handling personal secret inheritance, minding both most desired properties simultaneously:

- **Hard timelock**: as long as the owner (sender) is active, no one — including the intended heir (receiver) — can access the secret.
- **Reliable recoverability**: once the set timelock period elapses without sender activity, the receiver can recover the secret — and no one else can.

This is achieved by using a multi-signature pattern over an arbitrarily large set of timelocked server-held symmetric keys, encrypting the secret such that multiple pre-declared subsets of them can independently unlock it.


## Design goals

- **Minimized trust**: servers are passive and stateless beyond simple key-state storage and timestamp operations.
- **Interoperability**: standardized crypto suite, message structures, and transport; agnostic to server/client implementation internals.
- **Small API surface**: servers expose just three endpoints for basic operations — `set`, `ping`, `get`.
- **High privacy**: each party learns minimal information; random identifiers are heavily used to prevent linkage.


## Actors and roles

- **Sender**: 
   - owner who creates the secret
   - keeps it locked by sending regular pings
- **Receiver**: 
   - intended heir
   - can recover the secret after timelock expiry using any pre-declared subset of keys
- **Servers**: 
   - multiple independent passive servers each holding a single 256-bit AES-GCM key per secret, plus minimal metadata
   - in charge of generating key, extending unlock timestamp upon `ping`, and revealing key to receiver after unlocked


## Lifecycle overview

1) **Set (creation)**
   - Sender selects one or more servers and the policy for the subsets of decrypting keys.
   - Sender requests each server to generate a fresh AES-256-GCM key and two random identifiers for it: one for the sender and one for the receiver.
   - Sender constructs:
     - a minimal private local record sufficient to ping the keys
     - a shareable package containing what the receiver needs to probe the state of each key and recover its key material after unlocked

2) **Share (sender → receiver)**
   - Sender transmits the sharable package out-of-band to the receiver.
   - Only the format is standardized for interoperability.
   - Transport-agnostic (e.g. email, physical drive, messenger app, QR code, etc.).

3) **Ping (sender-side lock-extension)**
   - Sender requests a server to extend a key's unlock timestamp (`unlocksAt`) by the configured `timelock` period.
   - Signals that the owner is still active.

4) **Get (receiver-side probing and recovery)**
   - The receiver queries a server to check the key state.
   - After unlock, the key material is obtained. 
   - When enough keys are unlocked, the receiver uses any available subset to decrypt the secret.


## API surface (high level)

All client-server interactions occur over WebSocket (WS) connections initiated by the client (either sender or receiver), and the server performs the following operations:

- `set` (sender): generate a key and return it, along with its identifiers (`senderId`, `receiverId`).

- `ping` (sender): extend `unlocksAt` by the set `timelock` period.

- `get` (receiver): return key status (`locked` or `unlocked`) and, if `unlocked`, also the key material.

Detailed endpoint specifications and WS message sequence are provided below.


## Transport and framing

- **Transport**: WebSocket over TLS (HTTPS upgrade).

- **Messaging**: JSON objects with key values encoded as hex strings.

- **Auth**: Client can verify server identity through a challenge-response scheme using public key cryptography. Server can trust client has access to key since it knows the associated id (`senderId` or `receiverId`).


## Cryptographic sketch

- **Content encryption**: The cleartext secret is locally encrypted multiple times separately, one for each of the different subsets of decrypting keys. For each subset, the encryption is done using an AES-256-GCM key constructed as the byte-wise modular sum of all member keys, ensuring order-independent results. The resulting ciphertexts are only shared with the receiver out-of-band, never with the servers.

- **Server keys**: Servers generate and store unique AES-256-GCM keys and their metadata: `senderId`, `receiverId`, `timelock` and `unlocksAt`.

- **Unlock condition**: The value of a key is revealed upon a `get` request from the receiver if and only if the currently stored unlock timestamp (`unlocksAt`) is already in the past. Once `unlocked`, it cannot be locked again by a `ping` from the sender.


## Data model (server-side minimal state)

For every issued key, the server stores:

- `senderId` (opaque, random) — used by the sender to `ping`.
- `receiverId` (opaque, random) — used by the receiver to `get`.
- `key` — key material
- `timelock` (sender-specified at creation) - duration of the timelock period, i.e. how much is `unlocksAt` increased upon each `ping`.
- `unlocksAt` - moved forward by the timelock period upon each `ping` from the sender, as long as it is still `locked`.

## Timelock semantics

- On `set` (key creation), server initializes `unlocksAt = now + timelock`.

- On `ping`, server updates `unlocksAt = now + timelock`, as long as status is `locked`, and returns status (`locked` or `unlocked`); `status` is never changed.

- On `get`, server returns current `status` (`locked` or `unlocked`) and, if `status = unlocked`, also includes the key material (`key`).



## Security and privacy considerations

- **Passive servers**: servers do not push events; clients control timing and coordination, reducing attack surface.

- **Minimal disclosure**: before unlock, servers never reveal key material; only status.

- **Identifier unlinkability**: sender and receiver handles (`senderId` and `receiverId`) are distinct random identifiers per key, preventing linkage.

- **Clock trust**: unlock policy depends on server clocks.

- **Confidentiality**: ciphertext is never sent to servers; initially generated by sender and afterwards only held by receiver.

- **Integrity**: AES-GCM encryption ensures data protection and authenticity; WS/TLS with signed challenge provides secure communication and trustless server identity verification. 


## Compliance and interoperability

An implementation is compliant if it:

- Implements WS message frames as specified.
- Implements the `set`, `ping`, `get` endpoints with the specified semantics.
- Adheres to the cipher suite and data encoding rules.
- Produces/consumes the sharable receiver secret format.


## Endpoint specifications

The following sections will define endpoint data schemas.


### set

Request (sender → server):

```json
{
    "timelock": int (unix-timestamp [s])
}
```

Response (server → sender):

```json
{
    "sender": string (sender id),
    "receiver": string (receiver id),
    "key": string (key material)
}
```

### ping

Request (sender → server):

```json
{
    "id": string (sender id)
}
```

Response (server → sender):

```json
{
    "status": "locked" | "unlocked" (key status)
}
```


### get

Request (receiver → server):

```json
{
    "id": string (receiver id)
}
```

Response (server → receiver):

```json
{
    "status": "locked" | "unlocked" (key status),
    "key": string | undefined (key material)
}
```


## WebSocket message sequence

### Identity verification handshake (challenge → signature)

- Connection: client opens a WebSocket connection to the server's root path `/` (e.g., `wss://server.example/`).
- Challenge: immediately after connect, the client sends a fresh random string (challenge).
- Proof: the server signs the challenge with its private key and returns the signature.
- Verification: the client verifies the signature against the server's public key (internally stored as part of the key metadata).
- Decision: if verification fails, the client closes the WebSocket, otherwise it proceeds with the intended operation.


### Operation invocation (set | ping | get)

- After a valid handshake, the client requests an operation by naming the endpoint.
- Then it sends the payload object, as defined in the data schema specifications for the given endpoint (`set`, `ping`, `get`).
- Finally, the server executes the requested operation and replies with the corresponding response object.


Operational rules:

- The server does not push unsolicited messages; both response messages (proof and result) are triggered by the client's request messages (challenge and payload respectively).
- Each WS connection shall only be used for a single operation.

## Share format

Defines the exact structure of the data shared out-of-band from sender to receiver in order to instruct the receiver on how to recover the secret.

Example JSON:

```json
{
  "keys": [
    { "host": "server.example:80", "pk": "server-public-key", "id": "random-uuid" },
    ...
  ],
  "paths": [
    { "keys": [0, 2, 3], "data": "encrypted-secret" },
    ...
  ]
}
```

- `keys`: keys used by the secret
  - `host`: server host (`hostname`:`port`)
  - `pk`: server public key
  - `id`: key receiver identifier
- `paths`: subsets of decrypting keys
  - `keys`: keys referenced by their zero-based index in the `keys` list
  - `data`: ciphertext encrypted with this subset of keys


## Operational guidance

- **Ping cadence**: intervals between pings should be kept well below the timelock period to withstand failures, e.g. connection outages.
- **Redundancy**: multiple servers should be used across operators/jurisdictions to reduce correlated failure risk.


## Resilience

  - No single server compromise or reveals content; multiple keys are always required.
  - WS/TLS plus identity verification thwarts MitM before sensitive data is exposed.


## Glossary

- **Timelock period**: Fixed interval, always counting from current time, that `unlocksAt` is moved forward upon each `ping`, i.e. max time the sender can spend without executing a `ping` to a key, for it to remain `locked`.
- **Subset policy**: Declarative set of allowed key combinations that can recover the encrypted secret.
- **Handle**: Random identifier for a key; distinct for sender and receiver.


## Out of scope

- How sender and receiver exchange the share package (e.g. email, physical drive, messenger app, QR code, etc.).
- How clients store their local state (e.g. database, file, browser storage, etc.).
- Server and client implementation details (programming languages, deployment platforms, etc.).

