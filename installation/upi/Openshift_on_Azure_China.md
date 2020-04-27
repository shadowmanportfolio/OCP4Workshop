# Openshift on Azure China



截至目前Microsoft和红帽官方正在密切合作，紧锣密鼓地推进OCP 4在Azure全球和Azure中国的技术认证工作。在Azure中国区域中，裸金属安装方式已经通过认证并获得官方技术支持，对IPI安装方式的支持还在推进当中。本文讨论的是在中国区使用UPI方式进行安装的过程，这个功能目前处于技术预览阶段，相信很快就能GA并同广大客户见面。这次尝试我们采用的是Openshift 4.3企业版最新版，内容仅供对广大技术用户和技术爱好者参考，最终信息还以官方公布为准。



## 安装准备和环境规划

### 创建Azure账户

过程省略，可参考链接内容操作

https://github.com/openshift/installer/blob/master/docs/user/azure/account.md

### 准备域名

没有现成可用域名需要自行申请，如：xxx.online。这个是根域，也可使用子域，cluster.xxx.online。

用此域名在Azure创建公共DNS区域，记下四个公共服务器域名。再到域名服务商的管理服务自助页面将原有的Nameserver替换为上述四个Azure服务器域名。

大约等待10多分钟后，DNS配置数据被自动同步到Server，我们用命令验证一下：

```shell
$ nslookup -type=SOA xxx.online

Server:         168.63.129.16
Address:        168.63.129.16#53

Non-authoritative answer:
ocp4.online
        origin = ns1-06.azure-dns-1.cn
        mail addr = azuredns-hostmaster.microsoft.com
        serial = 1
        refresh = 3600
        retry = 300
        expire = 2419200
        minimum = 300

Authoritative answers can be found from:
ns1-06.azure-dns-1.cn   internet address = 40.73.192.6
```

### 账户限制

Azure账户的资源限制会对OCP集群资源分配有影响，下面链接内容可以先了解下。

https://docs.microsoft.com/en-us/azure/azure-resource-manager/management/azure-subscription-service-limits

### 相关资源

具体OCP集群安装中需要用到的资源如下：

#### 虚拟网络

每个群集都创建自己的VNet。每个区域的VNet的默认限制为1000，这表示最多允许创建1000个群集。当需要1000个以上的群集时，就需要增加此限制。默认每个集群安装将创建一个VNet。

#### 网络接口

安装默认规模集群时将创建6个网络接口。每个区域的默认限制是65536。使用这些网络接口的目的是用于绑定虚拟网络中的主机节点和负载均衡器。

#### 公共和私有IP地址

默认情况下，安装程序会[在一个region的所有可用zone内](https://azure.microsoft.com/en-us/global-infrastructure/availability-zones/)分布控制平面和计算机，来配置群集高可用性。可以参阅[此地图](https://azure.microsoft.com/en-us/global-infrastructure/regions/)以获取具有可用区计数的当前区域地图，建议选择具有3个或更多可用zone的region。目前中国还没提供多个zone的region，因此这个建议对中国不适用。

安装程序将创建两个外部负载均衡器和一个内部负载均衡器。每个外部负载均衡器都有一个公共IP地址，而内部负载均衡器都有一个私有IP地址。在VNet中创建两个子网，一个用于控制平面节点，另一个用于计算节点。

每个VM都有一个专用IP地址。默认安装会为总共6个私有地址创建6个VM。内部负载平衡器具有私有IP地址，而外部负载平衡器具有公共IP地址。总共为7个私有IP地址和2个公共IP地址。

在安装过程中还会为bootstrap计算机创建一个公共IP地址。以便在安装过程中出现任何问题，可以通过SSH排错。安装完成后，为节省资源，bootstrap计算机和公用IP地址就可以删除了。

#### 网络安全组

每个群集为VNet中的每个子网创建网络安全组。默认安装会为控制平面和计算节点子网分别创建网络安全组。帐户的默认限制为5000，这可以支持创建大量群集。默认安装的网络安全组为：

1. controlplane
   - 允许从任意位置访问控制平面端口6443；
2. node
   - 允许在Internet通过端口80和443访问工作节点。

#### 集群结点组成

默认情况下，集群将创建：

- 一台Standard_D4s_v3 bootstrap 结点（安装后删除）
- 三个Standard_D8s_v3 master节点。
- 三个Standard_D2s_v3 worker节点。

VM大小（Dsv3系列）的规格如下：

- Standard_D8s_v3
  - 8个vCPU，32GiB内存
  - IOP /吞吐量（Mbps）：（已缓存）16000/128
  - IOP /吞吐量（Mbps）：（未缓存）12800/192
  - NIC /带宽（Mbps）：4/4000
  - 64 GiB临时存储（SSD）
  - 最多16个数据磁盘
- Standard_D4s_v3
  - 4个vCPU，16GiB内存
  - IOP /吞吐量（Mbps）：（已缓存）8000/512
  - IOP /吞吐量（Mbps）：（未缓存）6400/1152
  - NIC /带宽（Mbps）：2/2000
  - NIC /带宽（Mbps）：2/1000
  - 32 GiB临时存储（SSD）
  - 最多8个数据磁盘
- Standard_D2s_v3
  - 2个vCPU，8GiB内存
  - IOP /吞吐量（Mbps）：（已缓存）4000/256
  - IOP /吞吐量（Mbps）：（未缓存）3200/384
  - NIC /带宽（Mbps）：2/1000
  - 16 GiB临时存储（SSD）
  - 最多4个数据磁盘

有关VM大小的更多详细信息，请参见[此处](https://docs.microsoft.com/en-us/azure/virtual-machines/windows/sizes-general)。

azure订阅默认允许20个vCPU，而OCP4至少需要[使用](https://github.com/openshift/installer/blob/master/docs/user/azure/limits.md#increasing-limits)22个。如果打算增加更多的worker节点，开启自动扩展和大型工作负载或其他实例类型，就要确保在实例类型的限制数量内拥有可用的资源。可通过提交case要求Azure来放宽限制。

#### 负载均衡

负载均衡器相关信息在[此处](https://github.com/openshift/installer/blob/master/docs/user/azure/limits.md#load-balancing)。默认情况下，每个群集将创建3个网络负载平衡器。每个区域对LB的默认限制是1000。集群需要创建以下负载均衡器：

1. 默认

- 公用IP地址，用于负载均衡工作节点之间对端口80和443的请求

2. 内部

- 内部IP地址，用于负载均衡控制平面节点上对端口6443和22623的请求

3. 外部

- 公用IP地址，用于负载均衡控制平面节点上对端口6443的请求

其他Kuberntes LoadBalancer服务对象将创建其他[负载平衡器](https://docs.microsoft.com/en-us/azure/load-balancer/load-balancer-overview)。

### Service Principal

在继续进行OpenShift安装之前，应按照此处链接概述的步骤为订阅创建具有管理权限的service Principal。

[Azure：创建Service Principal](https://docs.microsoft.com/en-us/azure-stack/user/azure-stack-create-service-principals)

#### 步骤1：创建Service Principal

可以使用Azure [门户](https://docs.microsoft.com/en-us/azure-stack/user/azure-stack-create-service-principals#create-service-principal-for-azure-ad)或Azure [CLI](https://docs.microsoft.com/en-us/cli/azure/create-an-azure-service-principal-azure-cli?view=azure-cli-latest#create-a-service-principal) 来创建

```shell
az ad sp create-for-rbac --name ServicePrincipalOCP4
Changing "ServicePrincipalOCP4" to a valid URI of "http://ServicePrincipalOCP4", which is the required format used for service principal names
Creating a role assignment under the scope of "/subscriptions/XXXXXXXX-XXXX-XXXX-XXXX-XXXXXXXXXXXX"
  Retrying role assignment creation: 1/36
AppId                                 DisplayName           Name                         Password                              Tenant
------------------------------------  --------------------  ---------------------------  ------------------------------------  ------------------------------------
XXXXXXXX-XXXX-XXXX-XXXX-XXXXXXXXXXXX  ServicePrincipalOCP4  http://ServicePrincipalOCP4  XXXXXXXX-XXXX-XXXX-XXXX-XXXXXXXXXXXX  XXXXXXXX-XXXX-XXXX-XXXX-XXXXXXXXXXXX
```

#### 步骤2：向租户管理员请求Service Principal的权限

为了向OCP群集中的组件提供正确的安全凭证，需要先向Service Principal申请`Azure Active Directory Graph -> Application.ReadWrite.OwnedBy`应用程序[权限](https://docs.microsoft.com/en-us/azure/active-directory/develop/v1-permissions-and-consent)，然后才能在Azure上部署OpenShift。可以使用Azure门户或Azure cli来操作。

##### 使用Azure CLI请求权限

使用以下命令找到Service Principal的AppId：

```
$  az ad sp show --id <Service Principal ID> -otable
AccountEnabled    AppDisplayName        AppId                                 AppOwnerTenantId                      AppRoleAssignmentRequired    DisplayName           Homepage                      ObjectId                              ObjectType        Odata.metadata                                                                                           Odata.type                                    PublisherName    ServicePrincipalType    SignInAudience
----------------  --------------------  ------------------------------------  ------------------------------------  ---------------------------  --------------------  ----------------------------  ------------------------------------  ----------------  -------------------------------------------------------------------------------------------------------  --------------------------------------------  ---------------  ----------------------  ----------------
True              ServicePrincipalOCP4  XXXXXXXX-XXXX-XXXX-XXXX-XXXXXXXXXXXX  XXXXXXXX-XXXX-XXXX-XXXX-XXXXXXXXXXXX  False                        ServicePrincipalOCP4  https://ServicePrincipalOCP4  XXXXXXXX-XXXX-XXXX-XXXX-XXXXXXXXXXXX  ServicePrincipal  https://graph.chinacloudapi.cn/XXXXXXXX-XXXX-XXXX-XXXX-XXXXXXXXXXXX/$metadata#directoryObjects/@Element  Microsoft.DirectoryServices.ServicePrincipal  John.Doe    Application             AzureADMyOrg
```

使用可以通过以下方式请求`Application.ReadWrite.OwnedBy`许可权限

```
az ad app permission add --id <AppId> --api 00000002-0000-0000-c000-000000000000 --api-permissions 824c81eb-e3f8-4ee6-8f6d-de7f50d565b7=Role
```



#### 步骤3：附加管理角色

Azure安装程序为群集创建了新的标识，因此需要访问权限才能创建和分配新的角色。这要求服务Service Principal至少具有`Contributor`和`User Access Administrator` [角色](https://docs.microsoft.com/en-us/azure/role-based-access-control/built-in-roles)。

可以使用Azure [门户](https://docs.microsoft.com/en-us/azure-stack/user/azure-stack-create-service-principals#assign-the-service-principal-to-a-role)或Azure [CLI](https://docs.microsoft.com/en-us/cli/azure/create-an-azure-service-principal-azure-cli?view=azure-cli-latest#manage-service-principal-roles)为Service Principal创建角色分配

```shell 
az role assignment list --assignee <APP_ID>
# Contributor role creation
az role assignment create --assignee <APP_ID> --role Contributor
# User Access Adminstrator role creation
az role assignment create --assignee <APP_ID> --role "User Access Administrator"
```

#### 步骤4：获取Client Secret

需要用之前保存下来的Client Secret值来配置本地计算机以运行安装程序。如果错过步骤未得到这些信息，也可以为Azure门户中的Service Principal添加新的凭证。可以使用Azure [门户](https://docs.microsoft.com/en-us/azure-stack/user/azure-stack-create-service-principals#get-credentials)或Azure [CLI](https://docs.microsoft.com/en-us/cli/azure/create-an-azure-service-principal-azure-cli?view=azure-cli-latest#reset-credentials)获取Service Principal的Client Secret。

## 配置

### 创建密钥

选择将来用于登录集群的ssh key。如果没有现成可用的，可以用下面的命令生成。

```
ssh-keygen -t rsa -b 4096 -N '' -f ocp4/ssh.key
```

指明生成的key的位置，例如"ocp4/ssh.key"。复制密钥供后续定义初始化文件时使用。



### 创建安装配置

#### 定制配置文件

我们提供一个现成[准备](https://github.com/openshift/installer/blob/master/docs/user/azure/customization.md)好的[install-config.yaml](https://github.com/openshift/installer/blob/master/docs/user/overview.md#multiple-invocations)文件，在当中使用特定区域。

```shell
apiVersion: v1
baseDomain: xxx.online
compute:
- hyperthreading: Enabled
  name: worker
  platform: {}
  replicas: 0
controlPlane:
  hyperthreading: Enabled
  name: master
  platform: {}
  replicas: 3
metadata:
  creationTimestamp: null
  name: testcluster
networking:
  clusterNetwork:
  - cidr: 10.128.0.0/14
    hostPrefix: 23
  machineCIDR: 10.0.0.0/24
  networkType: OpenShiftSDN
  serviceNetwork:
  - 172.30.0.0/16
platform:
  azure:
    region: chinaeast2
    baseDomainResourceGroupName: ocp4
publish: External
pullSecret: '............................................................................................................................................................................................'
sshKey: 'ssh-rsa ....................................................................................................'
```

定义的内容包含**云平台无关配置内容**，如OpenShift，Kubernetes，Platform，OS部分，详细可以参考 https://github.com/openshift/installer/blob/master/docs/user/customization.md#examples，与**Azur平台相关配置内容**的详细信息可以参考https://github.com/openshift/installer/blob/master/docs/user/azure/customization.md

### 从install config定义环境变量

后续安装步骤会用到install configuration文件中定义的一些参数信息，所以要用Export生成相应的环境变量：

```shell
export CLUSTER_NAME=`yq r install-config.yaml 'metadata.name'`
export AZURE_REGION=`yq r install-config.yaml 'platform.azure.region'`
export SSH_KEY=`yq r install-config.yaml 'sshKey'| xargs`
export BASE_DOMAIN=`yq r install-config.yaml 'baseDomain'`
export BASE_DOMAIN_RESOURCE_GROUP=`yq r install-config.yaml 'platform.azure.baseDomainResourceGroupName'`
```

创建一个空文件夹，例如20200324，把install-config.yaml文件复制进去。然后运行下面的命令，生成集群的Kubernetes manifest

```shell
openshift-install create manifests --dir=20200324
```

删除control plane和worker node相关的声明文件, 目的是屏蔽自动生成资源步骤，而采用后续手工方式创建资源。

```shell
rm -f 20200324/openshift/99_openshift-cluster-api_master-machines-*.yaml
rm -f 20200324/openshift/99_openshift-cluster-api_worker-machineset-*.yaml
```

编辑20200324/manifests/cluster-scheduler-02-config.yml，把mastersSchedulable的值改为false.

编辑20200324/manifests/cluster-dns-02-config.yml文件，注释掉privateZone和publicZone部分。这样后面我们将单独添加ingress DNS记录。

### Resource Group 和 Infra ID

安装程序为OpenShift 集群分配了一个带 `-`号的唯一标识，这个就是"Infra ID"。安装过程创建的所有相关资源都是在这个标识的基础上命名的。用Export定义Infra ID环境变量以备后用：

```shell
export INFRA_ID=`yq r 20200324/manifests/cluster-infrastructure-02-config.yml 'status.infrastructureName'`
```

同时，所有在Azure部署的资源都应该从属于 [resource group](https://docs.microsoft.com/en-us/azure/azure-resource-manager/management/overview#resource-groups). Resource group名也是以Infra ID为命名基础的，是在其名称后面加上 `-rg`。用Export定义resource group环境变量以备后用：

```shell
export RESOURCE_GROUP=`yq r 20200324/manifests/cluster-infrastructure-02-config.yml 'status.platformStatus.azure.resourceGroupName'`
```

### 生成Ignition文件

运行下面的命令，生成ignition文件。

```shell
openshift-install create ignition-configs --dir=20200324
```

### 创建Resource Group 和 identity

使用以下命令在Azure region中创建resource group：

```shell
az group create --name $RESOURCE_GROUP --location $AZURE_REGION
```

还要创建identity，它将会被用来给集群operator授权：

```shell
az identity create -g $RESOURCE_GROUP -n ${INFRA_ID}-identity
```

### 上传文件到Storage Account

部署步骤中需要从blob读取Red Hat Enterprise Linux CoreOS 的 VHD (virtual hard disk) 镜像和bootstrap的ignition 配置文件。创建一个storage account用于保存上述文件，export生成account 的 key环境变量。

```shell
az storage account create -g $RESOURCE_GROUP --location $AZURE_REGION --name ${CLUSTER_NAME}sa --kind Storage --sku Standard_LRS
export ACCOUNT_KEY=`az storage account keys list -g $RESOURCE_GROUP --account-name ${CLUSTER_NAME}sa --query "[0].value" -o tsv`
```

#### 上传VHD镜像

RHCOS VHD大约16G，所是它保存在本地进行远程部署是不现实的。必须复制存储到 storage container。首先创建blob storage container，然后再上传VHD。

选择适用的RHCOS版本，export生成VHD的URL环境变量。本次使用的是正式 4.3 版：

```shell
export VHD_URL=`curl -s https://raw.githubusercontent.com/openshift/installer/release-4.3/data/data/rhcos.json | jq -r .azure.url`
```

把选好的 VHD 上传到 blob：

```shell
az storage container create --name vhd --account-name ${CLUSTER_NAME}sa --account-key $ACCOUNT_KEY
az storage blob copy start --account-name ${CLUSTER_NAME}sa --account-key $ACCOUNT_KEY --destination-blob "rhcos.vhd" --destination-container vhd --source-uri "$VHD_URL"
```

用下面的命令跟踪上传过程：

```shell
status="unknown"
while [ "$status" != "success" ]
do
  status=`az storage blob show --container-name vhd --name "rhcos.vhd" --account-name ${CLUSTER_NAME}sa --account-key $ACCOUNT_KEY -o tsv --query properties.copy.status`
  echo $status
done
```

#### 上传bootstrap ignition

创建一个blob storage container 并 上传20200324目录中生成的 `bootstrap.ign` 文件：

```shell
az storage container create --name files --account-name ${CLUSTER_NAME}sa --public-access blob --account-key $ACCOUNT_KEY
az storage blob upload --account-name ${CLUSTER_NAME}sa --account-key $ACCOUNT_KEY -c "files" -f "20200324/bootstrap.ign" -n "bootstrap.ign"
```

### 创建 DNS zones

UPI（user-provisioned infrastructure）需要配置几条DNS records。可以灵活选择适用的DNS策略。

本次使用 [Azure's 自带的 DNS 方案](https://docs.microsoft.com/en-us/azure/dns/dns-overview)，首先创建 public DNS zone 实现外部域名 (internet) 可达，然后创建 private DNS zone 用于集群内部解析。 public zone不必和集群在同一个resource group中定义，甚至可以使用已有的base domain。这样就可以在保证之前正确生成[install config的前提](https://github.com/openshift/installer/blob/master/docs/user/azure/customization.md#cluster-scoped-properties)下，跳过创建public DNS zone的步骤。

在以`BASE_DOMAIN_RESOURCE_GROUP`变量命名的resource group中创建新的*public* DNS zone，如果准备使用现成的public DNS zone，就跳过这一步：

```
az network dns zone create -g $BASE_DOMAIN_RESOURCE_GROUP -n ${CLUSTER_NAME}.${BASE_DOMAIN}
```

在和部署资源同一个resource group下面创建 *private* zone：

```
az network private-dns zone create -g $RESOURCE_GROUP -n ${CLUSTER_NAME}.${BASE_DOMAIN}
```

### 为identity赋予角色

为Azure identity赋予 *Contributor* role，这样 Ingress Operator 就可以用有权在azure创建 public IP 和它的 load balancer （*后面发现ingress operator 拿不到IP，就是因为访问OCP代码域名写死，去错地方了！*）

```shell
export PRINCIPAL_ID=`az identity show -g $RESOURCE_GROUP -n ${INFRA_ID}-identity --query principalId --out tsv`
export RESOURCE_GROUP_ID=`az group show -g $RESOURCE_GROUP --query id --out tsv`
az role assignment create --assignee "$PRINCIPAL_ID" --role 'Contributor' --scope "$RESOURCE_GROUP_ID"
```

## 部署

UPI部署的关键内容就是 [Azure Resource Manager](https://docs.microsoft.com/en-us/azure/azure-resource-manager/template-deployment-overview) （ARM）模板，它负责部署各种资源 。这是一组命名为 "NN_name.json" 的json文件。下面就是用[az (Azure CLI)](https://docs.microsoft.com/en-us/cli/azure/) 和之前定义的环境变量按顺序依次执行这些模板部署集群的过程。

### 部署Virtual Network

创建 Virtual Network 和 Openshift 集群的subnets。如果准备使用已经存在的VNet或想修改`01_vnet.json`模板（比如，修改subnets的CIDR格式地址前缀）时，可以跳过这一步骤。

复制 [`01_vnet.json`](https://github.com/openshift/installer/blob/master/upi/azure/01_vnet.json) ARM template 到本地OCP4目录。

用`az` client创建deployment：

```shell
az group deployment create -g $RESOURCE_GROUP \
  --template-file "01_vnet.json" \
  --parameters baseName="$INFRA_ID"
```

链接刚创建的VNet到 private DNS zone：

```shell
az network private-dns link vnet create -g $RESOURCE_GROUP -z ${CLUSTER_NAME}.${BASE_DOMAIN} -n ${INFRA_ID}-network-link -v "${INFRA_ID}-vnet" -e false
```

### 部署image

复制 [`02_storage.json`](https://github.com/openshift/installer/blob/master/upi/azure/02_storage.json) 到本地OCP4目录。

用`az` client创建deployment：

```shell
export VHD_BLOB_URL=`az storage blob url --account-name ${CLUSTER_NAME}sa --account-key $ACCOUNT_KEY -c vhd -n "rhcos.vhd" -o tsv`

az group deployment create -g $RESOURCE_GROUP \
  --template-file "02_storage.json" \
  --parameters vhdBlobURL="$VHD_BLOB_URL" \
  --parameters baseName="$INFRA_ID"
```

### 部署load balancers

复制 [`03_infra.json`](https://github.com/openshift/installer/blob/master/upi/azure/03_infra.json) ARM 到本地OCP4目录。

用`az` client部署load balancers 和 public IP addresses：

```shell
az group deployment create -g $RESOURCE_GROUP \
  --template-file "03_infra.json" \
  --parameters privateDNSZoneName="${CLUSTER_NAME}.${BASE_DOMAIN}" \
  --parameters baseName="$INFRA_ID"
```

在*public* zone创建一条 `api` DNS record 用于平台API的public load balancer。 `BASE_DOMAIN_RESOURCE_GROUP` 必须指向public DNS zone所在的resource group。

```shell
export PUBLIC_IP=`az network public-ip list -g $RESOURCE_GROUP --query "[?name=='${INFRA_ID}-master-pip'] | [0].ipAddress" -o tsv`
az network dns record-set a add-record -g $BASE_DOMAIN_RESOURCE_GROUP -z ${CLUSTER_NAME}.${BASE_DOMAIN} -n api -a $PUBLIC_IP --ttl 60
```

或者在把集群加入已经存在的public zone里面时，用如下命令：

```
export PUBLIC_IP=`az network public-ip list -g $RESOURCE_GROUP --query "[?name=='${INFRA_ID}-master-pip'] | [0].ipAddress" -o tsv`
az network dns record-set a add-record -g $BASE_DOMAIN_RESOURCE_GROUP -z ${BASE_DOMAIN} -n api.${CLUSTER_NAME} -a $PUBLIC_IP --ttl 60
```

### 启动临时的集群bootstrap

复制 [`04_bootstrap.json`](https://github.com/openshift/installer/blob/master/upi/azure/04_bootstrap.json) ARM 到本地OCP4目录。

用`az` client创建deployment：

```shell
export BOOTSTRAP_URL=`az storage blob url --account-name ${CLUSTER_NAME}sa --account-key $ACCOUNT_KEY -c "files" -n "bootstrap.ign" -o tsv`
export BOOTSTRAP_IGNITION=`jq -rcnM --arg v "2.2.0" --arg url $BOOTSTRAP_URL '{ignition:{version:$v,config:{replace:{source:$url}}}}' | base64 -w0`

az group deployment create -g $RESOURCE_GROUP \
  --template-file "04_bootstrap.json" \
  --parameters bootstrapIgnition="$BOOTSTRAP_IGNITION" \
  --parameters sshKeyData="$SSH_KEY" \
  --parameters baseName="$INFRA_ID"
```

### 启动集群 control plane mater

复制 [`05_masters.json`](https://github.com/openshift/installer/blob/master/upi/azure/05_masters.json) ARM 到本地OCP4目录。

用`az` client创建deployment：

```
export MASTER_IGNITION=`cat 20200324/master.ign | base64`

az group deployment create -g $RESOURCE_GROUP \
  --template-file "05_masters.json" \
  --parameters masterIgnition="$MASTER_IGNITION" \
  --parameters sshKeyData="$SSH_KEY" \
  --parameters privateDNSZoneName="${CLUSTER_NAME}.${BASE_DOMAIN}" \
  --parameters baseName="$INFRA_ID"
```

### 等待 bootstrap 部署完成

等待，直到集群的控制平面引导完成：

```shell
$ openshift-install wait-for bootstrap-complete --dir=20200324 --log-level debug
DEBUG OpenShift Installer v4.3.5
DEBUG Built from commit 82f9a63c06956b3700a69475fbd14521e139aa1e
INFO Waiting up to 30m0s for the Kubernetes API at https://api.testcluster.xxx.online:6443...
INFO API v1.16.2 up
INFO Waiting up to 30m0s for bootstrapping to complete...
DEBUG Bootstrap status: complete
INFO It is now safe to remove the bootstrap resources
```

一但成功引导， 用export生成 `kubeconfig` 环境变量然后替换 `default` Ingress Controller，新ingress controller定义`endpointPublishingStrategy`为 `HostNetwork`类型。 它将禁止创建面向Azure的公共 Load Balancer并且允许能够自定义安全规则而不被k8s覆盖。（*这一步源于一篇blog，目的是解决获取IP地址Pending的问题，但实际操作中发现原有Ingress Controller无法被删除，因为Operator会不停地重建它，所以只能忽略掉这个删除和创建步骤*）

```shell
export KUBECONFIG=$(pwd)/20200324/auth/kubeconfig
oc delete ingresscontroller default -n openshift-ingress-operator
oc create -f ingresscontroller-default.yaml
```

禁用cloud-credential Operator，定义三个secrets，分别加入三个Operator所在namespace (*这一步源于同事在AWS安装经验*)

```shell
oc edit configmap cloud-credential-operator-config -n openshift-cloud-credential-operator
kind: ConfigMap
metadata:
  name: cloud-credential-operator-config
  namespace: openshift-cloud-credential-operator
  annotations:
    release.openshift.io/create-only: "true"
data:
  disabled: "true"

$ cat openshift-image-registry.yaml
apiVersion: v1
data:
  azure_client_id: NzQwMzI1ZjAtODNkYy00OTJiLTgwYTUtMDFjZTc0M2I1ZWFiCg==
  azure_client_secret: NGFiNWRjZTMtM2VhZi00MGMwLWI1NGYtOTNhMGNkMzYzNjUzCg==
  azure_tenant_id: NWRkMjJhM2UtZTMxNS00NTNmLTg2ZTEtOTQ5ZTQwNzI3ZWFjCg==
  azure_subscription_id: NDI3MDhmMGUtMTg0Ni00ODhkLWFlN2UtNGYyOGFiZmMwYzIyCg==
  azure_resource_group: dGVzdGNsdXN0ZXItc2Z3cWQtcmcK
  azure_location: Y2hpbmFlYXN0Mgo=
kind: Secret
metadata:
  name: installer-cloud-credentials
  namespace: openshift-image-registry
type: Opaque
oc create -f openshift-image-registry.yaml

$ cat openshift-ingress-operator.yaml
apiVersion: v1
data:
  azure_client_id: NzQwMzI1ZjAtODNkYy00OTJiLTgwYTUtMDFjZTc0M2I1ZWFiCg==
  azure_client_secret: NGFiNWRjZTMtM2VhZi00MGMwLWI1NGYtOTNhMGNkMzYzNjUzCg==
  azure_tenant_id: NWRkMjJhM2UtZTMxNS00NTNmLTg2ZTEtOTQ5ZTQwNzI3ZWFjCg==
  azure_subscription_id: NDI3MDhmMGUtMTg0Ni00ODhkLWFlN2UtNGYyOGFiZmMwYzIyCg==
  azure_resource_group: dGVzdGNsdXN0ZXItc2Z3cWQtcmcK
  azure_location: Y2hpbmFlYXN0Mgo=
kind: Secret
metadata:
  name: cloud-credentials
  namespace: openshift-ingress-operator
type: Opaque

oc create -f openshift-ingress-operator.yaml

$ cat openshift-machine-api.yaml
apiVersion: v1
data:
  azure_client_id: NzQwMzI1ZjAtODNkYy00OTJiLTgwYTUtMDFjZTc0M2I1ZWFiCg==
  azure_client_secret: NGFiNWRjZTMtM2VhZi00MGMwLWI1NGYtOTNhMGNkMzYzNjUzCg==
  azure_tenant_id: NWRkMjJhM2UtZTMxNS00NTNmLTg2ZTEtOTQ5ZTQwNzI3ZWFjCg==
  azure_subscription_id: NDI3MDhmMGUtMTg0Ni00ODhkLWFlN2UtNGYyOGFiZmMwYzIyCg==
  azure_resource_group: dGVzdGNsdXN0ZXItc2Z3cWQtcmcK
  azure_location: Y2hpbmFlYXN0Mgo=
kind: Secret
metadata:
  name: azure-cloud-credentials
  namespace: openshift-machine-api
type: Opaque

oc create -f openshift-machine-api.yaml
```

完成引导控制平面处理流程后，可以取消分配并删除bootstrap资源：

```shell
az network nsg rule delete -g $RESOURCE_GROUP --nsg-name ${INFRA_ID}-controlplane-nsg --name bootstrap_ssh_in
az vm stop -g $RESOURCE_GROUP --name ${INFRA_ID}-bootstrap
az vm deallocate -g $RESOURCE_GROUP --name ${INFRA_ID}-bootstrap
az vm delete -g $RESOURCE_GROUP --name ${INFRA_ID}-bootstrap --yes
az disk delete -g $RESOURCE_GROUP --name ${INFRA_ID}-bootstrap_OSDisk --no-wait --yes
az network nic delete -g $RESOURCE_GROUP --name ${INFRA_ID}-bootstrap-nic --no-wait
az storage blob delete --account-key $ACCOUNT_KEY --account-name ${CLUSTER_NAME}sa --container-name files --name bootstrap.ign
az network public-ip delete -g $RESOURCE_GROUP --name ${INFRA_ID}-bootstrap-ssh-pip
```

### 验证OpenShift API

现在可以使用 `oc` or `kubectl` 命令调用OpenShift API。管理员凭证在`auth/kubeconfig`目录中，例如：

```shell
export KUBECONFIG=$(pwd)/20200324/auth/kubeconfig
oc get nodes
oc get clusteroperator
```

API co会正常启动，OpenShift web console 会在即将安装的计算节点部署。

### 启动群集compute nodes

创建计算节点，可以分别启动单个实例或者通过集群外部的自动化处理 (例如：Auto Scaling Groups)。也可以利用OpenShift集群内置的缩放机制和主机API。

这次我们手工用ARM模板启动三个计算节点实例。 增加的实例数量可以修改 `06_workers.json` 文件来调整。

复制 [`06_workers.json`](https://github.com/openshift/installer/blob/master/upi/azure/06_workers.json) ARM 到本地OCP4目录。

用`az` client创建deployment：

```shell
export WORKER_IGNITION=`cat 20200324/worker.ign | base64`

az group deployment create -g $RESOURCE_GROUP \
  --template-file "06_workers.json" \
  --parameters workerIgnition="$WORKER_IGNITION" \
  --parameters sshKeyData="$SSH_KEY" \
  --parameters baseName="$INFRA_ID"
```

### 批准 worker CSRs

即使计算节点启起来后，他们也不会在 `oc get nodes`命令结果中找到。

而是会创建等待批准的certificate signing requests (CSRs) 。最终会有 `Pending` 的条目出现，就像下面看到的一样。使用 `watch oc get csr -A` 命令查看是否 pending CSR's 可用。

```shell
$ oc get csr -A
NAME        AGE    REQUESTOR                                                                   CONDITION
csr-8bppf   2m8s   system:serviceaccount:openshift-machine-config-operator:node-bootstrapper   Pending
csr-dj2w4   112s   system:serviceaccount:openshift-machine-config-operator:node-bootstrapper   Pending
csr-ph8s8   11s    system:serviceaccount:openshift-machine-config-operator:node-bootstrapper   Pending
csr-q7f6q   19m    system:node:master01                                                        Approved,Issued
csr-5ztvt   19m    system:node:master02                                                        Approved,Issued
csr-576l2   19m    system:node:master03                                                        Approved,Issued
csr-htmtm   19m    system:serviceaccount:openshift-machine-config-operator:node-bootstrapper   Approved,Issued
csr-wpvxq   19m    system:serviceaccount:openshift-machine-config-operator:node-bootstrapper   Approved,Issued
csr-xpp49   19m    system:serviceaccount:openshift-machine-config-operator:node-bootstrapper   Approved,Issued
```

需要用`oc describe csr `命令检查每个 pending 的 CSR，确认它们是由需要批准的节点产生的，不应该批准不明来源节点的CSR:

```shell 
$ oc adm certificate approve csr-8bppf csr-dj2w4 csr-ph8s8
certificatesigningrequest.certificates.k8s.io/csr-8bppf approved
certificatesigningrequest.certificates.k8s.io/csr-dj2w4 approved
certificatesigningrequest.certificates.k8s.io/csr-ph8s8 approved
```

被批准过的节点应该出现在命令 `oc get nodes`的结果中，但它们的状态还是 `NotReady` 。然后它们会分别产生每二个CSR，同样也需要被批准。重复上面的检查和批准动作。

一但所有相关的CSR都被批准，节点状态就都应该变为 `Ready` 并投入使用。

```shell
$ oc get nodes
NAME       STATUS   ROLES    AGE     VERSION
master01   Ready    master   23m     v1.14.6+cebabbf7a
master02   Ready    master   23m     v1.14.6+cebabbf7a
master03   Ready    master   23m     v1.14.6+cebabbf7a
node01     Ready    worker   2m30s   v1.14.6+cebabbf7a
node02     Ready    worker   2m35s   v1.14.6+cebabbf7a
node03     Ready    worker   2m34s   v1.14.6+cebabbf7a
```

### 增加 Ingress DNS Records

在 *public* and *private* zones 创建 DNS records 指向 ingress load balancer。用适用的 A, CNAME, 等方法来定义 records。可以使用通配符 `*.apps.{baseDomain}.` ，也可为每个route（或如下所示的，多个route使用同一个records）都配置 [特定的 records](https://github.com/openshift/installer/blob/master/docs/user/azure/install_upi.md#specific-route-records) 。

首先等待 ingress default router 创建 load balancer 并且得到 `EXTERNAL-IP` 值：

```shell
$ oc -n openshift-ingress get service router-default
NAME             TYPE           CLUSTER-IP      EXTERNAL-IP     PORT(S)                      AGE
router-default   LoadBalancer   172.30.20.10   35.130.120.110   80:32288/TCP,443:31215/TCP   20
```

在*public* DNS zone增加 `*.apps` record：

```shell
export PUBLIC_IP_ROUTER=`oc -n openshift-ingress get service router-default --no-headers | awk '{print $4}'`
az network dns record-set a add-record -g $BASE_DOMAIN_RESOURCE_GROUP -z ${CLUSTER_NAME}.${BASE_DOMAIN} -n *.apps -a $PUBLIC_IP_ROUTER --ttl 300
```

或者可以用下面的命令替代，这适用于在现有public zone中加入集群, use instead:

```shell
export PUBLIC_IP_ROUTER=`oc -n openshift-ingress get service router-default --no-headers | awk '{print $4}'`
az network dns record-set a add-record -g $BASE_DOMAIN_RESOURCE_GROUP -z ${BASE_DOMAIN} -n *.apps.${CLUSTER_NAME} -a $PUBLIC_IP_ROUTER --ttl 300
```

最后，在*private* DNS zone加入 `*.apps` record：

```shell
export PUBLIC_IP_ROUTER=`oc -n openshift-ingress get service router-default --no-headers | awk '{print $4}'`
az network private-dns record-set a create -g $RESOURCE_GROUP -z ${CLUSTER_NAME}.${BASE_DOMAIN} -n *.apps --ttl 300
az network private-dns record-set a add-record -g $RESOURCE_GROUP -z ${CLUSTER_NAME}.${BASE_DOMAIN} -n *.apps -a $PUBLIC_IP_ROUTER
```

#### 集群使用的route records

如果希望加入DNS明确的域名而不是通配，需要为集群当前每个路由分别创建配置。可以用下面的命令检查都有哪些路由：

```shell
$ oc get --all-namespaces -o jsonpath='{range .items[*]}{range .status.ingress[*]}{.host}{"\n"}{end}{end}' routes
oauth-openshift.apps.cluster.basedomain.com
console-openshift-console.apps.cluster.basedomain.com
downloads-openshift-console.apps.cluster.basedomain.com
alertmanager-main-openshift-monitoring.apps.cluster.basedomain.com
grafana-openshift-monitoring.apps.cluster.basedomain.com
prometheus-k8s-openshift-monitoring.apps.cluster.basedomain.com
```

### 等待安装完成

等待，直到集群就绪：

```shell
$ openshift-install wait-for install-complete --log-level debug --dir=20200324
DEBUG Built from commit 6b629f0c847887f22c7a95586e49b0e2434161ca
INFO Waiting up to 30m0s for the cluster at https://api.cluster.basedomain.com:6443 to initialize...
DEBUG Still waiting for the cluster to initialize: Working towards 4.2.12: 99% complete, waiting on authentication, console, monitoring
DEBUG Still waiting for the cluster to initialize: Working towards 4.2.12: 100% complete
DEBUG Cluster is initialized
INFO Waiting up to 10m0s for the openshift-console route to be created...
DEBUG Route found in openshift-console namespace: console
DEBUG Route found in openshift-console namespace: downloads
DEBUG OpenShift console route is created
INFO Install complete!
INFO To access the cluster as the system:admin user when using 'oc', run 'export KUBECONFIG=${PWD}/auth/kubeconfig'
INFO Access the OpenShift web-console here: https://console-openshift-console.apps.ocp4.xxx.online
INFO Login to the console with user: kubeadmin, password: REDACTED
```



## 参考资料

https://cloud.redhat.com/openshift/install/azure/user-provisioned

https://github.com/openshift/installer/blob/master/docs/user/azure/install_upi.md

https://github.com/openshift/installer/blob/master/docs/user/azure/install.md#create-configuration

https://github.com/openshift/installer/blob/master/docs/user/azure/customization.md

https://docs.microsoft.com/en-us/azure/dns/dns-delegate-domain-azure-dns

https://github.com/openshift/installer/blob/master/docs/user/azure/README.md

https://docs.openshift.com/container-platform/4.3/installing/installing_azure/installing-azure-customizations.html

https://blog.openshift.com/openshift-4-1-upi-environment-deployment-on-microsoft-azure-cloud/

https://docs.microsoft.com/en-us/azure/azure-resource-manager/management/azure-subscription-service-limits

https://github.com/nwcdlabs/openshift-on-aws-cn/blob/master/ocp-v4/README.md

https://github.com/openshift/cloud-credential-operator/blob/master/docs/disabled-operator.md

https://github.com/JuozasA/ocp4-azure-upi

https://docs.openshift.com/container-platform/4.3/authentication/certificates/replacing-default-ingress-certificate.html

https://blog.openshift.com/openshift-4-install-experience/

https://blog.openshift.com/openshift-4-2-on-azure-preview/

https://docs.openshift.com/container-platform/3.11/install_config/configuring_azure.html

https://rcarrata.com/openshift/ocp4_route_sharding/