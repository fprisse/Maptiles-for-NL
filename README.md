# Offline Map Tiles for Node-RED Worldmap
## Setup Guide

---

## 1. Download Raster Map Tiles

Use **MOBAC (Mobile Atlas Creator)** to create a free raster `.mbtiles` file.

1. Install Java 11 64-bit from https://adoptium.net/temurin/releases/?version=11
2. Download MOBAC from https://mobac.sourceforge.io/ and extract it
3. Launch `Mobile_Atlas_Creator.exe`
4. When prompted for atlas format select **MBTiles SQLite**, name it `Nederland`
5. In the Map Source dropdown find and select **OSM Standard**
   - If not present, create `mapsources\osm_standard.xml` in the MOBAC folder:
   ```xml
   <?xml version="1.0" encoding="UTF-8"?>
   <customMapSource>
       <name>OSM Standard</name>
       <minZoom>0</minZoom>
       <maxZoom>18</maxZoom>
       <tileType>png</tileType>
       <tileUpdate>None</tileUpdate>
       <url>https://tile.openstreetmap.org/{$z}/{$x}/{$y}.png</url>
       <backgroundColor>#000000</backgroundColor>
   </customMapSource>
   ```
   Restart MOBAC after saving the file.
6. Navigate the map to the Netherlands
7. Hold **Ctrl** and drag a rectangle over the Netherlands
8. Tick zoom levels **6 through 14** in the left panel
9. Click **Add Selection** then **Create Atlas**
10. Output file is in the `atlases\Nederland\` folder inside the MOBAC directory

---

For Open Seamaps 
```xml
<?xml version="1.0" encoding="UTF-8"?>
<customMapSource>
    <name>OpenSeaMap</name>
    <minZoom>0</minZoom>
    <maxZoom>18</maxZoom>
    <tileType>png</tileType>
    <tileUpdate>None</tileUpdate>
    <url>https://tiles.openseamap.org/seamark/{$z}/{$x}/{$y}.png</url>
    <backgroundColor>#000000</backgroundColor>
</customMapSource>
```

Save as `openseamap.xml` in the `mapsources` folder inside your MOBAC directory, then restart MOBAC.

Remember: OpenSeaMap is a **transparent overlay** showing nautical marks and buoys — it has no background. It is designed to sit on top of OSM Standard, not replace it.
You can keep it totaaly seperate and comine in teh WorldMap node in Node-red

## 2. Copy Tiles to Linux Server

Using WinSCP (drag and drop) or from Windows command prompt:

```
scp C:\MOBAC\atlases\Nederland\Nederland.mbtiles user@192.168.1.140:/opt/tiles/
```

Create the tiles folder first if needed:

```bash
sudo mkdir -p /opt/tiles
sudo chown $USER:$USER /opt/tiles
```

---

## 3. Install MBTileServer

Find the correct binary filename for your system:

```bash
curl -s https://api.github.com/repos/consbio/mbtileserver/releases/latest | grep "browser_download_url"
```

Download the `linux_amd64` version (or `linux_arm64` for Raspberry Pi):

```bash
wget https://github.com/consbio/mbtileserver/releases/download/vX.X.X/mbtileserver_vX.X.X_linux_amd64 -O ~/mbtileserver
chmod +x ~/mbtileserver
sudo mv ~/mbtileserver /usr/local/bin/mbtileserver
```

Test it:

```bash
mbtileserver -d /opt/tiles
```

Then open `http://192.168.1.140:8000/services` — you should see your tileset listed.

---

## 4. Run as a System Service

```bash
sudo nano /etc/systemd/system/mbtileserver.service
```

Paste this, replacing `user` with your Linux username:

```ini
[Unit]
Description=MBTile Server
After=network.target

[Service]
ExecStart=/usr/local/bin/mbtileserver -d /opt/tiles
Restart=always
User=user
WorkingDirectory=/opt/tiles

[Install]
WantedBy=multi-user.target
```

Enable and start:

```bash
sudo systemctl daemon-reload
sudo systemctl enable mbtileserver
sudo systemctl start mbtileserver
sudo systemctl status mbtileserver
```

Optional — allow service control without password. Add to sudoers (`sudo visudo`):

```
user ALL=(ALL) NOPASSWD: /usr/bin/systemctl start mbtileserver, /usr/bin/systemctl stop mbtileserver, /usr/bin/systemctl restart mbtileserver
```

---

## 5. Configure Node-RED Worldmap

### Option A — Via worldmap node settings

Double-click the worldmap node and fill in:

| Field | Value |
|---|---|
| **Map name** | `Nederland OSM` |
| **Map URL** | `http://192.168.1.140:8000/services/Nederland/tiles/{z}/{x}/{y}.png` |
| **Map options** | `{"maxZoom":14,"attribution":"© OpenStreetMap contributors"}` |
| **Map list** | add `Nederland OSM` to existing entries |
| **Base map** | select `Custom Map Provider` |

### Option B — Via inject + function node (auto-registers on deploy)

Add an **inject** node (fire once, delay 1s) connected to a **function** node, wired to the worldmap node:

```javascript
msg.payload = {
    command: {
        map: {
            name: "Local",
            url: "http://192.168.1.140:8000/services/Nederland/tiles/{z}/{x}/{y}.png",
            opt: { maxZoom: 14 }
        }
    }
};
return msg;
```

Also add `Local` to the **Map list** field in the worldmap node, and add `ADS-B` to the **Overlays** list.

### Selecting the map

Open the worldmap at `http://192.168.1.140:1880/worldmap`, open the layers panel (top right) and select **Local** or **Nederland OSM** as the base map.
