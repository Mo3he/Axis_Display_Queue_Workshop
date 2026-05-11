# Axis Display Speaker - Queue Display Workshop

Node-RED (already running) subscribed to MQTT Broker (10.129.174.38) and posting to Axis Display Speaker via VAPIX.

## What This Workshop Does

A restaurant-style order queue system. Node-RED subscribes to live queue state over MQTT and drives an Axis Display Speaker using the VAPIX Speaker Display Notification API.

```
restaurant/queue  -->  Node-RED  -->  PREPARING:  #3  #7  #12    (white on dark blue, scrolling)
restaurant/ready  -->  Node-RED  -->  ORDER READY:  #2  #5        (black on green, 8 s timer)
                                  -->  reverts to queue view       (after 8 seconds)
```

### Components

| Component | Role |
|---|---|
| MQTT Broker (10.129.174.38:1883) | Publishes live queue and ready order lists as JSON arrays |
| Node-RED (http://localhost:1880) | Subscribes to MQTT, formats messages, calls VAPIX API |
| Axis Display Speaker | Shows queue status on screen |

## Prerequisites

- Axis Display Speaker (factory defaulted, one per participant)
- AXIS IP Utility (already installed)
- Node-RED already running at http://localhost:1880 from Workshop 1

## Setup

### 1. Find Your Speaker's IP and Set Credentials

What happens: Identify the speaker on the network and create your login credentials.

1. Power on your Axis Display Speaker
2. Open **AXIS IP Utility** on your workstation
3. Note the IP address of your speaker
4. Open the speaker web interface: `http://<speaker-ip>`
5. When prompted, set a username and password of your choosing
6. Keep these credentials handy for the next step

### 2. Import the Node-RED Flow

What happens: Load the pre-built queue display flow into your Node-RED instance.

1. Open Node-RED: http://localhost:1880
2. In the top right select the burger menu (☰) then **Import**
3. Select **Upload** and choose `queue_display_flow.json` from this repository  
   (or paste the contents directly into the import dialog)
4. Click **Import**

### 3. Configure the MQTT Broker

What happens: Node-RED connects to the shared MQTT broker where queue data is already being published.

> For the TTC workshop the queue simulator is already running and publishing to `10.129.174.38:1883`. You only need to point your MQTT nodes at it.

1. Double-click the **Queue topic** MQTT node
2. Click the pencil (✎) next to "Server"
3. Set the **Server** to `10.129.174.38` and **Port** to `1883`
4. Click **Update** then **Done**
5. Repeat for the **Ready topic** MQTT node (it will share the same broker config once set)

### 4. Configure the Speaker Node

What happens: Node-RED will call the VAPIX API on your speaker to update the display.

1. Double-click the **POST to speaker display** http request node
2. Change the URL to:  
   `http://<your-speaker-ip>/config/rest/speaker-display-notification/v1/simple`
3. In the **Credentials** section enter your speaker username and password
4. Click **Done**

### 5. Deploy and Test

1. Click **Deploy** in the top right corner
2. Check the debug panel on the right side - you should see MQTT messages arriving and API responses appearing
3. Watch your speaker display cycle between the queue view and ready alerts

No data on the debug panel? Check the MQTT broker address and that the broker node shows a green "connected" indicator.

---

## Task 2: Customize the Display

What happens: Modify the Node-RED function nodes to change how the display looks and behaves.

> All display settings are in the two **function** nodes: **Build queue message** and **Build ready message**.

### Change Colors

1. Double-click **Build queue message**
2. Edit the `backgroundColor` and `textColor` fields. Values are RGB hex strings:
   - Dark blue (current): `#1A1A2E`
   - Try navy: `#003366` or purple: `#4B0082`
3. Click **Done** then **Deploy**

Try setting a matching color change in **Build ready message** too.

### Change Scroll Speed

- `scrollSpeed` accepts values from `0` (static) to `10` (fastest)
- `scrollDirection` options: `fromRightToLeft`, `fromBottomToTop`, `fromLeftToRight`

### Change Text Size

- `textSize` options: `small`, `medium`, `large`

### Change the Message Format

- Edit the `text` variable in the function node to display different content
- For example, change `'PREPARING:  '` to `'NOW COOKING:  '`

### Change the Ready Alert Duration

1. Double-click **Build ready message**
2. Find the `setTimeout(function() {... }, 8000)` line and change `8000` to a different value in milliseconds
3. Update the `duration: { type: 'time', value: 8000 }` field to match
4. Click **Done** then **Deploy**

---

## Task 3: Display Air Quality Data on the Speaker

What happens: Combine both workshops. Pull live air quality data from the D6310 sensors (already flowing through the MQTT broker from Workshop 1) and show it on your speaker.

### Import the Air Quality Display Flow

1. In Node-RED, top right menu (☰) then **Import**
2. Select **Upload** and choose `aq_display_flow.json`
3. Click **Import** - a new **Air Quality Display** tab will appear

### Configure the MQTT Broker

1. Double-click the **D6310 Air Quality** MQTT node
2. Click the pencil (✎) next to "Server" and select the existing `10.129.174.38` broker entry
3. Click **Update** then **Done**

The topic is pre-set to `axis/+/event/tns:axis/AirQualityMonitor/Metadata/#` which matches all four D6310 sensors.  
To show data from only one sensor replace `+` with its serial:

| Location | Serial |
|---|---|
| Play Space | E827251A7B8B |
| Learn Space | E827251A7B09 |
| Server Rack | E827251AA4C6 |
| Entrance | E827251A8AF7 |

### Configure the Speaker Node

1. Double-click the **POST to speaker display** node in the Air Quality tab
2. Update the URL and credentials to match your speaker (same as Task 1 setup)
3. Click **Done**

### Deploy and Test

1. Click **Deploy**
2. The display updates at most once every 30 seconds (rate limited to avoid spamming the display)
3. You should see messages like:  
   `AQI: 12   CO2: 487 ppm   Temp: 21.4°C   [E827251A7B09]`

### Understanding the Flow

```
D6310 sensor (MQTT every 1 s)
      |
      v
axis/+/event/tns:axis/AirQualityMonitor/Metadata/#
      |
      v
Rate limit (drops all but 1 message per 30 s)
      |
      v
Format AQ message (extracts AQI, CO2, Temperature)
      |
      v
POST to speaker display (VAPIX API)
```

### Customize the Air Quality Display

- Edit **Format AQ message** to show different fields: `VOC`, `Humidity`, `PM25`, etc.
- Change the background color (`#2D1B69` is currently a deep purple) to visually distinguish it from the queue display
- Adjust the rate limit node to update more or less frequently

---

## Troubleshooting

| Symptom | Check |
|---|---|
| MQTT nodes show "disconnected" | Verify broker address is `10.129.174.38`, port `1883`, no TLS |
| No messages in debug panel | Confirm the simulator is running - ask your workshop host |
| Speaker shows nothing | Verify the speaker IP in the URL of the POST node |
| API response shows 401 | Wrong username or password in the POST node credentials |
| API response shows 404 | Wrong URL path, confirm it ends in `/v1/simple` |
| Display not updating on Task 3 | Wait up to 30 s for the rate limiter to pass a message through |
| Digest auth error | Axis speakers require HTTP Digest - confirm the POST node auth type is set to **Digest** |

---

## Reference

### VAPIX Speaker Display Notification API

All display updates call:
```
POST /config/rest/speaker-display-notification/v1/simple
```
Authentication: HTTP Digest

| Field | Description |
|---|---|
| `message` | Text to display (max 1000 characters) |
| `textColor` | RGB hex, e.g. `#FFFFFF` |
| `backgroundColor` | RGB hex, e.g. `#1A1A2E` |
| `textSize` | `small`, `medium`, or `large` |
| `scrollDirection` | `fromRightToLeft`, `fromBottomToTop`, `fromLeftToRight` |
| `scrollSpeed` | `0` (static) to `10` (fastest) |
| `duration.type` | `time` (milliseconds), `repetitions`, or `timeCompleteMessage` |

Full reference: https://developer.axis.com/vapix/device-configuration/speaker-display-notification/

### MQTT Queue Topics

| Topic | Payload | Description |
|---|---|---|
| `restaurant/queue` | `[3, 7, 12]` | JSON array of order numbers being prepared |
| `restaurant/ready` | `[2, 5]` | JSON array of order numbers ready for pickup |

### D6310 Air Quality Topics (Workshop 1)

| Topic | Description |
|---|---|
| `axis/+/event/tns:axis/AirQualityMonitor/Metadata/#` | All sensors (wildcard) |
| `axis/E827251A7B8B/event/tns:axis/AirQualityMonitor/Metadata/#` | Play Space |
| `axis/E827251A7B09/event/tns:axis/AirQualityMonitor/Metadata/#` | Learn Space |
| `axis/E827251AA4C6/event/tns:axis/AirQualityMonitor/Metadata/#` | Server Rack |
| `axis/E827251A8AF7/event/tns:axis/AirQualityMonitor/Metadata/#` | Entrance |

## What You've Built

- **Task 1:** A live queue display that reacts instantly to order state changes over MQTT
- **Task 2:** A customized display tuned to your visual preferences
- **Task 3:** A unified IoT data pipeline bridging two Axis sensor systems through a single MQTT broker onto one display

---

Developed for the Axis TTC Workshop
