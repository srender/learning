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
当前环境目标未类浏览器环境。
整体流程， installedChunks[chunkId] 判断模块状态。如未加载，执行下载流程。在web环境下
采用创建script标签，jsonp方式去拉取。

在继续看下__webpack_require__.e相关引用。
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

上面的生成代码__webpack_require__.e 代码生成之处。
首先查看编译环节类web环境如何定义。如果是web环境，就引入JsonpTemplatePlugin。
```javascript
class WebpackOptionsApply extends OptionsApply {
   process(options, compiler) {
       if (typeof options.target === "string") {
           let JsonpTemplatePlugin;
			let FetchCompileWasmTemplatePlugin;
			let ReadFileCompileWasmTemplatePlugin;
			let NodeSourcePlugin;
			let NodeTargetPlugin;
			let NodeTemplatePlugin;

			switch (options.target) {
				case "web":
					JsonpTemplatePlugin = require("./web/JsonpTemplatePlugin");
					FetchCompileWasmTemplatePlugin = require("./web/FetchCompileWasmTemplatePlugin");
					NodeSourcePlugin = require("./node/NodeSourcePlugin");
					new JsonpTemplatePlugin().apply(compiler);
					new FetchCompileWasmTemplatePlugin({
						mangleImports: options.optimization.mangleWasmImports
					}).apply(compiler);
					new FunctionModulePlugin().apply(compiler);
					new NodeSourcePlugin(options.node).apply(compiler);
					new LoaderTargetPlugin(options.target).apply(compiler);
                    break;
            .........
       }
   } 
}
```
```javascript
//lib/Compilation.js
const MainTemplate = require("./MainTemplate");
class Compilation extends Tapable {
    this.outputOptions = options && options.output;
    this.mainTemplate = new MainTemplate(this.outputOptions);
}
```
看下JsonpTemplatePlugin的定义，引入了以下三个模块注册相应的hooks。
JsonpMainTemplatePlugin，JsonpChunkTemplatePlugin，JsonpHotUpdateChunkTemplatePlugin
```javascript
"use strict";
// ./web/JsonpTemplatePlugin
const JsonpMainTemplatePlugin = require("./JsonpMainTemplatePlugin");
const JsonpChunkTemplatePlugin = require("./JsonpChunkTemplatePlugin");
const JsonpHotUpdateChunkTemplatePlugin = require("./JsonpHotUpdateChunkTemplatePlugin");

class JsonpTemplatePlugin {
	apply(compiler) {
		compiler.hooks.thisCompilation.tap("JsonpTemplatePlugin", compilation => {
			new JsonpMainTemplatePlugin().apply(compilation.mainTemplate);
			new JsonpChunkTemplatePlugin().apply(compilation.chunkTemplate);
			new JsonpHotUpdateChunkTemplatePlugin().apply(
				compilation.hotUpdateChunkTemplate
			);
		});
	}
}

module.exports = JsonpTemplatePlugin;
```
与该段代码生成的是
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
执行 this.hooks.requireExtension 内部调用hooks.requireEnsure。相关代码写入过程
```javascript
//webpack/MainTemplate.js
module.exports = class MainTemplate extends Tapable {
    constructor(outputOptions) {
        this.hooks.requireExtensions.tap("MainTemplate", (source, chunk, hash) => {
            const buf = [];
			const chunkMaps = chunk.getChunkMaps(); 
            // Check if there are non initial chunks which need to be imported using require-ensure
            if (Object.keys(chunkMaps.hash).length) {
                buf.push("// This file contains only the entry chunk.");
                buf.push("// The chunk loading function for additional chunks");
				buf.push(`${this.requireFn}.e = function requireEnsure(chunkId) {`);
				buf.push(Template.indent("var promises = [];"));
				buf.push(
					Template.indent(
						this.hooks.requireEnsure.call("", chunk, hash, "chunkId")
					)
				);
				buf.push(Template.indent("return Promise.all(promises);"));
				buf.push("};");
            }
            ....
        }

        this.requireFn = "__webpack_require__";
    }   
}
```
chunk相关逻辑，主要用于判断当前chunk是否下载。
jsonpScript 通过jsonp下载chunk代码
```javascript
//webpack/lib/web/JsonpMainTemplatePlugin.js
class JsonpMainTemplatePlugin {
    mainTemplate.hooks.requireEnsure.tap(
        "JsonpMainTemplatePlugin load",
        (source, chunk, hash) => {
            return Template.asString([
                    source,
                    ......
                    mainTemplate.hooks.jsonpScript.call("", chunk, hash）
                ])
        }
    )

    mainTemplate.hooks.jsonpScript.tap(
			"JsonpMainTemplatePlugin",
			(_, chunk, hash) => {
                ...
                //下载script的相关逻辑
            })

}
```