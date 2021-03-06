.. SPDX-License-Identifier: GPL-2.0

====================================================
pin_user_pages() and related calls
====================================================

.. contents:: :local:

Overview
========

This document describes the following functions::

 pin_user_pages()
 pin_user_pages_fast()
 pin_user_pages_remote()

Basic description of FOLL_PIN
=============================

FOLL_PIN and FOLL_LONGTERM are flags that can be passed to the get_user_pages*()
("gup") family of functions. FOLL_PIN has significant interactions and
interdependencies with FOLL_LONGTERM, so both are covered here.

FOLL_PIN is internal to gup, meaning that it should not appear at the gup call
sites. This allows the associated wrapper functions  (pin_user_pages*() and
others) to set the correct combination of these flags, and to check for problems
as well.

FOLL_LONGTERM, on the other hand, *is* allowed to be set at the gup call sites.
This is in order to avoid creating a large number of wrapper functions to cover
all combinations of get*(), pin*(), FOLL_LONGTERM, and more. Also, the
pin_user_pages*() APIs are clearly distinct from the get_user_pages*() APIs, so
that's a natural dividing line, and a good point to make separate wrapper calls.
In other words, use pin_user_pages*() for DMA-pinned pages, and
get_user_pages*() for other cases. There are four cases described later on in
this document, to further clarify that concept.

FOLL_PIN and FOLL_GET are mutually exclusive for a given gup call. However,
multiple threads and call sites are free to pin the same struct pages, via both
FOLL_PIN and FOLL_GET. It's just the call site that needs to choose one or the
other, not the struct page(s).

The FOLL_PIN implementation is nearly the same as FOLL_GET, except that FOLL_PIN
uses a different reference counting technique.

FOLL_PIN is a prerequisite to FOLL_LONGTERM. Another way of saying that is,
FOLL_LONGTERM is a specific case, more restrictive case of FOLL_PIN.

Which flags are set by each wrapper
===================================

For these pin_user_pages*() functions, FOLL_PIN is OR'd in with whatever gup
flags the caller provides. The caller is required to pass in a non-null struct
pages* array, and the function then pin pages by incrementing each by a special
value. For now, that value is +1, just like get_user_pages*().::

 Function
 --------
 pin_user_pages          FOLL_PIN is always set internally by this function.
 pin_user_pages_fast     FOLL_PIN is always set internally by this function.
 pin_user_pages_remote   FOLL_PIN is always set internally by this function.

For these get_user_pages*() functions, FOLL_GET might not even be specified.
Behavior is a little more complex than above. If FOLL_GET was *not* specified,
but the caller passed in a non-null struct pages* array, then the function
sets FOLL_GET for you, and proceeds to pin pages by incrementing the refcount
of each page by +1.::

 Function
 --------
 get_user_pages           FOLL_GET is sometimes set internally by this function.
 get_user_pages_fast      FOLL_GET is sometimes set internally by this function.
 get_user_pages_remote    FOLL_GET is sometimes set internally by this function.

Tracking dma-pinned pages
=========================

Some of the key design constraints, and solutions, for tracking dma-pinned
pages:

* An actual reference count, per struct page, is required. This is because
  multiple processes may pin and unpin a page.

* False positives (reporting that a page is dma-pinned, when in fact it is not)
  are acceptable, but false negatives are not.

* struct page may not be increased in size for this, and all fields are already
  used.

* Given the above, we can overload the page->_refcount field by using, sort of,
  the upper bits in that field for a dma-pinned count. "Sort of", means that,
  rather than dividing page->_refcount into bit fields, we simple add a medium-
  large value (GUP_PIN_COUNTING_BIAS, initially chosen to be 1024: 10 bits) to
  page->_refcount. This provides fuzzy behavior: if a page has get_page() called
  on it 1024 times, then it will appear to have a single dma-pinned count.
  And again, that's acceptable.

This also leads to limitations: there are only 31-10==21 bits available for a
counter that increments 10 bits at a time.

TODO: for 1GB and larger huge pages, this is cutting it close. That's because
when pin_user_pages() follows such pages, it increments the head page by "1"
(where "1" used to mean "+1" for get_user_pages(), but now means "+1024" for
pin_user_pages()) for each tail page. So if you have a 1GB huge page:

* There are 256K (18 bits) worth of 4 KB tail pages.
* There are 21 bits available to count up via GUP_PIN_COUNTING_BIAS (that is,
  10 bits at a time)
* There are 21 - 18 == 3 bits available to count. Except that there aren't,
  because you need to allow for a few normal get_page() calls on the head page,
  as well. Fortunately, the approach of using addition, rather than "hard"
  bitfields, within page->_refcount, allows for sharing these bits gracefully.
  But we're still looking at about 8 references.

This, however, is a missing feature more than anything else, because it's easily
solved by addressing an obvious inefficiency in the original get_user_pages()
approach of retrieving pages: stop treating all the pages as if they were
PAGE_SIZE. Retrieve huge pages as huge pages. The callers need to be aware of
this, so some work is required. Once that's in place, this limitation mostly
disappears from view, because there will be ample refcounting range available.

* Callers must specifically request "dma-pinned tracking of pages". In other
  words, just calling get_user_pages() will not suffice; a new set of functions,
  pin_user_page() and related, must be used.

FOLL_PIN, FOLL_GET, FOLL_LONGTERM: when to use which flags
==========================================================

Thanks to Jan Kara, Vlastimil Babka and several other -mm people, for describing
these categories:

CASE 1: Direct IO (DIO)
-----------------------
There are GUP references to pages that are serving
as DIO buffers. These buffers are needed for a relatively short time (so they
are not "long term"). No special synchronization with page_mkclean() or
munmap() is provided. Therefore, flags to set at the call site are: ::

    FOLL_PIN

...but rather than setting FOLL_PIN directly, call sites should use one of
the pin_user_pages*() routines that set FOLL_PIN.

CASE 2: RDMA
------------
There are GUP references to pages that are serving as DMA
buffers. These buffers are needed for a long time ("long term"). No special
synchronization with page_mkclean() or munmap() is provided. Therefore, flags
to set at the call site are: ::

    FOLL_PIN | FOLL_LONGTERM

NOTE: Some pages, such as DAX pages, cannot be pinned with longterm pins. That's
because DAX pages do not have a separate page cache, and so "pinning" implies
locking down file system blocks, which is not (yet) supported in that way.

CASE 3: Hardware with page faulting support
-------------------------------------------
Here, a well-written driver doesn't normally need to pin pages at all. However,
if the driver does choose to do so, it can register MMU notifiers for the range,
and will be called back upon invalidation. Either way (avoiding page pinning, or
using MMU notifiers to unpin upon request), there is proper synchronization with
both filesystem and mm (page_mkclean(), munmap(), etc).

Therefore, neither flag needs to be set.

In this case, ideally, neither get_user_pages() nor pin_user_pages() should be
called. Instead, the software should be written so that it does not pin pages.
This allows mm and filesystems to operate more efficiently and reliably.

CASE 4: Pinning for struct page manipulation only
-------------------------------------------------
Here, normal GUP calls are sufficient, so neither flag needs to be set.

page_dma_pinned(): the whole point of pinning
=============================================

The whole point of marking pages as "DMA-pinned" or "gup-pinned" is to be able
to query, "is this page DMA-pinned?" That allows code such as page_mkclean()
(and file system writeback code in general) to make informed decisions about
what to do when a page cannot be unmapped due to such pins.

What to do in those cases is the subject of a years-long series of discussions
and debates (see the References at the end of this document). It's a TODO item
here: fill in the details once that's worked out. Meanwhile, it's safe to say
that having this available: ::

        static inline bool page_dma_pinned(struct page *page)

...is a prerequisite to solving the long-running gup+DMA problem.

Another way of thinking about FOLL_GET, FOLL_PIN, and FOLL_LONGTERM
===================================================================

Another way of thinking about these flags is as a progression of restrictions:
FOLL_GET is for struct page manipulation, without affecting the data that the
struct page refers to. FOLL_PIN is a *replacement* for FOLL_GET, and is for
short term pins on pages whose data *will* get accessed. As such, FOLL_PIN is
a "more severe" form of pinning. And finally, FOLL_LONGTERM is an even more
restrictive case that has FOLL_PIN as a prerequisite: this is for pages that
will be pinned longterm, and whose data will be accessed.

Unit testing
============
This file::

 tools/testing/selftests/vm/gup_benchmark.c

has the following new calls to exercise the new pin*() wrapper functions:

* PIN_FAST_BENCHMARK (./gup_benchmark -a)
* PIN_BENCHMARK (./gup_benchmark -b)

You can monitor how many total dma-pinned pages have been acquired and released
since the system was booted, via two new /proc/vmstat entries: ::

    /proc/vmstat/nr_foll_pin_requested
    /proc/vmstat/nr_foll_pin_requested

Those are both going to show zero, unless CONFIG_DEBUG_VM is set. This is
because there is a noticeable performance drop in unpin_user_page(), when they
are activated.

References
==========

* `Some slow progress on get_user_pages() (Apr 2, 2019) <https://lwn.net/Articles/784574/>`_
* `DMA and get_user_pages() (LPC: Dec 12, 2018) <https://lwn.net/Articles/774411/>`_
* `The trouble with get_user_pages() (Apr 30, 2018) <https://lwn.net/Articles/753027/>`_

John Hubbard, October, 2019
