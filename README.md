# Run-Nexus-CLI-Node-on-ETH-storage-
How to Run Nexus CLI Node on ETH Storage full details step by step with coding 

Nexus CLI node contributes zk proofs to the Nexus Layer-1 Testnet III and can earn NEX points if you link your account.
EthStorage es-node is a storage rollup built atop Ethereum (Sepolia testnet by default). We’ll run it via Docker. 

1) Server prep (Ubuntu)

# Update base OS:
sudo apt update && sudo apt -y upgrade

# Install utilities
sudo apt -y install curl git ufw

# Optional: open ports used by EthStorage (adjust if you harden later)
sudo ufw allow 22/tcp
sudo ufw allow 9545/tcp    # es-node RPC
sudo ufw allow 9222/tcp    # es-node p2p/ctrl
sudo ufw allow 30305/udp   # es-node p2p
sudo ufw --force enable

2) Install Docker (for EthStorage):
curl -fsSL https://get.docker.com | sh
sudo usermod -aG docker $USER
# log out/in (or run: newgrp docker)

3) Run EthStorage es-node (Sepolia)

A) Get RPC endpoints + wallets

Create two wallets:

>Miner address → receives mining rewards.

>Signer private key → signs txs (needs Sepolia ETH).


>Fund signer with Sepolia ETH from a faucet.

>Get Sepolia EL RPC (e.g., BlockPI free key; enable Archive Mode) and Sepolia Beacon (CL) RPC (e.g., QuickNode free).

B) One-liner (Docker)
Replace the placeholders and run:

# init (creates data dirs and downloads params)
docker run --rm \
  -v $PWD/es-data:/es-node/es-data \
  -v $PWD/zkey:/es-node/build/bin/snark_lib/zkey \
  -e ES_NODE_STORAGE_MINER=0xYOUR_MINER_ADDRESS \
  --entrypoint /es-node/init.sh \
  ghcr.io/ethstorage/es-node:v0.2.2 \
  --l1.rpc https://YOUR_SEPOLIA_EL_RPC

# start the node
docker run --name es -d \
  -v $PWD/es-data:/es-node/es-data \
  -v $PWD/zkey:/es-node/build/bin/snark_lib/zkey \
  -e ES_NODE_STORAGE_MINER=0xYOUR_MINER_ADDRESS \
  -e ES_NODE_SIGNER_PRIVATE_KEY=0xYOUR_SIGNER_PRIVKEY \
  -p 9545:9545 -p 9222:9222 -p 30305:30305/udp \
  --entrypoint /es-node/run.sh \
  ghcr.io/ethstorage/es-node:v0.2.2 \
  --l1.rpc https://YOUR_SEPOLIA_EL_RPC \
  --l1.beacon https://YOUR_SEPOLIA_BEACON_RPC

# logs
docker logs -f es

That’s exactly the flow recommended by the official tutorial (init → run, with EL/CL RPC). You’ll see phases: data sync → sampling (mining) → proof submission. 

> Tip: Want a compose file? Use the same env/volumes as above; just map the same ports.

4) Install & run a Nexus CLI node (prover)
The current, supported install is a single script; the binary is called nexus-cli in the latest releases. You can start with an existing node ID or register from the CLI and link it to your Nexus account to earn NEX Points.

A) Quick install:

# Interactive install (accept Terms on first run)
curl https://cli.nexus.xyz/ | sh

B) Start with an existing node id:

nexus-cli start --node-id YOUR_NODE_ID

C) Or register & start from scratch:

# 1) Link your wallet (the one you want to track points with)
nexus-cli register-user --wallet-address 0xYOUR_WALLET

# 2) Create node id locally
nexus-cli register-node

# 3) Start proving
nexus-cli start

You can always view/help or logout:

nexus-cli --help
nexus-cli logout

> Older docs may show nexus-network in examples; the current README and releases use nexus-cli. If your binary is named differently after install, run which nexus-cli and which nexus-network to confirm.

5) Make both services auto-start (systemd)
A) Nexus CLI node as a service

sudo tee /etc/systemd/system/nexus-cli.service >/dev/null <<'EOF'
[Unit]
Description=Nexus CLI prover
After=network-online.target
Wants=network-online.target

[Service]
User=%i
WorkingDirectory=/home/%i
Environment=NONINTERACTIVE=1
# Optional: hardcode your node id to avoid prompts
# Environment=NEXUS_NODE_ID=YOUR_NODE_ID
ExecStart=/usr/bin/env bash -lc 'command -v nexus-cli >/dev/null && \
  ( [ -z "$NEXUS_NODE_ID" ] && nexus-cli start || nexus-cli start --node-id "$NEXUS_NODE_ID" )'
Restart=always
RestartSec=5

[Install]
WantedBy=multi-user.target
EOF

# Replace 'ubuntu' with your actual username:
sudo systemctl enable nexus-cli.service
sudo systemctl start nexus-cli.service
journalctl -u nexus-cli -f

B)  EthStorage as a service (wrap the docker run):

sudo tee /etc/systemd/system/ethstorage.service >/dev/null <<'EOF'
[Unit]
Description=EthStorage es-node (Docker)
After=docker.service network-online.target
Requires=docker.service

[Service]
User=%i
WorkingDirectory=/home/%i/ethstorage
Environment=ES_MINER=0xYOUR_MINER_ADDRESS
Environment=ES_SIGNER_PRIV=0xYOUR_SIGNER_PRIVKEY
Environment=EL_RPC=https://YOUR_SEPOLIA_EL_RPC
Environment=CL_RPC=https://YOUR_SEPOLIA_BEACON_RPC
ExecStartPre=/usr/bin/docker rm -f es || true
ExecStart=/usr/bin/docker run --name es \
  -v %h/ethstorage/es-data:/es-node/es-data \
  -v %h/ethstorage/zkey:/es-node/build/bin/snark_lib/zkey \
  -e ES_NODE_STORAGE_MINER=${ES_MINER} \
  -e ES_NODE_SIGNER_PRIVATE_KEY=${ES_SIGNER_PRIV} \
  -p 9545:9545 -p 9222:9222 -p 30305:30305/udp \
  --entrypoint /es-node/run.sh \
  ghcr.io/ethstorage/es-node:v0.2.2 \
  --l1.rpc ${EL_RPC} --l1.beacon ${CL_RPC}
ExecStop=/usr/bin/docker stop es
Restart=always
RestartSec=5

[Install]
WantedBy=multi-user.target
EOF

mkdir -p ~/ethstorage/{es-data,zkey}
sudo systemctl enable ethstorage.service
sudo systemctl start ethstorage.service
journalctl -u ethstorage -f

6) Verify everything
EthStorage logs should move to “Sampling … done” and occasionally “Mining transaction success!” after proof submission. 
Nexus should display tasks/proofs and your node should show up in your account after linking. Points accrue only when linked.

7) Upgrades & maintenance
Nexus CLI: re-run the install script to fetch the latest release; new CLIs are not backward compatible across some testnet phases.

curl https://cli.nexus.xyz/ | sh
systemctl restart nexus-cli

EthStorage: stop the container, pull a newer tag (check docs/releases), re-run init.sh only when instructed, then start again. 

systemctl restart ethstorage
docker logs -f es

Must be Noted 
Keep the signer private key in systemd env or a local .env file you protect. Never commit it.
EthStorage is CPU-intensive during sync/sampling; if running both on one box, give yourself headroom (≥8 cores/16 GB RAM is comfortable). (Guideline derived from EthStorage’s minimums and typical prover loads.) 
To earn NEX points, you must link your CLI to your Nexus account (anonymous proving doesn’t count). 
