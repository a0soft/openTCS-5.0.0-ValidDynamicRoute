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

## 3. 对提及文件的描述(Descriptions for files mentioned)

以下对相关文件逐一描述.

### (01)tn-demo-lb-1234.xml

这是测试用的交通运输网模型文件，其改编自openTCS官方自带的0.0.3版的Demo-01.xml.

![tn-demo-lb-1234-v1](https://github.com/a0soft/openTCS-5.0.0-ValidDynamicRoute/blob/main/images/tn-demo-lb-1234-v1.png)

以下是对车辆Ve0001的主要描述:

    <vehicle name="Ve0001"
    length="1000"
    energyLevelCritical="30"
    energyLevelGood="90"
    energyLevelFullyRecharged="95"
    energyLevelSufficientlyRecharged="45"
    maxVelocity="1000" maxReverseVelocity="1000">
        <property name="ip" value="127.0.0.1"/>
        <property name="loopback:initialPosition" value="2"/>
        <property name="loopback:loadOperation" value="Load cargo"/>
        <property name="loopback:unloadOperation" value="Unload cargo"/>
        <property name="port" value="50200"/>
        <property name="tcs:preferredAdapterClass" value=
        "org.opentcs.virtualvehicle.LoopbackCommunicationAdapterFactory"/>
    </vehicle>

### **(02)DefaultDispatcherConfiguration.java**

派发器配置中, 为重新寻路触发器加入相关配置项.

    @ConfigurationEntry(
          type = "String",
          description = {
            /*...*/
            "TOPOLOGY_CHANGE: Vehicles get rerouted immediately
            on topology changes."
            /// --------------------------------------------------------------------
            , "ROUTE_STEP_FINISHED: Each relevant vehicle get rerouted as soon as
            the arrival of the destination of a route step."
            /// --------------------------------------------------------------------
          },
          orderKey = "1_orders_special_2")
      RerouteTrigger rerouteTrigger();
      /*...*/
    enum RerouteTrigger {
        /*...*/
        TOPOLOGY_CHANGE
        /// ------------------------------------------------------------------------
        , ROUTE_STEP_FINISHED
        /// ------------------------------------------------------------------------
        ;
    }

### **(03)opentcs-kernel-defaults-baseline.properties**

为方便测试, 修改配置文件.

    #kernelapp.autoEnableDriversOnStartup = false
    kernelapp.autoEnableDriversOnStartup = true
    ####
    #kernelapp.updateRoutingTopologyOnPathLockChange = false
    kernelapp.updateRoutingTopologyOnPathLockChange = true
    ####
    #defaultdispatcher.rerouteTrigger = NONE
    defaultdispatcher.rerouteTrigger = ROUTE_STEP_FINISHED
    ####
    #defaultrouter.routeToCurrentPosition = false
    defaultrouter.routeToCurrentPosition = true
    ####
    #virtualvehicle.simulationTimeFactor = 1.0
    virtualvehicle.simulationTimeFactor = 4.0
    ####

### **(04)1-tn-demo-lb-1234-set-vil.sh**

该批处理调用curl, 将对应车辆的集成级别设置为使用.

    #!/bin/sh

    curl -X PUT "http://localhost:55200/v1/vehicles/Ve0001/integrationLevel
    ?newValue=TO_BE_UTILIZED" -H "accept: */*"

    curl -X PUT "http://localhost:55200/v1/vehicles/Ve0002/integrationLevel
    ?newValue=TO_BE_UTILIZED" -H "accept: */*"

    curl -X PUT "http://localhost:55200/v1/vehicles/Ve0003/integrationLevel
    ?newValue=TO_BE_UTILIZED" -H "accept: */*"

    curl -X PUT "http://localhost:55200/v1/vehicles/Ve0004/integrationLevel
    ?newValue=TO_BE_UTILIZED" -H "accept: */*"

### **(05)2-tn-demo-lb-1234-create-to-part1.sh**

该批处理调用curl, 生成几个运单, 其分别预订一辆指定的车.

    #!/bin/sh

    curl -X POST
    "http://localhost:55200/v1/transportOrders/TOrder-04E8RBM7CAFREKRFRCEYGD901"
    -H "accept: */*" -H "Content-Type: application/json"
    -d "{\"deadline\":\"2019-09-19T16:42:40.396Z\",\"intendedVehicle\":\"Ve0001\",
    \"destinations\":[{\"locationName\":\"GiN01\",\"operation\":\"Load cargo\"}]}"

    curl -X POST
    "http://localhost:55200/v1/transportOrders/TOrder-04E8RBM7CAFREKRFRCEYGD902"
    -H "accept: */*" -H "Content-Type: application/json"
    -d "{\"deadline\":\"2019-09-19T16:42:40.396Z\",\"intendedVehicle\":\"Ve0002\",
    \"destinations\":[{\"locationName\":\"GiN02\",\"operation\":\"Load cargo\"}]}"

    curl -X POST
    "http://localhost:55200/v1/transportOrders/TOrder-04E8RBM7CAFREKRFRCEYGD903"
    -H "accept: */*" -H "Content-Type: application/json"
    -d "{\"deadline\":\"2019-09-19T16:42:40.396Z\",\"intendedVehicle\":\"Ve0003\",
    \"destinations\":[{\"locationName\":\"GiS01\",\"operation\":\"Load cargo\"}]}"

    curl -X POST
    "http://localhost:55200/v1/transportOrders/TOrder-04E8RBM7CAFREKRFRCEYGD904"
    -H "accept: */*" -H "Content-Type: application/json"
    -d "{\"deadline\":\"2019-09-19T16:42:40.396Z\",\"intendedVehicle\":\"Ve0004\",
    \"destinations\":[{\"locationName\":\"St01\",\"operation\":\"Load cargo\"}]}"

以下三个文件中运单号不同, 且:
    6-tn-demo-lb-1234-create-to-part1.sh, 与之类似;
    4-tn-demo-lb-1234-create-to-nv-part1.sh, 与之类似但未预订车辆;
    8-tn-demo-lb-1234-create-to-nv-part1.sh, 与之类似但未预订车辆.

### **(06)3-tn-demo-lb-1234-create-to-part2.sh**

该批处理调用curl, 生成配合(05)的几个运单.

    #!/bin/sh

    curl -X POST
    "http://localhost:55200/v1/transportOrders/TOrder-04E8RBM7CAFREKRFRCEYGD905"
    -H "accept: */*" -H "Content-Type: application/json"
    -d "{\"deadline\":\"2019-09-19T16:42:40.396Z\",\"intendedVehicle\":\"Ve0001\",
    \"destinations\":[{\"locationName\":\"St01\",\"operation\":\"Unload cargo\"},
    {\"locationName\":\"Re01\",\"operation\":\"CHARGE\"}]}"

    curl -X POST
    "http://localhost:55200/v1/transportOrders/TOrder-04E8RBM7CAFREKRFRCEYGD906"
    -H "accept: */*" -H "Content-Type: application/json"
    -d "{\"deadline\":\"2019-09-19T16:42:40.396Z\",\"intendedVehicle\":\"Ve0002\",
    \"destinations\":[{\"locationName\":\"St02\",\"operation\":\"Unload cargo\"},
    {\"locationName\":\"Re02\",\"operation\":\"CHARGE\"}]}"

    curl -X POST
    "http://localhost:55200/v1/transportOrders/TOrder-04E8RBM7CAFREKRFRCEYGD907"
    -H "accept: */*" -H "Content-Type: application/json"
    -d "{\"deadline\":\"2019-09-19T16:42:40.396Z\",\"intendedVehicle\":\"Ve0003\",
    \"destinations\":[{\"locationName\":\"St01\",\"operation\":\"Unload cargo\"},
    {\"locationName\":\"Re03\",\"operation\":\"CHARGE\"}]}"

    curl -X POST
    "http://localhost:55200/v1/transportOrders/TOrder-04E8RBM7CAFREKRFRCEYGD908"
    -H "accept: */*" -H "Content-Type: application/json"
    -d "{\"deadline\":\"2019-09-19T16:42:40.396Z\",\"intendedVehicle\":\"Ve0004\",
    \"destinations\":[{\"locationName\":\"St02\",\"operation\":\"Unload cargo\"},
    {\"locationName\":\"Re04\",\"operation\":\"CHARGE\"}]}"

以下三个文件中运单号不同, 且:

    9-tn-demo-lb-1234-create-to-part2.sh, 与之类似;
    5-tn-demo-lb-1234-create-to-nv-part2.sh, 与之类似但未预订车辆;
    7-tn-demo-lb-1234-create-to-nv-part2.sh, 与之类似但未预订车辆.

### **(07)DefaultRouter.java**

1)修改了以下2个方法.

      @Override
      public Optional<List<DriveOrder>> getRoute(
        Vehicle vehicle, Point sourcePoint, TransportOrder transportOrder) {
        /*...*/
        try {
          /// --------------------------------------------------------------------
          /// new:
          PointRouter pointRouter = getActualPointRouter(
              sourcePoint,
              vehicle
          );
          /// --------------------------------------------------------------------

          rwLock.readLock().lock();
          List<DriveOrder> driveOrderList = transportOrder.getFutureDriveOrders();
          DriveOrder[] driveOrders = driveOrderList.toArray(
            new DriveOrder[driveOrderList.size()]);

          /// --------------------------------------------------------------------
          /// original:
          //PointRouter pointRouter = pointRoutersByVehicleGroup.get(
          //  getRoutingGroupOfVehicle(vehicle));
          /// --------------------------------------------------------------------
          OrderRouteParameterStruct params = new OrderRouteParameterStruct(
            driveOrders, pointRouter);
          /*...*/
      }

      @Override
      public Optional<Route> getRoute(
        Vehicle vehicle, Point sourcePoint, Point destinationPoint) {
        /*...*/
        try {
          rwLock.readLock().lock();
          /// --------------------------------------------------------------------
          /// original:
          //PointRouter pointRouter = pointRoutersByVehicleGroup.get(
          //  getRoutingGroupOfVehicle(vehicle));
          /// --------------------------------------------------------------------
          /// new:
          PointRouter pointRouter = getActualPointRouter(sourcePoint, vehicle);
          /// --------------------------------------------------------------------
          long costs = pointRouter.getCosts(sourcePoint, destinationPoint);
          /*...*/
      }

2)新增以下2个方法.

      /// ------------------------------------------------------------------------
      private Collection<Point> getUnusablePoints(
        Point sourcePoint, Vehicle vehicle) {
        Collection<Point> unusablePoints = new HashSet<>();
        Set<Vehicle> allVehicles = objectService.fetchObjects(Vehicle.class);
        Set<Vehicle> occupyUnusablePointsVehicles1 = new HashSet<>();
        Set<Vehicle> occupyUnusablePointsVehicles2 = new HashSet<>();
        for (Vehicle ve : allVehicles) {
          /// Skip the given vehicle
          if (ve.getName() == vehicle.getName()) {continue;}
          if (ve.getCurrentPosition() == null) {continue;}

          /// For idle vehicle
          if (ve.hasState(Vehicle.State.IDLE)) {
            occupyUnusablePointsVehicles1.add(ve);
            continue;
          }

          /// For vehicle processing order
          if (ve.hasProcState(Vehicle.ProcState.PROCESSING_ORDER)) {
            boolean willAdd = true;
            for (TransportOrder to: objectService
              .fetchObjects(TransportOrder.class)) {
              /// The vehicle is intended for TOrder
              if ((to.hasState(TransportOrder.State.ACTIVE)
                || to.hasState(TransportOrder.State.DISPATCHABLE))
                  && to.getIntendedVehicle().getName() == ve.getName()) {
                willAdd = false;
                break;
              }
            }
            if (willAdd) {occupyUnusablePointsVehicles2.add(ve);}
          }
        }
        //Get all idle vehicle's CurrentPosition
        for (Vehicle ve : occupyUnusablePointsVehicles1) {
          for (Point curPoint : objectService.fetchObjects(Point.class)) {
            /// Skip the source point
            if (curPoint.getName().equals(ve.getCurrentPosition().getName())
              && !sourcePoint.getName().equals(
                ve.getCurrentPosition().getName())) {
              unusablePoints.add(curPoint);
              break;
            }
          }
        }

        /// Get each vehicle which is processing TO and no subsequent TO for it,
        /// the destination is a dead point.
        for (Vehicle ve : occupyUnusablePointsVehicles2) {
          TransportOrder to = objectService.fetchObject(
            TransportOrder.class, ve.getTransportOrder().getName());
          List<DriveOrder> dos = to.getAllDriveOrders();
          Point destinationPoint
            = dos.get(dos.size() - 1).getRoute().getFinalDestinationPoint();

          /// 若车辆vehicle的当前位置是车辆ve尚未走完的路段的端点之一,
          /// 则ve的终点并非真的不可用
          String vehiclePositionName = vehicle.getCurrentPosition().getName();
          boolean foundPN = false;
          boolean will_skip = false;

          /// 对于车辆ve,构造出其尚未走完的路段的终点的集合
          Set<String> namesOfPointToTravel = new HashSet<>();
          String positionName = ve.getCurrentPosition().getName();
          if (positionName.equals(dos.get(0).getRoute().getSteps().get(0)
            .getSourcePoint().getName())) {
            foundPN = true;
          }
          for (DriveOrder do_: dos) {
            List<Route.Step> steps = do_.getRoute().getSteps();
            for (Route.Step step: steps) {
              String pointName = step.getDestinationPoint().getName();
              if (!foundPN && positionName.equals(pointName)) {
                foundPN = true;
                continue;
              }
              if (foundPN) {namesOfPointToTravel.add(pointName);}
            }
          }

          /// 车辆vehicle的当前位置先于ve到达二者的公共路段
          for (String name: namesOfPointToTravel) {
            if (vehiclePositionName.equals(name)) {
              will_skip = true;
              break;
            }
          }
          if (will_skip) {continue;}

          for (Point curPoint : objectService.fetchObjects(Point.class)) {
            /// Skip the source point
            if (curPoint.getName().equals(destinationPoint.getName())
              && !sourcePoint.getName().equals(
                ve.getCurrentPosition().getName())) {
              unusablePoints.add(curPoint);
              break;
            }
          }
        }

        /// Skip the source point
        if (unusablePoints.contains(sourcePoint)) {
          unusablePoints.remove(sourcePoint);
        }
        return unusablePoints;
      }

      private PointRouter getActualPointRouter(
        Point sourcePoint,
        Vehicle vehicle)
      {
          Collection<Point> unusablePoints
            = getUnusablePoints(sourcePoint, vehicle);
          return pointRouterFactory.createPointRouter(vehicle, unusablePoints);
      }
      /// ------------------------------------------------------------------------

需要:

    import java.util.Collection;

### **(08)PointRouterFactory.java**

1)声明接口.

      /// ------------------------------------------------------------------------
      /// Creates a point router for the given vehicle
      ///     and excludes the unusablePoints.
      /// added by Catching Xu with email of a0soft@163.com.
      /// @param vehicle The vehicle.
      /// @param unusablePoints The unusable points.
      /// @return The point router.
      PointRouter createPointRouter(
          Vehicle vehicle,
          Collection<Point> unusablePoints
      );
      /// ------------------------------------------------------------------------

需要:

    import java.util.Collection;
    import org.opentcs.data.model.Point;

注：removeUnusablePoints，这个《改造OpenTCS之如何避免无效路由of基于OpenTCS的二次开发实践》一文中提及的方法，已不再需要，其逻辑应在该接口的实现中体现即可.

### **(09)AbstractPointRouterFactory.java**

1)修改方法.

      @Override
      public PointRouter createPointRouter(Vehicle vehicle) {
        /*...*/
        /// ----------------------------------------------------------------------
        /// original:
        //Graph<String, ModelEdge> graph = mapper.translateModel(
        //  points, objectService.fetchObjects(Path.class), vehicle);
        //PointRouter router = new ShortestPathPointRouter(
        //  createShortestPathAlgorithm(graph), points);
        // Make a single request for a route from one point to a different one
        // to make sure the point router is primed.
        // (Some implementations are initialized lazily.)
        //if (points.size() >= 2) {
        //  Iterator<Point> pointIter = points.iterator();
        //  router.getRouteSteps(pointIter.next(), pointIter.next());
        //}
        /// ----------------------------------------------------------------------
        /// new:
        PointRouter router = createPointRouterInternal(vehicle, points);
        /// ----------------------------------------------------------------------
        /*...*/
      }

2)新增以下方法和数据.

      /// ------------------------------------------------------------------------
      /// Creates a point router for the given vehicle
      ///     and excludes the unusablePoints.
      /// added by Catching Xu with email of a0soft@163.com.
      /// @param vehicle The vehicle.
      /// @param unusablePoints The unusable points.
      /// @return The point router.
      @Override
      public PointRouter createPointRouter(
          Vehicle vehicle,
          Collection<Point> unusablePoints)
      {
          requireNonNull(vehicle, "vehicle");
          requireNonNull(unusablePoints, "unusablePoints");
          long timeStampBefore = System.currentTimeMillis();
          Set<Point> points = objectService.fetchObjects(Point.class);
          for (Point pt : unusablePoints)
          {
              if (points.contains(pt))
              {
                  points.remove(pt);
              }
          }
          PointRouter router = createPointRouterInternal(
              vehicle, points);

          LOG.debug("Created point router for {} in {} milliseconds.",
                    vehicle.getName(),
                    System.currentTimeMillis() - timeStampBefore);
          return router;
      }
      /// 注：removeUnusablePoints，其逻辑已在该方法中体现.

      private static final String DEFAULT_ROUTING_GROUP = "";
      private String getRoutingGroupOfVehicle(Vehicle vehicle) {
        String propVal = vehicle.getProperty(PROPKEY_ROUTING_GROUP);
        return propVal == null ? DEFAULT_ROUTING_GROUP : propVal.strip();
      }
      private PointRouter createPointRouterInternal(
        Vehicle vehicle,
        Collection<Point> points)
      {
        boolean usingGroupFilter = true;
        String groupName = getRoutingGroupOfVehicle(vehicle);
        /// 若组名为空, 则对整个图进行计算
        if (groupName.isBlank() || groupName.isEmpty()) {
          usingGroupFilter = false;
        }
        if (usingGroupFilter) {
          Set<Group> allGroups = objectService.fetchObjects(Group.class);
          for (Group group: allGroups) {
            if (group.getName().equals(groupName)) {
              Set<TCSObjectReference<?>> members = group.getMembers();
              break;
            }
          }
        }

        Set<Path> allPaths = objectService.fetchObjects(Path.class);
        allPaths.removeIf(
          value->(
            points.contains(value.getSourcePoint())
            || points.contains(value.getDestinationPoint())));
        Graph<String, ModelEdge> graph = mapper.translateModel(
          points, allPaths, vehicle);

        PointRouter router = new ShortestPathPointRouter(
          createShortestPathAlgorithm(graph), points);
        // Make a single request for a route from one point to a different one
        // to make sure the point router is primed.
        // (Some implementations are initialized lazily.)
        if (points.size() >= 2) {
          Iterator<Point> pointIter = points.iterator();
          router.getRouteSteps(
              pointIter.next(),
              pointIter.next());
        }

        return router;
      }
      /// ------------------------------------------------------------------------

需要:

    import java.util.Collection;
    import static org.opentcs.components.kernel.Router.PROPKEY_ROUTING_GROUP;
    import org.opentcs.data.TCSObjectReference;
    import org.opentcs.data.model.Group;

### **(10)ShortestPathPointRouter.java**

1)修改以下方法.

      @Override
      public long getCosts(TCSObjectReference<Point> srcPointRef,
                           TCSObjectReference<Point> destPointRef) {
        /*...*/
        /// ------------------------------------------------------------------------
        /// Catch and handle exceptions.
        /// modified by Catching Xu with email of a0soft@163.com.
        /// original:
        //PointRouter router = new ShortestPathPointRouter(
        //  createShortestPathAlgorithm(graph), points);
        //GraphPath<String, ModelEdge> graphPath = algo.getPath(
        //  srcPointRef.getName(), destPointRef.getName());
        //if (graphPath == null) {
        //  return INFINITE_COSTS;
        //}
        /// new:
        GraphPath<String, ModelEdge> graphPath = null;
        try {
          graphPath = algo.getPath(srcPointRef.getName(), destPointRef.getName());
          if (graphPath == null) {
            return INFINITE_COSTS;
          }
        }
        catch (IllegalArgumentException e) {
          System.out.println(e.getMessage());
          return INFINITE_COSTS;
        }
        /// ------------------------------------------------------------------------
        return (long) graphPath.getWeight();
      }

<font color=#ff0000 size=4 face="黑体">注：网友们提到的不能派单或不能自动分配车辆现象，可能因为前言所述两篇文章忽略了此部分而导致。</font>

### **(11)DefaultModelGraphMapper.java**

1)修改以下方法.

      @Override
      public Graph<String, ModelEdge> translateModel(
        Collection<Point> points, Collection<Path> paths, Vehicle vehicle) {
        /*...*/
        Graph<String, ModelEdge> graph = new DirectedWeightedMultigraph<>(
          ModelEdge.class);
        /// ----------------------------------------------------------------------
        Set<String> pointNames = new HashSet<>();
        /// ----------------------------------------------------------------------
        for (Point point : points) {
          graph.addVertex(point.getName());
          /// --------------------------------------------------------------------
          pointNames.add(point.getName());
          /// --------------------------------------------------------------------
        }
        boolean allowNegativeEdgeWeights = configuration.algorithm()
          .isHandlingNegativeCosts();
        for (Path path : paths) {
          /// --------------------------------------------------------------------
          /// skip the path with vertex not existing.
          if (!pointNames.contains(path.getSourcePoint().getName())
              || !pointNames.contains(path.getDestinationPoint().getName())) {
              continue;
          }
          /// --------------------------------------------------------------------
          /*...*/
        }
        /*...*/
      }

需要:

    import java.util.HashSet;
    import java.util.Set;

### **(12)Dispatcher.java**

1)声明接口.

      /// ------------------------------------------------------------------------
      /// Support reroute while a vehicle arrives
      /// the destination of a step of the route.
      void vehicleUpdatedProgressIndex();
      /// ------------------------------------------------------------------------

### **(13)DefaultDispatcher.java**

1)新增以下方法.

      /// ------------------------------------------------------------------------
      @Override
      public void vehicleUpdatedProgressIndex() {
        if (configuration.rerouteTrigger() == ROUTE_STEP_FINISHED)
        {
          LOG.debug("Scheduling reroute task...");
          kernelExecutor.submit(() ->
          {
            LOG.debug(
              "Rerouting vehicles due to a vehicle updated progress index...");
            /// Sorry for the inconvenience of displaying my relevant code
            /// here, include the update for method "getUnusablePoints"!
            /// TODO: to be optimized!!!
            rerouteUtil.reroute(vehicleService.fetchObjects(Vehicle.class));
          });
        }
      }
      /// ------------------------------------------------------------------------

需要:

    import static org.opentcs.strategies.basic.dispatching.DefaultDispatcherConfiguration.RerouteTrigger.ROUTE_STEP_FINISHED;

### **(14)DefaultVehicleController.java**

1)修改以下方法.

      private void updatePositionWithOrder(String position, Point point) {
          /*...*/
          vehicleService.updateVehicleRouteProgressIndex(
            vehicle.getReference(), moveCommand.getStep().getRouteIndex());
            /// ------------------------------------------------------------------
            dispatcherService.vehicleUpdatedProgressIndex();
            ///-------------------------------------------------------------------
          // Let the scheduler know where we are now.
          scheduler.updateProgressIndex(
            this, moveCommand.getStep().getRouteIndex());
          /*...*/
      }

### **(15)DispatcherService.java**

1)声明接口.

      /// ------------------------------------------------------------------------
      /// Support reroute while a vehicle arrives
      /// the destination of a step of the route.
      void vehicleUpdatedProgressIndex();
      /// ------------------------------------------------------------------------

### **(16)RemoteDispatcherService.java**

1)声明接口.

      /// ------------------------------------------------------------------------
      /// Support reroute while a vehicle arrives
      /// the destination of a step of the route.
      /// added by Catching Xu with email of a0soft@163.com.
      void vehicleUpdatedProgressIndex(
        ClientID clientId)
        throws RemoteException;
      /// ------------------------------------------------------------------------

### **(17)StandardDispatcherService.java**

1)新增以下方法

      /// ------------------------------------------------------------------------
      /// Returns the index of this command in the route.
      /// Support reroute while a vehicle arrives
      /// the destination of a step of the route.
      /// added by Catching Xu with email of a0soft@163.com.
      public void vehicleUpdatedProgressIndex() {
        dispatcher.vehicleUpdatedProgressIndex();
      }
      /// ------------------------------------------------------------------------

### **(18)RemoteDispatcherServiceProxy.java**

1)新增以下方法

      /// ------------------------------------------------------------------------
      /// Support reroute while a vehicle arrives
      /// the destination of a step of the route.
      /// added by Catching Xu with email of a0soft@163.com.
      public void vehicleUpdatedProgressIndex() {
        try {
          getRemoteService().vehicleUpdatedProgressIndex(getClientId());
        }
        catch (RemoteException ex) {
          throw findSuitableExceptionFor(ex);
        }
      }
      /// ------------------------------------------------------------------------

### **(19)StandardRemoteDispatcherService.java**

1)新增以下方法

      /// ------------------------------------------------------------------------
      /// Support reroute while a vehicle arrives
      /// the destination of a step of the route.
      /// added by Catching Xu with email of a0soft@163.com.
      public void vehicleUpdatedProgressIndex(ClientID clientId) {
        userManager.verifyCredentials(clientId, UserPermission.MODIFY_ORDER);
        try {
          kernelExecutor.submit(() ->
            dispatcherService.vehicleUpdatedProgressIndex()).get();
        }
        catch (InterruptedException | ExecutionException exc) {
          throw findSuitableExceptionFor(exc);
        }
      }
      /// ------------------------------------------------------------------------
