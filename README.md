# Cosmos Horcrux Validator Setup

This repository contains Ansible playbooks for setting up a Cosmos validator node with Horcrux signing infrastructure.

## Prerequisites

- Ansible 2.9 or higher
- SSH access to target hosts
- Python 3.x on target hosts

## Installation

1. Install required Ansible collections:
```bash
ansible-galaxy collection install devsec.hardening
ansible-galaxy collection install community.docker
ansible-galaxy role install geerlingguy.docker
```

## Configuration

1. Update `inventory.yml` with your hosts:
   - Configure validator node(s) under `validators` group
   - Configure signer node(s) under `signers` group
   - Set appropriate SSH users and ports
   - Configure chain-specific settings

## Deployment Steps

### 1. Base System Setup

Run the base system setup playbook to configure OS hardening, SSH hardening, and Docker:

```bash
ansible-playbook -i inventory.yml playbooks/setup_base.yml
```

This playbook:
- Configures SSH hardening
- Applies OS hardening settings
- Sets up Docker
- Creates required users and groups

### 2. Deploy Noble Testnet Validator

Deploy the Noble testnet validator node:

```bash
ansible-playbook -i inventory.yml playbooks/playbooks/deploy_noble_testnet.yml
```

This playbook:
- Initializes the validator node
- Configures chain settings
- Sets up Docker container
- Configures private validator settings

### 3. Restore from Snapshot (Optional)

If you want to restore from a snapshot to speed up synchronization:

```bash
ansible-playbook -i inventory.yml playbooks/restore_snapshot.yml -e "target_group=validator-3"
```

This playbook:
- Downloads the latest snapshot
- Extracts it to the chain directory
- Restarts the node

### 4. Deploy Horcrux Signers

Deploy the Horcrux signing infrastructure:

```bash
ansible-playbook -i inventory.yml playbooks/deploy_signer.yml
```

This playbook:
- Sets up Horcrux on signer nodes
- Configures key shards
- Establishes connection with validator
