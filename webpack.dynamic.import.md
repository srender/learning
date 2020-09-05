# webpack import()
最近在学习webpack相关的，本文将展示import()，在webpack生成代码如何做转换
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