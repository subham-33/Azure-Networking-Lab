### IP Address Plan

|    Resource    | Address Space |                      Notes                        |
|      :---      |     :---      |                       :---                        |
|      VNet      |  10.20.0.0/16 |  65,534 usable IPs — big enough to carve subnets  |
|    VM Subnet   |  10.20.1.0/24 |  251 usable IPs (Azure reserves 5 per subnet)     |
|   JumpBox VM   |   10.20.1.4   | First usable IP (Azure reserves .0 .1 .2 .3 .255) |
|      VM2       |   10.20.1.5   |                   Private only                    |
|      VM3       |   10.20.1.6   |                   Private only                    |
|      VM4       |   10.20.1.7   |                   Private only                    |


### Resource Naming Plan

|      Resource     |       Name        |
|       :---        |       :---        |
|   Resource Group  |   rg-private-lab  |
|       VNet        |      vnet-lab     |
|      Subnet       |      snet-vms     |
|   Subnet-jumpbox  |     snet-jumpbox  |
|   NSG (JumpBox)   |    nsg-jumpbox    |
| NSG (Private VMs) |    nsg-private    |
|    Route Table    | rt-private-subnet |
|     JumpBox VM    |     vm-jumpbox    |
|        VM2        |   vm-private-01   |
|        VM3        |   vm-private-02   |
|        VM4        |   vm-private-03   |
