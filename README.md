# My Raspberry Pi server setup

Documentation to install my Raspberry Pi server setup including
* Photoprism + Syncthing for open-source alternative to Google Photos
  * Automatic backup
  * View online on any device
* Calibre server for ebook library and management
  * Website for management and viewing/downloading on any device
  * Standard OPDS support for KOReader (Kobo) and Moon Reader (Android phone)

## Photoprism + Syncthing (required as base image)

1. Flash the [PhotoprismPi image](https://docs.photoprism.app/getting-started/raspberry-pi/microsd-image/)
2. Install [Syncthing as a service](https://pimylifeup.com/raspberry-pi-syncthing/)
    ```bash
    sudo apt-get update -y
    sudo apt-get full-upgrade -y
    sudo apt-get install apt-transport-https
    curl -s https://syncthing.net/release-key.txt | gpg --dearmor | sudo tee /usr/share/keyrings/syncthing-archive-keyring.gpg >/dev/null
    echo "deb [signed-by=/usr/share/keyrings/syncthing-archive-keyring.gpg] https://apt.syncthing.net/ syncthing stable" | sudo tee /etc/apt/sources.list.d/syncthing.list
    sudo apt-get update -y
    sudo apt-get install syncthing
    ```
    Change IP to local one to only allow LAN connections
    ```bash
    syncthing &
    vim ~/.config/syncthing/config.xml
    ```
    and change `<address>127.0.0.1:8384</address>` to `<address>192.168.x.x:8384</address>`.
    Set up as a service
    ```bash
    sudo vim /lib/systemd/system/syncthing.service
    ```
    and add
    ```ini
    [Unit]
    Description=Syncthing - Open Source Continuous File Synchronization
    Documentation=man:syncthing(1)
    After=network.target
    
    [Service]
    User=pi
    ExecStart=/usr/bin/syncthing -no-browser -no-restart -logflags=0
    Restart=on-failure
    RestartSec=5
    SuccessExitStatus=3 4
    RestartForceExitStatus=3 4
    
    # Hardening
    ProtectSystem=full
    PrivateTmp=true
    SystemCallArchitectures=native
    MemoryDenyWriteExecute=true
    NoNewPrivileges=true
    
    [Install]
    WantedBy=multi-user.target
    ```
    And finally enable with
    ```bash
    sudo systemctl enable syncthing
    sudo systemctl start syncthing
    ```
    Access GUI with `192.168.x.x:8384`.
3. Copy or adjust Photoprism settings from `photoprism/docker-compose.yml` to `/boot/firmware/docker-compose/photoprism/docker-compose.yml` and edit Syncthing correspondingly
4. Edit the crontab
    ```bash
    crontab -e

    0 6 * * * cd /boot/firmware/docker-compose/photoprism && docker-compose exec photoprism photoprism index
    ```
    And add scheduled rebooting with sudo privileges
    ```bash
    sudo crontab -e

    0 4 * * * /sbin/shutdown -r now
    ```
5. (Optional) Refresh domain with a free domain service (`dy.fi` for Finnish IPs).
    ```bash
    crontab -e

    0 1 * * * wget --delete-after --no-check-certificate --no-proxy --user=EMAIL --password=PASSWORD https://www.dy.fi/nic/update?hostname=DOMAIN.dy.fi
    @reboot sleep 300 && wget --delete-after --no-check-certificate --no-proxy --user=EMAIL --password=PASSWORD https://www.dy.fi/nic/update?hostname=DOMAIN.dy.fi
    ```
6. Perform possible `fstab` changes for mounting USB hard-drive
    ```bash
    sudo /etc/fstab

    # Add something like the following for where to mount,  adjust file system (ext4 here) and UUID
    # These can be seen with `lsblk -f`
    UUID=049e038e-bbcb-46ae-b331-7c0f6028d8aa  /media/b/      ext4    defaults,nofail,errors=remount-ro 0       1
    ```


## Calibre server

Implemented using Docker compose using [linuxserver/docker-calibre-web](https://github.com/linuxserver/docker-calibre-web).
Copy the `calibre` folder in the repo to the server and consider the following steps.
The modified `init` is required for having correct permissions to edit book metadata.

1. Create the `config`, `library`, and `addbooks` folders under some folder `ebooks/` (`/media/b/Media/ebooks/` in example).
2. Run `docker-compose up -d` in the `calibre` folder
3. Schedule importing books
    ```bash
    crontab -e

    0 3 * * * cd ~/calibre-web && docker-compose exec calibre-web calibredb add -r "/addbooks" --library-path="/books" && rm -rf /media/b/Media/ebooks/addbooks/*
    ```
4. Start using Calibre through the default port 8083. This can be forwarded in the Docker compose YML or in your router.
    The OPDS API is located at `HOSTNAME:8083/opds`.

### Transmission torrent client

Transmission is setup to download torrents given in some folder, and move them to a given output folder upon completion.
This is setup using Docker compose with [jaymoulin/transmission](https://hub.docker.com/r/jaymoulin/transmission/).

1. Create the `to_download`, `incomplete`, and `config` folders under some folder `transmission/`. The output is setup as the `addbooks` folder in [Calibre server](#calibre-server).
2. Run `docker-compose up -d` once.
3. Edit the `rpc-whitelist` filed in `config/settings.json` to the following
    ```json
    "rpc-password": "{sha1 of password",
    "rpc-username": "username"
    ...
    "rpc-whitelist": "*.*.*.*",
    ```
4. Start using Transmission through the default ports 9091 and 51413. These can be forwarded in the Docker compose YML or in your router.


## Improvements
* Implement Syncthing with Docker compose instead of manual install.