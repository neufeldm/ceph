# RGW Architecture Notes
This document outlines some key architectural features of the
Rados Gateway (_RGW_) daemon code.

## Major Components

### Application State and Startup
The startup code for RGW is contained in three main files. The code that
runs as the actual "main" routine is contained in [rgw_main.h](rgw_main.h)
and [rgw_main.cc](rgw_main.cc). The implementation of the ```AppMain```
class is in [rgw_appmain.cc](rgw_appmain.cc). ```AppMain``` encapsulates
global application state information as well as the code to initialize it.
The [global_init.h](../global/global_init.h) and
[global_init.cc](../global/global_init.cc) contain some initialization code
used by multitple RADOS clients. In particular note the fatal signal handlers
in [signal_handler.h](../global/signal_handler.h) and
[signal_handler.cc](../global/signal_handler.cc).

### Front End
The front end web server code that actually handles _S3_ requests from
clients is modular, and multiple implementations are possible. See
[rgw_frontend.h](rgw_frontend.h) and [rgw_frontend.cc](rgw_frontend.cc).
The default frontend is based on
[Boost Beast](https://github.com/boostorg/beast), which in turn is integrated
with [Boost Asio](https://github.com/boostorg/asio). The Beast frontend
code is contained in [rgw_asio_frontend.h](rgw_asio_frontend.h) and
[rgw_asio_frontend.cc](rgw_asio_frontend.cc). Beast/Asio can use different
multitasking schemes, RGW uses stackful coroutines. See
```AsioFrontend::accept``` and the ```handle_connection``` function for
more details about how connections are handled on the front end.

### API/Web Server Handling
A connection coming in from the front end is handled by code in
[rgw_process.h](rgw_process.h) and [rgw_process.cc](rgw_process.cc).
The rough sequence of events in the function ```process_request```
for handling a connection event is:
* Get the environment/handler for the request
    * More on handlers later in this section
* Run Lua plugin _preRequest_ script
* Run ```schedule_request``` routine
    * This uses ```dmclock::SchedulerCompleter``` (see
      [rgw_dmclock_scheduler.h](rgw_dmclock_scheduler.h) and
      [rgw_dmclock_async_scheduler.h](rgw_dmclock_async_scheduler.h))
    * For rate limiting code see [rgw_ratelimit.h](rgw_ratelimit.h))
* Authorize/authenticate in ```rgw_process_authenticated```
* Wait for request to complete
* Run lua _postRequest_ script
* Run ```complete_request``` on ```RGWRestfulIO```, which communicates
  the result of the operation with the client. See
  [rgw_asio_client.h](rgw_asio_client.h) and
  [rgw_asio_client.cc](rgw_asio_client.cc).

A _handler_ provides context and support for an _operation_. The base classes
```RGWHandler``` and ```RGWOp``` are contained in [rgw_op.h](rgw_op.h) and
[rgw_op.cc](rgw_op.cc). An ```RGWOp``` can be many things, see
[rgw_op_type.h](rgw_op_type.h) for a full list of available operations.
```RGWHandler``` subclasses generally provide an abstraction for performing
REST operations for different APIs, _e.g._ _S3_ and _Swift_. Operations
also rely on a _driver_ for the actual storage back end. Drivers are part
of the _Storage Abstraction Layer_ (*SAL*), which abstracts the underlying
object store that backs up the client interface. The constants for S3 API names
and error codes are contained in [rgw_common.h](rgw_common.h) and
[rgw_common.cc](rgw_common.cc).

#### Notifications
_RGW_ can provide event notifications via Kafka and/or the Advanced
Mesesage Queuing Protocol. ```XXX mneufeld point out where these notifications
are implemented.```


### Storage Abstraction Layer (_SAL_)
The _SAL_ allows RGW to potentially use different backing object stores.
The primary back end is _RADOS_, but the full list contains more:
* RADOS (Primary)
* DBStore (SQLite as a data store)
* CORTX Motr (https://github.com/Seagate/cortx-motr)
* DAOS (Distributed Asynchronous Object Storage - https://docs.daos.io/v2.2/)
* Filter (not sure what this one does yet)

Generally speaking, backends other than _RADOS_ are secondary and not
as well-supported.

The files [rgw_sal.h](rgw_sal.h) and [rgw_sal.cc](rgw_sal.cc) contain the
base classes for the _SAL_ abstraction. There are some basic "starter"
_SAL_ subclasses defined in [rgw_sal_store.h](rgw_sal_store.h) and
[rgw_sal_store.cc](rgw_sal_store.cc) that are used, _e.g._ in the _RADOS_
_SAL_ implemented in
[driver/rados/rgw_sal_rados.h](driver/rados/rgw_sal_rados.h) and
[driver/rados/rgw_sal_rados.cc](driver/rados/rgw_sal_rados.cc)
Also see [rgw_sal_filter.h](rgw_sal_filter.h) and
[rgw_sal_filter.cc](rgw_sal_filter.cc), which generally appear to
delegate to a "next" object of the same base type. I'm guessing this
is to allow for multiple hooks to be attached to a SAL operation
to customize API handling.

### Ceph Classes
[Ceph Classes](https://docs.ceph.com/en/latest/architecture/#extending-ceph)
are a way of adding functionality directly at the Object Storage
Daemon. Clients can invoke remote calls provided by Classes on a connected
OSD. These may be added dynamically at runtime, or compiled directly
into the OSD. An example test class is provided in
[cls_sdk.cc](../cls/sdk/cls_sdk.cc). The header file for Classes is
[objclass.h](../include/rados/objclass.h). An example Class is provided
in [cls_hello.cc](../cls/hello/cls_hello.cc).

RGW has some Classes directly compiled into the OSD. See the following for
specific implementations of Classes for RGW:
* [cls_rgw_const.h](../cls/rgw/cls_rgw_const.h)
* [cls_rgw_ops.h](../cls/rgw/cls_rgw_ops.h)
* [cls_rgw_ops.cc](../cls/rgw/cls_rgw_ops.cc)
* [cls_rgw_types.h](../cls/rgw/cls_rgw_types.h)
* [cls_rgw_types.cc](../cls/rgw/cls_rgw_types.cc)
* [cls_rgw.cc](../cls/rgw/cls_rgw.cc)
* [cls_rgw_gc_const.h](../cls/rgw/cls_rgw_gc_const.h)
* [cls_rgw_gc_ops.h](../cls/rgw/cls_rgw_gc_ops.h)
* [cls_rgw_gc_types.h](../cls/rgw/cls_rgw_gc_types.h)
* [cls_rgw_gc.cc](../cls/rgw/cls_rgw_gc.cc)

There are also some wrapper functions around the invocation of the Classes
that may be used on the client (RGW) daemon:
* [cls_rgw_client.h](../cls/rgw/cls_rgw_client.h)
* [cls_rgw_client.cc](../cls/rgw/cls_rgw_client.cc)
* [cls_rgw_gc_client.h](../cls/rgw_gc/cls_rgw_gc_client.h)
* [cls_rgw_gc_client.cc](../cls/rgw/cls_rgw_gc_client.cc)

Inside the OSD the Class calls look to be handled in
[objclass.cc](../osd/objclass.cc). Also see
[PrimaryLogPG.h](../osd/PrimaryLogPG.h) and
[PrimaryLogPG.cc](../osd/PrimaryLogPG.cc) for some core data structures.

## Services
RGW Services are a broad category of tasks that run in the background.
Examples include synchronizing across RADOS instances and bucket
indexing/sharding. The core abstractions for a service are in:
* [rgw_service.h](driver/rados/rgw_service.h)
* [rgw_service.cc](driver/rados/rgw_service.cc)

RGW services are collected in the [services](services) subdirectory.

## Fundamental Objects

* [rgw_basic_types.h](rgw_basic_types.h)
* [rgw_user_types.h](rgw_user_types.h)
* [rgw_bucket_types.h](rgw_bucket_types.h)
* [rgw_obj_types.h](rgw_obj_types.h)
* [rgw_pool_types.h](rgw_pool_types.h)
* [rgw_zone_types.h](rgw_zone_types.h)
* [rgw_placement_types.h](rgw_placement_types.h)
* [rgw_acl_types.h](rgw_acl_types.h)
* [rgw_compression_types.h](rgw_compression_types.h)


## Tracing
Ceph has support for distributed tracing, using
[Jaeger](https://www.jaegertracing.io/) and
[OpenTelemetry](https://opentelemetry.io/). The common wrapper for
tracing is contained in [tracer.h](../common/tracer.h) and
[tracer.cc](../common/tracer.cc). The RGW-specific imports of it
are wrapped in [rgw_tracer.h](rgw_tracer.h) and
[rgw_tracer.cc](rgw_tracer.cc).

## Some Files of General Interest

### [rgw_admin.cc](rgw_admin.cc)
Handles the attributes you can set via the admin app. A bit of a
"garbage dump" which makes it useful for getting information about
varied parts of the system.

### [rgw_common.h](rgw_common.h)
Includes a bunch of stuff used across the system.

### [rgw_rest_s3.h](rgw_rest_s3.h)/[rgw_rest_s3.cc](rgw_rest_s3.cc)
Appears to manage a bunch of core stuff for the S3 HTTP server.