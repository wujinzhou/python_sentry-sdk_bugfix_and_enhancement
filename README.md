<p align="center">
    <a href="https://github.com/wujinzhou/python_sentry-sdk_bugfix_and_enhancement/tree/0.7.6" target="_blank" align="center">
        <img src="https://avatars1.githubusercontent.com/u/9814452" width="280">
    </a>
</p>

# Sentry SDK 0.7.6 Bug Fix & Enhancement 

### Fixed: Sentry SDK out of memory (OOM) problem
**What's the problem**
1. Worker queue uses an infinite queue to store captured **<u>event callbacks</u>**, the insertion will not be blocked whenever an events callback is put into the  queue 
2. However, the event callback functions in the queue are blocked functions, this implies, if the worker thread cannot digest these blocked functions fast enough, **<u>the queue size will keep increasing</u>**
3. The blocked callback functions in the queue uses urllib3 PoolManager to send HTTP requests to server, by default, when a connection to server is failed, up to 3 times of **retries** will be made, and also by default, the connection timeout has **not** been set
4. In bad internet conditions (for example, your sentry server is unreachable) the retry mechanism in (3) will take a very long time, and what make things even worse, since the timeout has not been set, your HTTP requests is being blocked until...(**海枯石烂**),  As a consequence, this lead (2) to happen -- the worker queue size become larger and larger....
5. Finally, client side **OOM**

---
**What's the fixes**
Add 3 extra configuration  options:
-**'qmaxsize'**, for worker queue, default value **-1** (infinite queue, same as original)
-**'timeout'**, for HTTP request/response , default value **None** (no timeout queue, same as original)
-**'retries'**: for HTTP connection, default value **False** (no retry, the original value in urllib3 is **3**)

---
**How to use**
--**Basically the same as the original, just add those 3 additional options in SDK init** 
```python
import logging  
import sentry_sdk  
from sentry_sdk.integrations.logging import LoggingIntegration  
from urllib3.util.retry import Retry  
from urllib3.util.timeout import Timeout

def init_sentry(config):  

  logging.getLogger("urllib3").setLevel(logging.WARNING)   
  sentry_dsn = config.SentryDsn       # http:.....my.app.com/1234'
  
  # All of this is already happening by default!  
  sentry_logging = LoggingIntegration(  
      level=logging.INFO,             # Capture this and above as breadcrumbs  
      event_level=logging.WARNING     # Send as events  
  )  
  
  sentry_sdk.init(  
      dsn=sentry_dsn,  
      integrations=[sentry_logging],  
      qmaxsize=4, #default -1, infinite 
      timeout=Timeout(connect=2.0, read=2.0), #default None, no timeout 
      retries=Retry(total=3, connect=1, read=1, redirect=1), #default False, no retry
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
