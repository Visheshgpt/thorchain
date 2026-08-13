az network vnet list-usage \
  --resource-group AZRG-ALL-ITS-VNET-SYS \
  --name VNET-CUS-COINIAAKSDEV-10.202.17.192-27 \
  --output table

kubectl delete namespace arweave
kubectl get pv | Select-String "arweave"

az disk list --query "[?diskState=='Unattached'].{name:name,rg:resourceGroup,gb:diskSizeGb}" -o table
az disk delete --name <disk> --resource-group <rg> --yes


az aks nodepool add \
  --resource-group azrg-cus-coinia-dev \
  --cluster-name cuscoiniadevaks \
  --name arweave --node-count 1 \
  --node-vm-size Standard_E2s_v5 --node-osdisk-size 64 \
  --labels workload=arweave \
  --node-taints workload=arweave:NoSchedule


kubectl get nodes -l workload=arweave
kubectl describe node <new-node> | Select-String -Pattern "Taints|memory:" -Context 0,2


kubectl logs -n arweave arweave-node-0 -c install-arweave -f
kubectl logs -n arweave arweave-node-0 -c arweave-node -f

kubectl port-forward -n arweave pod/arweave-node-0 1984:1984


curl.exe -s http://localhost:1984/info | ConvertFrom-Json | Format-List
kubectl top pod arweave-node-0 -n arweave


