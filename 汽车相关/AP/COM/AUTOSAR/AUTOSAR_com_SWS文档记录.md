# 1 Introduction and functional overview
This document contains the `requirements` on the functionality, `API` and the `configuration` of the AUTOSAR Adaptive Communication Management as part of the Adaptive AUTOSAR platform foundation.
The Communication Management realizes `Service Oriented Communication` between Adaptive AUTOSAR Applications for all levels of communication, e.g. IntraProcess, InterProcess, InterMachine. It consists of potentially generated Service Provider Skeletons and Service Requester Proxies and optionally the generic Communication Manager software for central brokering and configuration.
The Communication Management provides a built-in safety mechanism (`E2E` protection), which can be used for all levels of communication for events and methods.
The documentation of the Communication Management consists of two documents:
* the ARAComAPI explanatory document [1], providing explanations of the design and behavior descriptions of the ara::com API,
* this document, providing the requirements on the ara::com API.
Therefore it is recommended to read the ARAComAPI explanatory document first to get an overview and understanding, and to read this document afterward.









# 7 Functional specification
## 7.1 General description
The AUTOSAR Adaptive architecture organizes the software of the AUTOSAR Adaptive foundation as functional clusters.
These clusters offer common functionality as `services` to the applications.
The Communication Management (CM) for AUTOSAR Adaptive is such a functional cluster and is part of "AUTOSAR Runtime for Adaptive Applications" - ARA.
It is responsible for `the construction and supervision of communication paths` between applications, both local and remote.
The CM provides the infrastructure that enables communication between Adaptive AUTOSAR Applications within one machine and with software entities on other machines, e.g. other Adaptive AUTOSAR applications or Classic AUTOSAR SWCs. All communication paths can be established at design- , start-up- or run-time.
This specification includes the syntax of the API, the relationship of API to the model and describes semantics, e.g. through state machines, and assumption of pre-, post- conditions and use of APIs. The specification does not provide constraints on the SW architecture of a platform implementation, so there is no definition of basic software modules and no specification of implementation or internal technical architecture of the Communication Management.








































