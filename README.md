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
docker compose exec noble-testnet-cosigner-1 horcrux address grand-1 noble

# Example output:
# {
#   "HexAddress": "1E8B17F49F36526CD6CEA3122EFB739F437BA735",
#   "PubKey": "{\"@type\":\"/cosmos.crypto.ed25519.PubKey\",\"key\":\"CMkI5XIcfprOpg1arNaPQevBvRiz3usA4PrWdXPoPsE=\"}",
#   "ValConsAddress": "noblevalcons1r6930aylxefxe4kw5vfza7mnnaphhfe4zn6ahf",
#   "ValConsPubAddress": "noblevalconspub1zcjduepqprys3etjr3lf4n4xp4d2e450g84ur0gck00wkq8qltt82ulg8mqsgpagey"
# }

# Convert HexAddress to operator address (can be run anywhere with Docker):
docker run --rm ghcr.io/strangelove-ventures/heighliner/noble:v8.0.0-rc.4 nobled debug addr 1E8B17F49F36526CD6CEA3122EFB739F437BA735 --prefix noble

# Example output:
# Address: [30 139 23 244 159 54 82 108 214 206 163 18 46 251 115 159 67 123 167 53]
# Address (hex): 1E8B17F49F36526CD6CEA3122EFB739F437BA735
# Bech32 Acc: noble1r6930aylxefxe4kw5vfza7mnnaphhfe4cs93dk
# Bech32 Val: noblevaloper1r6930aylxefxe4kw5vfza7mnnaphhfe4kqfpmg
# Bech32 Con: noblevalcons1r6930aylxefxe4kw5vfza7mnnaphhfe4zn6ahf
```

2. **Create Validator**

First, create a new wallet and get testnet tokens:
```bash
# Create new wallet
docker run --rm -it ghcr.io/strangelove-ventures/heighliner/noble:v8.0.0-rc.4 nobled keys add testnet-wallet

# Example output:
# - address: noble1amsmcwzeaglledv738qkvyy4hzc08y9zy80nhy
#   name: testnet-wallet
#   pubkey: '{"@type":"/cosmos.crypto.secp256k1.PubKey","key":"AzRWfR8po0y0OUXEG16KfL7PXaseEZoNifbJn1YzeUOH"}'
#   type: local

Get testnet tokens from faucet:
Visit https://faucet.circle.com/ and request tokens for your address

# Check balance
```bash
docker run --rm ghcr.io/strangelove-ventures/heighliner/noble:v8.0.0-rc.4 nobled query bank balances noble1amsmcwzeaglledv738qkvyy4hzc08y9zy80nhy
```

Create validator configuration file (validator.json):
```json
{
    "pubkey": {"@type":"/cosmos.crypto.ed25519.PubKey","key":"CMkI5XIcfprOpg1arNaPQevBvRiz3usA4PrWdXPoPsE="},
    "amount": "100000uusdc",
    "moniker": "tsvalidator",
    "identity": "optional identity signature (ex. UPort or Keybase)",
    "website": "validator website",
    "security": "validator's (optional) security contact email",
    "details": "Validator description",
    "commission-rate": "0.1",
    "commission-max-rate": "0.2",
    "commission-max-change-rate": "0.01",
    "min-self-delegation": "1"
}
```

Create validator:
```bash
nobled tx staking create-validator /validator.json --from testnet-wallet --chain-id grand-1 --fees 20000uusdc
```

3. **Monitor Status**

Check validator logs:
```bash
# From validator node
docker compose logs -f

# From signer node
docker compose logs -f noble-testnet-cosigner-1
```

Note: Initially, the validator may show "unable to get pubkey from private validator". This is normal:
- Validator will restart automatically
- Waits for cosigner connection
- Starts syncing once any cosigner connects
- Becomes ready for staking after sync

## Troubleshooting

1. **Connection Issues**
- Check signer IP addresses in configuration
- Verify port 2024 is accessible between validator and signers
- Check network connectivity between nodes

2. **Signing Issues**
- Check Horcrux logs for errors
- Verify 3/5 threshold is maintained
- Check network connectivity between cosigners

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
