# openTCS-5.0.0-ValidDynamicRoute
It realizes valid and dynamic route finding in openTCS 5.0.0.

## 0. 前言

有几位网友关于拙作[《改造OpenTCS之如何避免无效路由of基于OpenTCS的二次开发实践》](https://baijiahao.baidu.com/s?id=1664195047709896169)、[《改造OpenTCS之动态更新路由of基于OpenTCS的二次开发实践》](https://baijiahao.baidu.com/s?id=1664560931613429318)留了言，笔者以本文、已上传到github的项目openTCS-5.0.0-ValidDynamicRoute，以及视频《有效路由的动态生成与更新体验》作为回应.

## 1. 开发环境(Development Environment)

笔者的相关开发在以下环境中进行.

{

  \"OS\":\"Ubuntu 20.04.2\",

  \"IDE\":\"Apache Netbeans IDE 12.0\",

  \"JdkVersion\":\"Java 11.0.10\",

  \"openTCSVersion\":\"5.0.0\",

  \"CurlVersion\":\"7.68.0\"

}

## 2. 相关的文件(Files related)

(1)修改的源代码文件(Modified source code files)

/openTCS-API-Base

/src/main/java/org/opentcs

/access/rmi/services/RemoteDispatcherService.java (16)

/access/rmi/services/RemoteDispatcherServiceProxy.java (18)

/components/kernel/services/DispatcherService.java (15)

/components/kernel/Dispatcher.java (12)

/openTCS-Kernel/src/main/java/org/opentcs/kernel

/distribution/config/opentcs-kernel-defaults-baseline.properties (03)

/services/StandardDispatcherService.java (17)

/vehicles/DefaultVehicleController.java (14)

/openTCS-Kernel-Extension-RMI-Services/src/main/java/org/opentcs/kernel

/extensions/rmi/StandardRemoteDispatcherService.java (19)

/openTCS-Strategies-Default/src

/main/java/org/opentcs/strategies/basic

/dispatching/DefaultDispatcherConfiguration.java (02)

/dispatching/DefaultDispatcher.java (13)

/routing/jgrapht/AbstractPointRouterFactory.java (09)

/routing/jgrapht/ShortestPathPointRouter.java (10)

/routing/jgrapht/DefaultModelGraphMapper.java (11)

/routing/DefaultRouter.java (07)

/routing/PointRouterFactory.java (08)

(2)用于测试的文件(Files used for testing)

/data

/script/1-tn-demo-lb-1234-sil.sh (04)

/script/2-tn-demo-lb-1234-create-to-part1.sh (05)

/script/4-tn-demo-lb-1234-create-to-nv-part1.sh

/script/6-tn-demo-lb-1234-create-to-part1.sh

/script/8-tn-demo-lb-1234-create-to-nv-part1.sh

/script/3-tn-demo-lb-1234-create-to-part2.sh (06)

/script/5-tn-demo-lb-1234-create-to-nv-part2.sh

/script/7-tn-demo-lb-1234-create-to-nv-part2.sh

/script/9-tn-demo-lb-1234-create-to-part2.sh

/tn/tn-demo-lb-1234.xml (01)

注：加载tn-demo-lb-1234.xml、持久化到内核并进入操作模式后，可以执行分四种情形测试：

    始终预订车辆：

        ./2-tn-demo-lb-1234-create-to-part1.sh

        ./3-tn-demo-lb-1234-create-to-part2.sh

    始终不预订车辆：

        ./4-tn-demo-lb-1234-create-to-nv-part1.sh

        ./5-tn-demo-lb-1234-create-to-nv-part2.sh

    先预订车辆后不预订车辆：

        ./6-tn-demo-lb-1234-create-to-part1.sh

        ./7-tn-demo-lb-1234-create-to-nv-part2.sh

    先不预订车辆后预订车辆：

        ./8-tn-demo-lb-1234-create-to-nv-part1.sh

        ./9-tn-demo-lb-1234-create-to-part2.sh
