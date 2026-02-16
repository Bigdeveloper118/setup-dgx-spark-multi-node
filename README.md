# Spark Standalone Cluster Setup (2 Nodes – Production Detailed Guide)

---

# 1. Overview

This document describes the complete setup and validation of a 2-node Apache Spark Standalone Cluster.
## Network Config

https://docs.nvidia.com/dgx/dgx-spark/spark-clustering.html

## Architecture

| Node | Hostname   | IP              | Role   |
|------|-----------|----------------|--------|
| Node 1 | gx10-ddc3 | 192.168.100.10 | Master |
| Node 2 | gx10-0d32 | 192.168.100.11 | Worker |

OS: Ubuntu 24.04  
Java: OpenJDK 11  
Spark: 3.5.8  
Cluster Mode: Standalone  

---

# 2. How Spark Standalone Works (Important Concept)

## Component Roles

### Master
- Manages cluster resources
- Schedules applications
- Tracks worker status
- Default ports:
  - 7077 → Cluster communication
  - 8080 → Web UI

### Worker
- Registers to master
- Provides CPU & memory
- Launches executors

### Driver
- Submits job
- Coordinates tasks
- Can run on master or client machine

---

# 3. Network Requirements

## Required Ports (Master Node)

| Port | Purpose |
|------|--------|
| 7077 | Spark Master RPC |
| 8080 | Spark Master Web UI |
| 22   | SSH |

Worker uses dynamic ports for executors.

---

# 4. Install Java (Both Nodes)

sudo apt update
sudo apt install openjdk-11-jdk -y
`

Verify:

java -version

Expected:

openjdk version "11.x"

---

# 5. Install Spark (Both Nodes)

Download:

wget https://downloads.apache.org/spark/spark-3.5.8/spark-3.5.8-bin-hadoop3.tgz

Extract:

tar -xzf spark-3.5.8-bin-hadoop3.tgz
mv spark-3.5.8-bin-hadoop3 spark

---

# 6. Environment Variables (Both Nodes)

Edit:

nano ~/.bashrc

Add:

export SPARK_HOME=$HOME/spark
export PATH=$SPARK_HOME/bin:$SPARK_HOME/sbin:$PATH

Reload:

source ~/.bashrc

Validate:

which spark-submit

update spark-env.sh:
# ไฟล์: $SPARK_HOME/conf/spark-env.sh ใน DGX 1 และ 2

# 1. ระบุ IP ของ DGX ในวงปกติ (1.x) เพื่อเอาไว้รับคำสั่งจาก Master
export SPARK_WORKER_HOST=192.168.1.xx

# 2. หัวใจสำคัญ: ระบุ IP วง Stack (100.x) เพื่อให้ Spark ใช้รับส่งข้อมูลหนักๆ
# ข้อมูลจะวิ่งท่อนี้เท่านั้น ไม่ผ่าน Master
export SPARK_LOCAL_IP=192.168.100.xx 

# 3. ตั้งค่า MTU ให้ Worker คุยกันเองด้วยกล่องใหญ่ (ถ้ามีใน Config)
# ปกติถ้า interface bond1 เป็น 9000 อยู่แล้ว Spark จะใช้ตามนั้นเอง

---

# 7. Fix Hostname Resolution (CRITICAL)

Wrong hostname mapping causes:

* Worker cannot register
* Loopback warnings
* Connection timeouts

## On Master

sudo nano /etc/hosts

Correct:

192.168.100.10 gx10-ddc3

Remove:

127.0.1.1 gx10-ddc3

## On Worker

sudo nano /etc/hosts

Correct:

192.168.100.11 gx10-0d32

---

# 8. Configure Spark Master (Node .10)

Create config:

cp $SPARK_HOME/conf/spark-env.sh.template $SPARK_HOME/conf/spark-env.sh

Edit:

nano $SPARK_HOME/conf/spark-env.sh

Add:

export SPARK_MASTER_HOST=192.168.100.10
export SPARK_MASTER_PORT=7077
export SPARK_LOCAL_IP=192.168.100.10

Why?

* Prevents hostname advertising
* Ensures correct IP binding
* Avoids loopback issues

---

# 9. Configure Worker (Node .11)

cp $SPARK_HOME/conf/spark-env.sh.template $SPARK_HOME/conf/spark-env.sh
nano $SPARK_HOME/conf/spark-env.sh

Add:

export SPARK_LOCAL_IP=192.168.100.11

---

# 10. Configure Firewall (Master Only)

sudo ufw allow 22
sudo ufw allow 7077
sudo ufw allow 8080
sudo ufw reload

Check:

sudo ufw status

---

# 11. Start Cluster

## Start Master

start-master.sh

Verify:

jps

Expected:

Master

Verify port:

ss -tlnp | grep 7077

Expected:

LISTEN 192.168.100.10:7077

Web UI:

http://192.168.100.10:8080

Must show:

spark://192.168.100.10:7077

---

## Start Worker

start-worker.sh spark://192.168.100.10:7077

Verify:

jps

Expected:

Worker

---

# 12. Validate Cluster

Web UI must show:

* Workers: 1
* Status: ALIVE
* 20 cores
* ~118GB RAM
* Worker IP: 192.168.100.11

---

# 13. Test Distributed Execution

spark-submit \
  --class org.apache.spark.examples.SparkPi \
  --master spark://192.168.100.10:7077 \
  $SPARK_HOME/examples/jars/spark-examples_2.12-3.5.8.jar \
  10

Success output:

Pi is roughly 3.14xxxx

Check Web UI:

* Running Applications
* Executor on worker node

---

# 14. Resource Tuning (Optional)

Limit worker cores:

start-worker.sh -c 16 spark://192.168.100.10:7077

Limit memory:

start-worker.sh -m 64g spark://192.168.100.10:7077

---

# 15. Troubleshooting Guide

## Worker Not Registering

Check:

ss -tlnp | grep 7077

Check firewall:

sudo ufw status

Check logs:

cat $SPARK_HOME/logs/*Worker*.out

## Loopback Warning

Fix /etc/hosts

## Connection Refused

Master not running or port blocked.

---

# 16. Stop Cluster

On Worker:

stop-worker.sh

On Master:

stop-master.sh

---

# 17. Production Recommendations

* Use static IP
* Use passwordless SSH
* Use systemd service
* Monitor with Prometheus
* Consider Spark on Kubernetes for scaling

---

# Final Validation Checklist

✔ Master running
✔ Worker ALIVE
✔ 7077 port listening
✔ Firewall configured
✔ Hostname resolved correctly
✔ Distributed job executed successfully

Spark Standalone Cluster Fully Operational

```
