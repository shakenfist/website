Changes between v0.2.0 and v0.2.1
=================================

* Fix crash in cleaner daemon when an etcd compaction fails. [Bug 152](https://github.com/shakenfist/shakenfist/issues/152).
* Fix HTTP 500 errors when malformed authorization headers are passed on API calls. [Bug 154](https://github.com/shakenfist/shakenfist/issues/154).
* Avoid starting instances on the network node if possible. [Bug 156](https://github.com/shakenfist/shakenfist/issues/156).
* Track and expose instance power states. Show instance state and power state in instance listings. [Bug 159](https://github.com/shakenfist/shakenfist/issues/159).
* Correct resource-in-use errors for specific IP requests. [Bug 162](https://github.com/shakenfist/shakenfist/issues/162).
* Network mesh event logging was too verbose. Only log additions and removals from the mesh. Also cleanup old mesh events on upgrade. [Bug 163](https://github.com/shakenfist/shakenfist/issues/163).
* Nodes now report what version of Shaken Fist they are running via etcd. [Bug 164](https://github.com/shakenfist/shakenfist/issues/164).