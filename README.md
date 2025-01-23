# Cosmos Horcrux Validator Setup

Ansible-based deployment for Cosmos validator with distributed Horcrux signing infrastructure (5 cosigners, 3/5 threshold) using Docker containers.

> ⚠️ **IMPORTANT: Development Status**
> - This setup is currently in testing phase and NOT production-ready
> - Several security features are disabled or not yet implemented:
>   - UFW (firewall) is disabled and configs for it are also not present
>   - Sentry nodes are not implemented
>   - Network segmentation is not configured
>   - Monitoring and alerts are not set up
> - See [TODO.md](TODO.md) for all pending security and infrastructure improvements
> - Do not use this configuration in production without enabling appropriate security measures

## Architecture

### Runtime Environment
- Container orchestration: Docker + Docker Compose
- Horcrux configuration: 5 cosigners (3/5 threshold)
- Network ports: 2024 (priv_validator)
- Security: OS hardening, SSH hardening (UFW disabled during testing)

### Components
1. **Validator**
   - Cosmos node (external signing)
   - No local priv_validator_key.json
   - Network security: UFW currently disabled for testing

2. **Horcrux Signers**
   - Distributed threshold signing (3/5)
   - Isolated containers per signer
   - Independent state directories
   - Automatic failover/recovery

## Prerequisites
- Ansible 2.9+
- SSH access
- Python 3.x

## Installation
```bash
ansible-galaxy collection install devsec.hardening
ansible-galaxy collection install community.docker
ansible-galaxy role install geerlingguy.docker
```

## Configuration
Example inventory structure:
```yaml
validators:
  hosts:
    validator-1:
      ansible_host: <IP>
      network:
        priv_validator_port: 2053

signers:
  hosts:
    signer-[1:5]:
      ansible_host: <IP>
```

## Deployment

1. **Base Setup**
```bash
ansible-playbook -i inventory.yml playbooks/setup_base.yml
```
- OS/SSH hardening
- Docker setup
- UFW configuration (TODO)

2. **Validator Setup**
```bash
ansible-playbook -i inventory.yml playbooks/deploy_noble_testnet.yml
```
- Chain initialization
- External signing config
- Container deployment

3. **Snapshot Restore (Optional)**
```bash
ansible-playbook -i inventory.yml playbooks/restore_snapshot.yml -e "target_group=validator-3"
```

4. **Signer Setup**
```bash
ansible-playbook -i inventory.yml playbooks/deploy_signer.yml
```
- Key share generation/distribution
- Threshold config (3/5)
- Validator connection setup

> **Security Note**: To minimize attack surface, no private key is ever transferred between validator and signers. Instead:
> 1. Private key is generated from scratch on a signer node using CometBFT in Docker
> 2. Key is sharded directly on the signer VM using Horcrux
> 3. Shards are distributed to other signers
> 4. Validator only receives the public key for external signing

## Post-Deployment

1. **Get Validator Address and Public Key**
```bash
# SSH into any signer node
ssh user@signer-node

# Get validator address and pubkey from Horcrux signer
# Note: This MUST be done from a signer node, not the validator
docker compose exec noble-testnet-cosigner-1 horcrux address grand-1 noble

# Example output:
# Address: noblevaloper1...
# Pubkey: {"@type":"/cosmos.crypto.ed25519.PubKey","key":"..."}
```
Save both the validator address (starts with 'noblevaloper') and complete pubkey JSON for staking.

2. **Validator Startup**
```bash
# Monitor validator logs
docker compose logs -f
```
Note: Validator may show "unable to get pubkey from private validator" initially. This is normal:
- Validator restarts automatically
- Waits for cosigner connection
- Starts syncing once any cosigner connects
- Becomes ready for staking after sync

3. **Staking Setup**
- Use external wallet with sufficient funds
- Stake tokens to validator address from step 1
- Command example:
```bash
nobled tx staking create-validator \
  --amount=1000000utoken \
  --pubkey=<PUBKEY_FROM_HORCRUX> \
  --moniker="validator-name" \
  --chain-id=grand-1 \
  --from=<WALLET_ADDRESS> \
  --commission-rate="0.10" \
  --commission-max-rate="0.20" \
  --commission-max-change-rate="0.01" \
  --min-self-delegation="1"
```

## Monitoring
```bash
# Validator status (from validator node)
cd /home/validator
docker compose logs -f

# Signer status (from signer node)
cd /home/signer
docker compose logs -f

# Monitor specific signer
cd /home/signer
docker compose logs -f noble-testnet-cosigner-1

# Firewall rules
sudo ufw status
```

## Troubleshooting
1. **Connection Issues**
   - UFW rules
   - Signer IP verification
   - Port 2053 accessibility

2. **Signing Issues**
   - Horcrux logs
   - Threshold verification
   - Network connectivity
