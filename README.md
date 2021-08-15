# cardano-node-setup-riscv64
How to build, setup and run Cardano Node and tools on RISC-V.

These are the steps how I got a working binary running on a HiFive Unmatched

# Prerequisites

## HiFive Unmatched


## Something that can run QEMU

## OS


### Installing Ubuntu 21.04

Installing Ubuntu [source](https://blogjawn.stufftoread.com/install-ubuntu-on-hifive-unmatched.html)


#### Preparing the SD Card


```bash
wget https://cdimage.ubuntu.com/releases/21.04/release/ubuntu-21.04-preinstalled-server-riscv64+unmatched.img.xz 
unxz ubuntu-21.04-preinstalled-server-riscv64+unmatched.img.xz
```

Flashing the Image Via Command Line
To flash the image to the SD card via the command line, run

```bash
dd if=</path/to/image.img> of=/dev/mmcblk0 bs=1M status=progress
```

Booting for the First Time

Connect to the serial console using the supplied USB cable.
```bash
sudo screen /dev/ttyUSB1 115200
```

Installing Ubuntu to an NVMe drive

```bash
wget http://cdimage.ubuntu.com/ubuntu/releases/21.04/release/ubuntu-21.04-preinstalled-server-riscv64+unmatched.img.xz 
unxz /ubuntu-21.04-preinstalled-server-riscv64+unmatched.img.xz
```

Make sure the NVMe drive is present by running

```bash
ls -l /dev/nvme*
```

```bash
sudo dd if=/ubuntu-21.04-preinstalled-server-riscv64+unmatched.img of=/dev/nvme0n1 bs=1M status=progress
sudo mount /dev/nvme0n1p1 /mnt
sudo chroot /mnt
U_BOOT_ROOT="root=/dev/nvme0n1p1"
u-boot-update
exit
reboot
```


# GHC compiler

```bash
git clone git@github.com:RV64-Stake-Pool/ghc.git
cd ghc
git checkout ghc-8.10.4-release-risc5-patches
```

# cabal-install

# Building the cardano-node

Make sure the patched compiler is in the path, for the rest you can pretty much follow the instructions [here](https://www.coincashew.com/coins/overview-ada/guide-how-to-build-a-haskell-stakepool-node).

Compilation can take a while so make sure to build with `-j4` option.

```bash
$ cabal build -j4
```

## TODO

* How to build a static only version in order to reduce compile time.


# Running the cardano-node


Follow the [instructions](https://www.coincashew.com/coins/overview-ada/guide-how-to-build-a-haskell-stakepool-node) available online.

In section 7. Create startup scripts](https://www.coincashew.com/coins/overview-ada/guide-how-to-build-a-haskell-stakepool-node#7-create-startup-scripts) add the `RTS` options as shown below in `startBlockProducingNode.sh` and `startRelayNode1.sh`:

```bash
/usr/local/bin/cardano-node run --topology ...
```

```bash
/usr/local/bin/cardano-node +RTS -N3 --disable-delayed-os-memory-return -I0.3 -Iw600 -A32m -F1.5 -H3000M -T -S -RTS run --topology ...
```

## Monitoring & Alerting

### Prometheus


This prometheus configuration can scrape both nodes on mainnet as testnet. The net is added as a label so we can have one dashboard in grafana.

`/etc/prometheus/prometheus.yaml`
```yaml
# Sample config for Prometheus.

global:
  scrape_interval:     15s # Set the scrape interval to every 15 seconds. Default is every 1 minute.
  evaluation_interval: 15s # Evaluate rules every 15 seconds. The default is every 1 minute.
  # scrape_timeout is set to the global default (10s).

  # Attach these labels to any time series or alerts when communicating with
  # external systems (federation, remote storage, Alertmanager).
  external_labels:
      monitor: 'example'

# Alertmanager configuration
alerting:
  alertmanagers:
  - static_configs:
    - targets: ['localhost:9093']

# Load rules once and periodically evaluate them according to the global 'evaluation_interval'.
rule_files:
  - "cardano_node_rules.yml"

# Here it's Prometheus itself.
scrape_configs:
  # The job name is added as a label `job=<job_name>` to any timeseries scraped from this config.
  - job_name: 'prometheus'
    static_configs:
      - targets: ['<relay1 hostname or ip>:9100']
        labels:
          alias: 'relaynode1'
          type:  'cardano-node'
          net: 'mainnet'
      - targets: ['<relay1 hostname or ip>:9100']
        labels:
          alias: 'relaynode2'
          type:  'cardano-node'
          net: 'mainnet'
      - targets: ['<producer hostname or ip>:9100']
        labels:
          alias: 'block-producer-node'
          type:  'cardano-node'
          net: 'mainnet'
      - targets: ['<producer hostname or ip>:12798']
        labels:
          alias: 'block-producer-node'
          type:  'cardano-node'
          net: 'mainnet'
      - targets: ['rv64-relay1.internal:13798']
        labels:
          alias: 'relaynode1'
          type:  'cardano-node'
          net: 'mainnet'
      - targets: ['<relay 2 hostname or ip>:12798']
        labels:
          alias: 'relaynode2'
          type:  'cardano-node'
          net: 'mainnet'
    # Optional if you also have nodes running on testnet
      - targets: ['producer:9100']
        labels:
          alias: 'block-producer-node'
          type:  'cardano-node'
          net: 'testnet'
      # The test server runs both a relay and producer on one machine so we need two prometheus ports, 13798 for producer and the default 12798 for the relay
      - targets: ['<test server hostname or ip>:13798']
        labels:
          alias: 'block-producer-node'
          type:  'cardano-node'
          net: 'testnet'
      - targets: ['<test server hostname or ip>:12798']
        labels:
          alias: 'relaynode1'
          type:  'cardano-node'
          net: 'testnet'

```

### Alertmanager rules

Here as some examples of alert rules to monitor your pool. 

File: `/etc/prometheus/cardano_node_rules.yml`

```yaml
groups:
- name: Hardware alerts
  rules:
  - alert: Node down
    expr: up{net="mainnet",instance=~".*:9100"} == 0
    for: 3m
    labels:
      severity: critical
    annotations:
      title: Node {{ $labels.alias }} is down
      description: Failed to scrape {{ $labels.job }} on {{ $labels.instance }} for more than 3 minutes. Node seems down.
- name: Cardano Node
  rules:
  - alert: Cardano Node down
    expr: up{net="mainnet",instance=~".*:1[23]798"} == 0
    for: 3m
    labels:
      severity: critical
    annotations:
      title: Cardano Node {{ $labels.alias }} is down
      description: Failed to scrape {{ $labels.job }} on {{ $labels.instance }} for more than 3 minutes. Node seems down.
  - alert: Relay peer connections low
    expr: cardano_node_metrics_connectedPeers_int{alias=~"^relaynode.*",net="mainnet"} < 10
    for: 3m
    labels:
      severity: critical
    annotations:
      title: Cardano relay node {{ $labels.alias }} connected peers is low
      description: Relay node {{ $labels.alias }} is only connected to {{ $value }} peers.
  - alert: Block producer not conneted to all relays
    expr: cardano_node_metrics_connectedPeers_int{alias="block-producer-node",net="mainnet"} < 2
    for: 3m
    labels:
      severity: critical
    annotations:
      title: Cardano block producer node {{ $labels.alias }} connected peers is low
      description: Block producer node {{ $labels.alias }} is only connected to {{ $value }} relays.
```

### Grafana dashboard

The goal is to have all the info that is avaialble in `gLiveView`

Install grafana as documented by grafana.
Install additional plugins listed below.

```bash
  grafana-cli plugins install simpod-json-datasource
  grafana-cli plugins install grafana-clock-panel
  grafana-cli plugins install grafana-worldmap-panel
  systemctl restart grafana-server
```

Create a datasource for prometheus.
Import the [dashboard](grafana-dashboards/cardano-node.json).
The dashboard is based on the dashboard created by SNSKY, the following extensions are made:
* Selection of testnet and mainnet
* GHC garbage collection
* CPU Temperature
* Worldmap to show peers (work in progress)

#### TODO's

* Build riscv64 of grafana
* Put peers on the worldmap

  
## Download pre-compiled binaries

Cabal and GHC are only needed for development/compiling new releases.

* [Patched GHC 8.10.4](https://ipfs.io/ipfs/QmXSSxRZK9mPvgLxR5vuKm5gPBiy6NmKTcpUk5K4FXx1Tm?filename=ghc-8.10.4-patched.tar.gz)
* [Cabal 3.4.0.0](https://ipfs.io/ipfs/QmT94BAFioQ5XL9Foytk2QHHLtpfD2kUQ8Y958cg4Dc2Ki?filename=cabal-3.4.0.0-riscv64.tgz)
* [libsodium](https://ipfs.io/ipfs/QmXLkhEfs5fHyows7xZARiNxGMSpRP3uj25SQL5Kpk3srN?filename=libsodium.tgz)
* [Cardano Node 1.28.0](https://ipfs.io/ipfs/QmUxY1VFhWPpTsK8eGfBrHQTW6EqfgfAJxYgSFK1hTyFh9?filename=cardano-1.28.0.tgz)

