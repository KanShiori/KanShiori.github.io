# Job CronJob 实践


Job 用于运行一次性的任务，即“离线任务”。CronJob 在 Job 之上提供了周期性任务支持。

## 1 Job
目的：部署 Job 任务，使用并行运行（Batch）的功能。
1. 创建部署 Job 任务。
{{< find_img "img1.png" >}}
2. 可以看到，并行的两个 Pod 正在运行。
{{< find_img "img2.png" >}}
经过一段时间可以看到，因为设置 deadline 为 100，所以 pod 异常被退出。而 restartPolicy: Never 使得不会再次运行。
{{< find_img "img3.png" >}}
3. 去除 deadline 设置，重新部署，可以看到，本次 4 次成功完成。
{{< find_img "img4.png" >}}


## 2 CronJob
目的：体验 CronJob 的周期性任务功能，明确 CronJob 是基于 Job 管理实现的。
1. 创建部署 CronJob 。可以看到，CronJob 中需要指定 JobTemplate，因此 CronJob 完全是基于 Job 管理的。
{{< find_img "img5.png" >}}
2. 部署后可以看到，CronJob 创建了一个 Job：
{{< find_img "img6.png" >}}
3. 经过一分钟，新 Job 被创建，并运行成功：
{{< find_img "img7.png" >}}


