# AWS 学习 - 1 - 地理概念


## 1 Region

将所有的资源都放在一个 Data Center（简称 DC）不能保证可用性，如果该数据中心出现问题，那么意味所有的应用程序出现问题。

为此，AWS 在世界各地不同地方构建了许多的 DC，每个地方称为 Region。例如 us-west、us-east、eu-west 等。

所有 Region 之间的网络是可以连通的，但是默认下 Region 之间网络是隔离的。该特性是为了数据处理的合法合规。

在学习一个服务时，很重要的一点就是知晓是 Regional，还是 Zonal。一个 Regional Service 表明是其 Region 下所有 AZ 共享的，由 AWS 保证了 Service 的高可用。


## 2 Available Zone

Region 由一组 Available Zone（简称 AZ），AZ 表示一个或一组分离的 DC。例如 us-west-1、us-west-2 等。

一个 Region 下的多个 AZ 之间相隔数十英里，该距离保证多个 AZ 之间通信还是低延时的。


## 3 Region AZ DC 之间关系

三者之间的关系如下图：
{{< find_img "img1.png" >}}

* 一个 Region 包含多个 AZ，各个 AZ 之间物理位置上分离；
* 一个 AZ 包含一个或多个 IDC；

可以看到，为了容灾，业务应该至少在一个 Region 下的两个 AZ 中部署。


### 4 Edge Location

Edge Location 是分布在世界各地的边缘站点，用于支持一些加速用户访问的服务。

例如，CloudFront 与 Route53 都可以部署在 Edge Location。
{{< find_img "img2.png" >}}


