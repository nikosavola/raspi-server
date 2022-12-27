# Raspberry Pi server

Documentation to install my Raspberry Pi server setup

## Photoprism + Syncthing

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
3. Copy or adjust Photoprism settings from `photoprism.yml` to `/boot/firmware/docker-compose/photoprism/docker-compose.yml` and edit Syncthing correspondingly
4. Edit the crontab
    ```bash
    crontab -e

    0 6 * * * cd /boot/firmware/docker-compose/photoprism && docker-compose exec photoprism photoprism index
    0 1 * * * wget --delete-after --no-check-certificate --no-proxy --user=EMAIL --password=PASSWORD https://www.dy.fi/nic/update?hostname=DOMAIN.dy.fi
    @reboot sleep 300 && wget --delete-after --no-check-certificate --no-proxy --user=EMAIL --password=PASSWORD https://www.dy.fi/nic/update?hostname=DOMAIN.dy.fi
    ```
    And add scheduled rebooting
    ```bash
    sudo crontab -e

    0 4 * * * /sbin/shutdown -r now
    ```
5. Perform possible `fstab` changes
    ```bash
    sudo /etc/fstab

    # Add something like the following for where to mount,  adjust file system (ext4 here) and UUID
    # These can be seen with `lsblk -f`
    UUID=049e038e-bbcb-46ae-b331-7c0f6028d8aa  /media/b/      ext4    defaults,nofail,errors=remount-ro 0       1
    ```


## Calibre server

TODO with https://github.com/bcleonard/calibre-cops



### Transmission downloader

https://hub.docker.com/r/jaymoulin/transmission/