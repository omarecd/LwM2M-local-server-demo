# LwM2M Local Server Demo — Step-by-Step

This guide shows how to run a local Eclipse Leshan LwM2M server, connect a device over CoAP/DTLS (PSK), and stream live readings into JSON.

## Prerequisites
- A Linux VM with Docker & Docker Compose (Ubuntu 22.04/24.04 works great)
- Public IP (or port-forwarding) for UDP 5683/5684 and TCP 8080
- One LwM2M device (IMEI + PSK), or the Leshan demo client for testing


## Step 1 — Provision the VM
Use any provider (Hetzner, Azure, AWS…). A 2 vCPU / 4 GB VM is plenty.
Then, make sure that you have Docker ready.


## Step 2 — Get Leshan & Node-RED running (Compose)
Create /root/docker-compose.yml:


```
services:
  leshan:
    image: eclipse-temurin:17-jre
    container_name: leshan
    ports:
      - "8080:8080"                     # Leshan UI & REST
      - "5683-5684:5683-5684/udp"       # CoAP / CoAPs
    volumes:
      - /root/leshan/leshan-server-demo.jar:/app/leshan-server-demo.jar
      - leshan_data:/data               # persist server state (security, etc.)
    command: ["java","-jar","/app/leshan-server-demo.jar"]
    restart: unless-stopped

  nodered:
    image: nodered/node-red:latest
    container_name: nodered
    ports:
      - "1880:1880"
    volumes:
      - nodered_data:/data
    restart: unless-stopped

volumes:
  leshan_data:
  nodered_data:
```


Download the Leshan demo JAR and start the docker compose:

```
mkdir -p /root/leshan
curl -L -o /root/leshan/leshan-server-demo-2.0.0-M15-jar-with-dependencies.jar \
https://repo1.maven.org/maven2/org/eclipse/leshan/leshan-server-demo/2.0.0-M15/leshan-server-demo-2.0.0-M15-jar-with-dependencies.jar

# Rename for simplicity
mv /root/leshan/leshan-server-demo-2.0.0-M15-jar-with-dependencies.jar /root/leshan/leshan-server-demo.jar
```


```
docker compose up -d
docker ps
```


Open the UI: http://<YOUR_IP>:8080 check that the server is running.


## Step 3 — Verify ports & basic connectivity


From your laptop do as follows. If you get “succeeded!” it means reachable
```
nc -zvu <YOUR_IP> 5683
nc -zvu <YOUR_IP> 5684
```


## Step 4 — Configure Device Security (PSK)
- Go to Leshan UI → SECURITY.
- Add a record:
 - Endpoint: urn:imei:<YOUR_IMEI>
 - Security mode: psk
 - Identity: urn:imei:<YOUR_IMEI>
 - Key: 00000000000000000000000000000000 (or your real key)
- Save.

The /data volume ensures this survives container restarts.

## Step 5 — Configure your Device

On the device, set:
- Server URI: coaps://<YOUR_IP>:5684 (DTLS)
- Endpoint: urn:imei:<YOUR_IMEI>
- PSK identity: urn:imei:<YOUR_IMEI>
- PSK key: the same hex key as in Leshan

Reboot the device. In CLIENTS, you should see the endpoint show up and update periodically.


## Step 6 — Quick API checks (reads)

Read Temperature (Object 3303 / Instance 0 / Resource 5700):


```
curl http://<YOUR_IP>:8080/api/clients/urn:imei:<YOUR_IMEI>/3303/0/5700
# → {"content":{"type":"FLOAT","value":"22.9"}, ...}
```

Very likely you will have a response like this:

```
Invalid request: The destination client is sleeping, request cannot be sent.
```

The reason for this is that the API Call needs to be made at the exact moment when the device is transmiting... That is offcourse not very handy, reason why the next step comes.



## Step 7 — Live updates via SSE (Server-Sent Events)

Leshan exposes a stream of server events at:

http://<YOUR_IP>:8080/api/event?ep=urn%3Aimei%3A<YOUR_IMEI>


**Test from terminal (with required headers):**
```
curl -N \
  -H "Accept: text/event-stream" \
  -H "Cache-Control: no-cache" \
  "http://<YOUR_IP>:8080/api/event?ep=urn%3Aimei%3A<YOUR_IMEI>"
```

You should see lines like:

```
event: UPDATED
data: {...}

event: SEND
data: {"ep":"urn:imei:...","val":{"/3303/0/5700":{...}}}
```

The events type 'SEND' contain exactly the telemetry information that we need.

## Step 8 — Node-RED: subscribe & transform

Open Node-RED: http://<YOUR_IP>:1880
Install node-red-contrib-sse-client (Manage palette).

Add a flow:
- SSE client node
- URL: http://<YOUR_IP>:8080/api/event?ep=urn%3Aimei%3A<YOUR_IMEI>
- (Optional) Events: SEND (to receive only actual inbound data)
- Function node → transform payload
- Debug or MQTT out


```
// Extract all relevant sensor values into clean { value: … } format

const v = msg.payload?.val;
if (!v) return null;

// Optional: map LwM2M resource paths to friendly names
const alias = {
  "3303/0/5700": "temperature",   // °C
  "3304/0/5700": "humidity",      // %RH
  "3301/0/5700": "illuminance",   // lux
  "3325/0/5700": "concentration"  // ppm or µg/m³
  // Add more if needed
};

// Helper to safely convert values
function parse(entry) {
  if (!entry) return undefined;
  if (["FLOAT", "INTEGER"].includes(entry.type)) return Number(entry.value);
  return entry.value;
}

const out = {};
for (const [path, entry] of Object.entries(v)) {
  const key = alias[path.replace(/^\//, "")];
  if (key) {
    out[key] = { value: parse(entry) };
  }
}

msg.payload = out;
return msg;
```

Deploy. You should now see clean JSON like:

```
{
  "temperature":   { "value": 22.8 },
  "humidity":      { "value": 45.6 },
  "illuminance":   { "value": 97.0 },
  "concentration": { "value": 809.0 }
}
```

## Step 9 — (Optional) Publish to MQTT

Add an MQTT out node:
- Topic: devices/<IMEI>/telemetry
- Payload: msg.payload (JSON from the Function)


## What could be next ?

- Store telemetry in InfluxDB / Timescale
- Dashboard with Grafana
- Etc...
- 
If you get stuck anywhere, open an issue — happy to help!
