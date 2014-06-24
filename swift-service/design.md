Service API Proposal for python-swiftclient
===

The current ```python-swiftclient``` provides a low-level client API for
performing individual actions on the object store, and a command line utility.
The code for the command line utility contains a large amount of logic for
batch operations and multithreading, but none of this code is presented in a
way that is terribly useful for end users because it mixes logic and output,
and requires a new thread pool for each operation. This means that developers
using the ```python-swiftclient``` code in their own projects must recreate
the logic contained in ```shell.py```, which obviously has the disadvantage of
missing bug fixes and features later added in the client library.

This design document proposes a number of changes to make the high-level,
multithreaded logic in ```shell.py``` available to developers via a new
re-entrant "service" API.

### Contents

* [Goal](#goal)
* [Current API](#current-api)
* [Proposal](#proposal)
 * [Service API](#service-api)
 * [Operation Results](#operation-results)
 * [Multithreading](#multithreading)
* [Examples](#example-usage)
 * [Upload](#upload)
 * [Delete](#delete)
* [Review](#review)

## Goal

The goal of this document is to propose an update to ```python-swiftclient```
to make the high-level, multithreaded logic in ```shell.py``` available to
developers via a new API. We propose the following changes:

* Move swift operation logic from ```shell.py``` into a new
file, ```service.py```.
* Provide a new high-level re-entrant API in ```service.py``` for accessing
the object store from multiple threads in end user projects.
* Convert the existing ```shell.py``` code to make use of the new high-level
API as an example of usage.

## Current API

The current API provides low-level operations for performing individual actions
on a swift object store, and a command line utility that contains a large amount
of high-level application logic for performing operations on multiple objects
using a thread pool. This high level logic cannot however be easily integrated
into end-user projects because it mixes program logic with output for the
command line utility, and requires a new thread pool for each operation. 

## Proposal

The API proposed in this document has been implemented and is currently under
review [here](https://review.openstack.org/#/c/85453/).

We'll begin the description with a quick example which demonstrates a simple
case of the proposed usage of the service API. In the example below the end
user would benefit from automatic handling of large object deletes, multiple
connections to ```swift``` performing deletes in multiple threads, and an
iterator providing full details of the results of each operation performed by
the API.

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

Having shown the example above, the new API proposal is as follows:

* Move swift operation logic from ```shell.py``` into a new
file, ```service.py```.
* Redesign the threading code in ```multithreading.py``` to allow multiple
simultaneous operations.
* Provide a context manager ```SwiftService``` giving a re-entrant connection to
```swift``` with a managed thread and connection pool for multiple
operations.
* Deliver results in a standard dictionary based style containing all details of
the operation performed, an indication of whether the operation was performed
successfully and all information provided by the low level client API.  

### Service API

The ```SwiftService``` object provides a context manager for managing access to
a swift object store. This object can be created without any options, and will
default to a set of options that correspond to the default options of the ```swift```
command line tool.

```python
with SwiftService(options=None) as swift:
    # Do work here
```

Once created, the service API provides all the operations exposed by the ```swift```
command line tool, and each call takes an options dictionary that allows any of
the default/object options to be overridden at a per-operation level:

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

All the above operations behave in the obvious fashion, either taking a string
or list of strings to represent object/container names, except for upload, which
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

The default options for the ```SwiftService``` object are provided in ```service.py```
(as shown below), and any option may be overridden by supplying a dictionary
containing the updated options, either when instantiating the ```SwiftService```
object, or for each individual operation call.

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

Each operation provided by the service API either raises a ```SwiftError```
if the operation failed completely, or returns one of the following:

* A dictionary detailing the results of the operation.
* An iterator that produces result dictionaries (for calls that perform multiple
sub-operations).

A result dictionary can indicate both success and failure (detailed in
the ```success``` key), and will either contain the successful result, or
an ```error``` key detailing the error encountered.

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

Where the possible ```action``` values are as follows:

```python
[
    'stat'
    'post',
    'list',
    'download_object',
    'create_container',
    'upload_object',
    'upload_segment',
    'create_dir_marker',
    'delete_container',
    'delete_object',
    'delete_segment',
    'capabilities',
]
```

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
from the port of the existing command line utility to the new code. The purpose
of this section is twofold; we wish to demonstrate how this API can be used to
make end user code cleaner and more readable, and as a demonstration of how
much logic is currently tied up in the command line client.

Necessarily these examples are quite large, but they do nicely demonstrate the
separation of operations and output that can be achieved after separating the
program logic into operations on an instance of a swift connection, and the
command line utility input/output operations.

### Upload

We begin with the best example; uploading into the object store.

At present ```shell.py``` contains a great deal of logic for uploading files
into the object store, and mixes program output with the logic for deciding
whether an object should be uploaded as an SLO/DLO, as well as code for
creating containers and uploading object segments.

In contrast, when all this high level logic is moved into ```service.py```
the only thing the command line utility needs to do is parse command line 
arguments, provide a list of files to upload, and process the results in order
to give feedback to the user.

And onto the comparison, first the code from the current ```shell.py```:  

```python
    (options, args) = parse_args(parser, args)
    args = args[1:]
    if len(args) < 2:
        thread_manager.error(
            'Usage: %s upload %s\n%s', BASENAME, st_upload_options,
            st_upload_help)
        return

    def _segment_job(job, conn):
        if job.get('delete', False):
            conn.delete_object(job['container'], job['obj'])
        else:
            fp = open(job['path'], 'rb')
            fp.seek(job['segment_start'])
            seg_container = args[0] + '_segments'
            if options.segment_container:
                seg_container = options.segment_container
            etag = conn.put_object(job.get('container', seg_container),
                                   job['obj'], fp,
                                   content_length=job['segment_size'])
            job['segment_location'] = '/%s/%s' % (seg_container, job['obj'])
            job['segment_etag'] = etag
        if options.verbose and 'log_line' in job:
            if conn.attempts > 1:
                thread_manager.print_msg('%s [after %d attempts]',
                                         job['log_line'], conn.attempts)
            else:
                thread_manager.print_msg(job['log_line'])
        return job

    def _object_job(job, conn):
        path = job['path']
        container = job.get('container', args[0])
        dir_marker = job.get('dir_marker', False)
        object_name = job['object_name']
        try:
            if object_name is not None:
                object_name.replace("\\", "/")
                obj = object_name
            else:
                obj = path
                if obj.startswith('./') or obj.startswith('.\\'):
                    obj = obj[2:]
                if obj.startswith('/'):
                    obj = obj[1:]
            put_headers = {'x-object-meta-mtime': "%f" % getmtime(path)}
            if dir_marker:
                if options.changed:
                    try:
                        headers = conn.head_object(container, obj)
                        ct = headers.get('content-type')
                        cl = int(headers.get('content-length'))
                        et = headers.get('etag')
                        mt = headers.get('x-object-meta-mtime')
                        if ct.split(';', 1)[0] == 'text/directory' and \
                                cl == 0 and \
                                et == 'd41d8cd98f00b204e9800998ecf8427e' and \
                                mt == put_headers['x-object-meta-mtime']:
                            return
                    except ClientException as err:
                        if err.http_status != 404:
                            raise
                conn.put_object(container, obj, '', content_length=0,
                                content_type='text/directory',
                                headers=put_headers)
            else:
                # We need to HEAD all objects now in case we're overwriting a
                # manifest object and need to delete the old segments
                # ourselves.
                old_manifest = None
                old_slo_manifest_paths = []
                new_slo_manifest_paths = set()
                if options.changed or options.skip_identical \
                        or not options.leave_segments:
                    if options.skip_identical:
                        checksum = None
                        try:
                            fp = open(path, 'rb')
                        except IOError:
                            pass
                        else:
                            with fp:
                                md5sum = md5()
                                while True:
                                    data = fp.read(65536)
                                    if not data:
                                        break
                                    md5sum.update(data)
                            checksum = md5sum.hexdigest()
                    try:
                        headers = conn.head_object(container, obj)
                        cl = int(headers.get('content-length'))
                        mt = headers.get('x-object-meta-mtime')
                        if (options.skip_identical and
                                checksum == headers.get('etag')):
                            thread_manager.print_msg(
                                "Skipped identical file '%s'", path)
                            return
                        if options.changed and cl == getsize(path) and \
                                mt == put_headers['x-object-meta-mtime']:
                            return
                        if not options.leave_segments:
                            old_manifest = headers.get('x-object-manifest')
                            if config_true_value(
                                    headers.get('x-static-large-object')):
                                headers, manifest_data = conn.get_object(
                                    container, obj,
                                    query_string='multipart-manifest=get')
                                for old_seg in json.loads(manifest_data):
                                    seg_path = old_seg['name'].lstrip('/')
                                    if isinstance(seg_path, unicode):
                                        seg_path = seg_path.encode('utf-8')
                                    old_slo_manifest_paths.append(seg_path)
                    except ClientException as err:
                        if err.http_status != 404:
                            raise
                # Merge the command line header options to the put_headers
                put_headers.update(split_headers(options.header, '',
                                                 thread_manager))
                # Don't do segment job if object is not big enough
                if options.segment_size and \
                        getsize(path) > int(options.segment_size):
                    seg_container = container + '_segments'
                    if options.segment_container:
                        seg_container = options.segment_container
                    full_size = getsize(path)

                    slo_segments = []
                    error_counter = [0]
                    segment_manager = thread_manager.queue_manager(
                        _segment_job, options.segment_threads,
                        store_results=slo_segments,
                        error_counter=error_counter,
                        connection_maker=create_connection)
                    with segment_manager as segment_queue:
                        segment = 0
                        segment_start = 0
                        while segment_start < full_size:
                            segment_size = int(options.segment_size)
                            if segment_start + segment_size > full_size:
                                segment_size = full_size - segment_start
                            if options.use_slo:
                                segment_name = '%s/slo/%s/%s/%s/%08d' % (
                                    obj, put_headers['x-object-meta-mtime'],
                                    full_size, options.segment_size, segment)
                            else:
                                segment_name = '%s/%s/%s/%s/%08d' % (
                                    obj, put_headers['x-object-meta-mtime'],
                                    full_size, options.segment_size, segment)
                            segment_queue.put(
                                {'path': path, 'obj': segment_name,
                                 'segment_start': segment_start,
                                 'segment_size': segment_size,
                                 'segment_index': segment,
                                 'log_line': '%s segment %s' % (obj, segment)})
                            segment += 1
                            segment_start += segment_size
                    if error_counter[0]:
                        raise ClientException(
                            'Aborting manifest creation '
                            'because not all segments could be uploaded. %s/%s'
                            % (container, obj))
                    if options.use_slo:
                        slo_segments.sort(key=lambda d: d['segment_index'])
                        for seg in slo_segments:
                            seg_loc = seg['segment_location'].lstrip('/')
                            if isinstance(seg_loc, unicode):
                                seg_loc = seg_loc.encode('utf-8')
                            new_slo_manifest_paths.add(seg_loc)

                        manifest_data = json.dumps([
                            {'path': d['segment_location'],
                             'etag': d['segment_etag'],
                             'size_bytes': d['segment_size']}
                            for d in slo_segments])

                        put_headers['x-static-large-object'] = 'true'
                        conn.put_object(container, obj, manifest_data,
                                        headers=put_headers,
                                        query_string='multipart-manifest=put')
                    else:
                        new_object_manifest = '%s/%s/%s/%s/%s/' % (
                            quote(seg_container), quote(obj),
                            put_headers['x-object-meta-mtime'], full_size,
                            options.segment_size)
                        if old_manifest and old_manifest.rstrip('/') == \
                                new_object_manifest.rstrip('/'):
                            old_manifest = None
                        put_headers['x-object-manifest'] = new_object_manifest
                        conn.put_object(container, obj, '', content_length=0,
                                        headers=put_headers)
                else:
                    conn.put_object(
                        container, obj, open(path, 'rb'),
                        content_length=getsize(path), headers=put_headers)
                if old_manifest or old_slo_manifest_paths:
                    segment_manager = thread_manager.queue_manager(
                        _segment_job, options.segment_threads,
                        connection_maker=create_connection)
                    segment_queue = segment_manager.queue
                    if old_manifest:
                        scontainer, sprefix = old_manifest.split('/', 1)
                        scontainer = unquote(scontainer)
                        sprefix = unquote(sprefix).rstrip('/') + '/'
                        for delobj in conn.get_container(scontainer,
                                                         prefix=sprefix)[1]:
                            segment_queue.put(
                                {'delete': True,
                                 'container': scontainer,
                                 'obj': delobj['name']})
                    if old_slo_manifest_paths:
                        for seg_to_delete in old_slo_manifest_paths:
                            if seg_to_delete in new_slo_manifest_paths:
                                continue
                            scont, sobj = \
                                seg_to_delete.split('/', 1)
                            segment_queue.put(
                                {'delete': True,
                                 'container': scont, 'obj': sobj})
                    if not segment_queue.empty():
                        with segment_manager:
                            pass
            if options.verbose:
                if conn.attempts > 1:
                    thread_manager.print_msg('%s [after %d attempts]', obj,
                                             conn.attempts)
                else:
                    thread_manager.print_msg(obj)
        except OSError as err:
            if err.errno != ENOENT:
                raise
            thread_manager.error('Local file %r not found', path)

    def _upload_dir(path, object_queue, object_name):
        names = listdir(path)
        if not names:
            object_queue.put({'path': path, 'object_name': object_name,
                             'dir_marker': True})
        else:
            for name in listdir(path):
                subpath = join(path, name)
                subobjname = None
                if object_name is not None:
                    subobjname = join(object_name, name)
                if isdir(subpath):
                    _upload_dir(subpath, object_queue, subobjname)
                else:
                    object_queue.put({'path': subpath,
                                     'object_name': subobjname})

    create_connection = lambda: get_conn(options)
    conn = create_connection()

    # Try to create the container, just in case it doesn't exist. If this
    # fails, it might just be because the user doesn't have container PUT
    # permissions, so we'll ignore any error. If there's really a problem,
    # it'll surface on the first object PUT.
    try:
        conn.put_container(args[0])
        if options.segment_size is not None:
            seg_container = args[0] + '_segments'
            if options.segment_container:
                seg_container = options.segment_container
            conn.put_container(seg_container)
    except ClientException as err:
        msg = ' '.join(str(x) for x in (err.http_status, err.http_reason))
        if err.http_response_content:
            if msg:
                msg += ': '
            msg += err.http_response_content[:60]
        thread_manager.error(
            'Error trying to create container %r: %s', args[0],
            msg)
    except Exception as err:
        thread_manager.error(
            'Error trying to create container %r: %s', args[0],
            err)

    if options.object_name is not None:
        if len(args[1:]) > 1:
            thread_manager.error('object-name only be used with 1 file or dir')
            return
    object_name = options.object_name

    object_manager = thread_manager.queue_manager(
        _object_job, options.object_threads,
        connection_maker=create_connection)
    with object_manager as object_queue:
        try:
            for arg in args[1:]:
                if isdir(arg):
                    _upload_dir(arg, object_queue, object_name)
                else:
                    object_queue.put({'path': arg, 'object_name': object_name})
        except ClientException as err:
            if err.http_status != 404:
                raise
            thread_manager.error('Account not found')
```

And now the same version of the repository ported to the new service API:

```python
    (options, args) = parse_args(parser, args)
    args = args[1:]
    if len(args) < 2:
        output_manager.error(
            'Usage: %s upload %s\n%s', BASENAME, st_upload_options,
            st_upload_help)
        return
    else:
        container = args[0]
        files = args[1:]

    if options.object_name is not None:
        if len(files) > 1:
            output_manager.error('object-name only be used with 1 file or dir')
            return
        else:
            orig_path = files[0]

    _opts = vars(options)
    _opts['object_uu_threads'] = options.object_threads
    with SwiftService(options=_opts) as swift:
        try:
            objs = []
            dir_markers = []
            for f in files:
                if isfile(f):
                    objs.append(f)
                elif isdir(f):
                    for (_dir, _ds, _fs) in walk(f):
                        if not (_ds + _fs):
                            dir_markers.append(_dir)
                        else:
                            objs.extend([join(_dir, _f) for _f in _fs])
                else:
                    output_manager.error("Local file '%s' not found" % f)

            # Now that we've collected all the required files and dir markers
            # build the tuples for the call to upload
            if options.object_name is not None:
                objs = [
                    (o, o.replace(orig_path, options.object_name, 1))
                    for o in objs
                ]
                dir_markers = [
                    (None, d.replace(orig_path, options.object_name, 1),
                     True) for d in dir_markers
                ]
            else:
                objs = list(zip(objs, objs))
                dir_markers = [(None, d, True) for d in dir_markers]

            for r in swift.upload(container, objs + dir_markers):
                if r['success']:
                    if options.verbose:
                        if 'attempts' in r and r['attempts'] > 1:
                            if 'object' in r:
                                output_manager.print_msg(
                                    '%s [after %d attempts]' %
                                    (r['object'],
                                     r['attempts'])
                                )
                        else:
                            if 'object' in r:
                                output_manager.print_msg(r['object'])
                            elif 'for_object' in r:
                                output_manager.print_msg(
                                    '%s segment %s' % (r['for_object'],
                                                       r['segment_index'])
                                )
                else:
                    error = r['error']
                    if isinstance(error, SwiftError):
                        output_manager.error(error.value)
                    else:
                        output_manager.error("Unexpected Error during upload: "
                                             "%s" % error)

        except SwiftError as e:
            output_manager.error(e.value)
```

### Delete

Similarly to the upload example, the current ```shell.py``` code mixes output
with the program logic required to delete objects from the store, whilst also
dealing with SLO/DLO segments and removing empty containers. Again, when
ported to the new service API, program logic and output are separated, and all
the logic is made available to end users as an importable library.

Again, we start with the code from the current ```shell.py```:

```python
    (options, args) = parse_args(parser, args)
    args = args[1:]
    if (not args and not options.yes_all) or (args and options.yes_all):
        thread_manager.error('Usage: %s delete %s\n%s',
                             BASENAME, st_delete_options,
                             st_delete_help)
        return

    def _delete_segment(item, conn):
        (container, obj) = item
        conn.delete_object(container, obj)
        if options.verbose:
            if conn.attempts > 2:
                thread_manager.print_msg(
                    '%s/%s [after %d attempts]', container,
                    obj, conn.attempts)
            else:
                thread_manager.print_msg('%s/%s', container, obj)

    def _delete_object(item, conn):
        (container, obj) = item
        try:
            old_manifest = None
            query_string = None
            if not options.leave_segments:
                try:
                    headers = conn.head_object(container, obj)
                    old_manifest = headers.get('x-object-manifest')
                    if config_true_value(
                            headers.get('x-static-large-object')):
                        query_string = 'multipart-manifest=delete'
                except ClientException as err:
                    if err.http_status != 404:
                        raise
            conn.delete_object(container, obj, query_string=query_string)
            if old_manifest:
                segment_manager = thread_manager.queue_manager(
                    _delete_segment, options.object_threads,
                    connection_maker=create_connection)
                segment_queue = segment_manager.queue
                scontainer, sprefix = old_manifest.split('/', 1)
                scontainer = unquote(scontainer)
                sprefix = unquote(sprefix).rstrip('/') + '/'
                for delobj in conn.get_container(scontainer,
                                                 prefix=sprefix)[1]:
                    segment_queue.put((scontainer, delobj['name']))
                if not segment_queue.empty():
                    with segment_manager:
                        pass
            if options.verbose:
                path = options.yes_all and join(container, obj) or obj
                if path[:1] in ('/', '\\'):
                    path = path[1:]
                if conn.attempts > 1:
                    thread_manager.print_msg('%s [after %d attempts]', path,
                                             conn.attempts)
                else:
                    thread_manager.print_msg(path)
        except ClientException as err:
            if err.http_status != 404:
                raise
            thread_manager.error("Object '%s/%s' not found", container, obj)

    def _delete_container(container, conn, object_queue):
        try:
            marker = ''
            while True:
                objects = [o['name'] for o in
                           conn.get_container(container, marker=marker)[1]]
                if not objects:
                    break
                for obj in objects:
                    object_queue.put((container, obj))
                marker = objects[-1]
            while not object_queue.empty():
                sleep(0.05)
            attempts = 1
            while True:
                try:
                    conn.delete_container(container)
                    break
                except ClientException as err:
                    if err.http_status != 409:
                        raise
                    if attempts > 10:
                        raise
                    attempts += 1
                    sleep(1)
        except ClientException as err:
            if err.http_status != 404:
                raise
            thread_manager.error('Container %r not found', container)

    create_connection = lambda: get_conn(options)
    obj_manager = thread_manager.queue_manager(
        _delete_object, options.object_threads,
        connection_maker=create_connection)
    with obj_manager as object_queue:
        cont_manager = thread_manager.queue_manager(
            _delete_container, options.container_threads, object_queue,
            connection_maker=create_connection)
        with cont_manager as container_queue:
            if not args:
                conn = create_connection()
                try:
                    marker = ''
                    while True:
                        containers = [
                            c['name']
                            for c in conn.get_account(marker=marker)[1]]
                        if not containers:
                            break
                        for container in containers:
                            container_queue.put(container)
                        marker = containers[-1]
                except ClientException as err:
                    if err.http_status != 404:
                        raise
                    thread_manager.error('Account not found')
            elif len(args) == 1:
                if '/' in args[0]:
                    print(
                        'WARNING: / in container name; you might have meant '
                        '%r instead of %r.' % (
                            args[0].replace('/', ' ', 1), args[0]),
                        file=stderr)
                container_queue.put(args[0])
            else:
                for obj in args[1:]:
                    object_queue.put((args[0], obj))
```

And again, the same version ported to the new service API:

```python
    (options, args) = parse_args(parser, args)
    args = args[1:]
    if (not args and not options.yes_all) or (args and options.yes_all):
        output_manager.error('Usage: %s delete %s\n%s',
                             BASENAME, st_delete_options,
                             st_delete_help)
        return

    _opts = vars(options)
    _opts['object_dd_threads'] = options.object_threads
    with SwiftService(options=_opts) as swift:
        try:
            if not args:
                del_iter = swift.delete()
            else:
                container = args[0]
                if '/' in container:
                    output_manager.error(
                        'WARNING: / in container name; you '
                        'might have meant %r instead of %r.' % (
                        container.replace('/', ' ', 1), container)
                    )
                    return
                objects = args[1:]
                if objects:
                    del_iter = swift.delete(container=container,
                                            objects=objects)
                else:
                    del_iter = swift.delete(container=container)

            for r in del_iter:
                if r['success']:
                    if options.verbose:
                        if r['action'] == 'delete_object':
                            c = r['container']
                            o = r['object']
                            p = '%s/%s' % (c, o) if options.yes_all else o
                            a = r['attempts']
                            if a > 1:
                                output_manager.print_msg(
                                    '%s [after %d attempts]', p, a)
                            else:
                                output_manager.print_msg(p)

                        elif r['action'] == 'delete_segment':
                            c = r['container']
                            o = r['object']
                            p = '%s/%s' % (c, o)
                            a = r['attempts']
                            if a > 1:
                                output_manager.print_msg(
                                    '%s [after %d attempts]', p, a)
                            else:
                                output_manager.print_msg(p)

                else:
                    # Special case error prints
                    output_manager.error("An unexpected error occurred whilst"
                                         "deleting: %s" % r['error'])
        except SwiftError as err:
            output_manager.error(err.value)
```

## Review

This update is currently under review, so to provide comments or suggestions,
please see the review page [here](https://review.openstack.org/#/c/85453/).
