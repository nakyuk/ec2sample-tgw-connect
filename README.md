# AWS Transit Gateway Connect with FRR

> **⚠️ DISCLAIMER: FOR TESTING PURPOSES ONLY**
> 
> This configuration is intended for **testing and development environments only** and is **NOT suitable for production use**. 
> 
> **Important considerations:**
> - No high availability or redundancy configuration
> - Single point of failure (single EC2 instance)
> - No automated failover mechanisms
> - Security configurations may need hardening for production
> - Performance tuning required for production workloads
> - No monitoring or alerting setup included
> 
> For production deployments, consult AWS Well-Architected Framework and implement proper redundancy, monitoring, security controls, and operational procedures.

This guide demonstrates how to set up AWS Transit Gateway Connect using FRR (Free Range Routing) on Ubuntu 22.04.

## Prerequisites

- AWS Account with appropriate permissions
- VPC with a subnet
- Transit Gateway
- Ubuntu 22.04 AMI
- AWS CLI configured (optional, for automated setup)

## Step 1: Create AWS Resources

1. Launch an Ubuntu 22.04 instance with the following specifications:

- Enable "Allow tags in instance metadata"
- Ensure the following network connectivity between Transit Gateway

2. Create Transit Gateway Connect Attachment & Peer

3. Add the following tags to your EC2 instance:

| Key | Description |
|-----|-------------|
| `TGW_IP_GRE` | Transit Gateway Connect peer IP address |
| `TGW_IP_BGP` | Inside CIDR block for BGP peering |
| `TGW_ASN` | Transit Gateway ASN |
| `PEER_ASN` | EC2 instance BGP ASN |


## Step 2: Configure EC2 Instance

```bash
#!/bin/bash
sudo apt -y update
sudo apt install -y frr

TOKEN=`curl -s -X PUT "http://169.254.169.254/latest/api/token" -H "X-aws-ec2-metadata-token-ttl-seconds: 21600"`
export INSTANCE_IP=`curl -s -H "X-aws-ec2-metadata-token: $TOKEN" http://169.254.169.254/latest/meta-data/local-ipv4`
export TGW_IP_GRE=`curl -s -H "X-aws-ec2-metadata-token: $TOKEN" http://169.254.169.254/latest/meta-data/tags/instance/TGW_IP_GRE`
export TGW_IP_BGP=`curl -s -H "X-aws-ec2-metadata-token: $TOKEN" http://169.254.169.254/latest/meta-data/tags/instance/TGW_IP_BGP`
export TGW_ASN=`curl -s -H "X-aws-ec2-metadata-token: $TOKEN" http://169.254.169.254/latest/meta-data/tags/instance/TGW_ASN`
export PEER_ASN=`curl -s -H "X-aws-ec2-metadata-token: $TOKEN" http://169.254.169.254/latest/meta-data/tags/instance/PEER_ASN`

CIDR=$TGW_IP_BGP
NETWORK=$(echo $CIDR | cut -d'/' -f1)
IFS='.' read -r i1 i2 i3 i4 <<< "$NETWORK"
EC2_BGP_IP="$i1.$i2.$i3.$((i4 + 1))"
TGW_BGP_IP1="$i1.$i2.$i3.$((i4 + 2))"
TGW_BGP_IP2="$i1.$i2.$i3.$((i4 + 3))"

# Configure GRE Tunnel
sudo tee /etc/netplan/99-gre.yaml << EOF
network:
  version: 2
  tunnels:
    gre1:
      mode: gre
      local: $INSTANCE_IP
      remote: $TGW_IP_GRE
      ttl: 255
      addresses:
        - $EC2_BGP_IP/29
EOF
sudo chmod 600 /etc/netplan/99-gre.yaml
sudo netplan apply

# Configure FRR
sudo sed -i 's/bgpd=no/bgpd=yes/' /etc/frr/daemons
sudo tee /etc/frr/frr.conf << EOF
frr version 8.4.4
frr defaults traditional
hostname tgw-connect-peer
log syslog informational
!
interface gre1
 description TGW Connect Tunnel
exit
!
router bgp $PEER_ASN
 bgp router-id $INSTANCE_IP
 neighbor $TGW_BGP_IP1 remote-as $TGW_ASN
 neighbor $TGW_BGP_IP1 ebgp-multihop 255
 neighbor $TGW_BGP_IP1 timers 10 30
 neighbor $TGW_BGP_IP2 remote-as $TGW_ASN
 neighbor $TGW_BGP_IP2 ebgp-multihop 255
 neighbor $TGW_BGP_IP2 timers 10 30
 !
 address-family ipv4 unicast
  neighbor $TGW_BGP_IP1 route-map ALLOW-ALL in
  neighbor $TGW_BGP_IP1 route-map ALLOW-ALL out
  neighbor $TGW_BGP_IP2 route-map ALLOW-ALL in
  neighbor $TGW_BGP_IP2 route-map ALLOW-ALL out
 exit-address-family
exit
!
route-map ALLOW-ALL permit 10
exit
!
line vty
!
EOF

sudo chown frr:frr /etc/frr/frr.conf
sudo chmod 640 /etc/frr/frr.conf

sudo systemctl enable frr
sudo systemctl restart frr
```

### Check GRE Tunnel

```bash
# Check GRE Tunnel
ip addr show gre1

# Check BGP Status
sudo vtysh -c "show ip bgp summary"
```
