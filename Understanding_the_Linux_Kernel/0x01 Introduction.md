# 1. File

    at unix world all of the thing is file

# 2. Hard link and soft(symbolic) link

## 2.1 Hard link

_`# ln p1 p2`_

new p2 link to p1, both _p1_ _p2_ are the file, chang one another change too.

- can't pass two file system
- can't create hard link for directory

## 2.2 Soft(symbolic) link

> slove the above lacking of hard link

_`# ln -s p1 p2`_

# 3. File type

- regular file
- directory
- symbolic link
- pipe, named pipe(FIFO)
- block-oriented device file
- character-oriented device file
- socket

# 4. File description and index node(inode)

File not have the attribute infomation like file lens and EOF(end-of-file).
Fs(File system) save those in inode structure, all of file have there own inode.
fs use inode to locate file.

Fs must provide the attributes of file by POSIX
- file type
- number of hard link
- file length (byte)
- device description(which include the file)
- the node number to locate file in fs
- file owner id
- file owner group id
- several time-stamp to show node change time, last access time, last modify time
- access permission and file mode

# 5. access permission and file mode

There are three type about user:
- user of file owner
- same group user without owner
- other user

all of them have three permission read,write,exec 4 2 1 rwx
and append mark, suid(Set User ID), sgid(Set Group ID).

# 6. system call of file op

`FILE fd = fopen(path, mode);`

```shell
# man fopen
NAME
       fopen, fdopen, freopen - stream open functions

SYNOPSIS
       #include <stdio.h>

       FILE *fopen(const char *path, const char *mode);

       FILE *fdopen(int fd, const char *mode);

       FILE *freopen(const char *path, const char *mode, FILE *stream);

   Feature Test Macro Requirements for glibc (see feature_test_macros(7)):

       fdopen(): _POSIX_C_SOURCE >= 1 || _XOPEN_SOURCE || _POSIX_SOURCE

```

`lseek`
```shell
NAME
       lseek - reposition read/write file offset

SYNOPSIS
       #include <sys/types.h>
       #include <unistd.h>

       off_t lseek(int fd, off_t offset, int whence);

```

and so on
