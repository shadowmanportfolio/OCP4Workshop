# 安装步骤

## 1. 集群环境准备

您将需要请求两个目录项目：

- 第一是申请一个共享 OpenShift Cluster： 登录 rhpds.redhat.com，导航到 **Services → Catalogs → All Services → Openshift Workshop** 。然后选择 **OpenShift 4.6 Workshop**，提交请求。

- 第二个是创建 Red Hat Enterprise Linux Virtual Machine 用于命令行环境客户端。这个步骤每个学员都要操作。
  1. 登录 labs.opentlc.com，导航到**Services → Catalogs → All Services → OPENTLC OpenShift 4 Labs**.
  2. 在左侧选择 **OpenShift Student VM**.
  3. 右侧点击 **Order**
  4. 从 **OpenShift Release** 下拉列表选择 **4.6**
  5. 从 **Region** 下拉列表选择最近的地区
  6. 在底部点击 **Submit**

## 2. 公告板

部署Etherpad容器应用。

```shell
oc new-project gpte-etherpad --display-name "OpenTLC Shared Etherpad"

oc new-app --template=postgresql-persistent --param POSTGRESQL_USER=ether --param POSTGRESQL_PASSWORD=ether --param POSTGRESQL_DATABASE=etherpad --param POSTGRESQL_VERSION=10 --param VOLUME_CAPACITY=10Gi --labels=app=etherpad_db

sleep 15

# From Github Repo
oc new-app -f 'https://raw.githubusercontent.com/shadowmanportfolio/OCP4Workshop/master/learningPath/Red%20Hat%20Delivery%20Specialist%20-%20Container%20Platform%20(ISV)/02-%E7%BA%A2%E5%B8%BD%20OpenShift%204%20%E5%9F%BA%E7%A1%80/resources/etherpad-template.yaml' -p DB_TYPE=postgres -p DB_HOST=postgresql -p DB_PORT=5432 -p DB_DATABASE=etherpad -p DB_USER=ether -p DB_PASS=ether -p ETHERPAD_IMAGE=quay.io/samzhu/etherpad:latest -p ADMIN_PASSWORD=secret

# From a cloned repo
# oc new-app -f ./etherpad-template.yaml -p DB_TYPE=postgres -p DB_HOST=postgresql -p DB_PORT=5432 -p DB_DATABASE=etherpad -p DB_USER=ether -p DB_PASS=ether -p ETHERPAD_IMAGE=quay.io/samzhu/etherpad:latest -p ADMIN_PASSWORD=secret

```

## 3. 模板

部署mongodb模板，因此此模板在4.6之后都被弃用了，Demo应用要用的话需要手工安装。

```shell
oc create -f 'https://raw.githubusercontent.com/shadowmanportfolio/OCP4Workshop/master/learningPath/Red%20Hat%20Delivery%20Specialist%20-%20Container%20Platform%20(ISV)/02-%E7%BA%A2%E5%B8%BD%20OpenShift%204%20%E5%9F%BA%E7%A1%80/resources/mongodb-template.yaml' -n openshift
oc create -f 'https://raw.githubusercontent.com/shadowmanportfolio/OCP4Workshop/master/learningPath/Red%20Hat%20Delivery%20Specialist%20-%20Container%20Platform%20(ISV)/02-%E7%BA%A2%E5%B8%BD%20OpenShift%204%20%E5%9F%BA%E7%A1%80/resources/mongodb-persistent-template.json' -n openshift

```





