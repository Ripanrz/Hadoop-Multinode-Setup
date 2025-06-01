# Hadoop-Multinode-Setup
# Panduan Lengkap Instalasi dan Konfigurasi Hadoop 3.3.6 Multi-Node di Komputer Fisik (Master dan Slave)

## 1. Persiapan Awal
### a. Instalasi Sistem Operasi
Instal sistem operasi Ubuntu Server atau Ubuntu Desktop di setiap komputer (1 master, 2 slave). Pastikan semua komputer:
- Terhubung dalam jaringan LAN yang sama
- Memiliki alamat IP statis

### b. Buat user khusus untuk Hadoop
Login sebagai root atau user utama, lalu buat user `hduser`:
```bash
sudo adduser hduser
```

### c. Tambahkan user ke grup `hadoop` dan beri hak sudo
```bash
sudo addgroup hadoop
sudo usermod -aG hadoop hduser
sudo usermod -aG sudo hduser
```

## 2. Atur Hostname dan /etc/hosts
### a. Ubah hostname
Lakukan di masing-masing komputer:
```bash
# Di Master
sudo hostnamectl set-hostname master

# Di Slave 1
sudo hostnamectl set-hostname slave1

# Di Slave 2
sudo hostnamectl set-hostname slave2
```

### b. Tambahkan alamat IP dan hostname ke file /etc/hosts
```bash
sudo nano /etc/hosts
```
Tambahkan baris berikut (ubah sesuai IP masing-masing):
```
192.168.1.10 master
192.168.1.11 slave1
192.168.1.12 slave2
```

Restart semua komputer:
```bash
sudo reboot
```

## 3. Instalasi Java 11
Login sebagai `hduser` di setiap komputer:
```bash
sudo apt update
sudo apt install openjdk-11-jdk -y
java -version
```

## 4. Konfigurasi SSH
Masih sebagai `hduser`, lakukan:
```bash
ssh-keygen -t rsa -P ""
cat ~/.ssh/id_rsa.pub >> ~/.ssh/authorized_keys
ssh localhost
```

### a. Salin key dari master ke semua slave (lakukan hanya di master):
```bash
ssh-copy-id slave1
ssh-copy-id slave2
```
Uji koneksi tanpa password:
```bash
ssh slave1
ssh slave2
```

## 5. Instalasi Hadoop (lakukan hanya di Master)
```bash
wget https://dlcdn.apache.org/hadoop/common/hadoop-3.3.6/hadoop-3.3.6.tar.gz
tar -xvzf hadoop-3.3.6.tar.gz
mv hadoop-3.3.6 ~/hadoop
```

## 6. Konfigurasi Hadoop (di Master)
### a. core-site.xml
```xml
<configuration>
  <property>
    <name>fs.defaultFS</name>
    <value>hdfs://master:9000</value>
  </property>
</configuration>
```

### b. hdfs-site.xml
```xml
<configuration>
  <property>
    <name>dfs.replication</name>
    <value>2</value>
  </property>
  <property>
    <name>dfs.namenode.name.dir</name>
    <value>file:///home/hduser/hadoop/hdfs/namenode</value>
  </property>
  <property>
    <name>dfs.datanode.data.dir</name>
    <value>file:///home/hduser/hadoop/hdfs/datanode</value>
  </property>
</configuration>
```

### c. mapred-site.xml
```xml
<configuration>
  <property>
    <name>mapreduce.framework.name</name>
    <value>yarn</value>
  </property>
</configuration>
```

### d. yarn-site.xml
```xml
<configuration>
  <property>
    <name>yarn.nodemanager.aux-services</name>
    <value>mapreduce_shuffle</value>
  </property>
  <property>
    <name>yarn.resourcemanager.hostname</name>
    <value>master</value>
  </property>
</configuration>
```

### e. hadoop-env.sh
Tambahkan di bagian akhir:
```bash
export JAVA_HOME=/usr/lib/jvm/java-11-openjdk-amd64
```

### f. worker
```bash
nano ~/hadoop/etc/hadoop/workers
```
Isi:
```
slave1
slave2
```

## 7. Salin Hadoop ke Slave
```bash
scp -r ~/hadoop slave1:/home/hduser/
scp -r ~/hadoop slave2:/home/hduser/
```

## 8. Buat direktori HDFS
### Di Master:
```bash
mkdir -p ~/hadoop/hdfs/namenode
mkdir -p ~/hadoop/hdfs/datanode
```
### Di Slave:
***
mkdir -p ~/hadoop/hdfs/datanode
***

## 9. Atur environment di .bashrc (semua node)
---
nano ~/.bashrc
---
Tambahkan di bawah:
```bash
export HADOOP_HOME=$HOME/hadoop
export HADOOP_INSTALL=$HADOOP_HOME
export HADOOP_MAPRED_HOME=$HADOOP_HOME
export HADOOP_COMMON_HOME=$HADOOP_HOME
export HADOOP_HDFS_HOME=$HADOOP_HOME
export YARN_HOME=$HADOOP_HOME
export HADOOP_COMMON_LIB_NATIVE_DIR=$HADOOP_HOME/lib/native
export HADOOP_CONF_DIR=$HADOOP_HOME/etc/hadoop
export PATH=$PATH:$HADOOP_HOME/sbin:$HADOOP_HOME/bin
export JAVA_HOME=/usr/lib/jvm/java-11-openjdk-amd64
export PATH=$PATH:$JAVA_HOME/bin
```
Aktifkan:
```
source ~/.bashrc
```

## 10. Nonaktifkan Firewall
```
sudo ufw disable
```

## 11. Format dan Jalankan Hadoop
### Format HDFS (hanya di master)
```bash
hdfs namenode -format
```

### Start Hadoop
```
start-dfs.sh
start-yarn.sh
```

### Cek JPS
```
jps
```
- Di Master: Harus muncul `NameNode`, `ResourceManager`
- Di Slave: Harus muncul `DataNode`, `NodeManager`

## 12. Akses Web UI
- NameNode: `http://master:9870`
- ResourceManager: `http://master:8088`

---
Panduan ini sudah disesuaikan agar aman, stabil, dan cocok untuk implementasi di jaringan nyata. Jika ada kendala teknis atau ingin dijadikan skrip otomatis `.sh`, tinggal bilang saja!
