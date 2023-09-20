# SMH-Info-Dump  

**This is not intended as a guide as many of those are customized to my needs, but can be a good starting point for others.**  
This is a reminder for myself on how to set up my proxmox environement for SMH  
The ultimate goal is to run 108 5TB node on 3 separate PoET phases  
This guide will contain everything from hardware to monitoring  

## Hardware used  

AMD EPYC 7742  
SuperMicro H11SSL-i  
512GB 3200Mhz RAM  
2x SAS9300-16I  
4x 1TB NVMe (Probably 8 or 12 once more nodes are running)  
PCIe 16x to quad-NVMe (Probably more like above)  
120GB NVMe boot drive  

## Setup  

One container runs an apache server to send software updates to the nodes.  
One runs Prometheus and grafana for monitoring.  
Use round-robin on the NVMes for the node containers storing the DBs.  
Mount PoST separatly in the /root/post/data folder.  

**Node container**  
Download latest node in /root/spacemesh.  
Customize config to fit your needs, this particular setup uses 5TB plots in 32GB files.  
Create, enable and start services.  

**spacemesh-node service**  
```
[Unit]  
Description=Spacemesh Node  

[Service]  
User=root  
KillMode=process  
KillSignal=SIGINT  
WorkingDirectory=/root/spacemesh  
ExecStart=/root/spacemesh/go-spacemesh --listen /ip4/0.0.0.0/tcp/20000 --config ./config.mainnet.json --filelock ./spacemesh.lock --grpc-public-listener 0.0.0.0:30000 --grpc-private-listener 127.0.0.1:40000 --grpc-json-listener 0.0.0.0:50000 --smeshing-coinbase sm1xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx --smeshing-opts-maxfilesize 34359738368 --smeshing-opts-numunits 80 --smeshing-opts-provider 4294967295 --smeshing-start   
Restart=always  
RestartSec=5  
LimitNOFILE=65536  

[Install]  
WantedBy=multi-user.target  
```

**spacemesh-metrics service**  
```
[Unit]  
Description=Spacemesh Metrics  

[Service]  
User=root  
KillMode=process  
KillSignal=SIGINT  
WorkingDirectory=/root/spacemesh  
ExecStart=python3 metrics.py  
Restart=always  
RestartSec=5  
LimitNOFILE=65536  

[Install]  
WantedBy=multi-user.target  
```

**/root/spacemesh/metrics.py**  
```
import subprocess  
import json  
import time  
import threading  
import socket  
from prometheus_client import start_http_server, Gauge  

# Metrics with additional 'instance' label for the custom name  
CONNECTED_PEERS = Gauge('spacemesh_connected_peers', 'Number of connected peers', ['instance'])  
IS_SYNCED = Gauge('spacemesh_is_synced', 'Is the node synced', ['instance'])  
SYNCED_LAYER = Gauge('spacemesh_synced_layer', 'Synced Layer', ['instance'])  
TOP_LAYER = Gauge('spacemesh_top_layer', 'Top Layer', ['instance'])  
VERIFIED_LAYER = Gauge('spacemesh_verified_layer', 'Verified Layer', ['instance'])  

# Timeout in seconds  
custom_timeout = 2  

def fetch_spacemesh_metrics(port):  
    instance_name = socket.gethostname()  
    while True:  
        try:  
            cmd = f'grpcurl --plaintext -d \'{{}}\' localhost:{port} spacemesh.v1.NodeService.Status'  
            output = subprocess.run(cmd, shell=True, stdout=subprocess.PIPE, stderr=subprocess.PIPE, text=True, timeout=custom_timeout)  

            if output.returncode == 0:  
                response = json.loads(output.stdout)  
                status = response.get('status', {})  

                CONNECTED_PEERS.labels(instance=instance_name).set(status.get("connectedPeers", 0))  
                IS_SYNCED.labels(instance=instance_name).set(status.get("isSynced", 0))  
                SYNCED_LAYER.labels(instance=instance_name).set(status.get("syncedLayer", {}).get("number", 0))  
                TOP_LAYER.labels(instance=instance_name).set(status.get("topLayer", {}).get("number", 0))  
                VERIFIED_LAYER.labels(instance=instance_name).set(status.get("verifiedLayer", {}).get("number", 0))  
            else:  
                CONNECTED_PEERS.labels(instance=instance_name).set(0)  
                IS_SYNCED.labels(instance=instance_name).set(0)  
                SYNCED_LAYER.labels(instance=instance_name).set(0)  
                TOP_LAYER.labels(instance=instance_name).set(0)  
                VERIFIED_LAYER.labels(instance=instance_name).set(0)  

            time.sleep(15)  

        except Exception as e:  
            CONNECTED_PEERS.labels(instance=instance_name).set(0)  
            IS_SYNCED.labels(instance=instance_name).set(0)  
            SYNCED_LAYER.labels(instance=instance_name).set(0)  
            TOP_LAYER.labels(instance=instance_name).set(0)  
            VERIFIED_LAYER.labels(instance=instance_name).set(0)  

            time.sleep(60)  

if __name__ == '__main__':  
    start_http_server(45000)  
    fetch_spacemesh_metrics(30000)  
```

**prometheus.yml**  
```
  - job_name: 'Spacemesh Public'  
    static_configs:  
      - targets: ["10.0.2.100:45000","10.0.2.101:45000","10.0.2.102:45000","10.0.2.103:45000","10.0.2.104:45000","10.0.2.105:45000","10.0.2.106:45000","10.0.2.107:45000"]  
```

## Update  
Using the Apache server, we download the update once and trigger it on all nodes.  

**update.sh**  
```
sudo systemctl stop spacemesh-node && cd spacemesh && rm go-spacemesh libpost.so profiler && wget 10.0.2.201/update/go-spacemesh && wget 10.0.2.201/update/libpost.so && wget 10.0.2.201/update/profiler && chmod +x go-spacemesh && cd && sudo systemctl start spacemesh-node  
```
**trigger.sh**
```
for i in {150..175}; do  
  ip="10.0.2.$i"  
  sshpass -p password ssh root@$ip "/root/update.sh"  
done  
```

## Example private/public node config  

**Public**  
```
  "p2p": {  
    "disable-reuseport": false,  
    "p2p-disable-legacy-discovery": true,  
    "direct": [  
        "/ip4/10.0.2.100/tcp/20000/p2p/12D3KooWN6JkBhFmpZ3Nc1pZRcWkBko7LhUJgYL9zzSczFfJkLib",
        "/ip4/10.0.2.101/tcp/20000/p2p/12D3KooWSvgJkT87fbXY8wsUjKULxeVC6tLXfxWGbtkCKtEAecxU",  
        "/ip4/10.0.2.102/tcp/20000/p2p/12D3KooWGZ4yeomjoPyDap1qhozddwWMLDzm1utwVg4DsoLRArfm",  
        "/ip4/10.0.2.103/tcp/20000/p2p/12D3KooW9rWEP9tB2brz8qdB1VaRbYceL88hqQfshkQSJWPC7Tx4",  
        "/ip4/10.0.2.104/tcp/20000/p2p/12D3KooWM25EyyZtc9HG5UFNcMjS23EZmgGmeH5YUmVvT2fjZkLn",  
        "/ip4/10.0.2.105/tcp/20000/p2p/12D3KooWDjqFRdcYpVu1Jy3QW93oruHJG9usXePpvbX8hTsy3Mct",  
        "/ip4/10.0.2.106/tcp/20000/p2p/12D3KooWKHVtT3SbJ3cmxphexW8VeZW4Q1PZYAc5deMLW7hcSvzC",  
        "/ip4/10.0.2.107/tcp/20000/p2p/12D3KooWJ8Q7QaH8Mdr8SRRky88HW3cB9vMmrPwNdnDRYKvGvX4x"  
    ],  
    "bootnodes": [  
        "/dns4/mainnet-bootnode-10.spacemesh.network/tcp/5000/p2p/12D3KooWHK5m83sNj2eNMJMGAngcS9gBja27ho83t79Q2CD4iRjQ",  
        "/dns4/mainnet-bootnode-11.spacemesh.network/tcp/5000/p2p/12D3KooWFrCDS8tc29nxJEYf4sKFXhXw7wMSdhQP4S7tsbfh6ngn",  
        "/dns4/mainnet-bootnode-12.spacemesh.network/tcp/5000/p2p/12D3KooWG4gk8GtMsAjYxHtbNC7oEoBTMRLbLDpKgSQMQkYBFRsw",  
        "/dns4/mainnet-bootnode-13.spacemesh.network/tcp/5000/p2p/12D3KooWPfWYFAkvB5SoqntFr1FMv41U5gqsgx1xoa2EzYuZiSQr",  
        "/dns4/mainnet-bootnode-14.spacemesh.network/tcp/5000/p2p/12D3KooWRkZMjGNrQfRyeKQC9U58cUwAfyQMtjNsupixkBFag8AY",  
        "/dns4/mainnet-bootnode-15.spacemesh.network/tcp/5000/p2p/12D3KooWSZyCMiphAZeVyKv9CGsrULT3nb3TdryjLxGwrjTrU3Ni",  
        "/dns4/mainnet-bootnode-16.spacemesh.network/tcp/5000/p2p/12D3KooWDAFRuFrMNgVQMDy8cgD71GLtPyYyfQzFxMZr2yUBgjHK",  
        "/dns4/mainnet-bootnode-17.spacemesh.network/tcp/5000/p2p/12D3KooWEZ4XzrSMgUyu1xZGVXPYhEXQWAtfaxDFaxQtWsx7SUGn",  
        "/dns4/mainnet-bootnode-18.spacemesh.network/tcp/5000/p2p/12D3KooWMJmdfwxDctuGGoTYJD8Wj9jubQBbPfrgrzzXaQ1RTKE6",  
        "/dns4/mainnet-bootnode-19.spacemesh.network/tcp/5000/p2p/12D3KooWQwpnXv96RP2rdiboJdcEfWSbjMGsf5kwpoia6jxyjWe9"  
    ],  
    "min-peers": 30,  
    "low-peers": 40,  
    "high-peers": 50,  
    "inbound-fraction": 1.1,  
    "outbound-fraction": 1.1  
  },  
```

**Private**  
```
  "p2p": {  
    "disable-reuseport": false,  
    "p2p-disable-legacy-discovery": true,  
        "disable-dht": true,  
        "direct": [  
            "/ip4/10.0.2.100/tcp/20000/p2p/12D3KooWN6JkBhFmpZ3Nc1pZRcWkBko7LhUJgYL9zzSczFfJkLib",  
            "/ip4/10.0.2.101/tcp/20000/p2p/12D3KooWSvgJkT87fbXY8wsUjKULxeVC6tLXfxWGbtkCKtEAecxU",  
            "/ip4/10.0.2.102/tcp/20000/p2p/12D3KooWGZ4yeomjoPyDap1qhozddwWMLDzm1utwVg4DsoLRArfm"  
    ],  
    "bootnodes": [],  
    "min-peers": 3,  
    "low-peers": 4,  
    "high-peers": 5  
  },  
```

## Tips  
Use public/private nodes. I recommend 3 public and the rest private.  
Once you have setup your 3 public nodes, you can generate multiple p2p.key in advance for your future private nodes and pre-enter them in your config.  
Keep a container with the node service disabled and no p2p.key file for cloning when your next plot is ready.  
You can completely disconnect your private nodes from the internet and your local network if all your containers are within a private lan.  
When creating a dashboard, I use ID 10347 for proxmox stats, and add spacemesh metrics with `spacemesh_connected_peers` `spacemesh_verified_layer` and `spacemesh_is_synced` which are made available by spacemesh-metrics.  
Use startup delays and order your containers to not start all at the same time and wreck the NVMes.  
