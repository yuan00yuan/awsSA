---
typora-root-url: ../RedShift vs. MySQL
typora-copy-images-to: ../RedShift vs. MySQL
---

# Redshift & MySQL 性能对比实验

[TOC]

## 实验目的

提供Redshift和RDS MySQL的性能对比试验。

## 涉及组件

- EC2
- RedShift
- RDS
- S3
- VPC

## 实验步骤

> **重要**
>
> 本实验默认您已经拥有了 AWS 账户并创建了 IAM 用户
>
> 若未执行以上设置，可参考[这里](https://docs.aws.amazon.com/zh_cn/AWSEC2/latest/UserGuide/get-set-up-for-amazon-ec2.html#sign-up-for-aws)

### 配置 VPC

新建 VPC 安全组，具体步骤参考[适用于 Amazon VPC 的 IPv4 入门](https://docs.aws.amazon.com/zh_cn/AmazonVPC/latest/UserGuide/getting-started-ipv4.html#getting-started-create-security-group)

将安全组的**入站规则**设置为

- **Type**: ALL Traffice
- **Protocol**: ALL
- **Port Range**: ALL
- **Source**: 选择 **Custom IP**，然后键入 `0.0.0.0/0`。

> **重要**
>
> 除演示之外，建议不要使用 0.0.0.0/0，因为它允许从 Internet 上的任何计算机进行访问。在实际环境中，您需要根据自己的网络设置创建入站规则。

### 配置 RedShift

1.  为 Amazon Redshift 创建 IAM 角色

   - 登录 AWS 管理控制台 并通过以下网址打开 IAM 控制台 <https://console.aws.amazon.com/iam/>。
   - 在左侧导航窗格中，选择 **Roles**。 
   - 选择 **Create role**
   - 在 **AWS Service** 组中，选择 **Redshift**。 
   - 在 **Select your use case** 下，选择 **Redshift - Customizable**，然后选择 **Next: Permissions**。 
   - 在 **Attach permissions policies** 页面上，选择 **AdministratorAccess**，然后选择 **Next: Review**。 
   - 对于 **Role name**，为您的角色键入一个名称。在本教程中，请键入 `myRedshiftRole`。 
   - 检查信息，然后选择 **Create Role**。 
   - 选择新角色的角色名称。
   - 将 **Role ARN** 复制到您的剪贴板 — 此值是您刚刚创建的角色的 Amazon 资源名称 (ARN)。在之后的实验中，将会使用到该值。 

2. 登录 AWS 管理控制台 并通过以下网址打开 Amazon Redshift 控制台：<https://console.aws.amazon.com/redshift/> 。

3. 在主菜单中，选择您要在其中创建群集的区域。在本教程中，选择 **美国西部（俄勒冈）**。 ![](/Users/dicey/Documents/aws实习/RedShift vs. MySQL/rs-gsg-aws-region-selector.png)

4. 在 Amazon Redshift 仪表板上，选择 **Launch Cluster**。									“Amazon Redshift Dashboard”如下所示![](/Users/dicey/Documents/aws实习/RedShift vs. MySQL/rs-gsg-clusters-launch-cluster-10.png)

5. 在“Cluster Details”页面上，输入下列值，然后选择 **Continue**：

   - **Cluster Identifier**：键入 `rs-vs-mysql`。                                     
   - **Database Name**：将此框留空。Amazon Redshift 将会创建一个名为 `dev` 的默认数据库。                                     
   - **Database Port**：键入数据库将接受连接的端口号。您应该在本教程的先决条件步骤确定了端口号。在启动群集之后便无法更改端口号，因此请确保您知道防火墙中的一个开放端口号，这样才能从 SQL                                        客户端工具连接到群集中的数据库。                                     
   - **Master User Name**：键入 `masteruser`。在群集可供使用之后，您将使用此用户名和密码连接到您的数据库。                                     
   - **Master User Password** 和 **Confirm Password**：为主用户账户键入密码。

   ![](/Users/dicey/Documents/aws实习/RedShift vs. MySQL/conf-rs.jpg)

6. 在“Node Configuration”页面上，选择下列值，然后选择 **Continue**：

   - **Node Type**：**dc1.large**
   - **Cluster Type**：**Multi Node**
   - **Number of compute nodes**：**2**

   ![](/Users/dicey/Documents/aws实习/RedShift vs. MySQL/conf-rs-nodes.jpeg)

7. 在“**Additional Configuration**”页面上，

   - **Choose a VPC**：选择您在**配置 VPC**这一步骤中创建的安全组所对应的 VPC
   - **Publicly Accessible**：**Yes** 
   - **VPC Security Groups**：选择您在**配置 VPC**这一步骤中创建的安全组
   - **AvailableRoles**： 选择**myRedshiftRole**
   - **其他选项**采用**默认**选项即可

   然后选择**Continue**。

8. 在“Review”页面上，查看您进行的选择，然后选择 **Launch Cluster**。 

### 配置 RDS

1. 登录 AWS 管理控制台 并通过以下网址打开 Amazon RDS 控制台：<https://console.aws.amazon.com/rds/>。 

2. 在 Amazon RDS 控制台的右上角，选择您要在其中创建数据库实例的区域。这里为保证与之前创建 RedShift 的区域相同，选择 **美国西部（俄勒冈）**。 

3. 在导航窗格中，选择**实例**。 

4. 选择**启动数据库实例**。**启动数据库实例向导**在**选择引擎**页面打开。![](/Users/dicey/Documents/aws实习/RedShift vs. MySQL/conf-rds.jpeg)

5. 选择 **MySQL**，然后选择**下一步**。

6. **选择使用案例**页面询问您是否计划使用所创建的数据库实例进行生产。选择 **开发/测试**，然后选择 **下一步** 。  

7. 在**指定数据库详细信息**页面上，指定数据库实例信息。选择下列值，然后选择 **下一步**。 

   - **数据库实例类**：db.r4.xlarge
   - **存储类型**：预置 IOPS
   - **预置 IOPS**：1000
   - **数据库实例标识符**：键入 `rs-vs-mysql`。 
   - **主用户名**：键入 `masteruser`。 
   - **主密码**和**确认密码**：键入您的密码
   - **其他设置保持默认**：

   ![](/Users/dicey/Documents/aws实习/RedShift vs. MySQL/conf-rds-mysql.jpeg)![](/Users/dicey/Documents/aws实习/RedShift vs. MySQL/conf-rds-mysql-2.jpg)

8. 在**配置高级设置**页面上，提供 RDS 启动 MySQL 数据库实例所需的其他信息。选择下列值，然后选择 **下一步**。

   - **Virtual Private Cloud (VPC)**：选择您在**配置 VPC**这一步骤中创建的安全组所对应的 VPC
   - **公开可用性**：是
   - **VPC安全组**：选择现有 VPC 安全组，并且选择**配置 VPC**这一步骤中创建的安全组
   - **数据库名称**：键入`dbname`
   - **备份保留期**：1 天
   - **其他请选择默认**

### 配置 EC2

1. 启动实例

   - 打开 Amazon EC2 控制台 <https://console.aws.amazon.com/ec2/>。选择您要在其中创建EC2实例的区域。这里为保证与之前创建 RedShift、RDS 的区域相同，选择 **美国西部（俄勒冈）**。 

   - 从控制台控制面板中，选择 **启动实例**。

   - **Choose an Amazon Machine Image (AMI)** 页面显示一组称为 *Amazon 系统映像 (AMI)* 的基本配置，作为您的实例的模板。选择 Amazon Linux AMI 2 的 HVM 版本 AMI。 

   - 在**选择实例类型** 页面上，您可以选择实例的硬件配置。选择 `t2.small` 类型 

   - 在**配置实例详细信息**页面上，自动分配公有 IP 选择**启用**，其他选择默认

   - 在**配置安全组**页面选择**选择一个现有的安全组**，并在表格中选择**配置 VPC**这一步骤中创建的安全组

   - 在**审核**页面选择**启动**

   - 当系统提示提供密钥时，选择 **选择现有的密钥对**，然后选择合适的密钥对。若没有创建密钥对，请参考[创建密钥对](https://docs.aws.amazon.com/zh_cn/AWSEC2/latest/UserGuide/get-set-up-for-amazon-ec2.html#create-a-key-pair)

     准备好后，选中确认复选框，然后选择 **启动实例**。  

2. 连接到 EC2

   请参考[使用 SSH 连接到 Linux 实例](https://docs.aws.amazon.com/zh_cn/AWSEC2/latest/UserGuide/AccessingInstancesLinux.html)

3.  在实例中安装 MySQL

   **在连接到 EC2 实例后**，依次输入以下命令，在实例中安装 MySQL

   ```
   wget http://repo.mysql.com/mysql-community-release-el6-5.noarch.rpm
   
   sudo rpm -ivh mysql-community-release-el6-5.noarch.rpm
   
   sudo yum install mysql-community-server -y
   ```

4. 在实例中安装 psql 工具

   **在连接到 EC2 实例后**，输入以下命令，在实例中安装 psql

   ```
   sudo yum install postgresql-server -y
   ```

### 导入数据到 RDS

1. 切换目录

   输入命令

   ```
   cd ~
   ```

2. 通过 AWS 命令行界面（CLI）下载数据

   AWS CLI 已经预安装在了 Amazon Linux AMI 上，但您仍需要进行相应的配置，详情可参考[配置 AWS CLI](https://docs.aws.amazon.com/zh_cn/cli/latest/userguide/cli-chap-getting-started.html)

   在 EC2 中配置完成 AWS CLI 后，输入以下命令拷贝测试数据

   ```
   aws s3 cp s3://awssampledbuswest2/tickit/sales_tab.txt sales_tab.txt
   aws s3 cp s3://awssampledbuswest2/tickit/allevents_pipe.txt allevents_pipe.txt
   ```

3. 连接到 RDS

   参考[与运行 MySQL 数据库引擎的数据库实例连接](https://docs.aws.amazon.com/zh_cn/AmazonRDS/latest/UserGuide/USER_ConnectToInstance.html)

4. 创建表格

   输入命令

   ```
   use dbname;
   ```

   ```
   create table sales(
   	salesid integer not null,
   	listid integer not null ,
   	sellerid integer not null,
   	buyerid integer not null,
   	eventid integer not null,
   	dateid smallint not null ,
   	qtysold smallint not null,
   	pricepaid decimal(8,2),
   	commission decimal(8,2),
   	saletime timestamp);
   	
   create table event(
   	eventid integer not null,
   	venueid smallint not null,
   	catid smallint not null,
   	dateid smallint not null,
   	eventname varchar(200),
   	starttime timestamp);
   ```

5. 导入表格

   ```
   load data local infile 'sales_tab.txt' 
   into table sales 
   lines terminated by '\n';
   
   load data local infile 'allevents_pipe.txt' 
   into table event 
   fields terminated by'|'
   lines terminated by '\n';
   ```

   

```
EXPLAIN ANALYZE
SELECT eventname, total_price
FROM  (SELECT eventid, total_price
       FROM (SELECT eventid, sum(pricepaid) total_price
             FROM   sales
             GROUP BY eventid) AS A) AS Q INNER JOIN event AS E
       ON Q.eventid = E.eventid
ORDER BY total_price desc
LIMIT 20
;
```











## 准备工作

为保证Redshift和RDS MySQL的算力尽量相等，我们使用如下配置：

- Redshift: dc1.large with 2 compute node
- RDS MySQL: db.r4.xlarge, disk 100G, IOPS 1000

为尽可能提高RDS MySQL的性能，我们关闭Encryption和Enhanced monitoring, 关闭Multi-AZ部署。

- RDS 文档有坑,在远程连接 mysql 数据库时,提供的命令是错误的
- RDS 访问权限改为公开
- 一个小坑:
https://docs.aws.amazon.com/zh_cn/redshift/latest/mgmt/connecting-from-psql.html
"在左侧导航窗格中，单击 Clusters。单击您的群集以将其打开。在 Cluster Database Properties 下，记录 Endpoint、Port 和 Database Name 的值。"中Endpoint 的信息不在这个标签下,而是在最顶部

copy users from 's3://awssampledbuswest2/tickit/allusers_pipe.txt' 
credentials 'aws_iam_role=arn:aws:iam::398560366025:role/myRedshiftRole' 
delimiter '|' region 'us-west-2';


