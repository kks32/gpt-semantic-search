# Overview

Through the Files service users can perform file listing, uploading, and
various operations such as move, copy, mkdir and delete. The service
also supports transferring files from one Tapis system to another.

Currently the files service includes support for systems of type LINUX,
S3 and IRODS. Other system types such as GLOBUS will be included in
future releases.

Note that supported functionality varies by system type.

All file operations act upon Tapis *System* resources.[^1] For more
information on the Systems service please see
[Systems](https://tapis.readthedocs.io/en/latest/technical/systems.html)

# Basic File Operations

## Listing

Tapis supports listing files or objects on a Tapis system. The type for
items listed will depend on system type. For example, for LINUX they
will be posix files and for S3 they will be storage objects. See the
next section below for additional considerations for S3 type systems. On
S3 systems, for example, the recurse flag is ignored and all objects
with keys matching the path as a prefix are always included.

For system types that support directory hierarchies the maximum
recursion depth is 20.

To list the files in the effective *rootDir* directory of a Tapis
system:

Using the official Tapis Python SDK:

``` python
t.files.listing("/")
```

Or using curl:

``` shell
curl -H "X-Tapis-Token: $JWT" https://tacc.tapis.io/v3/files/ops/my-system/
```

And to list a sub-directory in the system, just add the path to the
request:

Using CURL

``` shell
curl -H "X-Tapis-Token: $JWT" https://tacc.tapis.io/v3/files/ops/aturing-storage/subDir1/subDir2/subDir3/
```

Query Parameters

limit

:   integer - Max number of results to return, default of 1000

offset

:   integer - Skip the first N listings

The JSON response of the API will look something like this:

``` json
{
  "status": "success",
  "message": "ok",
  "result": [
      {
        "mimeType": "text/plain",
        "type": "file",
        "owner": "1003",
        "group": "1003",
        "nativePermissions": "drwxrwxr-x",
        "url": "tapis://dev/aturing-storage/file1.txt",
        "lastModified": "2021-04-29T16:55:57Z",
        "name": "file1.txt",
        "path": "file1.txt",
        "size": 313
      },
      {
        "mimeType": "text/plain",
        "type": "file",
        "owner": "1003",
        "group": "1003",
        "nativePermissions": "-rw-rw-r--",
        "url": "tapis://dev/aturing-storage/file2.txt",
        "lastModified": "2020-12-17T22:46:29Z",
        "name": "file2.txt",
        "path": "file2.txt",
        "size": 21
      }
  ],
  "version": "1.1-84a31617",
  "metadata": {}
}
```

### Listings and S3 Support

File listings on S3 type systems have some special considerations.
Objects in an S3 bucket do not have a hierarchical structure. There are
no directories. Everything is an object associated with a key.

One thing to note is that, as mentioned above, for S3 the recurse flag
is ignored and all objects with keys matching the path as a prefix are
always included.

Note that for S3 this means that when the path is an empty string all
objects in the bucket with a prefix matching *rootDir* will be included.
This is especially important to keep in mind when using the delete
operation to remove objects matching a path.

The attribute *rootDir* is optional for S3 type systems. When defined it
will be prepended to all paths and the resulting path will become the
key.

::: note
::: title
Note
:::

When *rootDir* is defined for an S3 system it typically should not begin
with `/`. For S3 keys are typically created and manipulated using URLs
and do not have a leading `/`.
:::

## Move and Copy

To move or copy a file or directory using the files service, make a PUT
request using the path to the current location of the file or folder.

For example, to copy a file located at [/file1.txt]{.title-ref} to
[/subdir/file1.txt]{.title-ref}

``` shell
curl -H "X-Tapis-Token: $JWT" -X PUT -d @body.json "https://tacc.tapis.io/v3/files/ops/aturing-storage/file1.txt"
```

with a JSON body of

``` json
{
  "operation": "COPY",
  "newPath": "/subdir/file1.txt"
}
```

## Uploading

To upload a file use a POST request. The file will be placed at the
location specified in the [{path}]{.title-ref} parameter in the request.
Not all system types support this operation. For example, given the
system [my-system]{.title-ref}, to insert a file in a folder located at
\`/folderA/folderB/folderC\`:

Using the official Tapis Python SDK:

``` python
with open("experiment-results.hd5", "r") as f:
    t.files.upload("my-system", "/folderA/folderB/folderC/someFile.txt", f)
```

``` shell
curl -H "X-Tapis-Token: $JWT" -X POST -F "file=@someFile.txt" https://tacc.tapis.io/v3/files/ops/my-system/folderA/folderB/folderC/someFile.txt
```

For some system types (such as LINUX) any folders that do not exist in
the specified path will automatically be created.

Note that for an S3 system an object will be created with a key of
*rootDir*/{path}.

## Deleting

To delete a file or folder, issue a DELETE request for the path to be
removed.

``` shell
curl -H "X-Tapis-Token: $JWT" -X DELETE "https://tacc.tapis.io/v3/files/ops/aturing-storage/file1.txt"
```

The request above would delete `file1.txt`

For an S3 system, the path will represent either a single object or all
objects in the bucket with a prefix matching the system *rootDir* if the
path is the empty string.

::: warning
::: title
Warning
:::

For an S3 system if the path is the empty string, then all objects in
the bucket with a key matching the prefix *rootDir* will be deleted. So
if the *rootDir* is also the empty string, then all objects in the
bucket will be removed.
:::

## Creating a directory

To create a directory, use POST and provide the path to the new
directory in the request body. Not all system types support this
operation.

``` shell
$ curl -H "X-Tapis-Token: $JWT" -X POST -d @body.json -X POST https://tacc.tapis.io/v3/files/ops/my-system
```

with a JSON body of

``` json
{
  "path": "path/to/new/directory/"
}
```

## Getting Linux stat information

Get native stat information for a file or directory for a system of type
LINUX.

For example, for [/subdir/file1.txt]{.title-ref}

``` shell
curl -H "X-Tapis-Token: $JWT" "https://tacc.tapis.io/v3/files/utils/linux/aturing-storage/subdir/file1.txt"
```

## Running a Linux native operation

Run a native operation on a path. Operations are *chmod*, *chown* or
*chgrp*. For a system of type LINUX.

For example, to change the owner of a file located at
[/file1.txt]{.title-ref} to `aeinstein`

``` shell
curl -H "X-Tapis-Token: $JWT" -X POST -d @body.json "https://tacc.tapis.io/v3/files/utils/linux/aturing-storage/file1.txt"
```

with a JSON body of

``` json
{
  "operation": "CHOWN",
  "argument": "aeinstein"
}
```

# Content

Get file or directory contents as a stream of data. Not supported for
all system types.

## File Contents - Serving files

To return the actual contents (raw bytes) of a file:

``` shell
$ curl -H "X-Tapis-Token: $JWT" https://tacc.tapis.io/v3/files/content/my-system/image.jpg > image.jpg
```

Query Parameters

startByte

:   integer - Start at byte N of the file

count

:   integer - Return this number of bytes after startByte

zip

:   boolean - Zip the contents of a folder

Header Parameters

more

:   integer - Return 1 KB chunks of UTF-8 encoded text from a file
    starting after page *more*. This call can be used to page through a
    text based file. Note that if the contents of the file are not
    textual (such as an image file or other binary format), the output
    will be bizarre.

# Transfers

File transfers are used to move data between Tapis systems. They should
be used for bulk data operations that are too large for the REST api to
perform. Transfers occur *asynchronously*, and are executed concurrently
where possible to increase performance. As such, the order in which the
files are transferred is not deterministic.

When a transfer is initiated, a *bill of materials* is created that
creates a record of all the files from the *sourceUri* that are to be
transferred to the *destinationUri*. Unless otherwise specified, all
files in the *bill of materials* must transfer successfully in order for
the overall transfer to be considered successful. A transfer task has an
attribute named *status* which is updated as the transfer progresses.
The possible states for a transfer are:

ACCEPTED

:   The initial request has been processed and saved.

IN_PROGRESS

:   The bill of materials has been created and transfers are either in
    flight or waiting to begin.

FAILED

:   The transfer failed.

COMPLETED

:   The transfer completed successfully, all files have been transferred
    to the target system.

Unauthenticated HTTP endpoints are also possible to use as a source for
transfers. This method can be utilized to include outputs from other
APIs into Tapis jobs.

The number of files included in the *bill of materials* will depend on
the system types and the *sourceUri* values provided in the transfer
request. If the source system supports directories and *sourceUri* is a
directory then the directory will be processed recursively and all files
will be added to the *bill of materials*. If the source system is of
type S3 then all objects matching the *sourceUri* path as a prefix will
be included.

## System types and supported functionality

As discussed above the files included in a transfer will depend on the
source system types and the *sourceUri* values provided in the transfer
request. Here is a summary of the behavior:

*LINUX/IRODS to LINUX/IRODS*

:   When the *sourceUri* is a directory a recursive listing is made and
    the files and directory structure are replicated on the
    *destinationUri* system.

*S3 to LINUX/IRODS*

:   All objects matching the *sourceUri* path as a prefix will be
    created as files on the *destinationUri* system.

*LINUX/IRODS to S3*

:   When the *sourceUri* is a directory a recursive listing is made. For
    each entry in the listing the path relative to the source system
    rootDir is mapped to a key for the S3 destination system. In other
    words, a recursive listing is made for the directory on the
    *sourceUri* system and for each non-directory entry an object is
    created on the S3 *destinationUri* system.

*S3 to S3*

:   All objects matching the *sourceUri* path as a prefix will be
    re-created as objects on the *destinationUri* system.

*HTTP/S to ANY*

:   Transfer of a directory is not supported. The content of the object
    from the *sourceUri* URL is used to create a single file or object
    on the *destinationUri* system.

*ANY to HTTP/S*

:   Transfers not supported. Tapis does not support the use of protocol
    http/s for the *destinationUri*.

## Creating Transfers

Lets say our user `aturing` needs to transfer data between two systems
that are registered in tapis. The source system has an id of
`aturing-storage` with the results of an experiment located in directory
`/experiments/experiment-1/` that should be transferred to a system with
id `aturing-compute`

``` shell
curl -H "X-Tapis-Token: $JWT" -X POST -d @body.json https://tacc.tapis.io/v3/files/tranfers
```

``` json
{
  "tag": "An optional identifier",
  "elements": [
    {
      "sourceUri": "tapis://aturing-storage/experiments/experiment-1/",
      "destinationUri": "tapis://aturing-compute/"
    }
  ]
}
```

The request above will initiate a transfer that copies all files and
folders in the `experiment-1` folder on the source system to the root
directory of the destination system `aturing-compute`

### HTTP Source

Unauthenticated HTTP/S endpoints can also be used as a source for a file
transfer request. This can be useful, for instance, when the inputs for
a job are from a separate web service, or perhaps stored in a public S3
bucket. Note that in this case the *sourceUri* does not refer to a Tapis
system.

``` shell
curl -H "X-Tapis-Token: $JWT" -X POST -d @body.json https://tacc.tapis.io/v3/files/tranfers
```

``` json
{
  "tag": "An optional identifier",
  "elements": [
    {
      "sourceUri": "https://some-web-application.io/calculations/12345/results.csv",
      "destinationUri": "tapis://aturing-compute/inputs.csv"
    }
  ]
}
```

The request above will place the output of the source URI into a file
called `inputs.csv` in the `aturing-compute` system.

## Getting transfer information

To retrieve information about a transfer including status and bytes
transferred, simply make a GET request to the transfers API with the
UUID of the transfer.

``` shell
curl -H "X-Tapis-Token: $JWT"  https://tacc.tapis.io/v3/files/tranfers/{UUID}
```

The JSON response should look something like :

``` json
{
  "status": "success",
  "message": "ok",
  "result": {
    "id": 1,
    "username": "aturing",
    "tenantId": "tacc",
    "tag": "some tag",
    "uuid": "b2dcf71a-bb7b-409a-8c01-1bbs97e749fb",
    "status": "COMPLETED",
    "parentTasks": [
      {
        "id": 17,
        "tenantId": "tacc",
        "username": "aturing",
        "sourceURI": "tapis://sourceSystem/file1.txt",
        "destinationURI": "tapis://destSystem/folderA/",
        "totalBytes": 100000,
        "bytesTransferred": 100000,
        "taskId": 1,
        "children": null,
        "errorMessage": null,
        "uuid": "8fdccda6-a504-4ddf-9464-7b22sa66bcc4",
        "status": "COMPLETED",
        "created": "2021-04-22T14:21:58.933851Z",
        "startTime": "2021-04-22T14:21:59.862356Z",
        "endTime": "2021-04-22T14:22:09.389847Z"
      }
    ],
    "estimatedTotalBytes": 100000,
    "totalBytesTransferred": 100000,
    "totalTransfers": 1,
    "completeTransfers": 1,
    "errorMessage": null,
    "created": "2021-04-22T14:21:58.933851Z",
    "startTime": "2021-04-22T14:21:59.838928Z",
    "endTime": "2021-04-22T14:22:09.376740Z"
  },
  "version": "1.1-094fd38d",
  "metadata": {}
}
```

# File Permissions

The permissions model allows for fine grained access control of paths on
a Tapis system. The system owner may grant READ and MODIFY permission to
specific users. MODIFY implies READ.

Please note that Tapis permissions are independent of native permissions
enforced by the underlying system host.

## Getting permissions

Get the Tapis permissions for a user for the system and path. If no user
specified then permissions are retrieved for the user making the
request.

``` shell
curl -H "X-Tapis-Token: $JWT" https://tacc.tapis.io/v3/files/perms/aturing-storage/experiment1?username=aeinstein
```

## Granting permissions

Lets say our user `aturing` has a system with ID `aturing-storage`. Alan
wishes to allow his collaborator `aeinstein` to view the results of an
experiment located at `/experiment1`

``` shell
curl -H "X-Tapis-Token: $JWT" -d @body.json -X POST https://tacc.tapis.io/v3/files/perms/aturing-storage/experiment1/
```

with a JSON body with the following shape:

``` json
{
  "username": "aeinstein",
  "permission": "READ"
}
```

Other users can also be granted permission to write to the system by
granting the `MODIFY` permission. The JSON body would then be:

``` json
{
  "username": "aeinstein",
  "permission": "MODIFY"
}
```

## Revoking permissions

Our user `aturing` now wishes to revoke his former collaborators access
to the folder above. He can issue a DELETE request on the path and
specify the username in order to revoke access:

``` shell
curl -H "X-Tapis-Token: $JWT" -X DELETE https://tacc.tapis.io/v3/files/perms/aturing-storage/experiment1?username=aeinstein
```

# File Sharing

In addition to fine grained permissions support, Tapis also supports a
higher level approach to granting access. This approach is known simply
as *sharing*. The sharing API allows you to share a path with a set of
users as well as share publicly with all users in a tenant. Sharing a
path grants READ access to the path.

Please note that the underlying host associated with a system typically
also has it\'s own access controls.

## Getting share information

Retrieve all sharing information for a path on a system. This includes
all users with whom the path has been shared and whether or not the path
has been made publicly available.

``` shell
curl -H "X-Tapis-Token: $JWT" https://tacc.tapis.io/v3/files/share/aturing-storage/experiment1
```

## Sharing a path with users

Create or update sharing information for a path on a system. The path
will be shared with the list of users provided in the request body.
Requester must be owner of the system. For LINUX systems path sharing is
hierarchical.

``` shell
curl -H "X-Tapis-Token: $JWT" -d @body.json -X POST https://tacc.tapis.io/v3/files/share/aturing-storage/experiment1/
```

with a JSON body with the following shape:

``` json
{
  "users": [ "aeinstein", "rfeynman" ]
}
```

## Sharing a path publicly

Share a path on a system with all users in the tenant. Requester must be
owner of the system.

``` shell
curl -H "X-Tapis-Token: $JWT" -X POST https://tacc.tapis.io/v3/files/share_public/aturing-storage/experiment1/
```

## Unsharing a path with users

Update sharing information for a path on a system. The path will be
unshared with the list of users provided in the request body. Requester
must be owner of the system.

``` shell
curl -H "X-Tapis-Token: $JWT" -d @body.json -X POST https://tacc.tapis.io/v3/files/unshare/aturing-storage/experiment1/
```

with a JSON body with the following shape:

``` json
{
  "users": [ "rfeynman" ]
}
```

## Unsharing a path publicly

Remove public sharing for a path on a system. Requester must be owner of
the system.

``` shell
curl -H "X-Tapis-Token: $JWT" -X POST https://tacc.tapis.io/v3/files/unshare_public/aturing-storage/experiment1/
```

## Removing all shares for a path

Remove all shares for a path on a system including public access. If the
path is a directory this will also be done for all sub-paths.

``` shell
curl -H "X-Tapis-Token: $JWT" -X POST https://tacc.tapis.io/v3/files/unshare_all/aturing-storage/experiment1/
```