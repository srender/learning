# webpack import()
最近在学习webpack相关的，从webpack生成的代码如何实现import动态流程。
本文将展示import()，在webpack生成代码如何做转换。

先来看下生成的webpack源码
类web的浏览器环境。
__webpack_require__.e 通过 chunkId 引入chunk。
```javascript
//sourceMap文件中
__webpack_require__.e(/*! import() */ 0)
        .then(
            __webpack_require__.t.bind(
                null,
                /*! ./dynamicImport.js */ "./dynamicImport.js",
                7
            )
        )
```
来看下__webpack_require__.e 的具体实现。
```javascript
// This file contains only the entry chunk.
// The chunk loading function for additional chunks
__webpack_require__.e = function requireEnsure(chunkId) {
    var promises = [];
    // JSONP chunk loading for javascript
    var installedChunkData = installedChunks[chunkId];
    if(installedChunkData !== 0) { // 0 means "already installed".
        // a Promise means "currently loading".
        if(installedChunkData) {
            promises.push(installedChunkData[2]);
        }else{
            // setup Promise in chunk cache
            var promise = new Promise(function(resolve, reject) {
                installedChunkData = installedChunks[chunkId] = [resolve, reject];
            });
            promises.push(installedChunkData[2] = promise);
            // start chunk loading
            var script = document.createElement('script');
            var onScriptComplete;
            script.charset = 'utf-8';
            script.timeout = 120;
            if (__webpack_require__.nc) {
                script.setAttribute("nonce", __webpack_require__.nc);
            }
            script.src = jsonpScriptSrc(chunkId);
            // create error before stack unwound to get useful stacktrace later
            var error = new Error();
            onScriptComplete = function (event) {
                // avoid mem leaks in IE.
                script.onerror = script.onload = null;
                clearTimeout(timeout);
                var chunk = installedChunks[chunkId];
                if(chunk !== 0) {
                    var errorType = event && (event.type === 'load' ? 'missing' : event.type);
                    var realSrc = event && event.target && event.target.src;
                    error.message = 'Loading chunk ' + chunkId + ' failed.\n(' + errorType + ': ' + realSrc + ')';
                    error.name = 'ChunkLoadError';
                    error.type = errorType;
                    error.request = realSrc;
                    chunk[1](error);
                }
                installedChunks[chunkId] = undefined;
            }

            var timeout = setTimeout(function(){
                onScriptComplete({ type: 'timeout', target: script });
            }, 120000);
            script.onerror = script.onload = onScriptComplete;
            document.head.appendChild(script);
        }
        return Promise.all(promises);
}
```


```javascript
//webpack/lib/APIPlugin.js
const REPLACEMENTS = {
    __webpack_require__: "__webpack_require__",
	__webpack_public_path__: "__webpack_require__.p",
	__webpack_modules__: "__webpack_require__.m",
	__webpack_chunk_load__: "__webpack_require__.e",
	__non_webpack_require__: "require",
}
```


```javascript
//webpack/MainTemplate.js
// require function shortcuts:
// __webpack_require__.s = the module id of the entry point
// __webpack_require__.c = the module cache
// __webpack_require__.m = the module functions
// __webpack_require__.p = the bundle public path
// __webpack_require__.i = the identity function used for harmony imports
// __webpack_require__.e = the chunk ensure function
// __webpack_require__.d = the exported property define getter function
// __webpack_require__.o = Object.prototype.hasOwnProperty.call
// __webpack_require__.r = define compatibility on export
// __webpack_require__.t = create a fake namespace object
// __webpack_require__.n = compatibility get default export
// __webpack_require__.h = the webpack hash
// __webpack_require__.w = an object containing all installed WebAssembly.Instance export objects keyed by module id
// __webpack_require__.oe = the uncaught error handler for the webpack runtime
// __webpack_require__.nc = the script nonce
```