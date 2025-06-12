# Building CSC PyTorch container with fakeroot

Apptainer on Puhti and Mahti allow fakeroot, but not user namespaces:
https://docs.csc.fi/computing/containers/creating/#building-a-container-without-sudo-access-on-puhti-and-mahti

> Apptainer enables --fakeroot flag by default when building
> containers if sudo or namespaces are not available, this makes the
> user appear as root:root while building the container, thus enabling
> them to build images that require root file permissions e.g. to
> install packages via apt. However, this only makes the user appear
> as the root user, in the host system a user still has no additional
> permissions. By itself, fakeroot is not always sufficient, and
> building some containers may fail due to various reasons. For more
> details see the official Apptainer documentation.

The goal is to take a relatively complex real-world container and see
how far we get with only fakeroot and no user namespaces.

The [container recipe](pytorch_2.6_csc.def) is slightly modified from
the one used to create the
[pytorch/2.6](https://docs.csc.fi/apps/pytorch/) module on
Puhti. Modifications mainly needed to fix Python version changes since
it was originally used.

An important point seems to be that it uses Rocky Linux 8.10 as a
base, the same operating system as Puhti uses:

```console
Bootstrap: docker
From: rockylinux/rockylinux:8.10
```

## Starting

Instead of building on a login node I'm using an interactive session
in a compute node. The reason is that the memory is too limited on the
login node.

```console
$ sinteractive -A project_2001659 -c 8 -d 100 -m 32G -t 8:00:00
$ echo $TMPDIR
/run/nvme/job_28502350/tmp
$ export APPTAINER_CACHEDIR=$TMPDIR
$ export APPTAINER_TMPDIR=$TMPDIR
$ apptainer build pytorch_2.6_csc.sif pytorch_2.6_csc.def
INFO:    User not listed in /etc/subuid, trying root-mapped namespace
INFO:    Could not start root-mapped namespace
INFO:    The %post section will be run under the fakeroot command
INFO:    Using --fix-perms because building from a definition file
INFO:     without either root user or unprivileged user namespaces
INFO:    Starting build...
```

## Problem with rpm packages adding users

```console
  Running scriptlet: unbound-libs-1.16.2-5.8.el8_10.x86_64
groupadd: failure while writing changes to /etc/group
useradd: group 'unbound' does not exist
error: %prein(unbound-libs-1.16.2-5.8.el8_10.x86_64) scriptlet failed, exit status 6

Error in PREIN scriptlet in rpm package unbound-libs
  Installing       : python3-unbound-1.16.2-5.8.el8_10.x86_64
error: unbound-libs-1.16.2-5.8.el8_10.x86_64: install failed
```

Solution: we fake it by replacing `groupadd` and `useradd` commands
with commands that always return true - making the calling programs
assume everything worked fine, even though they do nothing.

```console
%post
  cp /usr/bin/true /usr/sbin/groupadd
  cp /usr/bin/true /usr/sbin/useradd
```

## Problem with /tmp

(Not related to fakeroot, but building on Puhti in general.)

```console
Downloading https://download.pytorch.org/whl/cu124/torch-2.6.0%2Bcu124-cp312-cp312-linux_x86_64.whl (768.4 MB)
   ━━━━━╺━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━ 100.7/768.4 MB 342.1 MB/s eta 0:00:02
ERROR: Exception:
Traceback (most recent call last):
  File "/usr/local/lib/python3.12/site-packages/pip/_vendor/urllib3/response.py", line 438, in _error_catcher
    yield
  File "/usr/local/lib/python3.12/site-packages/pip/_vendor/urllib3/response.py", line 561, in read
    data = self._fp_read(amt) if not fp_closed else b""
           ^^^^^^^^^^^^^^^^^^
  File "/usr/local/lib/python3.12/site-packages/pip/_vendor/urllib3/response.py", line 527, in _fp_read
    return self._fp.read(amt) if amt is not None else self._fp.read()
           ^^^^^^^^^^^^^^^^^^
  File "/usr/local/lib/python3.12/site-packages/pip/_vendor/cachecontrol/filewrapper.py", line 102, in read
    self.__buf.write(data)
  File "/usr/lib64/python3.12/tempfile.py", line 499, in func_wrapper
    return func(*args, **kwargs)
           ^^^^^^^^^^^^^^^^^^^^^
OSError: [Errno 28] No space left on device
```

I learned that the apptainer build process does not inherit `$TMPDIR`
from the shell's environment, and pip needs somewhere to temporarily
store huge downloads.

The easiest solution seems to be to just bind an appropriate place to
`/tmp` inside the container:

```console
apptainer build --bind=$TMPDIR:/tmp ...
```

## Success

```console
INFO:    Adding labels
INFO:    Adding environment to container
INFO:    Creating SIF file...
INFO:    Build complete: pytorch_2.6_csc.sif
```

## Resource usage

```console
Memory Utilized: 17.80 GB
```

## Testing

I ran [my normal benchmarks](https://github.com/mvsjober/ml-benchmarks) 
and everything looks good: PyTorch DataDistributed with 4 GPUs, 8 GPUs 
(2 nodes) and also DeepSpeed with 8 GPUs, which uses MPI, works!
