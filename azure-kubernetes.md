## Azure Kubernetes Memo  

### Azure Kubernetes node scale up  
This command will scale up your AKS cluster node to 2 nodes.  
```bash
az aks scale --resource-group <MYRESOURCEGROUP> --name <MYAKSCLUSTER> --node-count 2 --nodepool-name <MYNODEPOOLNAME>
```