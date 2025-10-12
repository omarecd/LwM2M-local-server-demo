# LwM2M Local Server Demo — Step-by-Step

This guide shows how to run a local Eclipse Leshan LwM2M server, connect a device over CoAP/DTLS (PSK), and stream live readings into JSON.

## Prerequisites
- A Linux VM with Docker & Docker Compose (Ubuntu 22.04/24.04 works great)
- Public IP (or port-forwarding) for UDP 5683/5684 and TCP 8080
- One LwM2M device (IMEI + PSK), or the Leshan demo client for testing


## Step 1 — Provision the VM

Use any provider (Hetzner, DO, AWS…). A 2 vCPU / 4 GB VM is plenty.




### Install Docker & Compose (Ubuntu)
apt update && apt install -y ca-certificates curl gnupg
install -m 0755 -d /etc/apt/keyrings
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | gpg --dearmor -o /etc/apt/keyrings/docker.gpg
echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] \
  https://download.docker.com/linux/ubuntu $(. /etc/os-release; echo $VERSION_CODENAME) stable" \
  > /etc/apt/sources.list.d/docker.list
apt update
apt install -y docker-ce docker-ce-cli containerd.io docker-compose-plugin
> 




## Step 2 — Get Leshan & Node-RED running (Compose)

Create /root/docker-compose.yml:


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



Download the Leshan demo JAR and start:


mkdir -p /root/leshan
curl -L -o /root/leshan/leshan-server-demo.jar \
https://repo1.maven.org/maven2/org/eclipse/leshan/leshan-server-demo/2.0.0-M15/leshan-server-demo-2.0.0-M15-jar-with-dependencies.jar

docker compose up -d
docker ps



Open the UI: http://<YOUR_IP>:8080

⸻

## Step 3 — Verify ports & basic connectivity

On the VM:


ss -u -lpn | grep 568
# Expect listeners on 5683 and 5684 (UDP)

nc -zvu <YOUR_IP> 5683
nc -zvu <YOUR_IP> 5684
# From your laptop: “succeeded!” means reachable


Optional: test with the Leshan demo client (from your laptop):
curl -L -o leshan-client.jar \
https://repo1.maven.org/maven2/org/eclipse/leshan/leshan-client-demo/2.0.0-M15/leshan-client-demo-2.0.0-M15-jar-with-dependencies.jar
java -jar leshan-client.jar -u coap://<YOUR_IP>:5683


You should see the client register in the Leshan UI.

⸻

## Step 4 — Configure Device Security (PSK)
	1.	Go to Leshan UI → SECURITY.
	2.	Add a record:
	•	Endpoint: urn:imei:<YOUR_IMEI>
	•	Security mode: psk
	•	Identity: urn:imei:<YOUR_IMEI>
	•	Key: 00000000000000000000000000000000 (or your real key)
	3.	Save.

The /data volume ensures this survives container restarts.

⸻

## Step 5 — Configure your Device

On the device, set:
	•	Server URI: coaps://<YOUR_IP>:5684 (DTLS)
	•	Endpoint: urn:imei:<YOUR_IMEI>
	•	PSK identity: urn:imei:<YOUR_IMEI>
	•	PSK key: the same hex key as in Leshan

Reboot the device. In CLIENTS, you should see the endpoint show up and update periodically.

⸻

## Step 6 — Quick API checks (reads)

Read Temperature (Object 3303 / Instance 0 / Resource 5700):


curl http://<YOUR_IP>:8080/api/clients/urn:imei:<YOUR_IMEI>/3303/0/5700
# → {"content":{"type":"FLOAT","value":"22.9"}, ...}


## Step 7 — Live updates via SSE (Server-Sent Events)

Leshan exposes a stream of server events:


http://<YOUR_IP>:8080/api/event?ep=urn%3Aimei%3A<YOUR_IMEI>



Test from terminal:


curl -N http://<YOUR_IP>:8080/api/event?ep=urn%3Aimei%3A<YOUR_IMEI>
# You’ll see events like: event: SEND / data: {...}


## Step 8 — Node-RED: subscribe & transform

Open Node-RED: http://<YOUR_IP>:1880
Install node-red-contrib-sse-client (Manage palette).

Add a flow:
	•	SSE client node
	•	URL: http://<YOUR_IP>:8080/api/event?ep=urn%3Aimei%3A<YOUR_IMEI>
	•	(Optional) Events: SEND (to receive only actual inbound data)
	•	Function node → transform payload
	•	Debug or MQTT out

Function code (maps Temperature/Humidity/Illuminance/Concentration):


// Transform Leshan SSE payload into { temperature:{value}, humidity:{value}, illuminance:{value}, concentration:{value} }

const v = msg.payload?.val;
if (!v) return null;

const alias = {
  "3303/0/5700": "temperature",    // °C
  "3304/0/5700": "humidity",       // %RH
  "3301/0/5700": "illuminance",    // lux
  "3325/0/5700": "concentration"   // ppm / µg·m⁻³
};

function parse(entry) {
  if (!entry) return undefined;
  if (["FLOAT","INTEGER"].includes(entry.type)) return Number(entry.value);
  return entry.value;
}

const out = {};
for (const [path, entry] of Object.entries(v)) {
  const key = alias[path.replace(/^\//, "")];
  if (key) out[key] = { value: parse(entry) };
}

msg.payload = out;
return msg;


Deploy. You should now see clean JSON like:

{
  "temperature":   { "value": 22.8 },
  "humidity":      { "value": 45.6 },
  "illuminance":   { "value": 97.0 },
  "concentration": { "value": 809.0 }
}



## Step 9 — (Optional) Publish to MQTT

Add an MQTT out node:
	•	Topic: devices/<IMEI>/telemetry
	•	Payload: msg.payload (JSON from the Function)

⸻

Troubleshooting Tips
	•	No packets? Check UDP 5683/5684 reachability (tcpdump -ni any udp port 5683).
	•	DTLS PSK errors? Identity/key mismatch; confirm both sides use the same identity and hex key.
	•	SSE shows only UPDATED/COAPLOG? In SSE node, set Events = SEND to get only data pushes.
	•	Persistence lost after restart? Ensure leshan_data:/data is mounted and you’re using the M15 demo JAR path correctly in Compose.

⸻

What’s next
	•	Store telemetry in InfluxDB / Timescale
	•	Dashboard with Grafana
	•	Secure reverse proxy (TLS) in front of Node-RED + Leshan
	•	Bidirectional ops (Execute/Write) from Node-RED back to device

⸻

If you get stuck anywhere, open an issue — happy to help!
