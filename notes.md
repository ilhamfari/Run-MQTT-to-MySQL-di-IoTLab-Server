# Run MQTT to MySQL di IoTLab Server

Dokumentasi lanjutan dari [MQTT-to-MySQL](https://github.com/ilhamfari/MQTT-to-MySQL).
Script Python untuk auto-run koneksi MQTT ke MySQL menggunakan systemd di server.

## Langkah Instalasi dan Setup
1. Install dependensi: python3, pip, venv.
2. Setup virtualenv dan install paho-mqtt, mysql-connector-python.
3. Letakkan file `mqtt_mysql_iotlabserver.py` di /home/iotlab/.
4. Buat systemd service `mqtt-mysql-iotlabserver.service`.
5. Enable dan start service.

## Perintah Berguna
- Start: `sudo systemctl start mqtt-mysql-iotlabserver.service`
- Stop: `sudo systemctl stop mqtt-mysql-iotlabserver.service`
- Status: `sudo systemctl status mqtt-mysql-iotlabserver.service`
- Log: `sudo journalctl -u mqtt-mysql-iotlabserver.service -f`
