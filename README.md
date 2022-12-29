# State Machine for Async Service

## Overview

In this tutorial, we implement an asynchronous service by using a State Machine for the service handler.  Upon receipt of a client request, this service invokes other services, and collects multiple service responses, and then assembles and asynchronously returns a client response by using a FTL message endpoint.

The following figure illustrates the components of this project interacting via FTL endpoints.  Note that `pubsub` endpoint is used for the service handler to receive responses from the mock service, and so multiple service handlers can be deployed to process client requests concurrently.

![architecture](images/architecture.png)

Detailed description for this tutorial can be found in [Wiki page](https://github.com/learn-tibco-cep/tutorials/wiki/State-Machine-for-Async-Service)

## Components

### Handler State Machine

The main application logic is depicted by the state-machine diagram of the async handler (see the above figure).  Upon receipt of a client request, the service agent will create a new handler and change its state to `Pending`.  The preprocessor of the the client request will also send a service request to a mock service provider, which will return 1 or more service responses as specified by the service request.

While in the `Pending` state, the async handler will wait a max amount of time configured by a `global variable`.  If all expected number of service responses are received within the time limit, the handler will transition out of the `Pending` state, and return a client response message including the collection of service responses.  Otherwise, if the collected number of service responses is less than the expected number when the max wait time has reached, the handler will return a `Timeout` response containing only the partial service responses collected upto the max wait time.

### Performace Stat Collector

The project folder [Stats](./Stats) contains a utility used to collect and printout detailed application performance statistics that may not be clearly captured by other tools such as BE profiler or OpenTelemetry.

This utility is implemented by using BE rule-functions. The function [addStat](./Stats/addStat.rulefunction) allows you to aggregate any application-specific performance values (such as elapsed time) under a unique name.

The function [updateStats](./RuleFunctions/updateStats.rulefunction) shows how this utility can be used to aggregate and printout performance stats of the async handler.

### Mock Service

The project folder [TestMock](./TestMock) contains only a preprocessor function that generates multiple service responses according to a received service request.  It is used to support the functional and performance testing of the async handler.

This component is also an illustration on how easy it is to implement mock services in BE.

### Test Driver

The project folder [TestDriver](./TestDriver) is a multi-threaded client simulator used to support performance testing of the async handler.  It shows 2 commonly used techniques in BE applications.

* Use timer events to schedule delayed tasks.  At the startup of the test driver, the function [startup](./TestDriver/startup.rulefunction) schedules multiple request message publishers at random delays, so each publisher will start at a different time.  The delayed timer event will trigger the rule [DelayedDispatch](./TestDriver/DelayedDispatch.rule).
* Use local channel to spawn multiple threads.  Instead of starting message publishers in the `DelayedDispatch` rule, which would execute in the single timer thread, the `DelayedDispatch` rule sends a `dispatch` event to a local channel destination, which will be processed by the preprocessor function [publishClientRequests](./TestDriver/publishClientRequests.rulefunction) on a different thread.  This is a common technique to spawn multiple concurrent threads since BE does not allow you to create threads explicitly.

The test driver also implements a preprocessor function [onClientResponse](./TestDriver/onClientResponse.rulefunction) that subscribes response messsages and collects and prints out performance statistics.

The test driver is configured using the following `global variables`, whose default values are overridden by properties of the `test` PU in [Demo.cdd](./Demo.cdd).

|Variable Name|Description|Default|
|---|---|---|
|Test/Publishers|Number of concurrent publisher threads|3|
|Test/RequestCount|Number of requests to send per thread|1001|
|Test/RequestInterval|ms to sleep between consecutive requests|20|
|Test/PrintStats|Print stats after receiving this number of responses|200|

## Start FTL Server

This tutorial uses TIBCO FTL to exchange messages between BE application components.  To test this application, you must follow the instruction in [Prerequisites](https://github.com/learn-tibco-cep/tutorials/wiki/Prerequisites), install and start a FTL server.  For example, you may start FTL server in Docker:

```
docker run -d -p 8585:8585 ftl-tibftlserver:6.9.1 --name ftls1@$(hostname):8585
```

## Test the Application

Clone this repository, and import it into BE studio workspace, `$WS`, as a `Existing TIBCO BE Studio Project`.  Build the enterprise archive, e.g., `$WS/Demo.ear`.

### Start Inference Engines

```
$BE_HOME/bin/be-engine --propFile $BE_HOME/bin/be-engine.tra -n demo-1 -u default -c ${WS}/AsyncService/Demo.cdd Demo.ear
```

Optionally, start more inference engines for load distribution:

```
$BE_HOME/bin/be-engine --propFile $BE_HOME/bin/be-engine.tra -n demo-2 -u default -c ${WS}/AsyncService/Demo.cdd Demo.ear
```

### Start Mock Service

```
$BE_HOME/bin/be-engine --propFile $BE_HOME/bin/be-engine.tra -n mock -u mock -c ${WS}/AsyncService/Demo.cdd Demo.ear
```

### Start Test Driver

```
$BE_HOME/bin/be-engine --propFile $BE_HOME/bin/be-engine.tra -n test -u test -c ${WS}/AsyncService/Demo.cdd Demo.ear
```

## Test Results

On a MacBook Pro with 8-Core Intel Core i9 processor, this application printed out the following statistics for the default configuration with 3 test driver threads and each thread publishes requests at 50/s, i.e., for a total client request rate of `150/s`.

### Single Inference Engine

The test driver received `0.2%` of timeout errors due to race conditions when service response is processed before the original client request is committed.  For the other `99.8%` of successful responses, the server elapsed time is `1.8 ms` on average.

```
[Complete-ServerElapsed] count: 2993, avg: 1.800, max: 81.000, min: 0.000
[Complete-ClientElapsed] count: 2994, avg: 3.215, max: 84.000, min: 1.000
[Timeout-ServerElapsed] count: 6, avg: 5014.000, max: 5043.000, min: 5004.000
[Timeout-ClientElapsed] count: 6, avg: 5023.167, max: 5045.000, min: 5006.000
```

More detailed stats from the handler state machine is as follows:

```
[ServiceElapsed] count: 8982, avg: 0.088, max: 4.000, min: 0.000
[ServiceEnd2End] count: 8982, avg: 1.511, max: 81.000, min: 0.000
[Complete-ExpectedResponses] count: 2995, avg: 2.998, max: 5.000, min: 1.000
[Complete-HandleElapsed] count: 2994, avg: 1.799, max: 81.000, min: 0.000
[Timeout-ExpectedResponses] count: 6, avg: 3.333, max: 5.000, min: 1.000
[Timeout-HandleElapsed] count: 6, avg: 5014.000, max: 5043.000, min: 5004.000
```

### Two Inference Engines

When 2 inference engines are deployed, each engine will process only half of the client requests.  Although the elapsed time for processing each request is similar to that of the single engine deployment, the total throughput would be doubled because the load is evenly distributed by 2 workers.

The stats printed by the test driver:

```
[Complete-ServerElapsed] count: 2986, avg: 1.939, max: 22.000, min: 0.000
[Complete-ClientElapsed] count: 2986, avg: 3.371, max: 41.000, min: 1.000
[Timeout-ServerElapsed] count: 14, avg: 5006.357, max: 5015.000, min: 5002.000
[Timeout-ClientElapsed] count: 14, avg: 5012.071, max: 5027.000, min: 5004.000
```

The detailed stats printed by the inference engine 1:

```
[ServiceElapsed] count: 4466, avg: 0.090, max: 5.000, min: 0.000
[ServiceEnd2End] count: 4466, avg: 1.647, max: 21.000, min: 0.000
[Complete-ExpectedResponses] count: 1493, avg: 2.989, max: 5.000, min: 1.000
[Complete-HandleElapsed] count: 1493, avg: 1.922, max: 22.000, min: 0.000
[Timeout-ExpectedResponses] count: 7, avg: 3.143, max: 5.000, min: 1.000
[Timeout-HandleElapsed] count: 7, avg: 5006.286, max: 5011.000, min: 5003.000
```

The detailed stats printed by the inference engine 2:

```
[ServiceElapsed] count: 4474, avg: 0.083, max: 1.000, min: 0.000
[ServiceEnd2End] count: 4474, avg: 1.668, max: 22.000, min: 0.000
[Complete-ExpectedResponses] count: 1493, avg: 2.997, max: 5.000, min: 1.000
[Complete-HandleElapsed] count: 1493, avg: 1.956, max: 22.000, min: 0.000
[Timeout-ExpectedResponses] count: 7, avg: 2.714, max: 5.000, min: 1.000
[Timeout-HandleElapsed] count: 7, avg: 5006.429, max: 5015.000, min: 5002.000
```