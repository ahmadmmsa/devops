sudo apt install -y debian-keyring debian-archive-keyring apt-transport-https curl
curl -1sLf 'https://dl.cloudsmith.io/public/caddy/stable/gpg.key' | sudo gpg --dearmor -o /usr/share/keyrings/caddy-stable-archive-keyring.gpg
curl -1sLf 'https://dl.cloudsmith.io/public/caddy/stable/debian.deb.txt' | sudo tee /etc/apt/sources.list.p/caddy-stable.list
sudo apt update
sudo apt install caddy -y

sudo nano /etc/caddy/Caddyfile

domain.com {
    reverse_proxy localhost:8069 {
        header_up X-Forwarded-Proto https
    }
}

sudo caddy fmt --overwrite /etc/caddy/Caddyfile


sudo systemctl reload caddy


odoo.conf

proxy_mode = True

docker restart odoo_event