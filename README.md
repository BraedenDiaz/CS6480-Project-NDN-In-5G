# CS6480-Project-NDN-In-5G

Project for CS6480 at the University of Utah.

Refer to the references at the bottom for links to the free5gc and UERANSIM repositories if needed.

### Prerequisites

1. Create a network setup with one or more machines running Ubuntu 18 with kernel 5.0.0-23-generic.

For my project, I am using the [PowderWireless Platform](https://powderwireless.net) ran by the FLUX research group at the University of Utah but these instructions will work with any other valid setup.

For the following instructions, I am using a setup containing the following five nodes. Please refer to example 2 in the [free5gc wiki] (https://github.com/free5gc/free5gc/wiki/SMF-Config) to see what the nodes refer to visually.

I have added the IP addresses next to the name of the nodes below in my specific setup which you can refer to in the configurations, however, be sure to use the correct addresses for your setup.

- `free5gc (10.10.1.2)` node which will contain free5gc, the 5G core.
- `sim-ran (10.10.1.1)` node which will contain UERANSIM used as the simulated external Radio Accesss Network (RAN/gNB) and User Equipment (UE).
- `upfb (10.10.1.3)` node which acts as a UPF branching point.
- `upf1 (10.10.1.4)` node which acts as a UPF anchor.
- `upf2 (10.10.1.5)` node which acts as another UPF anchor.

## Install free5gc on the `free5gc` node and on all 3 `UPF` nodes

Unless stated otherwise, perform the commands in this section for the `free5gc` node and all 3 `UPF` nodes.

1. Install gtp5g Linux module

```bash
git clone -b v0.2.0 https://github.com/PrinzOwO/gtp5g.git
cd gtp5g
make
sudo make install
```

2. Install Golang Version 1.14.4

```bash
wget https://dl.google.com/go/go1.14.4.linux-amd64.tar.gz
sudo tar -C /usr/local -zxvf go1.14.4.linux-amd64.tar.gz
mkdir -p ~/go/{bin,pkg,src}

echo 'export GOPATH=$HOME/go' >> ~/.bashrc
echo 'export GOROOT=/usr/local/go' >> ~/.bashrc
echo 'export PATH=$PATH:$GOPATH/bin:$GOROOT/bin' >> ~/.bashrc
source ~/.bashrc
```
### Install free5gc Dependencies

3. Install Control-Plane Supporting Packagaes. Not required for the UPF nodes.

```bash
sudo apt -y update
sudo apt -y install mongodb wget git
sudo systemctl start mongodb
```

4. Install User-Plane Supporting packages

```bash
sudo apt -y update
sudo apt -y install git gcc cmake autoconf libtool pkg-config libmnl-dev libyaml-dev
go get -u github.com/sirupsen/logrus
```
### Configure Linux Network Settings

5. Enable IP forwarding on Linux

```bash
sudo sysctl -w net.ipv4.ip_forward=1
sudo iptables -A FORWARD -j ACCEPT
```

6. Configure NAT to allow UEs connected to access the Internet. The file /var/emulab/boot/controlif contains the name of the internet-facing "control network" device.

```bash
sudo iptables -t nat -A POSTROUTING -o `cat /var/emulab/boot/controlif` -j MASQUERADE
```

7. Stop firewall service so it doesn't interfere with anything.

```
sudo systemctl stop ufw
```

### Install free5gc

8. Clone the free5gc repository.

```bash
cd ~
git clone --recursive -b v3.0.4 -j `nproc` https://github.com/free5gc/free5gc.git
```

9. Install all free5gc Golang module dependencies.

```bash
cd ~/free5gc
go mod download
```

10. Compile free5gc network function services (AMF, SMF, etc)

`free5gc` node:
```bash
make all
```

On all 3 `UPF` nodes:
```bash
cd ~/free5gc
make upf
```

If you accidentially compiled all functions on the UPF nodes, that's fine as you can just run the UPF funtion on those nodes by itself.

11. Lastly, you can test free5gc by running the UPF function on the UPF nodes and then running the entire free5gc core all-in-one.

On all 3 `UPF` nodes:
```bash
cd ~/free5gc/src/upf/build
sudo -E ./bin/free5gc-upfd
```

`free5gc` node:
```bash
cd ~/free5gc
./run.sh
```

## Install UERANSIM on the `sim-ran` node

Note: I use the same kernel version 5.0.0-23-generic for this PowderWireless node as well just for consistency.

1. Install the UERANSIM dependencies.

```bash
sudo apt update
sudo apt upgrade
sudo apt install make
sudo apt install g++
sudo apt install openjdk-11-jdk
sudo apt install maven
sudo apt install libsctp-dev lksctp-tools
```

2. Set the `JAVA_HOME` environment variable.

```bash
export JAVA_HOME=/usr/lib/jvm/java-11-openjdk-amd64
```

3. Clone UERANSIM.

```bash
cd ~
git clone https://github.com/aligungr/UERANSIM.git
```

4. Make the UERANSIM scripts executable

```bash
cd ~/UERANSIM
chmod 700 build.sh run.sh
```

5. Build UERANSIM
```bash
./build.sh
```

## Configure free5gc

1. Open `free5gc/config/amfcfg.conf` and change the `ngapIpList` value located right below the `configuration` section from `127.0.0.1` to the IP of the free5gc machine.

```yaml
configuration:
  amfName: AMF
  ngapIpList:
    - 10.10.1.2
```

2. In `free5gc/config/smfcfg.conf`, configure the `userplane_information` and `links` sections based on your setup. Again, I'm using example 2 provided in the [free5gc wiki](https://github.com/free5gc/free5gc/wiki/SMF-Config).
```yaml
  pfcp:
    addr: 10.10.1.2
  userplane_information:
    up_nodes:
      gNB1:
        type: AN
        an_ip: 10.10.1.1
      BranchingUPF:
        type: UPF
        node_id: 10.10.1.3
      AnchorUPF1:
        type: UPF
        node_id: 10.10.1.4
      AnchorUPF2:
        type: UPF
        node_id: 10.10.1.5
    links:
      - A: gNB1
        B: BranchingUPF
      - A: BranchingUPF
        B: AnchorUPF1
      - A: BranchingUPF
        B: AnchorUPF2
```

3. Still in `smfcfg.conf`, add `ulcl: true` at the very bottom.

```yaml
  nrfUri: http://nrf.free5gc.org:29510 # <-- This already exists
  ulcl: true
```

4. Configure the `pfcp` and `gtpu` addresses in `free5gc/src/upf/build/config/upfcfg.yaml` on each `UPF` node you have.

In my case, I have a UPF branching point with two UPF anchors, so my config would be the following:

Branching UPF `upfcfg.yaml`:
```yaml
  pfcp:
    - addr: 10.10.1.3

  gtpu:
    - addr: 10.10.1.3
```

Anchor UPF 1 `upfcfg.yaml`:
```yaml
  pfcp:
    - addr: 10.10.1.4

  gtpu:
    - addr: 10.10.1.4
```

Anchor UPF 2 `upfcfg.yaml`:
```yaml
  pfcp:
    - addr: 10.10.1.5

  gtpu:
    - addr: 10.10.1.5
```

5. Finally, configure `free5gc/config/uerouting.yaml`.

The file already has other existing UEs, but I add the followng one where the `AN` value is set to the IP address of the UPF anchor point I want the following UE to use to access the data network. Also where the `SUPI` value also matches the default one added in the free5gc web console when you add a new subscriber (shown below later when running the free5gc webconsole).

```yaml
  - SUPI: imsi-2089300007487
    AN: 10.10.1.4 # Use UPF anchor 1
    PathList:
      - DestinationIP: 60.60.0.101
        UPF: !!seq
          - BranchingUPF
          - AnchorUPF1

      - DestinationIP: 60.60.0.103
        UPF: !!seq
          - BranchingUPF
          - AnchorUPF2
```

## Configure UERANSIM

1. In the file `UERANSIM/config/profile.yaml` change the `selected-profile:` value to `free5gc`.

```yaml
selected-profile: 'free5gc'
```

2. In the file `UERANSIM/config/free5gc/gnb.yaml`, If you're running UERANSIM on another machine, change the `host: 127.0.0.1` to IP address of the interface on the same network as the free5gc core.

```yaml
host: 10.10.1.1
```

3. Still in the same file `gnb.yaml`under the `amfConfigs` section, change the host value to the host of the free5gc node where the AMF is located.

```yaml
amfConfigs:
  - guami:
      mcc: 208
      mnc: 93
      amfRegionId:
        hex: '2a'
      amfSetId:
        hex: '5580'
      amfPointer:
        hex: 'a8'
    host: 10.10.1.2
    port: 38412
```

## Run free5gc and UERANSIM

### Prerequisites

### Add a UE to the database.

1. On the `free5gc` node, start the web server.
 
```bash
cd ~/free5gc/webconsole
go run server.go
```

2. Go to the website hosting the database interface.

The server runs on port 5000 by default. You whatever your machine name is or use the ecternal IP address of your machine.

```
http://user@pc750.emulab.net:5000
```

3. Login to the interface

```
Username: Admin
Password: free5gc
```

4. Go to the `subscribers` tab and click on `add`.

5. Change the USIM Type from `OPc` to `Op`

```
USIM Type: Op
```

Make sure the other UE values (IMSI, key, op, etc) match the UE values found in the UERANSIM configuration `UERANSIM/config/free5gc/ue.yaml`.

Note: The free5gc web interface should already populate a UE with default values that are the same as the UE in the UERANSIM config, so I didn't have to
change anything else except the `USIM Type`.

6. Add the UE

7. Logout of the database web interface and shutdown the web server.

## Run free5gc and UERANSIM

1. On all 3 `UPF` nodes, start the UPF function.

```bash
cd ~/free5gc/src/upf/build
sudo -E ./bin/free5gc-upfd
```

2. On the `free5gc` node, start the free5gc core all-in-one.

```bash
cd ~/free5gc
./run.sh
```

3. On the `sim-ran` node, start the UERANSIM.

```bash
cd ~/UERANSIM
./run.sh
```

At this point, you can use the UERANSIM interface to register, deregister, and connect to the data network over the free5gc core. I have yet to figure out how to do other things such as run an application over the UERANSIM or perform other mobile operations. Refer to the next section below.

# Challenges

- Have yet to get an example application running over the UERANSIM and using the free5gc core.

# References

- [PowderWireless Platform](https://powderwireless.net)
- [free5gc Repository](https://github.com/free5gc/free5gc)
- [UERANSIM Repository](https://github.com/aligungr/UERANSIM)
