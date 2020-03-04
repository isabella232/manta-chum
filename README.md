# chum - storage load generator

`chum` is a load generator for WebDAV or S3 servers.

## How it works

`chum` creates a number of threads. Each of these threads will synchronously
upload or download files from the target server. The data uploaded is a
chunk of random bytes.

`chum` supports the S3 and WebDAV client protocols.

Upload file size distribution is an important part of how `chum` works. `chum`
includes a default object size distribution if one is not provided at the CLI.
The CLI option for specifying a file size distribution is `-d`. When a `chum`
thread uploads a file it will choose a random file size from the provided
size distribution list. To skew the result of the distribution more of a given
size should be specified at the CLI.

For example, take the file size distribution [128k, 256k, 512k]. Over the course
of three file upload loops a single `chum` thread will choose from this
distribution randomly and upload files. Maybe the first upload was chosen to be
512k, the second was 128k, and the third was 512k.

Now let's say that we want to simulate a workload where 80% of uploads are
128k, and the remaining 20% are 512k and 1m. We can use a distribution like
[128, 128, 128, 128, 128, 128, 128, 128, 512, 1024]. Assuming the random
selection is truly random in the limit, 8/10 uploads will be 128k in size,
1/10 will be 512k in size, and 1/10 will be 1m in size.

There are two ways to specify the previous distribution at the CLI.

Long form:
```
-d 128k,128k,128k,128k,128k,128k,128k,128k,512k,1024k 
```
Short form:
```
-d 128k:8,512k,1m
```

Using the short form, a given size AxB is interpreted as 'add B copies of
A to the distribution.' The long and short form examples provided result in
equivalent distributions.

Another thing to keep in mind is the ratio of read operations to write
operations. This is configurable with the `-w` flag and follows the same
shorthand as the file size distribution argument.

For example, a 50/50 read/write workload is specified like so:
```
-w r,w
```
and a 80/20 read/write workload could be specified like this:
```
-w r:8,w:2
```
A read-only workload like so:
```
-w r
```
A write-only workload like so:
```
-w w
```
And a 50/50 write/delete workload:
```
-w w,d
```

The ID of objects uploaded are added to a queue. IDs are taken from the queue
whenever a read request is started. The behavior of the queue can be changed to
simulate a specific workload: LRU, MRU, and random addressing. See the `q`
argument and queue.rs for more details.

## Running

First, make sure that the 'chum' directory is created in the file server root.
This is where `chum` writes files. For nginx the directory must be owned by
`nobody:nobody`.

```
(nginx) $ mkdir /manta/chum
(nginx) $ chown nobody:nobody /manta/chum
```

Getting help:

```
$ chum -h
chum - Upload files to a given file server as quickly as possible

Options:
    -t, --target [s3|webdav]:IP
                        target server
    -c, --concurrency NUM
                        number of concurrent threads, default: 1
    -s, --sleep NUM     sleep duration in millis between each upload, default:
                        0
    -d, --distribution NUM:COUNT,NUM:COUNT,...
                        comma-separated distribution of file sizes to upload,
                        default: 128k,256k,512k
    -i, --interval NUM  interval in seconds at which to report stats, default:
                        2
    -q, --queue-mode lru|mru|rand
                        queue mode for read operations, default: rand
    -w, --workload OP:COUNT,OP:COUNT
                        workload of operations, default: r,w
    -f, --format h|v|t  statistics output format, default: h
    -m, --max-data CAP  maximum amount of data to write to the target,
                        default: 0, '0' disables cap
    -r, --read-list FILE
                        path to a file listing files to read from server,
                        default: none (files are chosen from recent uploads)
    -h, --help          print this help message
```

A target is required at a minimum:
```
$ chum -t webdav:127.0.0.1
```

Target a local nginx server, 50 worker threads, an object size distribution of
[1m, 2m, 3m], each thread sleeping 1000ms between uploads:

```
$ chum -t webdav:127.0.0.1 -c 50 -d 1m,2m,3m -s 1000
```

`chum` defaults to using port 80 for WebDAV targets and port 9000 for S3
targets. S3 client credentials default to the MinIO default client creds. These
can be changed by setting the `AWS_SECRET_ACCESS_KEY` and `AWS_ACCESS_KEY_ID`
environment variables.

Valid values for the `--format` argument:
- `h` - human readable output
- `v` - verbose human readable output
- `t` - computer readable tabular output

## Building

On SmartOS we recommend using image `f3a6e1a2-9d71-11e9-9bd2-e7e5b4a5c141`,
a recent base-64 image.

The following packages are required to download and build `chum` in that base-64
zone. The can be installed via pkgsrc via `pkgin(1)`

```
build-essential git rust-1.35.0nb1
```

To build:
```
$ cd chum
$ cargo build
```

## Notes

The delete workload is currently only supported for the S3 client.

## License

"chum" is licensed under the
[Mozilla Public License version 2.0](http://mozilla.org/MPL/2.0/).
See the file LICENSE.
