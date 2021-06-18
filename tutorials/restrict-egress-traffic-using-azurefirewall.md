
### Restrict egress traffic using Azure firewall  
The command below is based on the tutorial from Microsoft documentation in this [link](https://docs.microsoft.com/en-us/azure/aks/limit-egress-traffic#restrict-egress-traffic-using-azure-firewall)  



```bash
PREFIX="aks-egress"
RG="${PREFIX}-rg"
LOC="eastus"
PLUGIN=azure
AKSNAME="${PREFIX}"
VNET_NAME="${PREFIX}-vnet"
AKSSUBNET_NAME="aks-subnet"
# DO NOT CHANGE FWSUBNET_NAME - This is currently a requirement for Azure Firewall.
FWSUBNET_NAME="AzureFirewallSubnet"
FWNAME="${PREFIX}-fw"
FWPUBLICIP_NAME="${PREFIX}-fwpublicip"
FWIPCONFIG_NAME="${PREFIX}-fwconfig"
FWROUTE_TABLE_NAME="${PREFIX}-fwrt"
FWROUTE_NAME="${PREFIX}-fwrn"
FWROUTE_NAME_INTERNET="${PREFIX}-fwinternet"
SUBID=`az account show --query id --output tsv`


# Create Resource Group
az group create --name $RG --location $LOC


# Dedicated virtual network with AKS subnet
az network vnet create \
    --resource-group $RG \
    --name $VNET_NAME \
    --location $LOC \
    --address-prefixes 10.42.0.0/16 \
    --subnet-name $AKSSUBNET_NAME \
    --subnet-prefix 10.42.1.0/24

# Dedicated subnet for Azure Firewall (Firewall name cannot be changed)
az network vnet subnet create \
    --resource-group $RG \
    --vnet-name $VNET_NAME \
    --name $FWSUBNET_NAME \
    --address-prefix 10.42.2.0/24

# Check whether both subnet created successfully
az network vnet subnet list --resource-group $RG --vnet-name $VNET_NAME -o table

#Sample result: 
    AddressPrefix    Name                 PrivateEndpointNetworkPolicies    PrivateLinkServiceNetworkPolicies    ProvisioningState    ResourceGroup
    ---------------  -------------------  --------------------------------  -----------------------------------  -------------------  ---------------
    10.42.1.0/24     aks-subnet           Enabled                           Enabled                              Succeeded            aks-egress-rg
    10.42.2.0/24     AzureFirewallSubnet  Enabled                           Enabled                              Succeeded            aks-egress-rg


# Create a standard SKU public IP resource that will be used as the Azure Firewall frontend address.
az network public-ip create -g $RG -n $FWPUBLICIP_NAME -l $LOC --sku "Standard"

# Check the result 
az network public-ip list -o table
    Name                                  ResourceGroup                    Location    Zones    Address        AddressVersion    AllocationMethod   IdleTimeoutInMinutes    ProvisioningState
    ------------------------------------  -------------------------------  ----------  -------  -------------  ----------------  ------------------  ----------------------  -------------------
    aks-egress-fwpublicip                 aks-egress-rg                    eastus               40.88.XX.XXX  IPv4              Static   4                       Succeeded


# Install Azure Firewall preview CLI extension
az extension add --name azure-firewall

# Deploy Azure Firewall
az network firewall create -g $RG -n $FWNAME -l $LOC --enable-dns-proxy true

# Check command result 
az network firewall list -o table 
    Network.DNS.EnableProxy    Location    Name           ProvisioningState    ResourceGroup    ThreatIntelMode
    -------------------------  ----------  -------------  -------------------  ---------------  -----------------
    true                       eastus      aks-egress-fw  Succeeded            aks-egress-rg    Alert


# Configure Firewall IP Config (it may take several minutes)
az network firewall ip-config create -g $RG -f $FWNAME -n $FWIPCONFIG_NAME --public-ip-address $FWPUBLICIP_NAME --vnet-name $VNET_NAME

# Capture Firewall IP Address for Later Use
FWPUBLIC_IP=$(az network public-ip show -g $RG -n $FWPUBLICIP_NAME --query "ipAddress" -o tsv)
FWPRIVATE_IP=$(az network firewall show -g $RG -n $FWNAME --query "ipConfigurations[0].privateIpAddress" -o tsv)
echo $FWPUBLIC_IP
echo $FWPRIVATE_IP


# Create UDR and add a route for Azure Firewall
az network route-table create -g $RG -l $LOC --name $FWROUTE_TABLE_NAME
az network route-table route create -g $RG --name $FWROUTE_NAME --route-table-name $FWROUTE_TABLE_NAME --address-prefix 0.0.0.0/0 --next-hop-type VirtualAppliance --next-hop-ip-address $FWPRIVATE_IP --subscription $SUBID
az network route-table route create -g $RG --name $FWROUTE_NAME_INTERNET --route-table-name $FWROUTE_TABLE_NAME --address-prefix $FWPUBLIC_IP/32 --next-hop-type Internet



# Add FW Network Rules
az network firewall network-rule create -g $RG -f $FWNAME --collection-name 'aksfwnr' -n 'apiudp' --protocols 'UDP' --source-addresses '*' --destination-addresses "AzureCloud.$LOC" --destination-ports 1194 --action allow --priority 100
az network firewall network-rule create -g $RG -f $FWNAME --collection-name 'aksfwnr' -n 'apitcp' --protocols 'TCP' --source-addresses '*' --destination-addresses "AzureCloud.$LOC" --destination-ports 9000
az network firewall network-rule create -g $RG -f $FWNAME --collection-name 'aksfwnr' -n 'time' --protocols 'UDP' --source-addresses '*' --destination-fqdns 'ntp.ubuntu.com' --destination-ports 123

# Add FW Application Rules
az network firewall application-rule create -g $RG -f $FWNAME --collection-name 'aksfwar' -n 'fqdn' --source-addresses '*' --protocols 'http=80' 'https=443' --fqdn-tags "AzureKubernetesService" --action allow --priority 100


# Associate route table with next hop to Firewall to the AKS subnet
az network vnet subnet update -g $RG --vnet-name $VNET_NAME --name $AKSSUBNET_NAME --route-table $FWROUTE_TABLE_NAME


# Create SP(Service Principal) and Assign Permission to Virtual Network
az ad sp create-for-rbac -n "${PREFIX}sp" --skip-assignment

# Sample result: 
    {
    "appId": "XXXX-XXXX-XXXX-XXXX-XXXX",
    "displayName": "XXXX-XXXX-XXXX-XXXX-XXXX",
    "name": "XXXX-XXXX-XXXX-XXXX-XXXX",
    "password": "XXXX-XXXX-XXXX-XXXX-XXXX",
    "tenant": "XXXX-XXXX-XXXX-XXXX-XXXX"
    }


If ERROR, change the name of your Service Principal  
More than one application have the same display name "XXXX-XXXX-XXXX-XXXX-XXXX",please remove them first  



APPID="<SERVICE_PRINCIPAL_APPID_GOES_HERE>"
PASSWORD="<SERVICEPRINCIPAL_PASSWORD_GOES_HERE>"
VNETID=$(az network vnet show -g $RG --name $VNET_NAME --query id -o tsv)

# Assign SP (Service Principal) Permission to VNET
az role assignment create --assignee $APPID --scope $VNETID --role "Network Contributor"


# Give the AKS service principal to the pre-created route table 
RTID=$(az network route-table show -g $RG -n $FWROUTE_TABLE_NAME --query id -o tsv)
az role assignment create --assignee $APPID --scope $RTID --role "Network Contributor"



## DEPLOY AKS!!
SUBNETID=$(az network vnet subnet show -g $RG --vnet-name $VNET_NAME --name $AKSSUBNET_NAME --query id -o tsv)

az aks create -g $RG -n $AKSNAME -l $LOC \
  --node-count 3 --generate-ssh-keys \
  --network-plugin $PLUGIN \
  --vm-set-type VirtualMachineScaleSets \
  --load-balancer-sku Standard \
  --outbound-type userDefinedRouting \
  --service-cidr 10.41.0.0/16 \
  --dns-service-ip 10.41.0.10 \
  --docker-bridge-address 172.17.0.1/16 \
  --vnet-subnet-id $SUBNETID \
  --service-principal $APPID \
  --client-secret $PASSWORD \
  --api-server-authorized-ip-ranges $FWPUBLIC_IP

```