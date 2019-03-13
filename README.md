<p align="center">
    <a href="https://github.com/wujinzhou/python_sentry-sdk_bugfix_and_enhancement/tree/0.7.6" target="_blank" align="center">
        <img src="https://avatars1.githubusercontent.com/u/9814452" width="280">
    </a>
</p>

# Sentry SDK 0.7.6 Bug Fix & Enhancement 

### Fixed: Sentry SDK out of memory (OOM) problem
**What's the problem in the official release**
1. Worker queue uses an infinite queue to store captured **<u>event callbacks</u>**, the insertion is not being blocked whenever an events callback is put into the queue. This implies, if other callbacks are generated (and the number is more than one) during each callbacks's execution, **<u>then the queue size will keep increasing</u>**
2. The callback functions in the queue implements urllib3 PoolManager to send HTTP requests to server, by default, when a connection to server is failed, another 3 **retries** will be made, and during each of the retry, a warning message will be generated and captured as an event and finally put into the queue. In short, for each 1 callback is being executed, another 3 additional callbacks will be generated, so the queue size is keep growing. 
3. In bad internet conditions (for example, your sentry server is unreachable) the retry mechanism in (2) will blast the worker queue. And what make it even worse, the connection **timeout** has **not** been set by default, so all HTTP requests are in 'keep trying' states and stuck in the memory until...(**<del>海枯石烂</del>** system timeout)
4. Finally, client side **OOM**

---
### What are the fixes
**Use 4 extra configuration  options:**
* **'qmaxsize'**, for worker queue, default value **-1** (infinite queue, same as the original's default value)
* **'timeout'**, for HTTP request/response , default value **None** (no timeout queue, same as the original's default value)
* **'retries'**, for HTTP connection, default value **False** (no retry, the original retries in urllib3 is **3**)
* **'ignore_errors'**, for client's '_is_ignored_error' function, default value **[]** (no exclusion, same as the original's default value)
  * If the function callback itself emits events, it may cause an 'endless loop' just like the scenario mentioned above 
  * Then it's necessary to use **'ignore_errors'** to avoid these emissions
  * Example -- if we want to exclude 
    1) all log messages from a **specific logger**, which named 'SocketListener' (not possible in the original implementation)
    2) all 'ValueError' **exceptions** from anywhere 
    3) all **exceptions** from **module** 'foo', witch 'foo' is the module that actually generate exceptions according to stacktrace (not possible in the original implementation)
    4) all 'ZeroDivisionError' **exceptions** from **module** 'bar'
    5) all **exceptions** from **class** 'baz' **<u>and its subclass</u>**
    * We can set: **ignore_errors=['SocketListener', 'ValueError', 'foo', 'bar.ZeroDivisionError', baz]**
    
---
### How to use
**Basically the same as the original, just apply those 4 options during SDK init** 
```python
import logging
import sentry_sdk
from sentry_sdk.integrations.logging import LoggingIntegration
from urllib3.util.retry import Retry
from urllib3.util.timeout import Timeout


def init_sentry(config):

    sentry_dsn = config.SentryDsn       # http:.....my.app.com/1234'

    # All of this is already happening by default!
    sentry_logging = LoggingIntegration(
        level=logging.INFO,             # Capture this and above as breadcrumbs
        event_level=logging.WARNING     # Send as events
    )

    sentry_sdk.init(
        dsn=sentry_dsn,
        integrations=[sentry_logging],
        qmaxsize=4,                                             # default -1, infinite queue size
        timeout=Timeout(connect=2.0, read=2.0),                 # default None, no timeout
        retries=Retry(total=3, connect=1, read=1, redirect=1),  # default False, no retry
        ignore_errors=['urllib3']                               # default [] (dangerous!), should ignore waring messages captured from urllib3, in bad network conditions, waring messages from urllib3 may blast the queue and make client side oom
    )
 
```
---

<p align="center">
    <a href="https://sentry.io" target="_blank" align="center">
        <img src="https://sentry-brand.storage.googleapis.com/sentry-logo-black.png" width="280">
    </a>
</p>

# sentry-python - Sentry SDK for Python

[![Build Status](https://travis-ci.com/getsentry/sentry-python.svg?branch=master)](https://travis-ci.com/getsentry/sentry-python)

To learn about how to use the SDK:

- [Getting started with the new SDK](https://docs.sentry.io/quickstart/?platform=python)
- [Configuration options](https://docs.sentry.io/error-reporting/configuration/?platform=python)
- [Setting context (tags, user, extra information)](https://docs.sentry.io/enriching-error-data/context/?platform=python)
- [Integrations](https://docs.sentry.io/platforms/python/)

Are you coming from raven-python?

- [Cheatsheet: Migrating to the new SDK from Raven](https://forum.sentry.io/t/switching-to-sentry-python/4733)

To learn about internals:

- [API Reference (using pdoc)](https://getsentry.github.io/sentry-python/)
- [API Reference (using sphinx)](https://www.pydoc.io/search/?package=sentry-sdk)

# License

Licensed under the BSD license, see `LICENSE`
