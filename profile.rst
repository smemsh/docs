Profiling
~~~~~~~~~

Besides the venerable ``gprof``, there is a google tool which can be
used even without recompile (as long as binary was compiled with -pg)::

  apt install libgoogle-perftools{4,-dev}

Supposedly can add ``-lprofiler`` to link, but that did not work for me.
instead can load the shared object via ld.so env var::

  LD_PRELOAD=/usr/lib/x86_64-linux-gnu/libprofiler.so.0.5.4 \
  CPUPROFILE_REALTIME=1 \
  CPUPROFILE=/tmp/prof123.out \
  dmenu -w $WINDOWID </dev/null

There are many other flags available, REALTIME shows wall rather than
CPU time (above was used to debug an apparent IO pause).  Once executed
it will leave the profile data file, which can then be analyzed::

  $ google-pprof dmenu /tmp/prof123.out | cat
  Using local file /tmp/src/dmenu/dmenu.
  Using local file /tmp/prof123.out.
  Total: 101 samples
        99  98.0%  98.0%       99  98.0% __GI___poll
         1   1.0%  99.0%        1   1.0% __GI___clock_nanosleep
         1   1.0% 100.0%        1   1.0% __strstr_sse2_unaligned
         0   0.0% 100.0%        1   1.0% XCloseDisplay
         0   0.0% 100.0%        2   2.0% XInitExtension
         0   0.0% 100.0%       97  96.0% XNextEvent
         0   0.0% 100.0%        2   2.0% XQueryExtension
         0   0.0% 100.0%        2   2.0% XRenderFindDisplay
         0   0.0% 100.0%        1   1.0% XRenderFreeGlyphSet
         0   0.0% 100.0%        1   1.0% XRenderQueryExtension
         0   0.0% 100.0%        1   1.0% XftDefaultHasRender
         0   0.0% 100.0%        1   1.0% XftDefaultSubstitute
         0   0.0% 100.0%        1   1.0% XftFontDestroy
         0   0.0% 100.0%        1   1.0% XftFontManageMemory
         0   0.0% 100.0%        1   1.0% XftFontMatch
         0   0.0% 100.0%        1   1.0% XftFontOpenName
         0   0.0% 100.0%        1   1.0% _XOpenLC
         0   0.0% 100.0%       97  96.0% _XReadEvents
         0   0.0% 100.0%        2   2.0% _XReply
         0   0.0% 100.0%        1   1.0% _XftCloseDisplay
         0   0.0% 100.0%        1   1.0% _XftDisplayInfoGet (inline)
         0   0.0% 100.0%        1   1.0% _XftDisplayInfoGet.part.0
         0   0.0% 100.0%        1   1.0% _XimLocalOpenIM
         0   0.0% 100.0%        1   1.0% _XimLocalWcLookupString
         0   0.0% 100.0%        1   1.0% _XimOpenIM
         0   0.0% 100.0%        1   1.0% _XlcMapOSLocaleName
         0   0.0% 100.0%        1   1.0% _Xlcmbstoutf8
         0   0.0% 100.0%        1   1.0% __GI___nanosleep
         0   0.0% 100.0%      101 100.0% __libc_start_call_main
         0   0.0% 100.0%      101 100.0% __libc_start_main_impl
         0   0.0% 100.0%      101 100.0% _start
         0   0.0% 100.0%        1   1.0% drw_fontset_create
         0   0.0% 100.0%        1   1.0% grabfocus
         0   0.0% 100.0%      101 100.0% main
         0   0.0% 100.0%       99  98.0% xcb_get_setup
         0   0.0% 100.0%       97  96.0% xcb_wait_for_event
         0   0.0% 100.0%        2   2.0% xcb_wait_for_reply64


-------
columns
-------

1. samples during which function was at top of stack
2. as percentage of total samples
3. sum of percentages so far
4. samples where function was anywhere in stack
5. as percentage of total samples
