# Queue Simulator

This folder contains the queue simulator that publishes fake restaurant order data to an MQTT broker. Use it to run the workshop on your own network without the TTC infrastructure.

Adapted from [Mo3he/axis-display-queue](https://github.com/Mo3he/axis-display-queue).

## Requirements

- Python 3.10+
- An MQTT broker (e.g. Mosquitto)

## Quick Start

### 1. Install an MQTT Broker

If you don't already have a broker running:

**macOS:**
```bash
brew install mosquitto
brew services start mosquitto
```

**Ubuntu/Debian:**
```bash
sudo apt install mosquitto mosquitto-clients
sudo systemctl start mosquitto
```

**Docker:**
```bash
docker run -d --name mosquitto -p 1883:1883 eclipse-mosquitto:2
```

### 2. Install Python Dependencies

```bash
cd simulator
python -m venv .venv
source .venv/bin/activate    # Windows: .venv\Scripts\activate
pip install -r requirements.txt
```

### 3. Configure

Edit `config.yaml` and set your broker address:

```yaml
mqtt:
  host: localhost    # change to your broker IP
  port: 1883
```

You can also adjust timing (how fast orders arrive, cook, and get picked up):

```yaml
orders:
  arrival_min: 60      # seconds between new orders (min)
  arrival_max: 120     # seconds between new orders (max)
  cook_time_min: 120   # how long an order stays in queue (min)
  cook_time_max: 360   # how long an order stays in queue (max)
  pickup_time_min: 30  # how long a ready order waits (min)
  pickup_time_max: 120 # how long a ready order waits (max)
```

For a faster demo (orders every 5-15 seconds), try:

```yaml
orders:
  arrival_min: 5
  arrival_max: 15
  cook_time_min: 20
  cook_time_max: 60
  pickup_time_min: 10
  pickup_time_max: 30
```

### 4. Run

```bash
python queue_system.py
```

Output:

```
08:30:01  Restaurant open. Broker localhost:1883
08:31:12  Order 01 placed | queue=[1] ready=[]
08:32:45  Order 02 placed | queue=[1, 2] ready=[]
08:33:20  Order 01 READY  | queue=[2] ready=[1]
08:34:01  Order 03 placed | queue=[2, 3] ready=[1]
08:34:50  Order 01 done   | queue=[2, 3] ready=[]
```

Press `Ctrl+C` to stop. The simulator clears retained messages on exit.

### 5. Update the Workshop Broker Address

In the workshop README, replace `10.129.174.38` with your broker's IP address in:
- Step 2.2 (MQTT broker configuration)
- Step 5.2 (only relevant if you also have D6310 sensors)

## What It Publishes

| Topic | Payload Example | Description |
|---|---|---|
| `restaurant/queue` | `[3, 7, 12]` | Orders currently being prepared |
| `restaurant/ready` | `[2, 5]` | Orders ready for pickup |

### Behavior

- New orders arrive at random intervals (configurable)
- Each order spends a random "cook time" in the queue, then moves to ready
- After a random "pickup time" the order is removed from ready
- Order numbers wrap back to 1 after reaching `max_number` (default 99)
- Messages are retained so new MQTT subscribers immediately get the current state
- On shutdown (`Ctrl+C`), both topics are cleared to empty arrays

## Verifying It Works

Subscribe to the topics with a mosquitto client:

```bash
mosquitto_sub -h localhost -t "restaurant/#" -v
```

You should see:

```
restaurant/queue [1, 2, 3]
restaurant/ready [1]
restaurant/queue [2, 3, 4]
restaurant/ready []
```
