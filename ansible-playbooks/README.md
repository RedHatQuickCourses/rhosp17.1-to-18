# Red Hat OpenStack Services on OpenShift (RHOSO) Prerequisites and Setup Ansible Playbooks

This repository contains Ansible playbooks that automate the prerequisites and initial setup for Red Hat OpenStack Services on OpenShift (RHOSO) based on the adoption documentation. The playbooks convert the step-by-step manual instructions from the following documentation sections into idempotent, automated workflows:

## Overview

These playbooks automate the following setup phases:

1. **Prerequisites** (`prereqs.adoc`) - Verify cluster access, verify existing operators (NMState, MetalLB), and install Cert-Manager operator
2. **Install Operators** (`install-operators.adoc`) - Deploy the OpenStack operators and initialize them
3. **Network Isolation** (`network-isolation.adoc`) - Configure OpenShift networking for OpenStack networks (NNCP, NAD, MetalLB pools, IP forwarding)

## Prerequisites

### Software Requirements

- Ansible 2.12 or newer
- Python 3.8+
- OpenShift CLI (`oc`) configured with cluster access
- SSH key-based authentication to lab nodes

### Lab Environment Requirements

- Operational OpenShift cluster with Multus CNI support
- Bastion host with `oc` command line tool configured with cluster access
- Access to the OpenShift cluster (via bastion or directly)
- NMState and MetalLB operators already installed in the cluster (as per prereqs.adoc)

### Required Files

The playbooks expect the following YAML files to be present in the `files/` directory at the repository root:

- `osp-ng-openstack-operator.yaml` - OpenStack operator subscription configuration
- `osp-ng-openstack-operator-init.yaml` - OpenStack operator initialization
- `osp-ng-nncp-w1.yaml`, `osp-ng-nncp-w2.yaml`, `osp-ng-nncp-w3.yaml` - Node network configuration policies
- `osp-ng-netattach.yaml` - Network attachment definitions
- `osp-ng-metal-lb-ip-address-pools.yaml` - MetalLB IP address pools
- `osp-ng-metal-lb-l2-advertisements.yaml` - MetalLB L2 advertisements

## Quick Start

### SSH Jump Host Scenario (Recommended)

This deployment method is designed for environments where the OpenShift cluster can only be reached through a bastion/jump host. The playbooks will connect to the bastion host and then use SSH proxy commands to reach internal hosts (NFS server, compute nodes).

#### Prerequisites

1. **SSH Access to Bastion**: You must be able to SSH to your bastion host
2. **sshpass**: Install `sshpass` on your workstation for password-based SSH connections
3. **SSH Keys**: Ensure your lab SSH keys are available on the bastion host at `/home/lab-user/.ssh/`

#### 1. Configure Inventory

Edit `inventory/hosts.yml` and update the following variables with your lab details:

```yaml
# REQUIRED: Update these values for your environment
lab_guid: "your-lab-guid"                     # e.g., "a1b2c"
bastion_hostname: "ssh.ocpvdev01.rhdp.net"    # Your actual bastion hostname
bastion_port: "31295"                         # Your actual SSH port
bastion_password: "your-bastion-password"     # Your actual password
bastion_user: "lab-user"                      # Usually lab-user

# REQUIRED: Add your Red Hat credentials
registry_username: "your-registry-service-account"
registry_password: "your-registry-token"
rhc_username: "your-rh-username"
rhc_password: "your-rh-password"

# OPTIONAL: Internal hostnames (usually defaults work)
nfs_server_hostname: "nfsserver"              # Internal hostname for NFS server
compute_hostname: "compute01"                 # Internal hostname for compute node

# OPTIONAL: External IPs for OpenShift worker nodes (update if different)
rhoso_external_ip_worker_1: "172.21.0.21"    # External IP for worker node 1
rhoso_external_ip_worker_2: "172.21.0.22"    # External IP for worker node 2
rhoso_external_ip_worker_3: "172.21.0.23"    # External IP for worker node 3
```

#### 2. Optional: Configure SSH (Alternative Method)

For better SSH management, you can copy `ssh-config.template` to `~/.ssh/config` and update it with your values. This allows direct SSH to internal hosts:

```bash
cp ssh-config.template ~/.ssh/config
# Edit ~/.ssh/config with your actual values
```

#### 3. Check Configuration

```bash
./deploy-via-jumphost.sh --check-inventory
```

#### 4. Run Deployment

```bash
# Full deployment
./deploy-via-jumphost.sh

# Or specific phases
./deploy-via-jumphost.sh prerequisites
./deploy-via-jumphost.sh control-plane

# Dry run (check mode)
./deploy-via-jumphost.sh --dry-run full
```

#### Troubleshooting SSH Jump Host Issues

1. **SSH Connection Failures**: Ensure `sshpass` is installed and your bastion credentials are correct
2. **Permission Denied**: Verify SSH keys are present on bastion at `/home/lab-user/.ssh/LAB_GUIDkey.pem`
3. **Proxy Command Errors**: Check that internal hostnames (nfsserver, compute01) are resolvable from bastion
4. **Timeout Issues**: Internal hosts may take time to boot; retry after a few minutes

### Direct Bastion Access Scenario

If you're running directly on the bastion host:

#### 1. Install Required Collections

```bash
ansible-galaxy collection install -r requirements.yml
```

#### 2. Configure Inventory

Edit `inventory/hosts.yml` for direct access (no SSH jump)

#### 3. Run Deployment

```bash
ansible-playbook site.yml
```

## Individual Role Execution

You can run individual roles for troubleshooting or partial deployments:

```bash
# Prerequisites only
ansible-playbook site.yml --tags prerequisites

# Install operators only
ansible-playbook site.yml --tags install-operators

# Network configuration only  
ansible-playbook site.yml --tags network-isolation
```

## Project Structure

```
ansible-playbooks/
├── site.yml                 # Main playbook orchestrating the three roles
├── ansible.cfg              # Ansible configuration
├── requirements.yml         # Required Ansible collections
├── inventory/
│   └── hosts.yml            # Inventory template (MUST be customized)
├── vars/
│   └── main.yml            # Global variables and configuration
└── roles/
    ├── prerequisites/       # Verify existing operators and install Cert-Manager
    ├── install-operators/   # Deploy OpenStack operators
    └── network-isolation/   # Set up network isolation (NNCP, NAD, MetalLB)
```

## Key Variables

### Lab Environment (`vars/main.yml`)

- `openstack_operators_namespace`: Namespace for OpenStack operators (default: `openstack-operators`)
- `openstack_namespace`: Namespace for OpenStack deployment (default: `openstack`)

### Timeouts

- `operator_wait_timeout`: 600 seconds (operator installation)
- Default retries and delays are configured in each role for waiting on resources

## Common Issues and Troubleshooting

### 1. Operator Installation Timeout

**Problem**: Operators fail to install within timeout period
**Solution**: Check cluster resources and verify operator catalogs are available. Increase retry counts in role tasks if needed.

### 2. Missing YAML Files

**Problem**: Playbook fails with "Required files not found" error
**Solution**: Ensure all required YAML files exist in the `files/` directory at the repository root

### 3. Network Configuration Issues

**Problem**: Network isolation setup fails or NNCP doesn't reach SuccessfullyConfigured state
**Solution**: 
- Verify OpenShift worker nodes have the required network interfaces
- Check NMState operator is running: `oc get pods -n openshift-nmstate`
- Verify NNCP status: `oc get nncp`

### 4. Cert-Manager Installation Issues

**Problem**: Cert-Manager operator fails to install
**Solution**: 
- Verify the operator catalog source is available: `oc get catalogsource -n openshift-marketplace`
- Check install plan status: `oc get installplan -n cert-manager-operator`

## Verification Commands

After successful execution, verify the setup:

```bash
# Verify prerequisites
oc get clusterserviceversion -n openshift-nmstate -o custom-columns=Name:.metadata.name,Phase:.status.phase
oc get clusterserviceversion -n metallb-system -o custom-columns=Name:.metadata.name,Phase:.status.phase
oc get pods -n cert-manager

# Verify OpenStack operators
oc get operators openstack-operator.openstack-operators
oc get pods -n openstack-operators

# Verify network isolation
oc get nncp
oc get netattachdef -n openstack
oc get ipaddresspool -n metallb-system
oc get l2advertisement -n metallb-system
```

## Support

This playbook automation is based on the RHOSO adoption documentation sections:
- `prereqs.adoc` - Prerequisites for Installation
- `install-operators.adoc` - Install the OpenStack Operator
- `network-isolation.adoc` - Preparing RHOCP for RHOSP Network Isolation

For issues:

1. Check the troubleshooting section above
2. Verify your lab environment meets all prerequisites  
3. Review Ansible output for specific error messages
4. Consult the original AsciiDoc documentation for manual steps

## Contributing

When modifying these playbooks:

1. Maintain idempotency - tasks should be safe to run multiple times
2. Use appropriate Ansible modules instead of shell commands where possible
3. Add proper error handling and verification steps
4. Update documentation for any new variables or requirements
5. Test thoroughly in a lab environment before production use
