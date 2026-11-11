# Raspberry Pi Network IDS & Secure API Architecture

## 1. Overview

This project uses a Raspberry Pi as a lightweight network intrusion detection system (IDS) and API server. It collects network traffic, identifies suspicious activity, records security events, and allows a separate computer to retrieve those events securely.

The system uses several components that work together:

- **Python IDS:** Captures and analyzes network packets.
- **Flask:** Provides the API endpoints and application logic.
- **Waitress:** Runs the Flask application as a production WSGI server.
- **SQLite:** Stores detected network events in a local database.
- **Nginx:** Acts as a reverse proxy and handles incoming HTTPS connections.
- **OpenSSL:** Generates the TLS certificate and private key used by Nginx.
- **Bearer Token Authentication:** Restricts access to authorized API clients.

## 2. Architecture

### Network layout

| Device | Role | Example address |
|---|---|---|
| Raspberry Pi 3B | IDS and API server | `192.168.1.140` |
| HP Laptop | Remote API client / management | `192.168.1.137` |
| Raspberry Pi loopback | Local API connection | `127.0.0.1:8080` |
| Nginx HTTPS listener | Remote API access | Port `443` |

*These IP addresses are example values and should be verified against the current network configuration.*

### Request flow

```text
                  HOME NETWORK
                       |
              +------------------+
              |    HP Laptop     |
              |  API Client      |
              +------------------+
                       |
                  HTTPS :443
                       |
              +------------------+
              |      NGINX       |
              | Reverse Proxy    |
              | TLS Termination  |
              +------------------+
                       |
                 HTTP :8080
                 (Loopback)
                       |
              +------------------+
              |     WAITRESS     |
              |  WSGI Server     |
              +------------------+
                       |
              +------------------+
              |    FLASK API     |
              | Authentication  |
              | Request Handling |
              +------------------+
                       |
              +------------------+
              |      SQLITE      |
              |  ids_events.db   |
              +------------------+

    Separate local traffic path:

    +------------------+
    |    Python IDS    |
    | Packet Capture   |
    | Threat Detection |
    +------------------+
             |
       HTTP :8080
       127.0.0.1
             |
       Waitress / Flask
             |
          SQLite
```

## 3. What Each Component Does

### Python IDS

The IDS monitors network packets through a selected network interface using PyShark and TShark.

Its responsibilities include:
- Capturing network traffic.
- Extracting source and destination IP addresses.
- Identifying protocols and ports.
- Detecting suspicious behavior, such as potential TCP SYN port scans.
- Creating structured security events.
- Sending detected events to the Flask API.

The IDS is the detection component. It does not host the API.

### Flask API

Flask handles the application's API functionality.

Its responsibilities include:
- Receiving JSON events from the IDS.
- Authenticating requests using a bearer token.
- Validating incoming data.
- Storing events in SQLite.
- Providing endpoints for retrieving stored events.

Example endpoints:

| Endpoint | Method | Purpose |
|---|---|---|
| `/health` | GET | Check API status |
| `/api/` | POST | Submit a security event |
| `/api/events` | GET | Retrieve recorded events |

### Waitress

Waitress is the production WSGI server that runs the Flask application.

Instead of relying on Flask's built-in development server, Waitress handles incoming application requests.

In this setup, it listens on:

```text
127.0.0.1:8080
```

Binding to loopback prevents remote devices from connecting directly to the Flask application through that port.

### SQLite

SQLite stores the IDS events in a local database file:

```text
ids_events.db
```

The database is created automatically when the API initializes its database schema.

It stores information such as:
- Event timestamp.
- Network interface.
- Protocol.
- Source and destination IP addresses.
- Source and destination ports.
- Detected alert information.

### Nginx

Nginx acts as a reverse proxy in front of Waitress.

Its responsibilities include:
- Listening for incoming HTTPS connections on port `443`.
- Handling TLS encryption.
- Forwarding authorized API requests to Waitress.
- Keeping the application server inaccessible directly from the LAN.

Nginx forwards requests to:

```text
http://127.0.0.1:8080
```

### OpenSSL

OpenSSL generates the TLS certificate and private key used by Nginx.

For a home lab, a self-signed certificate can be used.

The certificate allows the laptop to establish an encrypted HTTPS connection to the Raspberry Pi, provided the laptop trusts the certificate.

### Bearer Token Authentication

Bearer-token authentication protects the API endpoints.

The client includes an authorization header:

```http
Authorization: Bearer YOUR_SECRET_TOKEN
```

Flask checks the token before allowing access to the API.

HTTPS protects the token while it travels across the network. The token itself must also be stored securely on both devices.

## 4. How Everything Works Together

### Local IDS event submission

1. The Python IDS captures network traffic.
2. It analyzes packets and checks for suspicious behavior.
3. When it detects an alert, it creates a JSON event.
4. The IDS sends the event to `127.0.0.1:8080/api/`.
5. Waitress receives the request and passes it to Flask.
6. Flask validates the bearer token and event data.
7. Flask inserts the event into the SQLite database.

### Remote API access

1. The HP laptop sends an HTTPS request to the Raspberry Pi.
2. Nginx receives the request on port `443`.
3. Nginx handles the TLS connection using its OpenSSL-generated certificate.
4. Nginx forwards the request to Waitress on `127.0.0.1:8080`.
5. Flask verifies the bearer token.
6. Flask processes the request and retrieves the requested data from SQLite.
7. The API returns the response through Waitress and Nginx to the laptop.

## 5. Security Considerations

- Bind Waitress to `127.0.0.1`, not `0.0.0.0`.
- Use HTTPS for remote API connections.
- Require bearer-token authentication.
- Store API tokens in protected environment files rather than hardcoding them in Python.
- Restrict incoming firewall connections to trusted devices.
- Keep SSH access restricted to authorized users.
- Do not expose the API to the public internet through router port forwarding.
- Keep the Nginx private key protected.
- Remember that a self-signed certificate must be explicitly trusted by the client.
- Keep the Raspberry Pi's limited RAM and processing capacity in mind.

## 6. Final Component Breakdown

| Component | Primary responsibility |
|---|---|
| Python IDS | Packet capture and threat detection |
| PyShark / TShark | Network packet analysis |
| Flask | API application and request handling |
| Waitress | Runs the Flask application |
| SQLite | Stores security events |
| Nginx | Reverse proxy and HTTPS termination |
| OpenSSL | TLS certificate generation |
| Bearer Token | API authentication |
| UFW | Firewall and network access restrictions |

## 7. Overall Goal

The goal is to build a small, centralized network monitoring system using a Raspberry Pi.

The IDS detects suspicious network activity, Flask receives and manages security events, SQLite stores the data, and Nginx provides secure remote access from the HP laptop.

This architecture separates packet detection, application logic, data storage, and secure network access into independent components.
