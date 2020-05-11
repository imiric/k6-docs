---
title: "Archives for bundling/sharing a test"
excerpt: ""
---

## What is an archive?

When the complexity of a k6 test goes beyond a single JS file it quickly becomes cumbersome to
find and bundle up all the dependencies (JS, [open()](/javascript-api/init-context/open-filepath-mode)'ed data files, TLS
client certs, etc.). There is a need for a native way to bundle and distribute/share a test. That
is what k6 archives are for.

A k6 archive is simply a [tar](https://en.wikipedia.org/wiki/Tar_%28computing%29) file with all
files needed to execute a k6 test.

## How to create and run an archive

Let's say that you normally execute a test using:

<div class="code-group" data-props='{"labels": [], "lineNumbers": [true]}'>

```shell
$ k6 run script.js
```

</div>

Now if you replace `run` with `archive` k6 will run the [init stage](/using-k6/test-life-cycle) of
the code to determine which JS files are being imported and what data files are being 
[`open()`](/javascript-api/init-context/open-filepath-mode)'ed and bundles all of the files up
into a tar file:

<div class="code-group" data-props='{"labels": [], "lineNumbers": [true]}'>

```shell
$ k6 archive script.js
```

</div>

This would produce a tar file on disk called `archive.tar` (you can change that by setting
`-O filename.tar`). It's also easy to run an archive, as `k6 run` is compatible with archive
files you can execute:

> ### ⚠️ Overriding options
>
> As always you can override options using CLI flags or environment variables when
> running an archive.

<div class="code-group" data-props='{"labels": [], "lineNumbers": [true]}'>

```shell
$ k6 run archive.tar
```

</div>

## Use cases

Archive files have a variety of use cases, but they all share the common need to bundle
of a test's files into a single file for easy distribution.

### Sharing a test
By bundling up a test into an archive it's easy to share the test with your teammates by
simply storing or sending a single tar file. As we saw in the previous section, your teammates
can execute the archive by running `k6 run archive.tar`.

### Preparing tests for CI
If you have a complex CI pipeline and your load tests are separated from your application
code, you could store k6 archives as build artifacts whenever the load test source code
is changed, and then pull in those k6 archives from the artifacts storage for test execution
as needed.

### k6 Cloud Execution
k6 offers a commercial service for running large scale and geographically
distributed load tests on managed cloud infrastructure. Cloud executed tests are triggered
from the k6 command-line via the `k6 cloud script.js` command (similar to `k6 run`) which will
trigger an implicit creation of a k6 archive that is uploaded and distributed to k6 cloud
load generators for execution.

### Clustered Execution (*future*)
In the future (see [our roadmap](https://github.com/loadimpact/k6/wiki/Roadmap)) k6 will
support a clustered execution mode that will enable tests to be run across more than one
node. This mode of execution is also likely to make use of the archive functionality to
distribute the test files to all participating nodes.

## Contents of an archive file

An archive contains the original source of the JS code, any [`open()`](/javascript-api/init-context/open-filepath-mode)'ed
data files, [SSL/TLS client certificates](/using-k6/ssl-tls/ssl-tls-client-certificates) as well as a 
`metadata.json` with all the options (a cascading of the options set on the [CLI](/using-k6/options),
via [Environment variables](/using-k6/options) and [in-script options](/using-k6/options)
(`export let options = {...}`)).

Let's create an archive from the following sample test. Here is the layout in the filesystem
of the files:

<div class="code-group" data-props='{"labels": ["Sample test structure"], "lineNumbers": [true]}'>

```shell
/home/johndoe/tests/api-test $ tree
.
├── utils
|   └-- common.js
├── endpoints
|   ├── login.js
|   └-- search.js
├── node_modules
|   └-- somelib
|       └-- lib.js
├── data
|   └-- users.json
└-- script.js
```

</div>

Now, if the current working directory is `/home/johndoe/tests/api-test/` and we run
`k6 archive script.js` we'd get a tar file called `archive.tar` (you can change the name of the
file using `-O filename.tar`). The contents of the archive file would look like something like
this:


<div class="code-group" data-props='{"labels": ["Structure of archive.tar"], "lineNumbers": [true]}'>

```text
├-- data
├-- files
|   └-- home
|       └-- nobody <-- the username has been anonymized (see section further down)
|           └-- tests
|               └-- api-test
|                   └-- data
|                       └-- users.json
├-- metadata.json
└-- scripts
    └-- home
        └-- nobody <-- the username has been anonymized (see section further down)
            └-- tests
                └-- api-test
                    ├-- script.js
                    ├-- utils
                    |   └-- common.js
                    ├-- endpoints
                    |   ├-- login.js
                    |   └-- search.js
                    └-- node_modules
                        └-- somelib
                            └-- lib.js
```

</div>

Breaking down the file structure we get:

**data** contains the source code of the main JS file (`script.js` in this example).

**files** contains the full original directory tree of all [`open()`](/javascript-api/init-context/open-filepath-mode)'ed data files.

**metadata.json** The resolved "default" options for this test based on [CLI flags](/using-k6/options),
[Environment variables](/using-k6/options) and [in-script options](/using-k6/options).

***scripts*** contains the full original directory tree of all `import`'ed JS dependencies.

<div class="code-group" data-props='{"labels": ["metadata.json"], "lineNumbers": [true]}'>

```json
{
  "type": "js",
  "options": {
    "paused": null,
    "vus": null,
    "vusMax": null,
    "duration": null,
    "iterations": null,
    "stages": null,
    "setupTimeout": null,
    "teardownTimeout": null,
    "rps": null,
    "maxRedirects": null,
    "userAgent": null,
    "batch": null,
    "batchPerHost": null,
    "httpDebug": null,
    "insecureSkipTLSVerify": null,
    "tlsCipherSuites": null,
    "tlsVersion": {
      "min": "",
      "max": ""
    },
    "tlsAuth": null,
    "throw": null,
    "thresholds": null,
    "blacklistIPs": null,
    "hosts": null,
    "noConnectionReuse": null,
    "ext": null,
    "summaryTrendStats": null,
    "systemTags": [
      "url",
      "name",
      "check",
      "error",
      "tls_version",
      "method",
      "subproto",
      "status",
      "group",
      "proto"
    ],
    "tags": null
  },
  "filename": "/home/johndoe/tests/api-test/script.js",
  "pwd": "/home/johndoe/tests/api-test/",
  "env": {}
}
```

</div>

## What an archive file does not contain

We try to be cautious with what we include in an archive file. Some things we do to that end:
* We anonymize the username found in any path to JS and data file dependencies
* We don't include and environment variables from the system in the archive