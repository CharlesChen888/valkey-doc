---
title: "String internals"
linkTitle: "String internals"
description: Guide to the original implementation of Strings
---

**Note: this document was written by the creator of Valkey, Salvatore Sanfilippo, early in the development of Valkey (c. 2010). Virtual Memory has been deprecated since Redis OSS 2.6, so this documentation
is here only for historical interest.**

The implementation of Strings is contained in `sds.c` (`sds` stands for
Simple Dynamic Strings). The implementation is available as a standalone library
at [https://github.com/antirez/sds](https://github.com/antirez/sds).

The C structure `sdshdr` declared in `sds.h` represents a String:

    struct sdshdr {
        long len;
        long free;
        char buf[];
    };

The `buf` character array stores the actual string.

The `len` field stores the length of `buf`. This makes obtaining the length
of a String an O(1) operation.

The `free` field stores the number of additional bytes available for use.

Together the `len` and `free` field can be thought of as holding the metadata of the `buf` character array.

Creating Strings
---

A new data type named `sds` is defined in `sds.h` to be a synonym for a character pointer:

    typedef char *sds;

`sdsnewlen` function defined in `sds.c` creates a new String:

    sds sdsnewlen(const void *init, size_t initlen) {
        struct sdshdr *sh;

        sh = zmalloc(sizeof(struct sdshdr)+initlen+1);
    #ifdef SDS_ABORT_ON_OOM
        if (sh == NULL) sdsOomAbort();
    #else
        if (sh == NULL) return NULL;
    #endif
        sh->len = initlen;
        sh->free = 0;
        if (initlen) {
            if (init) memcpy(sh->buf, init, initlen);
            else memset(sh->buf,0,initlen);
        }
        sh->buf[initlen] = '\0';
        return (char*)sh->buf;
    }

Remember a String is a variable of type `struct sdshdr`. But `sdsnewlen` returns a character pointer!!

That's a trick and needs some explanation.

Suppose I create a String using `sdsnewlen` like below:

    sdsnewlen("redis", 5);

This creates a new variable of type `struct sdshdr` allocating memory for `len` and `free`
fields as well as for the `buf` character array.

    sh = zmalloc(sizeof(struct sdshdr)+initlen+1); // initlen is length of init argument.

After `sdsnewlen` successfully creates a String the result is something like:

    -----------
    |5|0|redis|
    -----------
    ^   ^
    sh  sh->buf

`sdsnewlen` returns `sh->buf` to the caller.

What do you do if you need to free the String pointed by `sh`?

You want the pointer `sh` but you only have the pointer `sh->buf`.

Can you get the pointer `sh` from `sh->buf`?

Yes. Pointer arithmetic. Notice from the above ASCII art that if you subtract
the size of two longs from `sh->buf` you get the pointer `sh`.

The `sizeof` two longs happens to be the size of `struct sdshdr`.

Look at `sdslen` function and see this trick at work:

    size_t sdslen(const sds s) {
        struct sdshdr *sh = (void*) (s-(sizeof(struct sdshdr)));
        return sh->len;
    }

Knowing this trick you could easily go through the rest of the functions in `sds.c`.

The String implementation is hidden behind an interface that accepts only character pointers. The users of Strings need not care about how it's implemented and can treat Strings as a character pointer.
