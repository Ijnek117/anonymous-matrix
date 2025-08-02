# TBD

### Summer Research Internship Project: Achieving User Pseudonymity in Federated End-to-End Encrypted Messaging Platforms.

This repository serves as the central documentation for the research project. It brings together 2 open-source components within the Matrix Network to demonstrate whether it is possible to achieve user anonymity from remote homeservers in a federated end-to-end encrypted messaging platform.

---

## 1. Introduction & Research Goal

The goal of this project is to investigate and develop a federated end-to-end encrypted messaging platform that supports user anonymity from remote homeservers. This was achieved by modifying components of the Matrix Network where federation already exists but where anonymity is not yet available. 

For more information please see the [Report](REPORT.md).

## 2. Component Overview

The project consists of the following components. 

| Component Name| Original GitHub Repo | Description of Modifications| Link to Modified Repo |
| --------------------- | ------------------------------ | --------------------------------------------------------- | ------------------------------- |
| `Dendrite Homeserver` | `element-hq/dendrite` | Modified API endpoints to support client-side matrix userID encryption in invite events, and `mxid_mappings` to not include userIDs. | `github.com/Ijnek117/dendrite/tree/anonymity-research` |
| `gomatrixserverlib`| `matrix-org/gomatrixserverlib` | Changed federation events to not reveal invitee's userIDs in `mxid_mappings`. | `github.com/Ijnek117/gomatrixserverlib/tree/anonymity-research` |
| `Gomuks Client` | `gomuks/gomuks`| Added support for rooms of type MSC4014 and implemented client-side encryption of matrix user IDs for invite events. | `github.com/Ijnek117/gomuks/tree/anonymity-research` |
| `mautrixgo`| `mautrix/go` | Added support for encrypted matrix user IDs | `github.com/Ijnek117/mautrixgo/tree/anonymity-research` |

## 3. Getting Started (Running the Full Project)

Follow these steps to set up and run the entire research project. This will set up two matrix homeservers that will be able to federate with each other and a matrix client. Please note that currently the servers will not be able to federate with other servers in the Matrix Network.

### 3.1 Prerequisites

General
- git
- sqlite
- Go v1.24+ (if you plan to modify the code)

Gomuks specific
- latest LTS of Node.js
- libolm or libolm-dev

### 3.2 Installation

Clone all the modified repositories into a single directory, and checkout the anonymity-research branch in all of them.
```
git clone [https://github.com/Ijnek117/dendrite.git](https://github.com/Ijnek117/dendrite.git)
git clone [https://github.com/Ijnek117/gomuks.git](https://github.com/Ijnek117/gomuks.git)
git clone [https://github.com/Ijnek117/gomatrixserverlib.git](https://github.com/Ijnek117/gomatrixserverlib.git)
git clone [https://github.com/Ijnek117/mautrix.git](https://github.com/Ijnek117/mautrix.git).git
```
### 3.3 Configuration 
#### 3.3.1 Configure Dendrite 

> [!NOTE]
> All paths must be different for each homeserver, or they must all be run in different data directories.
>

Open the `go.mod` file and make `gomatrixserverlib` point to your local instance:

```go
replace github.com/matrix-org/gomatrixserverlib => ../gomatrixserverlib
```

Do the following for each homeserver you want to set up. 

- Generate a Matrix signing key for federation 

    ```./bin/generate-keys --private-key matrix_key.pem```
- Generate a self-signed certificate 

    ```./bin/generate-keys --tls-cert server.crt --tls-key server.key```
- Copy and modify the config file. You'll need to set a server name and paths to the server keys, database connection strings, JetStream topic name, and storage path.
    ```cp research.yaml dendrite.yaml```
    - For the easiest setup, the `server_name` should be an IP address and a port number for handling TLS connections (e.g., `127.0.0.1:8448` for localhost). See the official documentation for instructions on using a domain name.

#### 3.3.2 Configure Gomuks
- Open the go.mod file and make mautrix point to your local instance
```
replace maunium.net/go/mautrix => /pathto/mautrixgo
``` 

### 3.4 Building the Binaries
Navigate into each directory and build the binary 
#### 3.4.1 Building Dendrite 
```
cd dendrite
go build -o bin/ ./cmd/...
go install ./cmd/dendrite
```
#### 3.4.2 Building Gomuks
```
cd ../gomuks
./build.sh
```

### 3.5 Running the Project
#### 3.5.1 Start Dendrite
For each server, run the binary with the parameters filled in the background .
```
./bin/dendrite --tls-cert <path/to/server.crt> --tls-key <path/to/server.key> --config <path/to/dendrite.yaml> -really-enable-open-registration -http-bind-address 127.0.0.1:<port> -https-bind-address <servername> &
```

e.g. For a homeserver with servername: `127.0.0.1:8448`
```
./bin/dendrite --tls-cert server.crt --tls-key server.key --config dendrite.yaml -really-enable-open-registration -http-bind-address 127.0.0.1:8009 -https-bind-address 127.0.0.1:8448 &
```
Now create a user and password on each server

```
./bin/create-account -config <dendrite.yaml> -url http://<http-bind-address> -username <username>
```

#### 3.5.2 Start Gomuks
```
./gomuks
```

### 3.6 Expected Output

You should see Dendrite homeserver logs, the NATS server starting up, and Gomuks server logs.

### 3.7 Common issues 

Gomuks [see here for a list of common issues](https://docs.mau.fi/gomuks/installation.html#common-compilation-issues)


## 4. Creating a room 

These are instructions on how to create a new room of type `org.matrix.msc4014`, and invite a user from another homeserver to join it.

### 4.1 Loging into Gomuks 

To log into the accounts you created with Gomuks you need a recovery key for each account. The easiest way I have found is to use a more established Matrix client such as [Element Web]() or [Element](). 

1. Sign in using 'Other homeserver' with `http://127.0.0.1:<http-port>`

2. Fill in the username and password you created 

3. You will be prompted to set up recovery or do so my navigating to Settings -> Encryption -> Set up Recovery. Follow the instructions

4. Make sure to save the recovery key 

Open gomuks web by going to the address listed in the gomuks terminal it should say:

```
Server started address=localhost:29325
```

Log in using the recovery key from Element..

### 4.2 Setting up a room of type `org.matrix.msc4014`
To create the room

1. Click the + symbol in the top left

2. Name your room and untick encrypted 

3. Select "advanced options" and set the `room version` to be `org.matrix.msc4014`

4. Click Create

5. Right click on the room name in the room menu and select "invite a user"

6. Fill in the username for the user on your other homeserver such as @\<username\>:\<servername\>

### 4.3 Accepting the invitation 

Login to the user account you just invited on Element, and accept the invite. 

Open the new room, you should now see the following state events on Element

```
@a:127.0.0.1:8449 created this room. This is the start of TestRoom.
Invite to this room
Expand
@a:127.0.0.1:8449 created and configured the room.
@a:127.0.0.1:8449 invited @alice:127.0.0.1:8448
```

You can now send messages between the two users.

### 4.4 Access the matrix state and message events

To examine into what matrix state and message events are on the dendrite 
homeservers, you can open the room_server database with sqlite3 using the 
connection path set up in the config.

e.g.
```
sqlite3 room_server.db
```

Similarly for the sync_api and other databases.