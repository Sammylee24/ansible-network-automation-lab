# Ansible Multi-AS Network Automation Lab

This repository contains a set of Ansible playbooks designed to build and configure a multi-AS lab network from the ground up. The project demonstrates a data-driven approach to network automation, moving away from static scripts to idempotent, reusable, and scalable Infrastructure as Code (IaC).

The core of this project is to use a central set of YAML data files (a "single source of truth") to define the entire network, allowing for rapid and consistent deployment.

## Features & Capabilities

This automation suite handles the following configurations:

-   **System Foundation:** Deploys and configures an Ubuntu NTP server and synchronizes all network devices as clients.
-   **Device Standardization:** Applies consistent login banners and configures Loopback and point-to-point interfaces.
-   **IGP Underlay:** Establishes a full OSPF routing domain within the core network (AS 65001) to provide internal reachability.
-   **iBGP Overlay:** Builds a scalable, full-mesh iBGP within the core AS using Loopback addresses.
-   **eBGP Peering:** Configures eBGP peering with two external Autonomous Systems (AS 100 & 200).
-   **Policy Control:** Demonstrates conditional network advertisement based on the router's AS.

## Network Topology

The lab consists of 8 routers across 3 Autonomous Systems. The core network (AS 65001) runs iBGP and connects to the external ASNs via eBGP.

```
          +-----------------+          +-----------------+
          |     AS 100      |          |     AS 200      |
          |  (R1) --- (R3)  |          |  (R2) --- (R4)  |
          +------|------|---+          +------|------|---+
                 |      |                    |      |
                 |      | eBGP Peering       |      |
                 |      |                    |      |
          +------|------|--------------------|------|---+
          |      (R5) ======== iBGP ======== (R6)      |
          |       | \                         / |       |
          |       |  \                       /  |       |
          |       |   \                     /   |       |
          |       |    \                   /    |       |
          |       |     \                 /     |       |
          |      (R7) ======== iBGP ======== (R8)      |
          |                                            |
          |                  AS 65001                  |
          +--------------------------------------------+
```

## Prerequisites

-   Ansible 2.9+ installed on a Linux control node.
-   `sshpass` utility installed (`sudo apt-get install sshpass`).
-   Password-based or SSH-key-based connectivity established to all network devices.

## Project Structure

```
.
├── hosts.ini               # Inventory file with router groups
├── network_vars.yml        # All interface, IP, and BGP data
├── ospf-backbone.yaml      # Configures OSPF in the core
├── ibgp-full-mesh.yaml      # Configures iBGP in the core
├── ebgp-peering.yaml       # Configures interfaces and eBGP
└── README.md
```

## Configuration

This project is data-driven, with two main configuration files:

### 1. Inventory (`ebgp-hosts.ini`)

This file defines which routers exist and groups them by their function and AS number.

```ini
[as65001_routers]
R5
R6
R7
R8

[as100_routers]
R1
R3

[as200_routers]
R2
R4

[all_routers:children]
as65001_routers
as100_routers
as200_routers

[all_routers:vars]
ansible_connection = network_cli
ansible_network_os = cisco.ios.ios
ansible_user = your_username
ansible_password = your_password
```

### 2. Variables (`network_vars.yml`)

This file is the "single source of truth" for the network configuration. It contains all IP addresses, interface descriptions, and BGP information.

```yaml
---
interfaces:
  R1:
    bgp_as: 100
    loopback0_ip: '10.0.0.1'
    interfaces:
      - { intf: 'e0/1', desc: 'To_R5', ip: '10.0.15.1', mask: '255.255.255.252', peer_ip: '10.0.15.2', peer_as: 65001 }
  R5:
    bgp_as: 65001
    loopback0_ip: '10.0.0.5'
    interfaces:
      - { intf: 'e0/1', desc: 'To_R1', ip: '10.0.15.2', mask: '255.255.255.252', peer_ip: '10.0.15.1', peer_as: 100 }
      # ... other interfaces
```

## How to Run

The playbooks should be run in a logical order to build the network step-by-step. The `ebgp-peering.yaml` playbook requires an inventory file to be specified with the `-i` flag.

1.  **Configure OSPF in the core network:**
    ```bash
    ansible-playbook ospf-backbone.yaml
    ```

2.  **Configure the iBGP full-mesh in the core:**
    ```bash
    ansible-playbook ibgp-full-mesh.yaml
    ```

3.  **Configure all interfaces and eBGP peerings:**
    ```bash
    ansible-playbook -i ebgp-hosts ebgp-peering.yaml
    ```

## Playbook Descriptions

-   **`ospf-backbone.yaml`**: Configures OSPF on the core routers (AS 65001) to enable reachability between their Loopback interfaces.
-   **`ibgp-full-mesh.yaml`**: Configures a full mesh of iBGP peers between all routers in AS 65001, using their Loopback0 interfaces for peering.
-   **`ebgp-peering.yaml`**: A comprehensive playbook that reads from `network_vars.yml` to:
    -   Configure all physical and Loopback interfaces on every router.
    -   Set up the BGP process on all routers based on their AS number.
    -   Establish eBGP peering sessions for the specified external links.
    -   Conditionally advertise Loopback networks into BGP.
