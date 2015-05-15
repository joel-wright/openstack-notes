# Swift Service API Tutorial

The ```python-swiftclient``` provides two APIs; a low-level client API for
performing individual actions on the object store, and a higher level API
for performing common operations using a configurable thread pool.

This document focuses on solving problems using the latter high level API,
and will cover the following topics:

* Authentication
* Configuration
* Basic Operations
* Extending the Basic Operations
* Helpful Hints (And things to avoid)

### Contents

* [Introduction](#introduction)
* [Motivation](#motivation)
* [Configuration](#config)
 * [Authentication](#auth)
 * [Options](#options)
* [Basic Operations](#basic)
 * [Operation Return Values] (#operation-results)
 * [Stat](#stat-basic)
 * [Post](#post-basic)
 * [List](#list-basic)
 * [Upload](#upload-basic)
 * [Download](#download-basic)
 * [Delete](#delete-basic)
 * [Capabilities](#capabilities-basic)
* [Important Considerations](#important)

## Introduction

We'll begin with a quick example which demonstrates the usage of the service
API to perform a simple, but useful, operation on an object store.

In the example below the user deletes the given objects from the connected
storage location, whilst transparently benefitting from automatic handling of
large object deletes, multiple HTTP connections performing deletes
in multiple threads, and an iterator providing full details of the results of
each operation performed by the API.

```python
with SwiftService() as swift:
    container = 'tmp-container'
    objects_to_del = ['object1', 'path/to/object2']
    delete_iterator = swift.delete(container=container, objects=objects_to_del)
    for result in delete_iterator:
        if result['success']:
            print("Object successfully deleted: %s" % result['object'])
        else:
            print("Object failed to delete: %s" % result['object'])
```

<a name="motivation"></a>
## Motivation

What's the use case here?

The main use case for the python-swiftclient service API is for server side
tasks such as backups, archiving, parking etc. We have used this API to build
automated archiving services for large filesystems, as well as intermediate
storage space for workflows and for saving process logs to service accessible
locations.

<a name="config"></a>
## Configuration

In this section we'll cover how to provide authentication credentials to the
```SwiftService``` object, and detail some of the service options a user might
want to override.

### Authentication

Authentication can be performed in two main ways:

 * Configured environment variables
 * Loaded from config file and configured programmatically

### Configuration

When we create an instance of a ```SwiftService``` we can override a collection
of default options to suit our use case. Typically the defaults are sensible to
get us started, but depending on our needs we might want to tweak them to
improve performance (option affecting large objects and thread counts can
significantly alter performance in the right situation).

Service level defaults and some extra options can also be overridden on a
per-operation (or even in some cases per-object) basis, and we'll call out which
options affect which operations later in the document.


 * ```retries```: 5
  * The number of times that the library should attempt to retry HTTP actions
    before giving up and reporting a failure.
 * ```os_username```: ```environ.get('OS_USERNAME')```
 * ```os_password```: ```environ.get('OS_PASSWORD')```
 * ```os_tenant_name```: ```environ.get('OS_TENANT_NAME')```
 * ```os_auth_url```: ```environ.get('OS_AUTH_URL')```
  * Sohonet FileStore uses Keystone v2 auth, and these are the relevant
    options to configure access. As you can see from the default values, if
    these options are not set explicitly they will default to the values of the
    given environment variables.
 * ```container_threads```: 10
 * ```object_dd_threads```: 10
 * ```object_uu_threads```: 10
 * ```segment_threads```: 10
  * The above options determine the size of the available thread pools for
    performing swift operations. Container operations (such as listing a
    container) operate in the container threads, and a similar pattern applies
    to object and segment threads. Note that the object threads are separated
    into two separate thread pools: ```uu``` and ```dd```. This stands for
    "upload/update" and "download/delete", and the corresponding actions will
    be run on separate threads pools. We will discuss these in more detail
    when we cover extending functionality later in the document.
 * ```segment_size```: None
  * If specified, this option enables uploading of large objects. Should the
    object being uploaded be larger than 5G in size, this option is mandatory
    otherwise the upload will fail. This option should be specified as a size
    in bytes.
 * ```use_slo```: False
  * Used in combination with the above option, ```use_slo``` will upload large
    objects as static rather than dynamic. Only static large objects provide
    error checking for the downloaded object, so this option is recommended.
 * ```segment_container```: None
  * Allows the user to select the container into which large object segments
    will be uploaded. We do not recommend changing this value as it could make
    locating orphaned segments more difficult in the case of errors.
 * ```leave_segments```: False
  * Setting this option to true means that when deleting or overwriting a large
    object, its segments will be left in the object store and must be cleaned
    up manually. This option can be useful when sharing large object segments
    between multiple objects in more advanced scenarios, but must be treated
    with care, as it could lead to ever increasing storage usage.
 * ```changed```: None
  * This option affects uploads and simply means that those objects which
     already exist in the object store will not be overwritten if the mtime
     and size of the source is the same as the existing object.
 * ```skip_identical```: False
  * A slightly more thorough case of the above, but rather than mtime and size
     uses an object's md5sum (Note: currently does not work for large objects,
     although a patch is in progress to fix this).
 * ```yes_all```: False
  * This options affects only download and delete, and in each case must be
     specified in order to download/delete the entire contents of an account.
     This option has no effect on any other calls.
 * ```no_download```: False
  * This option only affects download and means that all operations proceed as
     normal with the exception that no data is written to disk.
 * ```header```: []
  * Used with upload and post operations to set headers on objects. Headers
     are specified as colon separated strings, e.g. "content-type:text/plain".
 * ```meta```: []
  * Used to set metadata on an object similarly to headers. Note that setting
     metadata is a destructive operation, so when updating one of many metadata
     values all desired metadata for an object must be re-applied.
 * ```long```: False
  * Affects only list operations, and results in more metrics being made
     available in the results at the expense of lower performance.
 * ```fail_fast```: False
  * Applies to delete and upload operations, and attempts to abort queued
     tasks in the event of errors.
 * ```prefix```: None
  * Affects list operations; only objects with the given prefix will be
     returned/affected. It is not advisable to set at the service level, as
     those operations that call list to discover objects on which they should
     operate will also be affected.
 * ```delimiter```: None
  * Affects list operations, and means that listings only contain results up
     to the first instance of the delimiter in the object name. This is useful
     for working with objects containing '/' in their names to simulate folder
     structures.
 * ```dir_marker```: False
  * Affects uploads, and allows empty 'pseudofolder' objects to be created
     when the source of an upload is ```None```. See later for further details
     on upload sources.
    
Other available options can be found in ```swiftclient/service.py``` in the
source code for ```python-swiftclient```, and each operation contains a
docstring showing which options modify its behaviour.

#### Future Options

Some patches are currently being developed that will introduce further options,
these are detailed below:
    
 * ```shuffle```: False
   * When downloading objects the current default behaviour is to shuffle lists
     of objects in order to spread the load on storage drives wehn multiple
     clients are downloading the same files to multiple locations (e.g. in the
     event of distributing an update). In a future version of the SwiftService
     code this behaviour will be optional, so that objects can be downloaded
     in listing order (when combined with a single download thread).

<a name="basic"></a>
### Basic Operations

The ```SwiftService``` object provides a context manager for performing common
operations on a swift object store.

```python
with SwiftService(options=None) as swift:
    # Do work here
```

Once created, the service object provides all the operations exposed by the
```swift``` command line tool, and each call takes an options dictionary that
allows any of the default/object options to be overridden at a per-operation
level:

```python
# Stat an account/container/object
SwiftService.stat(container=None, objects=None, options=None)

# Post to account/container/object
SwiftService.post(container=None, objects=None, options=None)

# List an account/container
SwiftService.list(container=None, options=None)

# Download an account/container/objects
SwiftService.download(container=None, objects=None, options=None)

# Upload objects to a container
SwiftService.upload(container, objects, options=None)

# Delete a container/objects
SwiftService.delete(container=None, objects=None, options=None)

# Get account capabilities
SwiftService.capabilities(url=None, options=None)
```

All the above operations above take an options `dict` to override those options
passed during the creation of the `swift` object for the current call and,
where appropriate, object lists may also take an options `dict` to override
options at a per-object level.

A swift service object is thread safe, so multiple threads can share access to
the object store through a single instance, and operations will be queued on
a single thread pool. For multiple thread pools, or different configurations
for varying workloads, multiple swift connections can safely be created by
calling ```SwiftService()``` multiple times (although in this case the user
must be aware of the potential impact of very high numbers of concurrent
connections to a single object store).

<a name="operation-results"></a>
#### Operation Return Values

Each operation provided by the service API may raise a ```SwiftError``` or
```ClientException``` for any call that fails completely (or a call which
performs only one operation at an account or container level). In the case of a
successful call an operation returns one of the following:

* A dictionary detailing the results of a single operation.
* An iterator that produces result dictionaries (for calls that perform
multiple sub-operations).

A result dictionary can indicate either the success or failure of an individual
operation (detailed in the ```success``` key), and will either contain the
successful result, or an ```error``` key detailing the error encountered
(usually an instance of Exception).

An example result dictionary is given below:

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

All the possible ```action``` values

```python
[
    'stat_account',
    'stat_container',
    'stat_object',
    'post_account',
    'post_container',
    'post_object',
    'list_part',          # list yields zero or more 'list_part' results
    'download_object',
    'create_container',   # from upload
    'create_dir_marker',  # from upload
    'upload_object',
    'upload_segment',
    'delete_container',
    'delete_object',
    'delete_segment',     # from delete_object operations
    'capabilities',
]
```

<a name="stat-basic"></a>
#### Stat

Stat can be called against an account, a container, or a list of objects to
get account stats, container stats or information about the given objects.
the first two cases a dictionary is returned containing the results of the
operation, and in the case of a list of object names being supplied, an
iterator over the results generated for each object is returned.

Information returned includes the amount of data used by the given 
object/container/account and any headers or metadata set (this includes
user set data as well as content-type and modification times).

The method docstring is given below:

```python
"""
Get account stats, container stats or information about a list of
objects in a container.

:param container: The container to query.
:param objects: A list of object paths about which to return
                information (a list of strings).
:param options: A dictionary containing options to override the global
                options specified during the service object creation.
                These options are applied to all stat operations
                performed by this call.

                    {
                        'human': False
                    }

:returns: Either a single dictionary containing stats about an account
          or container, or an iterator for returning the results of the
          stat operations on a list of objects.

:raises: SwiftError
"""
```

Valid calls for this method are as follows:

 * ```stat([options])```: Returns stats for the configured account.
 * ```stat(<container>, [options])```: Returns stats for the given container.
 * ```stat(<container>, <object_list>, [options])```: Returns stats for each
   of the given objects in the the given container (through the returned
   iterator).

<a name="post-basic"></a>
#### Post

Post can be used to set headers and metadata on a given
account/container/object. These can be purely for use by the user later, or
for specialised middleware such as the object expirer (which allows the user
to specify a time or delay, at which point the object will be deleted).

For example, the following code will set the objects ```['o1', 'o2', 'o3']```
in the container ```c1``` for the currently configured account to be deleted
after a 1 hour delay, and set a modified by metadata header to ```user```.

```python
opts = {
    "header": ["X-Delete-After:3600"]
    "meta": ["Modified-By:user"]
}
swift.post(container='c1', objects=['o1', 'o2', 'o3'], options=opts)
```

The method docstring is given below:

```python
"""
Post operations on an account, container or list of objects

:param container: The container to make the post operation against.
:param objects: A list of tuples containing an object name, and an
                options dict (can be None) to override the options for
                that individual post operation.

                    [(object, options), ...]

                The options dict is described below.
:param options: A dictionary containing options to override the global
                options specified during the service object creation.
                These options are applied to all post operations
                performed by this call, unless overridden on a per
                object basis. Possible options are given below:

                    {
                        'meta': [],
                        'headers': [],
                        'read_acl': None,   # For containers only
                        'write_acl': None,  # For containers only
                        'sync_to': None,    # For containers only
                        'sync_key': None    # For containers only
                    }

:returns: Either a single result dictionary in the case of a post to a
          container/account, or an iterator for returning the results
          of posts to a list of objects.

:raises: SwiftError
"""
```

Valid calls for this method are as follows:

 * ```post([options])```: Set the given headers/metadata for the configured
                          account.
 * ```post(<container>, [options])```: Set the given headers/metadata/acl for
                                       the given container.
 * ```post(<container>, <object_list>, [options])```: Set the given
   headers/metadata for the given objects in the the given container.

Note: A post with no metadata set will reset any metadata already set on
an object.

<a name="list-basic"></a>
#### List

The list method is used to retrieve the names and basic details about the
containers and objects within the configured account. The list
method returns an iterator that produces pages of results, which contain the 
details of up to 10,000 objects/containers. In the case of the method being
called without a specified container, the iterator will produce pages of
container names, and in the case that a container is specified the iterator
will produce pages of object names with basic details. The details returned
when listing objects includes the size and etag of the object in addition to
its name, allowing for basic validation without using the ```stat``` method.

The method docstring is given below:

```python
"""
List operations on an account, container.

:param container: The container to make the list operation against.
:param options: A dictionary containing options to override the global
                options specified during the service object creation.

                    {
                        'long': False,
                        'prefix': None,
                        'delimiter': None,
                    }

:returns: A generator for returning the results of the list operation
          on an account or container. Each result yielded from the
          generator is either a 'list_account_part' or
          'list_container_part', containing part of the listing.
"""
```

Valid calls for this method are as follows:

 * ```list([options])```: List the containers in the configured account.
 * ```list(<container>, [options])```: List the objects in the given container. 

Below is an example return value for a container list page followed by an
example of an object list page:

```python
    {
        'action': 'list_account_part',
        'container': None,
        'success': True,
        'marker': '',
        'listing': [
            {
                'count': 108849,
                'bytes': 368302494,
                'name': 'container1'
            },
            {
                'count': 1305,
                'bytes': 8876453,
                'name': 'container2'
            },
            ...
        ]
    }
```

```python
    {
        'action': 'list_container_part',
        'container': 'example_container',
        'prefix': 'test_obj',
        'success': True,
        'marker': '',
        'listing': [
            {
                'bytes': 10485760,
                'last_modified': '2014-12-11T12:02:38.774540',
                'hash': 'fb938269cbeabe4c234e1127bbd3b74a',
                'name': 'test_obj_1',
                'content_type': 'application/octet-stream'
            },
            {
                'bytes': 39874572,
                'last_modified': '2014-12-11T12:02:39.114540',
                'hash': 'f98797cd97232bbf9898989111aa166c',
                'name': 'test_obj_2',
                'content_type': 'application/octet-stream'
            },
            ...
        ],
    }
```

<a name="upload-basic"></a>
#### Upload

The upload method allows the user to create objects in the object store from
a variety of sources with complete control over the options used for each
upload.

Uploads are performed to a given container using a list containing a
possible mix of filesystem paths or ```SwiftUploadObject``` instances. Using 
```SwiftUploadObject```s allows a user to control the name of the object in
the object store, as well as upload from a ```File```-like object, or even
```None``` to create an empty object. The ```SwiftUploadObject``` also allows
a user to set per-object options for the upload operations.

```python
"""
Upload a list of objects to a given container.

:param container: The container to put the uploads into.
:param objects: A list of file/directory names (strings) or
                SwiftUploadObject instances containing a source for the
                created object, an object name, and an options dict
                (can be None) to override the options for that
                individual upload operation::

                    [
                        '/path/to/file',
                        SwiftUploadObject('/path', object_name='obj1'),
                        ...
                    ]

                The options dict is as described below.

                The SwiftUploadObject source may be one of:

                    file - A file like object (with a read method)
                    path - A string containing the path to a local file
                           or directory
                    None - Indicates that we want an empty object

:param options: A dictionary containing options to override the global
                options specified during the service object creation.
                These options are applied to all upload operations
                performed by this call, unless overridden on a per
                object basis. Possible options are given below::

                    {
                        'meta': [],
                        'headers': [],
                        'segment_size': None,
                        'use_slo': False,
                        'segment_container: None,
                        'leave_segments': False,
                        'changed': None,
                        'skip_identical': False,
                        'fail_fast': False,
                        'dir_marker': False  # Only for None sources
                    }

:returns: A generator for returning the results of the uploads.

:raises: SwiftError
:raises: ClientException
"""
```

<a name="download-basic"></a>
#### Download

```python
"""
Download operations on an account, optional container and optional list
of objects.

:param container: The container to download from.
:param options: A dictionary containing options to override the global
                options specified during the service object creation.

                    {
                        'yes_all': False,
                        'marker': '',
                        'prefix': None,
                        'no_download': False,
                        'header': [],
                        'skip_identical': False,
                        'out_file': None
                    }

:returns: A generator for returning the results of the download
          operations. Each result yielded from the generator is a
          'download_object' dictionary containing the results of an
          individual file download.

:raises: ClientException
:raises: SwiftError
"""
```

<a name="delete-basic"></a>
#### Delete

```python
"""
Delete operations on an account, optional container and optional list
of objects.

:param container: The container to delete from.
:param options: A dictionary containing options to override the global
                options specified during the service object creation.

                    {
                        'yes_all': False,
                        'leave_segments': False,
                    }

:returns: A generator for returning the results of the delete
          operations. Each result yielded from the generator is either
          a 'delete_container', 'delete_object' or 'delete_segment'
          dictionary containing the results of an individual delete
          operation.

:raises: SwiftError
:raises: ClientException
"""
```

<a name="capabilities-basic"></a>
#### Capabilities

```python
"""
List the cluster capabilities.

:param url: Proxy URL of the cluster to retrieve capabilities.

:returns: A dictionary containing the capabilities of the cluster.

:raises: ClientException
:raises: SwiftError
"""
```

<a name="important"></a>
## Important Considerations

This section covers some important considerations, helpful hints, and things
to avoid when integrating an object store into your workflow. Making sure you
understand the points raised in this section will save a lot of time and
re-architecting later.

#### An Object Store is not a Filesystem

It can't be stressed enough that your usage of the object store should reflect
the proper use case, and not treat the system like a filesystem. There are 2
main restrictions to bear in mind here when designing your use of the object
store:

 * Objects cannot be moved/renamed.
 * Objects cannot be modified.
 
Let's look at those statements in more detail:

 * Because of the way in which objects are stored and referenced by the object
   store, objects cannot be renamed because this would in most cases require
   multiple copies of the data to be moved between physical storage devices.
   Because of this a move operation is not provided, so if the user wants to
   move an object they must re-upload to the new location and delete the
   original.
 * Objects are stored in multiple locations and are checked for integrity
   based on the m5sum calculated during upload. Object creation is a 1-shot
   event, and in order to modify the contents of an object the entire new
   contents must be re-uploaded. In certain special cases it is possible to
   work around this restriction using large objects, but no general file-like
   access is available to modify a stored object.
   
#### Performance

Thread pool sizes and concurrent connections
Large objects (tarballs) vs smaller objects (upload and download speed)

#### SLO vs DLO

SLO is better for validation and specifying ordering (and creativity)
DLO is useful for log files or growing data, but has less obvious validation