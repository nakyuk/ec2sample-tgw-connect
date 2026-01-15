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

| Key | Example | Description |
|-----|-------|-------------|
| `TGW_IP_GRE` | `10.0.0.1` | Transit Gateway Connect peer IP address |
| `TGW_IP_BGP` | `169.254.0.0/29` | Inside CIDR block for BGP peering |
| `TGW_ASN` | `64500` | Transit Gateway ASN |
| `PEER_ASN` | `65000` | EC2 instance BGP ASN |


```bash
CONNECT_PEER_ID="tgw-connect-peer-01234567890123456"
INSTANCE_ID="i-01234567890123456"

PEER_INFO=$(aws ec2 describe-transit-gateway-connect-peers \
    --transit-gateway-connect-peer-ids $CONNECT_PEER_ID \
    --output json)

TGW_IP_GRE=$(echo $PEER_INFO | jq -r '.TransitGatewayConnectPeers[0].ConnectPeerConfiguration.TransitGatewayAddress')
TGW_IP_BGP=$(echo $PEER_INFO | jq -r '.TransitGatewayConnectPeers[0].ConnectPeerConfiguration.InsideCidrBlocks[0]')
TGW_ASN=$(echo $PEER_INFO | jq -r '.TransitGatewayConnectPeers[0].ConnectPeerConfiguration.BgpConfigurations[0].TransitGatewayAsn')
PEER_ASN=$(echo $PEER_INFO | jq -r '.TransitGatewayConnectPeers[0].ConnectPeerConfiguration.BgpConfigurations[0].PeerAsn')

aws ec2 create-tags \
    --resources $INSTANCE_ID \
    --tags \
        Key=TGW_IP_GRE,Value=$TGW_IP_GRE \
        Key=TGW_IP_BGP,Value=$TGW_IP_BGP \
        Key=TGW_ASN,Value=$TGW_ASN \
        Key=PEER_ASN,Value=$PEER_ASN
```


## Step 2: Configure EC2 Instance

Run the following command on the EC2 instance

```bash
#!/bin/bash
sleep 30
sudo apt -y update
sudo apt install -y frr
wget https://github.com/nakyuk/ec2sample-tgw-connect/archive/refs/heads/main.zip
unzip main.zip
cd ec2sample-tgw-connect-main/
sudo bash setup.sh
```

### Check GRE Tunnel

```bash
# Check GRE Tunnel
ip addr show gre1

# Check BGP Status
sudo vtysh -c "show ip bgp summary"
```
