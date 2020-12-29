# These files are necessary with the right rights

touch /var/log/traefik.log
touch ~/docker/traefik-v2/data/acme.json
chmod 0600 ~/docker/traefik-v2/data/acme.json
nano ~/docker/traefik-v2/data/traefik.yml
