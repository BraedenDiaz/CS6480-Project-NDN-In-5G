# CS6480-Project-NDN-In-5G

## Install free5gc on the `free5gc` node

### Prerequisites

0. Spin up a PowderWireless node using a linux profile with the kernel version 5.0.0-23-generic.

Note: You can also set up two connected nodes if you plan on using an external Radio Access Network (RAN) with free5gc as well. Please refer to my example profiles linked below.

[My Example PowderWireless Profiles](/PowderWireless%20Profiles)

For the following instructions, I am using my [end-to-end](/PowderWireless%20Profiles/free5gc-end-to-end-d430.py) profile containing the following two nodes.

- `free5gc` node which will contain free5gc.
- `sim-ran` node which will contain UERANSIM.

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

3. Install Control-Plane Supporting Packagaes

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
```

6. Configure NAT to allow UEs connected to access the Internet. The file /var/emulab/boot/controlif contains the name of the internet-facing "control network" device.

```bash
sudo iptables -t nat -A POSTROUTING -o `cat /var/emulab/boot/controlif` -j MASQUERADE
sudo systemctl stop ufw
```

### Install free5gc

7. Clone the free5gc repository.

```bash
cd ~
git clone --recursive -b v3.0.4 -j `nproc` https://github.com/free5gc/free5gc.git
cd free5gc
```

8. Install all free5gc Golang module dependencies.

```bash
cd ~/free5gc
go mod download
```

9. Compile all free5gc network function services (AMF, SMF, UPF, etc)

```bash
cd ~/free5gc
make all
```

10. Lastly, run the entire free5gc core all-in-one.

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
export JAVA_HOME=<path to the jdk 11 just installed>
```

3. Build UERANSIM.

```bash
cd ~/UERANSIM
./build.sh
```

Note: You may have to make `build.sh` executable.

```bash
chmod 700 build.sh
```

## Configure free5gc

1. Open `free5gc/config/amfcfg.conf` and change the `ngapIpList` value located right below the `configuration` section from `127.0.0.1` to the IP of your machine `10.10.1.1`.

```yaml
configuration:
  amfName: AMF
  ngapIpList:
    - 10.10.1.1
```

2. Still in `amfcfg.conf`, next change the port number under the `sbi` section to the port number UERANSIM uses `38412`

```yaml
  sbi:
    scheme: http
    registerIPv4: 127.0.0.1 # IP used to register to NRF
    bindingIPv4: 127.0.0.1  # IP used to bind the service
    port: 38412
```

## Configure UERANSIM

1. In the file `UERANSIM/config/profile.yaml` change the `selected-profile:` value to `free5gc`.

```yaml
selected-profile: 'free5gc'
```

2. In the file `UERANSIM/config/free5gc/gnb.yaml`under the `amfConfigs` section, change the host and port to the host and port of your free5gc node (where the AMF is located).

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
    host: 10.10.1.1 # Chnage this
    port: 38412
```
NOTE: Use the default port number as shown in the `gnb.yaml` file which should be 38412 as when I changed it, it wouldn't connect to the free5gc core even when I also changed the port number in the AMF configuration as well, so just leave this as port 38412 and only change the AMF port so that it is 38412.

## Run free5gc and UERANSIM

### Prerequisites

### Add a UE to the database.

1. On the `free5gc` node, start the web server.
 
```bash
cd ~/free5gc/webconsole
go run server.go
```

2. Go to the website hosting the database interface.

The server runs on port 5000 by default.

```
http://user@pc750.emulab.net:5000
```

3. Login to the interface

```
Username: Admin
Password: free5gc
```

4. Go to the `subscribers` tab and click on `add`.

5. Change the USIM Type to `Op` instad of `OPc`

```
USIM Type: OPc
```

Make sure the other UE values (IMSI, key, op, etc) match the UE values found in the UERANSIM configuration `UERANSIM/config/free5gc/ue.yaml`.

Note: The free5gc web interface should already populate a UE with default values that are the same as the UE in the UERANSIM config, so I didn't have to
change anything else except the `USIM Type`.

6. Add the UE

7. Logout of the database web interface and shutdown the web server.

## Run free5gc and UERANSIM

1. On the `free5gc` node, start the free5gc core all-in-one.

```bash
cd ~/free5gc
./run.sh
```

2. On the `sim-ran` node, start the UERANSIM.

```bash
cd ~/UERANSIM
./run.sh
```

Note: You may have to make `run.sh` executable.

```bash
chmod 700 run.sh
```

At this point, you can use the UERANSIM interface to register, deregister, and connect to the data network over the free5gc core. I have yet to figure out how to do other things such as run an application over the UERANSIM or perform other mobile operations. Refer to the next section below.

# In Progress

- Working on getting an example application running over the UERANSIM and using the free5gc core.
- Using the test files provided in free5gc to see how they get N3IWF-based applications working and how they trigger scenraios like handoff, UE simulation, etc.

# References

- [free5gc Repository](https://github.com/free5gc/free5gc)
- [UERANSIM Repositoru](https://github.com/aligungr/UERANSIM)
