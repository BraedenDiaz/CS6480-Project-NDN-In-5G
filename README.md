# CS6480-Project-NDN-In-5G

## Install free5gc on a PowderWireless node

### Prerequisites

0. Spin up a PowderWireless node using a linux profile with the kernel version 5.0.0-23-generic.

Note: You can also set up two connected nodes if you plan on using an external Radio Access Network (RAN) with free5gc as well. Please refer to my example profiles linked below.

[My Example PowderWireless Profiles](/PowderWireless%20Profiles)

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
