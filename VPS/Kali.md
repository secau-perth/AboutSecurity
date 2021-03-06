# Kali system infrastructure configuration.

---

- The penetration tool is placed in /pentest
- Residual files are placed in /tmp/test
- By default, all commands are passed through proxychains4

---

# Basic installation (基本配置)

```bash
# /pentest 作为存放渗透工具的文件夹
# /tmp/test 作为存放临时文件的文件夹
mkdir /pentest
mkdir /tmp/test

# apt mirror
tee /etc/apt/sources.list <<-'EOF'
deb https://mirrors.aliyun.com/kali kali-rolling main non-free contrib
deb-src https://mirrors.aliyun.com/kali kali-rolling main non-free contrib
deb http://mirrors.tuna.tsinghua.edu.cn/kali kali-rolling main contrib non-free
deb-src https://mirrors.tuna.tsinghua.edu.cn/kali kali-rolling main contrib non-free
deb http://http.kali.org/kali kali-rolling main non-free contrib
deb-src http://http.kali.org/kali kali-rolling main non-free contrib
EOF

rm -rf /var/cache/apt/archives/lock
rm -rf /var/lib/dpkg/lock-frontend
rm -rf /var/lib/dpkg/lock
rm /var/lib/dpkg/lock
rm /var/lib/apt/lists/lock

apt update

# SSH
echo "PermitRootLogin yes" >> /etc/ssh/sshd_config
echo "PasswordAuthentication yes" >> /etc/ssh/sshd_config
systemctl start ssh
systemctl enable ssh
ssh-keygen -t dsa -f /etc/ssh/ssh_host_dsa_key
ssh-keygen -t dsa -f /etc/ssh/ssh_host_rsa_key

# 安装依赖 (install dependence)
apt install -y gcc g++
apt install -y make cmake
apt install -y vim git curl lrzsz wget unzip resolvconf p7zip-full rlwrap
apt install -y apt-transport-https
apt install -y ca-certificates
apt install -y software-properties-common
apt install -y build-essential
apt install -y jq
apt install -y gdb socat telnet tree tcpdump iptraf iftop nethogs openssl libssl-dev libssh2-1-dev

# Change DNS
echo "nameserver 223.5.5.5" > /etc/resolvconf/resolv.conf.d/head
resolvconf -u

# Proxychains-ng
cd /tmp/test
# rz upload a proxychains-ng.zip
unzip proxychains-ng.zip
cd proxychains-ng-master
./configure
make && make install
cp ./src/proxychains.conf /etc/proxychains.conf
cd .. && rm -rf proxychains-ng
vim /etc/proxychains.conf    # Change it to your proxy.

# pip
wget https://bootstrap.pypa.io/get-pip.py
python2 get-pip.py
python2 -m pip install --upgrade pip
apt-get install -y python-dev
apt-get install -y python3-dev

# 安装中文字体 (Install fonts)
apt install -y xfonts-intl-chinese ttf-wqy-microhei ttf-wqy-zenhei xfonts-wqy

# pip3
cd /tmp/test
wget https://bootstrap.pypa.io/get-pip.py
python3 get-pip.py

# GO
cd /tmp/test
# rz upload a go1.13.linux-amd64.tar.gz
tar -C /usr/local -xzf go1.13.linux-amd64.tar.gz
export PATH=$PATH:/usr/local/go/bin
export GOROOT=/usr/local/go
export GOPATH=$HOME/Applications/Go
source $HOME/.profile
source ~/.bash_profile

ln -s /usr/local/go/bin/go /usr/bin/go
go version

# Docker
apt remove docker docker-engine docker.io
apt update
apt install -y \
  apt-transport-https \
  ca-certificates \
  curl \
  software-properties-common \
  gnupg

curl -fsSL https://mirrors.ustc.edu.cn/docker-ce/linux/ubuntu/gpg | apt-key add -
echo 'deb https://download.docker.com/linux/debian stretch stable'> /etc/apt/sources.list.d/docker.list
apt update
apt install -y docker-ce

docker version
systemctl start docker
systemctl enable docker

# Docker-Compose
cd /tmp/test
wget https://github.com/docker/compose/releases/download/1.25.5/docker-compose-Linux-x86_64
mv docker-compose-Linux-x86_64 /usr/local/bin/docker-compose
chmod +x /usr/local/bin/docker-compose
docker-compose version
```

---

# Change mirrors (换源)

> If your service does not go through any firewall, you know, this step is not needed

```bash
# pip mirrors
mkdir -p ~/.pip/
tee ~/.pip/pip.conf <<-'EOF'
[global]
index-url = https://mirrors.aliyun.com/pypi/simple/

[install]
trusted-host=mirrors.aliyun.com
EOF

# Docker mirrors
mkdir -p /etc/docker
tee /etc/docker/daemon.json <<-'EOF'
{
  "registry-mirrors": ["https://docker.mirrors.ustc.edu.cn"]
}
EOF
systemctl enable docker
systemctl restart docker
systemctl daemon-reload

# GO Proxy
go env -w GO111MODULE=on
go env -w GOPROXY="https://goproxy.io,direct"
```

---

# Security tool installation (安全工具安装配置)

```bash
mkdir /tmp/test
cd /tmp/test

# AboutSecurity
cd /pentest
git clone https://github.com/ffffffff0x/AboutSecurity.git

# Misc Tools
apt install -y foremost parallel owasp-mantra-ff btscanner
apt install -y rarcrack
apt install -y powershell
apt install -y redis
apt install -y xdot
gem install zsteg

# python module
pip install PyJWT pyshark requests sqlparse threadpool urllib3 lxml pyzbar bs4
pip install ftfy
pip3 install updog
python2 -m pip install yara pycrypto openpyxl ujson pil
python2 -m pip install Crypto
python2 -m pip install pycryptodome
python2 -m pip install pytz
python2 -m pip install Pillow
python3 -m pip install ciphey --upgrade

# distorm
cd /tmp/test
git clone https://github.com/gdabah/distorm
cd distorm
python2 -m pip install distorm3

# Volatility
cd /pentest
git clone https://github.com/volatilityfoundation/volatility.git
cd volatility
python setup.py build
python setup.py install
python vol.py --info

# hashcat、7z2hashcat
cd /pentest
wget https://hashcat.net/files/hashcat-6.1.1.7z --no-check-certificate
wget https://raw.githubusercontent.com/philsmd/7z2hashcat/master/7z2hashcat.pl
7za x hashcat-6.1.1.7z && rm -rf hashcat-6.1.1.7z
cd hashcat-6.1.1 && chmod +x hashcat.bin && cp hashcat.bin hashcat
ln -s /pentest/hashcat-6.1.1/hashcat /usr/sbin/hashcat

# RustScan
cd /tmp/test
wget https://github.com/RustScan/RustScan/releases/download/1.10.0/rustscan_1.10.0_amd64.deb
dpkg -i rustscan_1.10.0_amd64.deb

# ncat
apt install -y ncat
update-alternatives --set nc /usr/bin/ncat

# binwalk
cd /tmp/test
git clone https://github.com/ReFirmLabs/binwalk.git && cd binwalk
sudo ./deps.sh && python setup.py uninstall && python setup.py install

# unyaffs
cd /tmp/test
wget https://storage.googleapis.com/google-code-archive-downloads/v2/code.google.com/unyaffs/unyaffs
mv unyaffs /usr/local/bin/
chmod +x /usr/local/bin/unyaffs
unyaffs

# ffuf
cd /tmp/test
wget https://github.com/ffuf/ffuf/releases/download/v1.1.0/ffuf_1.1.0_linux_amd64.tar.gz
tar -zxvf ffuf_1.1.0_linux_amd64.tar.gz
mv ffuf /usr/local/bin/
chmod +x /usr/local/bin/ffuf
ffuf -V

# JSFinder
cd /pentest
git clone https://github.com/Threezh1/JSFinder.git

# SecretFinder
cd /pentest
git clone https://github.com/m4ll0k/SecretFinder.git
cd SecretFinder
python3 -m pip install -r requirements.txt
python3 SecretFinder.py

# WebAliveScan
cd /pentest
git clone https://github.com/broken5/WebAliveScan.git
cd WebAliveScan/
pip3 install -r requirements.txt -i https://mirrors.aliyun.com/pypi/simple/
python3 webscan.py

# oneforall
cd /pentest
git clone https://github.com/shmilylty/OneForAll
cd OneForAll/
python3 -m pip install -U pip setuptools wheel -i https://mirrors.aliyun.com/pypi/simple/
pip3 install -r requirements.txt -i https://mirrors.aliyun.com/pypi/simple/
python3 oneforall.py --help

# ksubdomain
cd /tmp/test
apt install -y libpcap-dev
wget https://github.com/knownsec/ksubdomain/releases/download/v0.5.1/ksubdomain_linux.zip
unzip ksubdomain_linux.zip
mv ksubdomain /usr/local/bin/
chmod +x /usr/local/bin/ksubdomain
ksubdomain

# Impacket
cd /pentest
git clone https://github.com/SecureAuthCorp/impacket.git
cd impacket
pip3 install .

# AWVS-13
docker pull secfa/docker-awvs
docker run -it -d -p 13443:3443 secfa/docker-awvs
    # 容器的相关信息 (Information about the container)
    # awvs13 username: admin@admin.com
    # awvs13 password: Admin123
    # AWVS Version：13.0.200217097
    # browser access：https://127.0.0.1:13443/

# AWVS13+nessus
docker pull leishianquan/awvs-nessus:v03
docker run -it -d -p 13443:3443 -p 8834:8834 leishianquan/awvs-nessus:v03 /bin/bash
docker ps -a
docker start [docker_id]
docker exec -it [docker_id] /bin/bash
/etc/init.d/nessusd start
cp /home/license_info.json /home/acunetix/.acunetix/data/license/
  # Nessus:
  # https://127.0.0.1:8834/#/
  # nessus username:leishi
  # nessus password:leishianquan

  # Awvs13.0.200625101：
  # https://127.0.0.1:13443/
  # awvs13 username: leishi@leishi.com
  # awvs13 password: Leishi123
```
