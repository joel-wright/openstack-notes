Service API Proposal for python-swiftclient
===

The current ```python-swiftclient``` provides a low-level client API for
performing individual actions on the object store, and a command line utility.
The code for the command line utility contains a large amount of logic for
batch operations and multithreading, but none of this code is presented in a
way that is terribly useful for end users. This means that developers using the
```python-swiftclient``` code in their own projects must recreate the logic
contained in ```shell.py```, which obviously has the disadvantage of missing
bug fixes and features later added in client library.

This design document proposes a number of changes to make this high-level logic
available to developers using the client library.

### Contents

 - [Goal](#Goal)
 - [Current API](#Current API)
 - [Proposal](#Proposal)
   - [Service API](#Service API)
   - [Operation Results](#Operation Results)
   - [Multithreading](#Multithreading)
 - [Examples](#Example Usage)
 - [Review](#Review)

## Goal

The goal of this document is to propose an update to ```python-swiftclient```
to make the logic in ```shell.py``` available to developers via a new API:

  - Move the vast majority of the operation logic from ```shell.py``` into a
    new file, ```service.py```.
  - Redesign the threading code in ```multithreading.py``` to allow multiple
    simultaneous operations.
  - Provide a new high-level re-entrant API in ```service.py``` for accessing
    the object store from end user projects.
  - Convert the existing ```shell.py``` code to make use of the new high-level
    API.

## Current API

The current API provides a low-level API for performing individual actions on
an object store, and a command line utility with high-level

...

## Proposal

Let's start this section off with a quick example which demonstrates a simple
case of the proposed usage of the service API. In the example below the end
user would benefit from automatic handling of large object deletes, multiple
connections to ```swift``` performing deletes in multiple threads, and an
iterator providing full details of the results of each operation performed by
the API.

```python
with SwiftService() as swift:
    container = 'tmp-container'
    files = ['/tmp/file1', '/tmp/file2']
    delete_iterator = swift.delete(container=container, objects=files)
    for result in delete_iterator:
        if result['success']:
            print("Object successfully deleted: %s" % result['object'])
        else:
            print("Object failed to delete: %s" % result['object'])
```

Having shown the example above, the new API proposal is as follows:

 - A context manager ```SwiftService``` providing a re-entrant connection to
   ```swift``` with a managed thread and connection pool for multiple
   operations.
 - A standard dictionary based result style containing all details of the
   operation performed, an indication of whether the operation was performed
   successfully and all information provided by the low level client API.  

### Service API

The ```SwiftService``` object provides a context manager for managing access to
a swift object store. This object can be created without any options, and will
default to a set of options that correspond to the default options that would
have been provided by the command line utility.

```python
with SwiftService(options=None, timeout=864000) as swift:
    # Do work here
```

Once created, the service API provides all the operations exposed by the
```swift``` command line utility, each having an option to override any of the
default/object options at a per-operation level:

```python
# Stat an account/container/object
SwiftService.stat(container=None, obj=None, options=None)

# Post to account/container/object
SwiftService.post(container=None, obj=None, options=None)

# List an account/container
SwiftService.list(container=None, options=None)

# Download an account/container/objects
SwiftService.download(container=None, objects=None, options=None)

# Upload objects to a container
SwiftService.upload(container, objs, options=None)

# Delete a container/objects
SwiftService.delete(container=None, objects=None, options=None)

# Get account capabilities
SwiftService.capabilities(url=None, options=None)
```

All the above operations behave in the obvious fashion except for upload, which
has a specification for the object list as follows (example usage of this API
follows later):

```python
"""
Upload a list of objects where an object is a tuple containing:

  (file, obj) - A file like object (with a read method) and the
                name of the object it will be uploaded into.
                Modified time will be 'now'.
  (path, obj) - A path to a local file and the name of the object
                it will be uploaded as. Modified time data will be
                taken from the path.
  (None, obj, dir) - Create an empty object with an mtime of 'now', and
                     create this as a dir marker if dir is True.
"""
```

The default options are provided in ```service.py```, and any option may be
overridden by supplying a dictionary containing the updated options.

```python
_default_global_options = {
    "snet": False,
    "verbose": 1,
    "debug": False,
    "info": False,
    "auth": environ.get('ST_AUTH'),
    "auth_version": environ.get('ST_AUTH_VERSION', '1.0'),
    "user": environ.get('ST_USER'),
    "key": environ.get('ST_KEY'),
    "retries": 5,
    "os_username": environ.get('OS_USERNAME'),
    "os_password": environ.get('OS_PASSWORD'),
    "os_tenant_id": environ.get('OS_TENANT_ID'),
    "os_tenant_name": environ.get('OS_TENANT_NAME'),
    "os_auth_url": environ.get('OS_AUTH_URL'),
    "os_auth_token": environ.get('OS_AUTH_TOKEN'),
    "os_storage_url": environ.get('OS_STORAGE_URL'),
    "os_region_name": environ.get('OS_REGION_NAME'),
    "os_service_type": environ.get('OS_SERVICE_TYPE'),
    "os_endpoint_type": environ.get('OS_ENDPOINT_TYPE'),
    "os_cacert": environ.get('OS_CACERT'),
    "insecure": config_true_value(environ.get('SWIFTCLIENT_INSECURE')),
    "ssl_compression": False,
    "segment_threads": 10,
    "object_dd_threads": 10,
    "object_uu_threads": 10,
    "container_threads": 10
}

_default_local_options = {
    'sync_to': None,
    'sync_key': None,
    'use_slo': False,
    'segment_size': None,
    'segment_container': None,
    'leave_segments': False,
    'changed': None,
    'skip_identical': False,
    'yes_all': False,
    'read_acl': None,
    'write_acl': None,
    'out_file': None,
    'no_download': False,
    'long': False,
    'totals': False,
    'marker': '',
    'header': [],
    'meta': [],
    'prefix': None,
    'delimiter': None,
    'fail_fast': True,
    'human': False
```



### Operation Results

Each operation provided by the service API returns a dictionary as a result,
either as via an iterator in the case that the operation may perform multiple
operations, or directly in the case of single operation calls. An example
result dictionary is given below:

```python
result = {
    'action': 'download_object',
    'success': True,
    'container': container,
    'object': obj,
    'path': path,
    'start_time': start_time,
    'finish_time': finish_time,
    'headers_receipt': headers_receipt,
    'auth_end_time': conn.auth_end_time,
    'read_length': bytes_read,
    'attempts': conn.attempts
}
```

Where the possible ```action``` values are as follows:

```python
[
    'post',
    'list',
    'download_object',
    'upload_object',
    'create_container',
    'create_dir_marker',
    'upload_segment',
    'upload_object',
    'delete_segment',
    'delete_object',
    'delete_container',
]
```

NOTES: Need to follow the API style for stat and capabilities, and update stat
to use 'options' not 'opts', and capabilities to take an 'options' key/value

### Multithreading

Before this update, the multithreading code provided two main functions;
performing multiple threaded operations and presenting a thread safe interface
to outputting to ```stderr``` and ```stdout```. Whilst the latter code can be
included in this update, the former has a number of issues:

 - Thread pools are created per _operation_ and it is therefore impossible to
   share a thread pool between multiple operations.
 - The current threading code assumes that a thread pool is created, used and
   destroyed for each operation, and therefore cannot be used for a long
   running re-entrant API.

This update therefore maintains the output interface as ```OutputManager``` and
introduces a new threading object, based on ```Concurrent.Futures```, called
```MultiThreadingManager``` which creates a thread pool for operations on each
of _container_, _object_ and _segment_ (actually, in order to avoid potential
deadlocks, the _object_ pools are split into two separate pools, one for
uploading/updating objects and one for downloading/deleting objects). Each
thread pool, ```ConnectionThreadPoolExecutor```, maintains and re-uses a
collection of connections to swift, lazily creating them based on demand.

The updated multithreading code can be found in ```multithreading.py```.

## Example Usage

This section show some example usage of the new service API with examples drawn
from the port of the existing command line utility to the new code.

...

## Review

This update is currently under review, so to provide comments or suggestions,
please see the review page [here](https://review.openstack.org/#/c/85453/).
