# Azure-Firewall-with-forced-tunneling

A step-by-step guide to deploying an Azure Firewall with forced tunneling, User Defined Routes (UDR), NSG, and a Linux VM. All CLI commands use placeholders; replace with your own resource names and values.

---

## Resources & Naming Conventions

* **Resource Group**: `rg-firewall-demo`
* **Virtual Network**: `vnet-firewall-demo` (10.0.0.0/16)

  * Subnet **AzureFirewallSubnet**: 10.0.1.0/24
  * Subnet **WorkloadSubnet**: 10.0.2.0/24
* **Azure Firewall**: `fw-demo` (private IP 10.0.1.4)
* **Route Table**: `rt-fw-demo`
* **Network Security Group**: `nsg-workload`
* **Linux VM**: `vm-workload` in WorkloadSubnet

![rg-azure-firewall-demo](https://github.com/user-attachments/assets/259766b2-64c7-4f5b-9b65-6185149e1250)

---

## 1. Create Resource Group & VNet

```bash
az group create \
  --name rg-firewall-demo \
  --location eastus2

az network vnet create \
  --resource-group rg-firewall-demo \
  --name vnet-firewall-demo \
  --address-prefix 10.0.0.0/16 \
  --subnet-name AzureFirewallSubnet \
  --subnet-prefix 10.0.1.0/24

az network vnet subnet create \
  --resource-group rg-firewall-demo \
  --vnet-name vnet-firewall-demo \
  --name WorkloadSubnet \
  --address-prefix 10.0.2.0/24
```

## 2. Deploy Azure Firewall & Public IP

```bash
# Public IP for Firewall
az network public-ip create \
  --resource-group rg-firewall-demo \
  --name fw-demo-pip \
  --sku Standard \
  --allocation-method Static

# Firewall
az network firewall create \
  --resource-group rg-firewall-demo \
  --name fw-demo \
  --location eastus2

az network firewall ip-config create \
  --resource-group rg-firewall-demo \
  --firewall-name fw-demo \
  --name fw-demo-ipcfg \
  --public-ip-address fw-demo-pip \
  --vnet-name vnet-firewall-demo
```

## 3. Create Route Table & Default Route

```bash
az network route-table create \
  --resource-group rg-firewall-demo \
  --name rt-fw-demo \
  --location eastus2

az network route-table route create \
  --resource-group rg-firewall-demo \
  --route-table-name rt-fw-demo \
  --name default-route \
  --address-prefix 0.0.0.0/0 \
  --next-hop-type VirtualAppliance \
  --next-hop-ip-address 10.0.1.4   # Firewall private IP
```

## 4. Associate Route Table to WorkloadSubnet

```bash
az network vnet subnet update \
  --resource-group rg-firewall-demo \
  --vnet-name vnet-firewall-demo \
  --name WorkloadSubnet \
  --route-table rt-fw-demo
```
![image](https://github.com/user-attachments/assets/61f44b02-b4a0-4c5d-9d4a-b1fc896ed89d)

## 5. Create NSG & SSH Rule

```bash
az network nsg create \
  --resource-group rg-firewall-demo \
  --name nsg-workload

az network nsg rule create \
  --resource-group rg-firewall-demo \
  --nsg-name nsg-workload \
  --name Allow-SSH-From-Office \
  --priority 100 \
  --direction Inbound \
  --access Allow \
  --protocol Tcp \
  --source-address-prefixes YOUR_OFFICE_IP/32 \
  --destination-port-ranges 22

az network vnet subnet update \
  --resource-group rg-firewall-demo \
  --vnet-name vnet-firewall-demo \
  --name WorkloadSubnet \
  --network-security-group nsg-workload
```
![image](https://github.com/user-attachments/assets/6e7e718d-438a-43d0-a197-234497856026)

## 6. Deploy Linux VM

```bash
az vm create \
  --resource-group rg-firewall-demo \
  --name vm-workload \
  --image UbuntuLTS \
  --vnet-name vnet-firewall-demo \
  --subnet WorkloadSubnet \
  --nsg nsg-workload \
  --public-ip-address vm-workload-pip \
  --generate-ssh-keys

# Open port 80 (or your test port)
az vm open-port \
  --resource-group rg-firewall-demo \
  --name vm-workload \
  --port 80
```
![image](https://github.com/user-attachments/assets/e00b0281-110c-4850-ba1b-877379d50efc)

## 7. Verification

1. **Inbound (DNAT)**: SSH to `vm-workload-pip` → you reach the VM.
2. **Outbound (SNAT)**: On VM run `curl https://ifconfig.me` → returns your `fw-demo-pip` public IP.

![image](https://github.com/user-attachments/assets/cbb39ed0-3c0d-4c33-8424-243011b303ae)


---

## 8. Cleanup

```bash
az group delete --name rg-firewall-demo --yes --no-wait
```

