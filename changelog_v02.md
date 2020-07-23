Changes between v0.2.1 and v0.2.2
=================================

v0.2.2 was released on 23 July 2020.

* Support for cascading delete of resources when you delete a namespace. [Bug 157](https://github.com/shakenfist/shakenfist/issues/157).
* Shaken Fist now tracks the power state of instances and exposes that via the REST API and command line client. We also kill stray instances which are left running after deletion. [Bug 173](https://github.com/shakenfist/shakenfist/issues/173); [bug 184](https://github.com/shakenfist/shakenfist/issues/184); [bug 192](https://github.com/shakenfist/shakenfist/issues/192); and [bug 197](https://github.com/shakenfist/shakenfist/issues/197).
* The API (and command line client) now display the version of Shaken Fist installed on each node. [Bug 175](https://github.com/shakenfist/shakenfist/issues/175).
* The network event cleanup code handles larger numbers of events needing to be cleaned on upgrade. [Bug 176](https://github.com/shakenfist/shakenfist/issues/176).
* Kernel Shared Memory is now configured by the deployer and used by Shaken Fist to over subscribe memory in cases where pages are successfully being shared. [Bug 177](https://github.com/shakenfist/shakenfist/issues/177); [deployer bug 28](https://github.com/shakenfist/deploy/issues/28); and [deployer bug 33](https://github.com/shakenfist/deploy/issues/33).
* Instance nodes are now correctly reported in cases where placement was forced. [Bug 179](https://github.com/shakenfist/shakenfist/issues/179).
* Instance scheduling is retried on a different node in cases where the churn rate is sufficient for the cached resource availability information to be inaccurate. [Bug 186](https://github.com/shakenfist/shakenfist/issues/186).
* A permissions denied error was corrected for shakenfist.json accesses in the comment line client. [Bug 187](https://github.com/shakenfist/shakenfist/issues/187).
* Incorrect database information for instances or network interfaces no longer crashes the start of new instances. [Bug 194](https://github.com/shakenfist/shakenfist/issues/194).
* Instance names must now be DNS safe. [Bug 200](https://github.com/shakenfist/shakenfist/issues/200).
* The deployer now checks that the configured MTU on the VXLAN mesh interface is sane. [Deployer bug 30](https://github.com/shakenfist/deploy/issues/30), and [deployer bug 32](https://github.com/shakenfist/deploy/issues/32).

Changes between v0.2.0 and v0.2.1
=================================

v0.2.1 was released on 16 July 2020.

* Fix crash in cleaner daemon when an etcd compaction fails. [Bug 152](https://github.com/shakenfist/shakenfist/issues/152).
* Fix HTTP 500 errors when malformed authorization headers are passed on API calls. [Bug 154](https://github.com/shakenfist/shakenfist/issues/154).
* Avoid starting instances on the network node if possible. [Bug 156](https://github.com/shakenfist/shakenfist/issues/156).
* Track and expose instance power states. Show instance state and power state in instance listings. [Bug 159](https://github.com/shakenfist/shakenfist/issues/159).
* Correct resource-in-use errors for specific IP requests. [Bug 162](https://github.com/shakenfist/shakenfist/issues/162).
* Network mesh event logging was too verbose. Only log additions and removals from the mesh. Also cleanup old mesh events on upgrade. [Bug 163](https://github.com/shakenfist/shakenfist/issues/163).
* Nodes now report what version of Shaken Fist they are running via etcd. [Bug 164](https://github.com/shakenfist/shakenfist/issues/164).