# Grafana 基础


## 1 基本概念

### 1.1 Data Source

Grafana 支持多种不同类型的时序数据库，称为[数据源]^(Data Source)。目前官方支持：Graphite、InfluxDB、OpenTSDB、Prometheus、Elasticsearch、CloudWatch。

可以将多个数据源的数据合并到一个单独的仪表盘（Dashboard）上，但是每个面板（Panel）都绑定到属于特定组织的特定数据源。

添加一个数据源时，分为两种访问模式：
* **Server** 模式（默认）：所有请求都从浏览器发送到 Grafana 后端服务器，后端服务器将请求转发到数据源。
* **Browser** 模式：浏览器访问模式，所有请求直接从浏览器发送到数据源。

### 1.2 Organization

Grafana 可以支持创建多个[组织]^(Organization)，每个组织可以有一个或多个数据源。所有的仪表盘都归特定组织拥有。

### 1.3 User

[用户]^(User)是 Grafana 的账户，一个用户可以隶属于一个或多个组织，通过角色（Role）为其分配不同级别的权限。

Grafana 也支持各种用户认证方式。

### 1.4 Panel

[面板]^(Panel)是**最基本的可视化模块**。每个 Panel 根据查询编辑器（例如 Prometheus 就是 PromQL）查询数据，并提供展示图表。

目前，Grafana 支持的 Panel 许多 Panel 类型，最常用的就是 Time series，展示基于时间的样本数据。

可以通过向 Grafana 用户分享链接，或者使用快照功能将正在查看的数据编码为静态和交互式 JSON 文件，来向其他人分享 Panel 以及数据。

{{< image src="img0-2.png" height=200 >}}

### 1.5 Row

[行]^(Row)用于在**仪表盘界面组织多个 Panel**，一般为 12 单位的宽度。

{{< image src="img0-0.png" height=40 >}}

### 1.6 Query Editor

每个 Panel 都有着一个或多个 Query Editor，通过编写语句来读取数据源的样本，将其展示到 Panel 上。

{{< image src="img0-1.png" height=200 >}}

### 1.7 Dashboard

多个 Panel 或者多个 Row 构成一个[仪表盘]^(Dashboard)，用于分类不同的指标项。

通过分享链接或者通过快照功能，可以向他人分析 Dashboard 以及数据。

你也可以仅仅通过 Export 功能，将 Dashboard 结构导出为 JSON 文件。他人可以通过 Import 该 JSON 文件直接构建对应的 Dashboard 。

{{< image src="img1.png" height=300 >}}

## 2 定制 Dashboard

1. 创建新的 Dashboard 
   
   点击 "Create" 菜单栏创建新的 Dashboard。点击 "Setting" 按钮进行配置。

2. 配置 General 项
   
   General 项包含 Dashboard 名称和一些常规配置。

   {{< image src="img10.png" height=330 >}}

   * Name         -  Dashboard 名称                                          
   * Description  -  Dashboard 描述                                          
   * Tages        -  Dashboard  label                                        
   * Editable     - Editable 可以编辑 Dashboard ，Read-only 不允许编辑 Dashboard  
   * Timezone     - 配置时区                                            
   * Auto-refresh - 定制相对时间显示和自动刷新选项，默认即可            

3. 配置 Variables 选项
   
   可以为 Dashboard 配置多个模板变量，在 Panel 中可以使用模板变量，变量会在 Dashboard 顶部展示为下拉选择框。

   {{< image src="img11.png" height=70 >}}

   Variables 的类型包括：

   * Interval - 表示时间跨度，来动态改变表达式中的时间段
   * Query - 允许用户编写数据源查询语句，根据查询结果变为变量的可选值
   * Datasource - 允许用户通过变量更改整个 Dashboard 的数据源
   * Custom - 用户自定义特定的可选变量值
   * Constant - 定义隐藏常量
   * Ad hoc filter - 仅用于某些特定数据源
   * Text box - 具有可选默认值的文本输入字段

## 3 定制 Panel 

### 3.1 Graph  Panel 

1. 配置 Query 项
   
   {{< image src="img12.png" height=200 >}}

   * Data source - 使用的数据源
   * Query options - 设置查询数据相关的参数，例如最小时间间隔等
   * Query - 设置查询语句，展示的数据就来自于通过查询语句查询数据源
     * Legend format - 可视化图折线的名称

2. 配置 Panel options
   
   {{< image src="img13.png" height=350 >}}

   * Title -  Panel 名称
   * Description -  Panel 描述

3. 配置 Axes
   
   Axes 用于配置坐标抽的显示。可以使用坐标轴左右两侧 Y 轴的单位。

4. 配置 Legend
   
   配置图例的显示方式，可以展示一些特殊的值。

5. 配置 Display
   
   Display 选项用于战术显示的样式。

## 参考

* [**《Prometheus 监控技术与实践》**](https://book.douban.com/subject/35034115/)
