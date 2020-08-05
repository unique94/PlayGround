# V2Ray with mKCP

## Create VMs
<img src="./images/create_VM.png" width="450"/>

## Install V2Ray
- Download script

    ```wget https://install.direct/go.sh```

    <img src="./images/V2Ray_Install_1.png"/>


- Install V2Ray

    ```sudo bash go.sh```

    <img src="./images/V2Ray_Install_2.png"/>

## Config V2Ray Server
- Update the V2Ray config in `/etc/v2ray/config.json`

    ```
    {
        "inbounds": [{
            "port": 29544, // Change the port
            "protocol": "vmess",
            "settings": {
            "clients": [
                {
                "id": "67ec54ac-bba6-4cf9-b6f5-f301b6dff23f", // Change the UUID. 
                "level": 1,
                "alterId": 64
                }
            ]
            },
            "streamSettings": {  // This part is for mKCP
            "network": "mkcp",
            "kcpSettings": {
                "uplinkCapacity": 5,
                "downlinkCapacity": 100,
                "congestion": true,
                "header": {
                "type": "none"
                }
            }
            }
        }],
        "outbounds": [{
            "protocol": "freedom",
            "settings": {}
        },{
            "protocol": "blackhole",
            "settings": {},
            "tag": "blocked"
        }],
        "routing": {
            "rules": [
            {
                "type": "field",
                "ip": ["geoip:private"],
                "outboundTag": "blocked"
            }
            ]
        }
    }
    ```

- Restart V2Ray service.

    ```systemctl restart v2ray```

## Update VM config.

- Disable auto-shutdown

    <img src="./images/V2Ray_Install_3.png" width="450"/>

- Add Inbound port rules

    In this case, we use port `29544` as the V2Ray inbound port. We need the add the rule in VM firewall.

    <img src="./images/V2Ray_Install_4.png" width="450"/>

## Install V2Ray Client

- Install Clients.

    Clients can be found [here](https://github.com/v2ray/v2ray-core/releases). Unzip the file and you can get the below files.

    <img src="./images/V2Ray_Install_5.png"/>

- Update Client Config.

    ```
    {
        "inbounds": [
            {
            "port": 1080,
            "protocol": "socks",
            "sniffing": {
                "enabled": true,
                "destOverride": ["http", "tls"]
            },
            "settings": {
                "auth": "noauth"
            }
            }
        ],
        "outbounds": [
            {
                "protocol": "vmess",
                "settings": {
                    "vnext": [
                    {
                        "address": "Your VM IP",
                        "port": 29544,
                        "users": [
                        {
                            "id": "67ec54ac-bba6-4cf9-b6f5-f301b6dff23f",
                            "alterId": 64
                        }
                        ]
                    }
                    ]
                },
                "streamSettings": {
                    "network": "mkcp",
                    "kcpSettings": {
                        "uplinkCapacity": 5,
                        "downlinkCapacity": 100,
                        "congestion": true,
                        "header": {
                            "type": "none"
                        }
                    }
                }
            }
        ]
    }
    ```

- Run v2ray.exe. Happy V2Ray.

## Ref
https://guide.v2fly.org/advanced/mkcp.html

