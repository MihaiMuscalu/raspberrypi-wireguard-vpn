# Persönlicher VPN-Server auf Raspberry Pi

## Projektbeschreibung
Dieses Projekt zeigt den Aufbau eines sicheren, remote zugänglichen VPN-Servers auf einem Raspberry Pi 4B (2GB).  
Der Server nutzt WireGuard für VPN-Verbindungen, Docker für einfache Verwaltung, und eine kostenlose DuckDNS-Domain für dynamische IP-Adressen.  
Zusätzlich wird die Web-UI WG-Easy für Client-Verwaltung eingesetzt, abgesichert über HTTPS mit Nginx und Let’s Encrypt.



## Installationsanleitung Schritt-für-Schritt

### 1. Raspberry Pi OS Lite installieren
- Raspberry Pi OS Lite (64-bit) auf SD-Karte flashen
- SSH aktivieren und Pi starten

### 2. System aktualisieren
bash
sudo apt update && sudo apt full-upgrade -y && sudo reboot


### 3. DuckDNS Domain einrichten

1. Kostenlos auf [DuckDNS.org](https://www.duckdns.org/) registrieren
2. Subdomain erstellen, z.B. `myvpn.duckdns.org`
3. Token merken

### 4. DuckDNS Updater Script erstellen

bash
mkdir -p ~/duckdns && cd ~/duckdns
cat > duck.sh <<'EOF'
echo url="https://www.duckdns.org/update?domains=MYVPN&token=YOURTOKEN&ip=" | curl -k -K -
EOF
chmod +x duck.sh
./duck.sh


* Cronjob hinzufügen, damit die IP alle 5 Minuten aktualisiert wird:

bash
(crontab -l 2>/dev/null; echo "*/5 * * * * $HOME/duckdns/duck.sh >/dev/null 2>&1") | crontab -


### 5. Docker & Docker Compose installieren

bash
curl -fsSL https://get.docker.com -o get-docker.sh
sudo sh get-docker.sh
sudo apt install -y docker-compose-plugin
sudo usermod -aG docker $USER
newgrp docker


### 6. WG-Easy Docker Container einrichten

bash
mkdir -p ~/wg-easy && cd ~/wg-easy
cat > docker-compose.yml <<'EOF'
version: "3.8"
services:
  wg-easy:
    image: ghcr.io/wg-easy/wg-easy:15
    container_name: wg-easy
    restart: unless-stopped
    ports:
      - "51820:51820/udp"
      - "51821:51821/tcp"
    cap_add:
      - NET_ADMIN
      - SYS_MODULE
    sysctls:
      - net.ipv4.conf.all.src_valid_mark=1
      - net.ipv4.ip_forward=1
    environment:
      - WG_HOST=myvpn.duckdns.org
      - PASSWORD=CHOOSE_A_STRONG_PASSWORD
    volumes:
      - ./data:/etc/wireguard
EOF


* Container starten:

bash
docker compose up -d
docker compose logs -f wg-easy


### 7. Router Port Forwarding

* UDP 51820 → VPN-Traffic
* TCP 80 & 443 → HTTPS für WG-Easy Web UI
* Prüfen, lokale Pi-IP:


### 8. Nginx Reverse Proxy & HTTPS

bash
sudo apt install -y nginx certbot python3-certbot-nginx
sudo tee /etc/nginx/sites-available/wg-easy <<'EOF'
server {
  listen 80;
  server_name myvpn.duckdns.org;
  location / {
    proxy_pass http://127.0.0.1:51821;
    proxy_set_header Host $host;
    proxy_set_header X-Real-IP $remote_addr;
    proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    proxy_set_header X-Forwarded-Proto $scheme;
  }
}
EOF
sudo ln -s /etc/nginx/sites-available/wg-easy /etc/nginx/sites-enabled/
sudo nginx -t && sudo systemctl reload nginx
sudo certbot --nginx -d myvpn.duckdns.org --non-interactive --agree-tos -m your@email.com




## Hinweise zu Ports, DuckDNS & HTTPS

* VPN Port: UDP 51820
* Web UI: TCP 51821 (intern), HTTPS über 443 extern
* DuckDNS: Aktualisiert automatisch alle 5 Minuten
* HTTPS: Let’s Encrypt für sichere Web-UI



## Test der VPN-Verbindung

1. Smartphone/PC WireGuard App:

   * QR-Code oder `.conf` aus WG-Easy Web UI importieren
   * Mit VPN verbinden → sollte „Connected“ anzeigen
2. Prüfung auf Pi:

bash
docker exec -it wg-easy wg show


* Zeigt aktive Peers, Handshake-Zeiten, Datenverkehr

3. Web UI Zugriff:

   * Browser → `https://myvpn.duckdns.org` → Clientliste und Einstellungen einsehen



* Video mit Demo: https://www.youtube.com/watch?v=imtbccfj7Ls


