### 2.7.3 Service Discovery
#### 2.7.3.1 Service Discovery Types
The MSRA Communication Management provides two different ways to locate services offered by applications that run on other ECUs:
1. **SOME/IP Service Discovery**
The standard SOME/IP Service Discovery conforms with the SOME/IP Service Discovery protocol specification and is described in detail in [7].
Once SOME/IP daemon is started, it listens "passively" to the incoming offers and monitors the offers for the configured services.
Once an application starts, all its configured required service instances are "actively" requested.
This triggers accordingly that service discovery for those instances enters the initial wait phase.
In case the service has not already been found, SOME/IP Service Discovery FindService entries will be sent out.

2. **Static Service Discovery**
This is an extension of the SOME/IP network binding available `only in MSRA`.
Basically, the standard SOME/IP Service Discovery is replaced with `static information` which is provided by the user of MSRA.
**The standard SOME/IP Service Discovery is completely disabled** in that case and there is no exchange of remote SOME/IP Service Discovery messages between clients and servers prior to the establishing of connections for the method and event transmission.
The aforementioned static information is represented as multiple tables containing all the necessary information to establish data connections with no help of the standard SOME/IP Service Discovery protocol. See 5.6.4 for a couple of examples what this information looks like.
The Static Service Discovery might be better than the standard SOME/IP Service Discovery in the following cases:
* The standard SOME/IP Service Discovery increases the time between the MSRA’s startup and the transmission of the first SOME/IP data message
* When some of the communication partners have not implemented the standard SOME/IP Service Discovery protocol yet
* The standard SOME/IP Service Discovery protocol is redundant because all data communication paths are known in advance and never change
* The standard SOME/IP Service Discovery protocol is redundant because all data communication paths are only between local applications served by the same SOME/IP daemon
* For testing purposes








### 2.7.7 Overload Protection
To protect applications from being overloaded by flow of incoming notifications, an `overload protection filter` can be used to `filter out unwanted notifications`.
Notifications are received via UDP or TCP first at SOME/IP daemon, and then they are forwarded to all
applications that have subscribed for such event (or field notifier). 
**If overload protection filter is active for that event, the time interval between two notifications will never be shorter than the configured minimum interval.**
The reception of events configured to use the overload protection feature, will work as follows:
* Events received when the overload protection for that specific event type is not active, will be
forwarded immediately to the application.
* Events received when the overload protection for that specific event type is active, will be buffered
locally for later transmission to the application. If there was already an event buffered, it will be
overwritten (discarded).
* When the overload protection time expires, the buffered (last received) event, if any, will be
forwarded to the application.
The activation/deactivation of the overload protection for an event type works as follows:
* Activation: whenever an event is forwarded to the application (either directly or after the expiration of
the overload protection time), the overload protection will be activated and the overload protection
time will start.
* Deactivation: whenever the overload protection time expires and there is no buffered event to send,
the overload protection will be deactivated.
Figure 2-7 depicts the described behavior of the feature.



### 5.6.4 Service Discovery
Service Discovery can be fully modeled in ARXML so that all the service instances could be located by either dynamically using SOME/IP Service Discovery protocol or statically using the provided information in the model. As a result, based on customer requirement when it is decided that a specific Required/Provided Service Instance shall not use dynamic service discovery, or in other words the static service discovery is intended to be used, the `staticServiceDiscovery` element shall be provided as it is shown in listings 5-3 and 5-4.
ServiceDiscoveryConfig shall be modelled in all Required-/Provided Service Instances. This will be used in case Service Discovery is active. For each SomeIpServiceInstanceToMachineMapping, this ServiceDiscoveryConfig can be overriden by introducing the staticServiceDiscovery element, which implicitly disables Service Discovery for that mapping, and instead, uses the pre-configured IP addresses and ports.










