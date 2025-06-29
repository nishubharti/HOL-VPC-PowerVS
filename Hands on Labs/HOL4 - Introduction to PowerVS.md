# Hands-on Lab 4: Introduction to IBM PowerVS (60 mins)

What you will learn:

* PowerVS fundamentals and service architecture.
* Virtual service instance deployment on PowerVS.
* PowerVS storage volume management.
* PowerVS management using console, CLI, and API interfaces.

## Overview

PowerVS workspaces are region-specific containers for all your Power Virtual Server resources. In this Hands on Lab (HOL) we will create a PowerVS virtual server instance, on a private network, protected by a network security group. The VSI will be connected to a public network initially so that it can install a postgres database via cloud-init.

## Resources that will be deployed in this HOL

In this HOL, you will deploy the following:

Resource Type | Name | Notes
---------|----------|---------
Power Workspace | <TEAM_NAME>-powervs-wksp
Private Subnet | <TEAM_NAME>-power-db-sn
PowerVS VSI | <TEAM_NAME>db-powervs-vsi

This document references:

- `<TEAM_NAME>` this is your team name e.g. `team-1`
- `<TEAM_ID_NUMBER>` this is your team number e.g. `1`

## Scenario

In this HOL we will:

* Deploy PowerVS resources:
    * Step 1: Create PowerVS Workspace
    * Step 2: Verify SSH Key
    * Step 3: Set Up Networks
    * Step 4: Deploy Virtual Server Instance (VSI)
    * Step 5: Configure Network Security groups
* Verify:
    * Step 1: Basic Storage Information and Health Checks
    * Step 2. **dd command** - Simple throughput testing
    * Step 3. **hdparm** - Basic disk performance
    * Step 4. **fio (Flexible I/O Tester)** - Most comprehensive

## Deploy PowerVS resources

### Step 1: Create PowerVS Workspace

1. Create a PowerVS workspace that will contain all your Power Virtual Server resources. Follow the instructions at[Creating a Power Virtual Server workspace](https://cloud.ibm.com/docs/power-iaas?topic=power-iaas-creating-power-virtual-server#creating-service) using the following parameters:

   - **Location type**: `IBM datacenter`
   - **Location**: Dallas (us-south)
   - **Workspace name**: <TEAM_NAME>-powervs-wksp
   - **Resource group**: <TEAM_NAME>-app1-rg
   - **User tags**: `env:app1`

### Step 2: Verify SSH Key

1. From the newly created workspace, navigate to **SSH Keys** and ensure yor SSH keys are listed

### Step 3: Set Up Networks

1. Follow the instructions at [Configuring a private network subnet](https://cloud.ibm.com/docs/power-iaas?topic=power-iaas-configuring-subnet) to create a private subnet using the following parameters:
   
   - **Network name**: <TEAM_NAME>-power-db-sn
   - **CIDR**: 10.<TEAM_NUMBER>.8.0/24
   - **Gateway**: 10.<TEAM_NUMBER>.8.1
   - **DNS servers**: Use the IP addresses from your private DNS custom resolvers
   - **MTU**: 9000

2. To verify use the following commands:

```bash
ibmcloud pi subnet list
ibmcloud pi subnet get <TEAM_NAME>-power-db-sn
```

### Step 4: Deploy Virtual Server Instance (VSI)

1. Follow the documentation at [Creating a Power Systems Virtual Server](https://cloud.ibm.com/docs/power-iaas?topic=power-iaas-creating-power-virtual-server) with the following parameters: 

   - **Instance name**: <TEAM_NAME>db-powervs-vsi
   - **User tags**: `env:app1`
   - **Virtual server pinning**: none
   - **SSH key**: <TEAM_NAME>-ssh-key-1
   - **Operating system**: IBM provides subscription - Linux.
   - **Image**: RHEL9-SP4
   - **Tier**: Tier 3
   - **Advanced configurations**:
     - **Specify cloud-init user data**: Enabled
     - **User data**: Paste contents of [db-powervs-vsi-user-data.yaml](Scripts/HOL4/db-powervs-vsi-user-data.yaml)
   - **Machine Type**: s922
   - **Core type**: Shared uncapped
   - **Profile**: System configuration
   - **Cores**: 0.25
   - **Memory**: 2
   - **Public networks**: Enable
   - **Network**: <TEAM_NAME>-power-db-sn

2. To verify use the following commands:
 
```bash
ibmcloud pi instance list
ibmcloud pi instance get <TEAM_NAME>db-powervs-vsi
```

### Step 5: Configure Network Security groups

Network Security groups enablement and configuration takes a little while, please be patient at each step and ensure the step completes before moving on to the next.

1. Follow the instructions at [Enabling or disabling NSG in a workspace](https://cloud.ibm.com/docs/power-iaas?topic=power-iaas-nsg#enable-disable-nsg)
2. Follow the instructions at [Network security groups](https://cloud.ibm.com/docs/power-iaas?topic=power-iaas-nsg) to enable Network Security groups using the following parameters:

    - **Network address groups**:
      - **Name**: mgmt-servers
      - **CIDR**: 10.<TEAM_NUMBER>.1.0.0/24
      - **Name**: app1-app-sn
      - **CIDR**: 10.<TEAM_NUMBER>.4.64/26
    - **Network security groups**:
      - **Inbound rules**:
        - **Any**:
          - **Action**: Allow
          - **Protocol**: Any
          - **Remote**: mgmt-servers
          - **Members**: <TEAM_NAME>db-powervs-vsi
        - **TCP**:
          - **Action**: Allow
          - **Protocol**: Any
          - **Remote**: mgmt-servers
          - **Source port range**: 5432-5432
          - **Members**: <TEAM_NAME>db-powervs-vsi

## Verify

In this section we will connect to the VSI and do some storage tests.

### Step 1: Basic Storage Information and Health Checks

First, gather information about your storage configuration:

```bash
# List all block devices
lsblk

# Show filesystem disk space usage
df -h

# Display detailed disk information
fdisk -l

# View LVM configuration if using logical volumes
sudo lvdisplay
sudo vgdisplay
sudo pvdisplay

# Check multipath configuration (common in enterprise environments)
sudo multipath -ll
```

### Step 2. **dd command** - Simple throughput testing

```bash
# Test write performance (be careful with target location)
dd if=/dev/zero of=/tmp/testfile bs=1M count=1024 oflag=direct

# Test read performance
dd if=/tmp/testfile of=/dev/null bs=1M iflag=direct

# Clean up
rm /tmp/testfile
```

### Step 3. **hdparm** - Basic disk performance
```bash
sudo dnf install hdparm

# Test cached read speed
sudo hdparm -t /dev/sda

# Test direct read speed (bypasses cache)
sudo hdparm -T /dev/sda
```

### Step 4. **fio (Flexible I/O Tester)** - Most comprehensive

Install and use fio for detailed I/O performance testing:

```bash
# Install fio
sudo dnf install fio

# Random read test
fio --name=random-read --ioengine=libaio --iodepth=16 --rw=randread --bs=4k --direct=1 --size=1G --numjobs=1 --runtime=60 --time_based

# Random write test
fio --name=random-write --ioengine=libaio --iodepth=16 --rw=randwrite --bs=4k --direct=1 --size=1G --numjobs=1 --runtime=60 --time_based

# Sequential read test
fio --name=sequential-read --ioengine=libaio --iodepth=16 --rw=read --bs=1M --direct=1 --size=1G --numjobs=1 --runtime=60 --time_based
```

## Questions

1. What is IBM Power Virtual Server in an IBM data centre?
   A. A physical server deployed in minutes.
   B. A virtual server (LPAR) offering flexible, secure, and scalable compute capacity for Power enterprise workloads.
   C. A cloud-native application platform.
   D. A service exclusively for SAP HANA workloads.
2. What is the primary function of Network Security Groups (NSGs) in an IBM Power Virtual Server workspace within an IBM data centre?
3. What is the default high availability solution supported by Power Virtual Server in IBM data centres?
4. Does IBM provide maintenance for the AIX, IBM i, or Linux operating systems running on Power Virtual Server in IBM data centres? If not, whose responsibility, is it?
5. Which IBM Cloud service can be integrated with Power Virtual Server to centrally manage an organization's security, risk, and compliance with regulatory standards and industry benchmarks?
    A. IBM Cloud Object Storage
    B. IBM Cloud Monitoring
    C. IBM Cloud Security and Compliance Center Workload Protection
    D. IBM Cloud IAM
6. What is a Power Edge Router (PER) in the context of Power Virtual Server in IBM data centres, and what are two key benefits of using a PER-enabled workspace?
7. Detail the various storage tiers available in IBM Power Virtual Server and their corresponding IOPS performance. What crucial consideration should be made when selecting a storage tier for production workloads?
8. Describe the architectural role and key benefits of Shared Processor Pools (SPPs) in Power Virtual Server.
9. How is virtual LAN (VLAN) isolation enforced between different tenants within the IBM Power Virtual Server infrastructure in IBM data centres?
10. How can a public network be added or removed from a Power Virtual Server instance in an IBM data centre, and what are the implications of toggling its status?

## Additional Information

### Cloud-init

SELinux can block PostgreSQL from accessing its data directory or configuration files. Without proper contexts, PostgreSQL might fail to start or function correctly. The restorecon commands ensure PostgreSQL files have the correct SELinux labels. The boolean settings allow PostgreSQL to perform necessary operations that SELinux might otherwise block

The cloud-init file we used:

* Installs PostgreSQL - Installs PostgreSQL server and client packages along with contrib modules
* Initializes Database - Runs the initial database setup using postgresql-setup --initdb
* Configures Authentication - Updates pg_hba.conf to allow password authentication for local connections
* Starts Services - Enables and starts the PostgreSQL service
* Creates Your Test Setup - Executes all your specified SQL commands to create:

    * testuser with password testpassword
    * testdb database owned by testuser
    * test_records table with the exact schema you specified
    * Proper privileges for the test user

* Security Configuration - Configures firewall if firewalld is active
* Status Script - Creates a utility script at /usr/local/bin/postgres-status.sh for checking the setup
* SELinux-specific additions:

    * Required Packages - Added policycoreutils-python-utils and setools-console for SELinux management tools
    * SELinux Status Check - Checks if SELinux is enabled before applying configurations
    * File Context Restoration - Uses restorecon -R /var/lib/pgsql/ to set proper SELinux contexts for PostgreSQL files
    * Context Restoration After Config - Restores contexts after modifying configuration files
    * SELinux Booleans - Sets important PostgreSQL-related booleans:

      * postgresql_can_rsync on - Allows PostgreSQL to use rsync for replication
      * nis_enabled on - Allows network connections (useful for future remote access)
