# sepolia-rpc

### 🧱 Step 1: 安装依赖软件

```bash
sudo apt-get update && sudo apt-get upgrade -y

sudo apt install curl iptables build-essential git wget lz4 jq make gcc nano automake autoconf tmux htop nvme-cli libgbm1 pkg-config libssl-dev libleveldb-dev tar clang bsdmainutils ncdu unzip -y
```

---

### 🐳 Step 2: 安装 Docker 和 Docker Compose

```bash
# 清理旧的 Docker
for pkg in docker.io docker-doc docker-compose podman-docker containerd runc; do sudo apt-get remove -y $pkg; done

# 添加 Docker 官方源
sudo apt-get install ca-certificates curl gnupg -y
sudo install -m 0755 -d /etc/apt/keyrings
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg
sudo chmod a+r /etc/apt/keyrings/docker.gpg

echo \
"deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] https://download.docker.com/linux/ubuntu \
$(. /etc/os-release && echo "$VERSION_CODENAME") stable" | \
sudo tee /etc/apt/sources.list.d/docker.list > /dev/null

# 安装 Docker 和 Compose 插件
sudo apt update -y && sudo apt install docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin -y

# 测试 Docker 是否安装成功
sudo docker run hello-world
sudo systemctl enable docker
sudo systemctl restart docker
```

---

### 📂 Step 3: 创建目录和 JWT 密钥

```bash
mkdir -p /root/ethereum/execution
mkdir -p /root/ethereum/consensus
openssl rand -hex 32 > /root/ethereum/jwt.hex
cat /root/ethereum/jwt.hex  # 确认存在
```

---

### 📝 Step 4: 编写 `docker-compose.yml`

```bash
cd /root/ethereum
nano docker-compose.yml
```

```bash
services:
  geth:
    image: ethereum/client-go:stable
    container_name: geth
    restart: unless-stopped
    ports:
      - 30303:30303
      - 30303:30303/udp
      - 18545:8545
      - 18546:8546
      - 18551:8551
    volumes:
      - /root/ethereum/execution:/data
      - /root/ethereum/jwt.hex:/data/jwt.hex
    command:
      - --sepolia
      - --http
      - --http.api=eth,net,web3
      - --http.addr=0.0.0.0
      - --http.port=8545
      - --authrpc.addr=0.0.0.0
      - --authrpc.port=8551
      - --authrpc.vhosts=*
      - --authrpc.jwtsecret=/data/jwt.hex
      - --syncmode=snap
      - --datadir=/data
    logging:
      driver: "json-file"
      options:
        max-size: "10m"
        max-file: "3"

  prysm:
    image: gcr.io/prysmaticlabs/prysm/beacon-chain
    container_name: prysm
    restart: unless-stopped
    ports:
      - 14000:4000
      - 13500:3500
      - 12001:12000/udp
    volumes:
      - /root/ethereum/consensus:/data
      - /root/ethereum/jwt.hex:/data/jwt.hex
    depends_on:
      - geth
    command:
      - --sepolia
      - --accept-terms-of-use
      - --datadir=/data
      - --disable-monitoring
      - --rpc-host=0.0.0.0
      - --execution-endpoint=http://geth:8551
      - --jwt-secret=/data/jwt.hex
      - --rpc-port=4000
      - --grpc-gateway-corsdomain=*
      - --grpc-gateway-host=0.0.0.0
      - --grpc-gateway-port=3500
      - --min-sync-peers=3
      - --checkpoint-sync-url=https://checkpoint-sync.sepolia.ethpandaops.io
      - --genesis-beacon-api-url=https://checkpoint-sync.sepolia.ethpandaops.io
    logging:
      driver: "json-file"
      options:
        max-size: "10m"
        max-file: "3"
```

### 🟢 Step 5: 启动节点

```bash
docker compose up -d
```

查看日志：

```bash
docker compose logs -fn 100
```

---

### 🔍 Step 6: 查看同步状态

### ✅ 检查 Geth 执行层是否同步：

```bash
curl -X POST -H "Content-Type: application/json" --data '{"jsonrpc":"2.0","method":"eth_syncing","params":[],"id":1}' http://localhost:8545
```

### ✅ 检查 Prysm 共识层是否同步：

```bash
curl http://localhost:3500/eth/v1/node/syncing
```

---

### 🔐 Step 7: 配置防火墙（UFW）

```bash
sudo ufw allow 22
sudo ufw allow ssh
sudo ufw allow 30303/tcp
sudo ufw allow 30303/udp
sudo ufw allow from 127.0.0.1 to any port 8545 proto tcp
sudo ufw allow from 127.0.0.1 to any port 3500 proto tcp
sudo ufw enable
sudo ufw reload
```

> 如需远程访问 RPC，请执行：
> 

```bash
sudo ufw allow from <your-client-ip> to any port 8545 proto tcp
sudo ufw allow from <your-client-ip> to any port 3500 proto tcp
```

---

### 📡 Step 8: 获取 RPC 访问地址

- **执行层 (Geth)**
    - 内部访问：`http://localhost:8545`
    - 外部访问：`http://<your-vps-ip>:8545`
- **共识层 (Prysm)**
    - 内部访问：`http://localhost:3500`
    - 外部访问：`http://<your-vps-ip>:3500`

---

### 📊 Step 9: 资源监控（可选）

```bash
htop                # 查看 CPU、内存
df -h               # 查看磁盘空间
docker exec -it geth du -sh /data
docker exec -it prysm du -sh /data
```
