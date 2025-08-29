# LakeFS × Excel (xlwings Lite) on localhost: how we beat the CORS wall 🚧

Local workbooks that call LakeFS over HTTP trip four modern browser defences at once:

* classic **CORS** pre-flight (`OPTIONS …`),
* the new **Private Network Access (PNA)** header set,
* LakeFS’ lack of an `OPTIONS /api/*` handler,
* and the browser WebView inside Excel that refuses to fall back.

Below is the full post-mortem and every workaround we tried, ending with the ultra-small **NGINX side-car proxy** that finally unblocked Excel.
