HOW TO UNDERSTAND ARCHITECTURE AND WORKFLOW
===========================================

## Fundamentals to undersatnd aplloauto - core

Autonomous vehicles \(AV\) dynamics controled by planning engine through Controller Area Network bus \(CAN bus\). Software reads data from
hardware registers and write them back just like we did in Assembly language. To compute concisely, Location Module, Perception Module, Planning Mudule as independent 
input sources and output sources work together though Peer2Peer. P2P supported by RPC networks application.

Apolloauto uses ROS1 as underlying network which means that apolloauto bowrrows Master-Nodes framework from ROS1. Since xmlRPC from ROS1 is really old \(compared
to the recent brpc and [grpc](https://yiakwy.github.io/blog/2017/10/01/gRPC-C-CORE)\), baidu develops its own protobuf version of RPC. 

In Baidu ApolloAuto, three statges develoment have already been described

1. Offline Simulation Engine Dreamviewer & apolloauto core softare module
   - Get first taste on how algorithm works for a cars
   - We don't need to touch a real car or hardware and start development immediately
2. Core modules Integrateion: 
   - Location
   - Perception \(support third parties' solution like mobileye ES4 chipe based camera for L2 development\) process point cloud data from `Lidar` and return segmented objects info on request 
   - Planning: compute fine tuned path, car dynamic controlling info for path segments from route service
   - Routine: local implementation of finding path segments through `Navigator` interface; Using A\*star algorihtm. 
3. Hdmap. One of the key differences from L2 level AV development. L4 AV machine needs Hdmap. Since a robot \(an autonomous vehicles\) needs to rebuild 
3d world \(please check opencv [SLAM]() chapter\) in its micro computer, reference object coordinates play a great role in relocating VA both in map and real world.
4. Cloud based Online Simulation Drive Senario Engine and Data Centre. 
   - As a partner of baidu, you will be granted docker credentials to commit new images and replay on cloud the algoritm you devleoped.
   - Create and manage complex senorios to simulate real world driving experiences

## ROS underlying Subscription and Publication machenism and apolloauto modules structure


#### ROS underlying Subscription and Publication machenism

So how does ROS1 based system communicate with each other and how does apolloauto make use of it ? ROS has [tutorial](http://wiki.ros.org/ROS/Tutorials), and I will explain it 
quickly before we analyze apolloauto modules structure.

ROS is a software, currently exclusively well supported by Ubuntu series. It has master roscore. 

> printenv | grep ROS

deafult ros master uri is "http://localhost:11311. One can create an independent binary by performing ros::init and start it by performing ros::spin \(some kind of Linux event loop\) 
using c++ or python. The binary behind the freshly created package is called ***ROS node***. The node will register its name and ip address in Master in case of other nodes querying. Nodes commuicate
with each by directly constructing a TCP connection.

If a node want to read data from others, we call it subscriper. The typical format is 

```
... bla bla bla
ros::NodeHandle h;
ros::Subscriber sub = h.subscribe("topic_name", q_size, cb)
.. bla bla bla
```

If a node want to provide data for subscribers to read, we call it publisehr. The typical format is 

```
... bla bla bla
ros::NodeHandle h;
ros::Publisher pub = h.advertise<generated_msg_format_cls>("topic_name", q_size)
... bla bla bla
```

cb here is a callback excuted when Linux kernel IO is ready. With these signatures bearing in mind, we can quickly analyze apolloauto
module structures before diving deep into core modules implementation.

#### apolloauto modules structure

I have conducted full research about it but I cannot show you all of them. ApolloAuto modules/common/ provide basic micros to control ros::spin for each 
module and /modules/common/adaptor contains the most infomation how a topic is registerd. Every modules will be registerd from the [point](https://github.com/yiakwy/apollo/blob/master/modules/common/adapters/adapter_manager.cc#L50)
. By reading configuration file defined ${MODULE_NAME}/conf, we can get basic infomation about topics a module subscribe and publish.

Each module starts by firing "Init" interface and register callbacks. If you wanto step by step debug apolloauto in gdb, make sure you have add breakpoints in those back. This also 
demonstrate that if you don't like what implemented by Baidu, just override the callback.

## Data preprocessing and Extedned Kalman Filter

Kalman Filter is mathematical interative methods to converge to real estimation without knowing the whole real\-time input sequence. No matter what kind of data you need to process, you can 
rely on Kalman Filter. Extended Kalman Filter is used for 3d rigid movemnts in matrix format. It is not hard. I recommend you a series tutorial from United States F15 director 
[Michel van Biezen](https://www.youtube.com/watch?v=CaCcOwJPytQ).

Since it is used in input data preprocessing, you might see it in hdmap, perception, planning and so on so forth.
 
## Selected modules analysis

#### HMI & Dreamviewer

There is not too much about hmi interface and dreamviewer but it is a good place to visualize the topics parameters. 

HMI is a simply simple python application based on Flask.
Instead of using http, it uses websocket to query ROS modlues application. If you have asynchronous http downloader, it is easy to understan, that an http connection is just a 
socket connection file descriptor which we have already write http headers, methods into that buffer. Once hmi flask backend receives a command, it will execute a subprocess
to execute corresponding binary.

Dreamviewer, in contrast, works a little bit like frontend app written in React, Webpack, and Threejs \( webgl, see /dreamvidw/backend/simulation_world, /dreamview/frontend/src/render \), 
techniques. It subscribes messages from ROS nodes and draws it a frame after a frame.

#### Percetion

Initially, this module implemented logics exclusively for Lidar and Radar processes. It is registered by AdapterManager as a ros node functioning as a info fusion system to
output observed Obstacles info. In the latest version of the codes, different hardware input handlers of ROS nodes are specified in /perception/obstacles/onboard and implemented in
different parallel locations, which consists of *Lidar, Radar, Traffic lights and GPS*.

1. Lidar:
   - Hadmap: get tranformation matrix convert point world coordinates to local cordinates and build map polygons
   - ROI filter: get ROI and perform Kalman Filter on input data
   - Segmentation: A U-Net based \(a lot of variants\) caffemodel will be loaded and perform forward computation based on data from Hdmap and ROI filtering results
   - Object Building: Lidar return points \(x, y, z\). Hence you need to goup them into "Obstacles" \(vector or set\)
   - Obstacles Tracker: Baidu using uisng HM sovler from Google. For large bipartite graph, KM algorithms in Lagrange foramt is usually deployed since
     SGD is extremely simple for that.

2. Radar:
   - Similar to Lidar with raw\_obstacles info from sensors.
   - ROI filter: get ROI objects and perform Kalman Filter on input data
   - Objects Tracker

3. Probability Fusion\(New in Apollo 1.5!\):
   - As far as I can understand, fusion system in apolloatuo 
   - It is typically the one of most important parts: collects all the info and makes final combination of information from sensors on mother board 
	 for track lists and rule based cognitive engine
   - Tha major process is association, hence HM algorithms here is used again as a bipartite graph.
   - Track lists are maintained along timestamps and each list will be updated based on probablistic rules engine
