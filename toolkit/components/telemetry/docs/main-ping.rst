
"main" ping
===========

This is the "main" Telemetry ping type, whose payload contains most of the measurements that are used to track the performance and health of Firefox in the wild.
It includes the histograms and other performance and diagnostic data.

This ping is triggered by different scenarios, which is documented by the ``reason`` field:

* ``aborted-session`` - this ping is regularly saved to disk (every 5 minutes), overwriting itself, and deleted at shutdown. If a previous aborted session ping is found at startup, it gets sent to the server. The first aborted-session ping is generated as soon as Telemetry starts
* ``environment-change`` - the :doc:`environment` changed, so the session measurements got reset and a new subsession starts
* ``shutdown`` - triggered when the browser session ends
* ``daily`` - a session split triggered in 24h hour intervals at local midnight. If an ``environment-change`` ping is generated by the time it should be sent, the daily ping is rescheduled for the next midnight
* ``saved-session`` - the *"classic"* Telemetry payload with measurements covering the whole browser session (only submitted for a transition period)

Most reasons lead to a session split, initiating a new *subsession*. We reset important measurements for those subsessions.

*Note:* ``saved-session`` is sent with a different ping type (``saved-session``, not ``main``), but otherwise has the same format as discussed here.

Structure::

    {
      version: 4,

      info: {
        reason: <string>, // what triggered this ping: "saved-session", "environment-change", "shutdown", ...
        revision: <string>, // the Histograms.json revision
        timezoneOffset: <number>, // time-zone offset from UTC, in minutes, for the current locale
        previousBuildId: <string>, // null if this is the first run, or the previous build ID is unknown

        sessionId: <uuid>,  // random session id, shared by subsessions
        subsessionId: <uuid>,  // random subsession id
        previousSessionId: <uuid>, // session id of the previous session, null on first run.
        previousSubsessionId: <uuid>, // subsession id of the previous subsession (even if it was in a different session),
                                      // null on first run.

        subsessionCounter: <number>, // the running no. of this subsession since the start of the browser session
        profileSubsessionCounter: <number>, // the running no. of all subsessions for the whole profile life time

        sessionStartDate: <ISO date>, // daily precision
        subsessionStartDate: <ISO date>, // daily precision, ISO date in local time
        sessionLength: <number>, // the session length until now in seconds, monotonic
        subsessionLength: <number>, // the subsession length in seconds, monotonic

        flashVersion: <string>, // obsolete, use ``environment.addons.activePlugins``
        addons: <string>, // obsolete, use ``environment.addons``
      },

      childPayloads: {...}, // only present with e10s; a reduced payload from content processes, null on failure
      simpleMeasurements: {...},

      // The following properties may all be null if we fail to collect them.
      histograms: {...},
      keyedHistograms: {...},
      chromeHangs: {...},
      threadHangStats: [...],
      log: [...],
      webrtc: {...},
      fileIOReports: {...},
      lateWrites: {...},
      addonDetails: {...},
      addonHistograms: {...},
      UIMeasurements: {...},
      slowSQL: {...},
      slowSQLstartup: {...},
    }

info
----

sessionLength
~~~~~~~~~~~~~
The length of the current session so far in seconds.
This uses a monotonic clock, so this may mismatch with other measurements that
are not monotonic like calculations based on ``Date.now()``.

If the monotonic clock failed, this will be ``-1``.

subsessionLength
~~~~~~~~~~~~~~~~
The length of this subsession in seconds.
This uses a monotonic clock, so this may mismatch with other measurements that are not monotonic (e.g. based on Date.now()).

If ``sessionLength`` is ``-1``, the monotonic clock is not working.

threadHangStats
~~~~~~~~~~~~~~~
Contains the statistics about the hangs in main and background threads. Note that hangs in this section capture the [C++ pseudostack](https://developer.mozilla.org/en-US/docs/Mozilla/Performance/Profiling_with_the_Built-in_Profiler#Native_stack_vs._Pseudo_stack) and an incomplete JS stack, which is not 100% precise.

To avoid submitting overly large payloads, some limits are applied:

* Identical, adjacent "(chrome script)" or "(content script)" stack entries are collapsed together. If a stack is reduced, the "(reduced stack)" frame marker is added as the oldest frame.
* The depth of the reported stacks is limited to 11 entries. This value represents the 95th percentile of the thread hangs stack depths reported by Telemetry.

Structure::

    "threadHangStats" : [
      {
        "name" : "Gecko",
        "activity" : {...}, // a time histogram of all task run times
        "hangs" : [
          {
            "stack" : [
              "Startup::XRE_Main",
              "Timer::Fire",
              "(content script)",
              "IPDL::PPluginScriptableObject::SendGetChildProperty",
              ... up to 11 frames ...
            ],
            "nativeStack": [...], // optionally available
            "histogram" : {...}, // the time histogram of the hang times
            "annotations" : [
              {
                "pluginName" : "Shockwave Flash",
                "pluginVersion" : "18.0.0.209"
              },
              ... other annotations ...
            ]
          },
        ],
      },
      ... other threads ...
     ]

chromeHangs
~~~~~~~~~~~
Contains the statistics about the hangs happening exclusively on the main thread of the parent process. Precise C++ stacks are reported. This is only available on Nightly Release on Windows, when building using "--enable-profiling" switch.

Some limits are applied:

* Reported chrome hang stacks are limited in depth to 50 entries.
* The maximum number of reported stacks is 50.

Structure::

    "chromeHangs" : {
      "memoryMap" : [
        ["wgdi32.pdb", "08A541B5942242BDB4AEABD8C87E4CFF2"],
        ["igd10iumd32.pdb", "D36DEBF2E78149B5BE1856B772F1C3991"],
        ... other entries in the format ["module name", "breakpad identifier"] ...
       ],
      "stacks" : [
        [
          [
            0, // the module index or -1 for invalid module indices
            190649 // the offset of this program counter in its module or an absolute pc
          ],
          [1, 2540075],
          ... other frames, up to 50 ...
         ],
         ... other stacks, up to 50 ...
      ],
      "durations" : [8, ...], // the hang durations (in seconds)
      "systemUptime" : [692, ...], // the system uptime (in minutes) at the time of the hang
      "firefoxUptime" : [672, ...], // the Firefox uptime (in minutes) at the time of the hang
      "annotations" : [
        [
          [0, ...], // the indices of the related hangs
          {
            "pluginName" : "Shockwave Flash",
            "pluginVersion" : "18.0.0.209",
            ... other annotations as key:value pairs ...
          }
        ],
        ...
      ]
    },
