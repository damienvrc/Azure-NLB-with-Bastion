# Azure-NLB-with-Bastion
Azure Network Load Balancer scenario with 3 web front IIS servers and one Azure Bastion form remote access

## Ressource group

az group create -l northeurope -n WORKSHOP

## VNET and SUBNETS
 
- VNET : az network vnet create -g WORKSHOP -n vnet_01 --address-prefix 172.16.3.0/24

- Subnet 01 : az network vnet subnet create --address-prefix 172.16.3.16/28 --name sub_01 --resource-group WORKSHOP --vnet-name vnet_01
- Subnet 02 : az network vnet subnet create --address-prefix 172.16.3.32/28 --name sub_02 --resource-group WORKSHOP --vnet-name vnet_01 

- Subnet Bastion : az network vnet subnet create --resource-group WORKSHOP --name AzureBastionSubnet --vnet-name vnet_01 --address-prefixes 172.16.3.48/28

## Bastion Public IP

az network public-ip create --resource-group WORKSHOP --name bastion-ip --sku Standard

## PaaS Bastion (15min)

az network bastion create --resource-group WORKSHOP --name Bastion01 --public-ip-address bastion-ip --vnet-name vnet_01 --location northeurope

## WEB VM (Windows 2019 with IIS)

### VM NSG
az network nsg create --resource-group WORKSHOP --name web-nsg

### NSG Rules
az network nsg rule create --resource-group WORKSHOP --nsg-name web-nsg --name webhttprule --protocol '*' --direction inbound --source-address-prefix '*' --source-port-range '*' --destination-address-prefix '*' --destination-port-range 80 --access allow --priority 200

### VM NIC
az network nic create --resource-group HAVAS-WORKSHOP --name nicVM01 --vnet-name vnet_01 --subnet sub_01 --network-security-group web-nsg
az network nic create --resource-group HAVAS-WORKSHOP --name nicVM02 --vnet-name vnet_01 --subnet sub_01 --network-security-group web-nsg
az network nic create --resource-group HAVAS-WORKSHOP --name nicVM03 --vnet-name vnet_01 --subnet sub_01 --network-security-group web-nsg

### Windows VM
az vm create --resource-group WORKSHOP --name VM01 --nics nicVM01 --image win2019datacenter --admin-username workshop --zone 1 --no-wait
az vm create --resource-group WORKSHOP --name VM02 --nics nicVM02 --image win2019datacenter --admin-username workshop --zone 1 --no-wait
az vm create --resource-group WORKSHOP --name VM03 --nics nicVM03 --image win2019datacenter --admin-username workshop --zone 1 --no-wait

## Network Load Balancer

### LoadBalancer public IP
az network public-ip create --resource-group WORKSHOP --name external-lb-ip --sku Standard --zone 1

### External LoadBalancer
az network lb create --resource-group WORKSHOP --name external-lb --sku Standard --public-ip-address external-lb-ip --frontend-ip-name FrontEnd --backend-pool-name BackEndPool

### Helth Check
az network lb probe create --resource-group WORKSHOP --lb-name external-lb --name HTTPHealthCheck --protocol tcp --port 80  

### LoadBalancer rule
az network lb rule create --resource-group WORKSHOP --lb-name external-lb --name HTTPRule --protocol tcp --frontend-port 80 --backend-port 80 --frontend-ip-name FrontEnd --backend-pool-name BackEndPool --probe-name HTTPHealthCheck --disable-outbound-snat true --idle-timeout 5 --enable-tcp-reset true

### Add VM to backend pool
az network nic ip-config address-pool add --address-pool BackEndPool --ip-config-name ipconfig1 --nic-name nicVM01 --resource-group WORKSHOP --lb-name external-lb
az network nic ip-config address-pool add --address-pool BackEndPool --ip-config-name ipconfig1 --nic-name nicVM02 --resource-group WORKSHOP --lb-name external-lb
az network nic ip-config address-pool add --address-pool BackEndPool --ip-config-name ipconfig1 --nic-name nicVM03 --resource-group WORKSHOP --lb-name external-lb

### Outbound NAT Public IP 
az network public-ip create --resource-group WORKSHOP --name PublicIPOutbound --sku Standard --zone 1

### LB config
az network lb frontend-ip create --resource-group WORKSHOP --name FrontEndOutbound --lb-name external-lb --public-ip-address PublicIPOutbound

### LB Backend Pool
az network lb address-pool create --resource-group WORKSHOP --lb-name external-lb --name BackendPoolOutbound

### LB Outbound Rule
az network lb outbound-rule create --resource-group WORKSHOP --lb-name external-lb --name OutboundRule --frontend-ip-configs FrontEndOutbound --protocol All --idle-timeout 5 --outbound-ports 10000 --address-pool BackEndPoolOutbound

### Add VM to Outbound pool
az network nic ip-config address-pool add --address-pool BackendPoolOutbound --ip-config-name ipconfig1 --nic-name nicVM01 --resource-group WORKSHOP --lb-name external-lb
az network nic ip-config address-pool add --address-pool BackendPoolOutbound --ip-config-name ipconfig1 --nic-name nicVM02 --resource-group WORKSHOP --lb-name external-lb
az network nic ip-config address-pool add --address-pool BackendPoolOutbound --ip-config-name ipconfig1 --nic-name nicVM03 --resource-group WORKSHOP --lb-name external-lb

## IIS Install via Bastion

Add-WindowsFeature Web-Server
Add-Content -Path C:\inetpub\wwwroot\Default.htm -Value $($env:computername)

## IIS loadbalancer site access
az network public-ip show --resource-group WORKSHOP --name havas-external-lb-ip --query ipAddress --output tsv

Copie IP in web browser 







