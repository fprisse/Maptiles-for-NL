# Offline Map for Node-RED Worldmap
## MBTileServer Setup Manual

---

## Table of Contents

1. [Download Raster Map Tiles](#1-download-raster-map-tiles)
2. [Install MBTileServer](#2-install-mbtileserver)
3. [Test MBTileServer Manually](#3-test-mbtileserver-manually)
4. [Run MBTileServer as a System Service](#4-run-mbtileserver-as-a-system-service)
5. [Configure Node-RED Worldmap Node](#5-configure-node-red-worldmap-node)

---

## 1. Download Raster Map Tiles

The Node-RED worldmap node is based on Leaflet, which requires **raster PNG tiles**. Vector tiles (used by MapLibre/Mapbox) are not compatible.

**Recommended source: MapTiler Data**

1. Go to https://data.maptiler.com/downloads/europe/netherlands/
2. Create a free account if you do not have one
3. Select the **OpenStreetMap** raster dataset for the Netherlands
4. Download the `.mbtiles` file (approximately 300–600 MB depending on zoom levels selected)

> If you want a wider area (e.g. BeNeLux or Western Europe), download that extract instead. The tile URL format is identical regardless of coverage.

---

## 2. Install MBTileServer

MBTileServer is a single self-contained binary with no dependencies. It reads `.mbtiles` files directly and serves tiles over HTTP.

### 2.1 Create a directory for your tiles

```bash
sudo mkdir -p /opt/tiles
sudo chown $USER:$USER /opt/tiles
```

Copy your downloaded `.mbtiles` file into this folder:

```bash
cp ~/Downloads/netherlands.mbtiles /opt/tiles/
```

### 2.2 Download the mbtileserver binary

Go to: https://github.com/consbio/mbtileserver/releases

Download the correct binary for your system:

**Raspberry Pi (ARM 64-bit):**
```bash
wget https://github.com/consbio/mbtileserver/releases/latest/download/mbtileserver_linux_arm64 -O mbtileserver
```

**Standard Linux PC (x86 64-bit):**
```bash
wget https://github.com/consbio/mbtileserver/releases/latest/download/mbtileserver_linux_amd64 -O mbtileserver
```

### 2.3 Install the binary

```bash
chmod +x mbtileserver
sudo mv mbtileserver /usr/local/bin/mbtileserver
```

Verify it is accessible:

```bash
mbtileserver --version
```

---

## 3. Test MBTileServer Manually

Before setting up the service, confirm everything works:

```bash
mbtileserver --dir /opt/tiles
```

You should see output similar to:

```
Serving tiles from /opt/tiles
Listening on port 8000
```

Open a browser and go to:

```
http://localhost:8000/services
```

You should see a JSON response listing your `.mbtiles` file, for example:

```json
{
  "netherlands": {
    "url": "http://localhost:8000/services/netherlands"
  }
}
```

The tile URL pattern for use in worldmap will be:

```
http://localhost:8000/services/netherlands/tiles/{z}/{x}/{y}.png
```

> The name `netherlands` in the URL comes from the filename of your `.mbtiles` file without the extension. If your file is named `OSM_netherlands.mbtiles`, the URL will use `OSM_netherlands`.

Press Ctrl+C to stop the manual test before proceeding to the next step.

---

## 4. Run MBTileServer as a System Service

### 4.1 Create the systemd service file

```bash
sudo nano /etc/systemd/system/mbtileserver.service
```

Paste the following content, adjusting `User` to your actual Linux username:

```ini
[Unit]
Description=MBTile Server
After=network.target

[Service]
ExecStart=/usr/local/bin/mbtileserver --dir /opt/tiles
Restart=always
User=pi
WorkingDirectory=/opt/tiles

[Install]
WantedBy=multi-user.target
```

Save and exit: Ctrl+O, Enter, Ctrl+X.

### 4.2 Enable and start the service

```bash
sudo systemctl daemon-reload
sudo systemctl enable mbtileserver
sudo systemctl start mbtileserver
```

### 4.3 Verify the service is running

```bash
sudo systemctl status mbtileserver
```

Expected output:

```
● mbtileserver.service - MBTile Server
     Loaded: loaded (/etc/systemd/system/mbtileserver.service; enabled)
     Active: active (running) since ...
```

The service will now start automatically at every boot.

### 4.4 Useful management commands

```bash
sudo systemctl stop mbtileserver        # stop the service
sudo systemctl restart mbtileserver     # restart after changes
sudo journalctl -u mbtileserver -f      # view live logs
```

---

## 5. Configure Node-RED Worldmap Node

### 5.1 Open the worldmap node settings

In the Node-RED editor, double-click your **worldmap** node to open its properties.

### 5.2 Settings to change

| Field | Value |
|---|---|
| **Map name** | `Local` |
| **Map URL** | `http://localhost:8000/services/netherlands/tiles/{z}/{x}/{y}.png` |
| **Map options** | `{"maxZoom":14}` |
| **Map list** | add `Local` to the existing list, e.g. `OSMG,OSMC,EsriC,Local` |

> Adjust the URL to match your actual `.mbtiles` filename if it differs from `netherlands`.

### 5.3 Deploy

Click **Done** then click the red **Deploy** button.

### 5.4 Select the offline map in the browser

Open the worldmap at:

```
http://<your-host>:1880/worldmap
```

Open the **layers panel** (icon in the top-right corner of the map). Under base maps, select **Local**. The map will switch to your offline tiles immediately.

> If the map shows blank grey tiles, confirm mbtileserver is running (`sudo systemctl status mbtileserver`) and that the URL in the worldmap node exactly matches the filename of your `.mbtiles` file.
