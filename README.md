# 🐘 Hadoop 3.3.6 Multi-Node Cluster Setup (Ubuntu)

Panduan lengkap instalasi dan konfigurasi **Apache Hadoop 3.3.6** pada arsitektur Multi-Node (1 Master, 2 Worker) menggunakan sistem operasi Ubuntu. Panduan ini mencakup konfigurasi HDFS, YARN, dan MapReduce.

## 📋 Topologi Jaringan

| Role | Hostname | IP Address (Contoh) | Deskripsi |
| :--- | :--- | :--- | :--- |
| **Master** | `master` | `192.168.1.10` | NameNode, ResourceManager |
| **Slave 1** | `slave1` | `192.168.1.11` | DataNode, NodeManager |
| **Slave 2** | `slave2` | `192.168.1.12` | DataNode, NodeManager |

> **Catatan:** Pastikan semua node terhubung dalam satu jaringan LAN dan saling bisa melakukan ping.

---

## 🚀 1. Persiapan Awal (Semua Node)

Lakukan langkah ini di **Master** dan semua **Slave**.

### Update OS & Install Java 11
```bash
sudo apt update && sudo apt upgrade -y
sudo apt install openjdk-11-jdk ssh pdsh vim -y
```
### Buat User Hadoop (hduser)
```bash
sudo adduser hduser
sudo usermod -aG sudo hduser
su - hduser
```
### Konfigurasi Hostname & Hosts
Edit /etc/hosts di setiap mesin agar saling mengenali nama hostname.
```bash
sudo nano /etc/hosts
```
Tambahkan:
```bash
192.168.1.10 master
192.168.1.11 slave1
192.168.1.12 slave2
```

---

## 🔑 2. Konfigurasi SSH (Hanya di Master)
Master harus bisa mengakses Slave (dan dirinya sendiri) tanpa password.
```bash
# Generate SSH Key (Tekan Enter untuk semua prompt)
ssh-keygen -t rsa -P "" -f ~/.ssh/id_rsa

# Copy ID ke semua node
ssh-copy-id master
ssh-copy-id slave1
ssh-copy-id slave2
```

---

## 📦 3. Instalasi Hadoop (Hanya di Master)
Download dan ekstrak Hadoop.
```bash
cd ~
wget [https://dlcdn.apache.org/hadoop/common/hadoop-3.3.6/hadoop-3.3.6.tar.gz](https://dlcdn.apache.org/hadoop/common/hadoop-3.3.6/hadoop-3.3.6.tar.gz)
tar -xvzf hadoop-3.3.6.tar.gz
mv hadoop-3.3.6 hadoop
```
### Setup Environment Variables
Edit .bashrc:
```bash
nano ~/.bashrc
```
Tambahkan di baris paling bawah:
```bash
export JAVA_HOME=/usr/lib/jvm/java-11-openjdk-amd64
export HADOOP_HOME=$HOME/hadoop
export HADOOP_INSTALL=$HADOOP_HOME
export HADOOP_MAPRED_HOME=$HADOOP_HOME
export HADOOP_COMMON_HOME=$HADOOP_HOME
export HADOOP_HDFS_HOME=$HADOOP_HOME
export YARN_HOME=$HADOOP_HOME
export HADOOP_COMMON_LIB_NATIVE_DIR=$HADOOP_HOME/lib/native
export HADOOP_CONF_DIR=$HADOOP_HOME/etc/hadoop
export HADOOP_OPTS="-Djava.library.path=$HADOOP_HOME/lib/native"
export PATH=$PATH:$HADOOP_HOME/sbin:$HADOOP_HOME/bin
export PDSH_RCMD_TYPE=ssh
```
Aktifkan perubahan:
```bash
source ~/.bashrc
```

---

## ⚙️ 4. Konfigurasi XML (Hanya di Master)
Edit file konfigurasi di ~/hadoop/etc/hadoop/.

hadoop-env.sh
Wajib tambahkan ini untuk mencegah error permission pada Hadoop 3.x:
```bash
export JAVA_HOME=/usr/lib/jvm/java-11-openjdk-amd64
export HDFS_NAMENODE_USER=hduser
export HDFS_DATANODE_USER=hduser
export HDFS_SECONDARYNAMENODE_USER=hduser
export YARN_RESOURCEMANAGER_USER=hduser
export YARN_NODEMANAGER_USER=hduser
```
core-site.xml
```bash
<configuration>
    <property>
        <name>fs.defaultFS</name>
        <value>hdfs://master:9000</value>
    </property>
    <property>
        <name>hadoop.tmp.dir</name>
        <value>/home/hduser/hadoop/tmp_data</value>
    </property>
</configuration>
```
hdfs-site.xml
```bash
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
mapred-site.xml
```bash
<configuration>
    <property>
        <name>mapreduce.framework.name</name>
        <value>yarn</value>
    </property>
    <property>
        <name>mapreduce.application.classpath</name>
        <value>$HADOOP_MAPRED_HOME/share/hadoop/mapreduce/*:$HADOOP_MAPRED_HOME/share/hadoop/mapreduce/lib/*</value>
    </property>
</configuration>
```
yarn-site.xml
```bash
<configuration>
    <property>
        <name>yarn.nodemanager.aux-services</name>
        <value>mapreduce_shuffle</value>
    </property>
    <property>
        <name>yarn.nodemanager.env-whitelist</name>
        <value>JAVA_HOME,HADOOP_COMMON_HOME,HADOOP_HDFS_HOME,HADOOP_CONF_DIR,CLASSPATH_PREPEND_DISTCACHE,HADOOP_YARN_HOME,HADOOP_HOME,PATH,LANG,TZ,HADOOP_MAPRED_HOME</value>
    </property>
    <property>
        <name>yarn.resourcemanager.hostname</name>
        <value>master</value>
    </property>
</configuration>
```
workers
Hapus localhost, ganti dengan hostname slave:
```bash
slave1
slave2
```

---

## 📡 5. Distribusi ke Slave
Salin konfigurasi yang sudah jadi dari Master ke Slave 1 dan Slave 2.
```bash
# Salin folder Hadoop
scp -r ~/hadoop hduser@slave1:~/
scp -r ~/hadoop hduser@slave2:~/

# Salin bashrc (agar environment variable sama)
scp ~/.bashrc hduser@slave1:~/
scp ~/.bashrc hduser@slave2:~/
```
>* Penting: Login ke slave1 dan slave2, lalu jalankan source ~/.bashrc.

---

## ▶️ 6. Menjalankan Cluster
1. Format NameNode (Hanya Sekali)
Lakukan ini di Master saat pertama kali instalasi.
```bash
hdfs namenode -format
```
2. Start Services
Di Master, jalankan:
```bash
start-dfs.sh
start-yarn.sh
```

---

## ✅ 7. Verifikasi
Cek JPS (Java Process Status)
Di Master:
```bash
$ jps
NameNode
ResourceManager
SecondaryNameNode
```
Di Slave:
```bash
$ jps
DataNode
NodeManager
```
Web Interface
Buka browser dan akses:
>* HDFS Dashboard: http://master:9870
>* YARN Dashboard: http://master:8088

---

## 🛠 Troubleshooting
Permission Denied: Pastikan folder ~/hadoop dimiliki oleh user hduser.
```bash
sudo chown -R hduser:hduser ~/hadoop
```
DataNode Tidak Muncul: Biasanya karena Cluster ID tidak cocok. Hapus folder data di slave dan restart.
```bash
# Di slave
rm -rf ~/hadoop/hdfs/datanode/*
```
