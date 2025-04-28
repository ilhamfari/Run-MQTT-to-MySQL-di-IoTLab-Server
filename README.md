# Run MQTT to MySQL di IoTLab Server

Dokumentasi ini merupakan **lanjutan** dari repository sebelumnya: [MQTT-to-MySQL](https://github.com/ilhamfari/MQTT-to-MySQL).

Proyek ini menjelaskan bagaimana menjalankan script Python untuk membaca data dari MQTT Broker dan menyimpan ke MySQL secara otomatis menggunakan **Systemd** service di server Ubuntu.

---

## 🛠️ Persiapan

### 1. Pastikan sudah tersedia:
- Ubuntu server (minimal 20.04)
- Mosquitto Broker (opsional, jika ingin testing lokal)
- MySQL Server atau Remote Hosting Database
- Python 3 dan pip
- Virtual environment (venv)


### 2. Install Dependensi

```bash
sudo apt update
sudo apt install python3 python3-pip python3-venv
```

Buat virtual environment:

```bash
python3 -m venv mqttenv
source mqttenv/bin/activate
```

Install library Python:

```bash
pip install paho-mqtt mysql-connector-python
```


---

## 🖊️ Struktur File

Letakkan script Python berikut di home directory Anda (`/home/iotlab/`):

Nama file: `mqtt_mysql_iotlabserver.py`

```python
from datetime import datetime
import paho.mqtt.client as mqtt
import mysql.connector
import json
import re
import time

partial_payloads = {}
PAYLOAD_TIMEOUT = 3

db = mysql.connector.connect(
    host="[IP Hosting atau Domain]",
    user="[User Database]",
    password="[Password Database]",
    database="[Nama Database]"
)
cursor = db.cursor()

def combine_and_save(slave_id, topic):
    global partial_payloads
    merged = {}
    for part in partial_payloads[slave_id]["data"]:
        merged.update(part)

    VA = merged.get("VA")
    VB = merged.get("VB")
    VC = merged.get("VC")
    IA = merged.get("IA")
    IB = merged.get("IB")
    IC = merged.get("IC")
    PA = merged.get("PA")
    PB = merged.get("PB")
    PC = merged.get("PC")
    PT = merged.get("PT")
    PFA = merged.get("PFA")
    PFB = merged.get("PFB")
    PFC = merged.get("PFC")
    PFT = merged.get("PFT")
    Freq = merged.get("Freq")
    Energy = merged.get("Energy")
    DayaT = merged.get("DayaT")

    query = """
        INSERT INTO parsed_data (
            slave_id, topic, VA, VB, VC, IA, IB, IC,
            PA, PB, PC, PT, PFA, PFB, PFC, PFT,
            Freq, Energy, DayaT
        ) VALUES (
            %s, %s, %s, %s, %s, %s, %s, %s,
            %s, %s, %s, %s, %s, %s, %s, %s,
            %s, %s, %s
        )
    """
    values = (
        slave_id, topic, VA, VB, VC, IA, IB, IC,
        PA, PB, PC, PT, PFA, PFB, PFC, PFT,
        Freq, Energy, DayaT
    )
    cursor.execute(query, values)
    db.commit()

    current_timestamp = datetime.now().strftime('%Y-%m-%d %H:%M:%S')
    print(f"✅ Data slave {slave_id} disimpan pada {current_timestamp}")

    del partial_payloads[slave_id]

def on_message(client, userdata, msg):
    topic = msg.topic
    payload = msg.payload.decode()
    try:
        data_json = json.loads(payload)
        slave_id = str(data_json.get("ID", ""))
        if not slave_id:
            match = re.search(r'slave(\d+)', topic)
            slave_id = match.group(1) if match else "unknown"
        now = time.time()
        if slave_id not in partial_payloads:
            partial_payloads[slave_id] = {"data": [], "timestamp": now, "topic": topic}
        partial_payloads[slave_id]["data"].append(data_json)
        partial_payloads[slave_id]["timestamp"] = now
        if len(partial_payloads[slave_id]["data"]) >= 3:
            combine_and_save(slave_id, topic)
    except Exception as e:
        print("❌ Error parsing/simpan:", e)

def check_timeout():
    now = time.time()
    for slave_id in list(partial_payloads.keys()):
        if now - partial_payloads[slave_id]["timestamp"] > PAYLOAD_TIMEOUT:
            print(f"⚠️ Timeout! Data slave {slave_id} dihapus.")
            del partial_payloads[slave_id]

client = mqtt.Client()
client.on_message = on_message
client.connect("[IP Broker MQTT]", 1883, 60)
client.subscribe("ged-4/monitoring-kWh/#")

print("🚀 Menunggu data MQTT...")

while True:
    client.loop(timeout=1.0)
    check_timeout()
```

---

## 🛅 Setup Systemd untuk Autostart

### 1. Buat file service:

```bash
sudo nano /etc/systemd/system/mqtt-mysql-iotlabserver.service
```

Isikan:

```ini
[Unit]
Description=MQTT to MySQL Connector Python Service
After=network.target

[Service]
User=iotlab
WorkingDirectory=/home/iotlab
ExecStart=/home/iotlab/mqttenv/bin/python /home/iotlab/mqtt_mysql_iotlabserver.py
Restart=always
RestartSec=5

[Install]
WantedBy=multi-user.target
```


### 2. Reload dan Enable Service

```bash
sudo systemctl daemon-reload
sudo systemctl enable mqtt-mysql-iotlabserver.service
sudo systemctl start mqtt-mysql-iotlabserver.service
```


### 3. Cek Status Service

```bash
sudo systemctl status mqtt-mysql-iotlabserver.service
```

Kalau berhasil akan muncul:
```
Active: active (running)
```


### 4. Melihat Log

```bash
sudo journalctl -u mqtt-mysql-iotlabserver.service -f
```

Untuk melihat error atau print output dari script.


---

## 🔄 Perintah Berguna

| Tujuan | Perintah |
|:---|:---|
| Start service | `sudo systemctl start mqtt-mysql-iotlabserver.service` |
| Stop service | `sudo systemctl stop mqtt-mysql-iotlabserver.service` |
| Restart service | `sudo systemctl restart mqtt-mysql-iotlabserver.service` |
| Status service | `sudo systemctl status mqtt-mysql-iotlabserver.service` |
| Lihat log realtime | `sudo journalctl -u mqtt-mysql-iotlabserver.service -f` |
| Disable autostart | `sudo systemctl disable mqtt-mysql-iotlabserver.service` |
| Hapus service | `sudo rm /etc/systemd/system/mqtt-mysql-iotlabserver.service && sudo systemctl daemon-reload` |

---

## 📅 Catatan Penting
- Pastikan semua path file dan virtual environment sudah sesuai.
- Jika script error, cek log dengan `journalctl` untuk melihat debug.
- Pastikan IP broker MQTT dan IP hosting database bisa diakses dari server.

---

> Dokumentasi oleh Ilham Fari | Lanjutan dari proyek [MQTT-to-MySQL](https://github.com/ilhamfari/MQTT-to-MySQL)

---

