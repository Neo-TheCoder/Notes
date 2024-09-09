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