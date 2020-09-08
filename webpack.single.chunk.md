# webpack single chunk
webpack单文件引入
```javascript
//index.js
import "./cast.js"
```
```javascript
//cast.js
console.log("js")
```
我们先来看下生成的代码的整体结构。
此处为自执行函数，传入参数为modules。即cast.js 和 index.js.
上述两份js代码生成key value形式，value为执行函数
```javascript
//build 整体结构。
(function(modules) { // webpackBootstrap

})({
    "./cast.js": (function(module, exports) { console.log("js") }),
    "./index.js":(function(module, __webpack_exports__, __webpack_require__) {
        "use strict";
        __webpack_require__.r(__webpack_exports__);
        var _cast_js__WEBPACK_IMPORTED_MODULE_0__ = __webpack_require__(/*! ./cast.js */ "./cast.js");
        var _cast_js__WEBPACK_IMPORTED_MODULE_0___default = /*#__PURE__*/__webpack_require__.n(_cast_js__WEBPACK_IMPORTED_MODULE_0__);
    })
})
```
详细看下webpackBootstrap里面的函数定义。
定义了相关暴露接口,模块暴露m。模块缓存c。模块定义d等。
下面解读t相关的
```javascript
(function(modules) { // webpackBootstrap」
    // The module cache
    var installedModules = {};

    // The require function
    function __webpack_require__(moduleId) {}

    // expose the modules object (__webpack_modules__)
    __webpack_require__.m = modules;

    // expose the module cache
    __webpack_require__.c = installedModules;

    // define getter function for harmony exports
    __webpack_require__.d = function(exports, name, getter) {}

    // define __esModule on exports
    __webpack_require__.r = function(exports) {}

    // create a fake namespace object
    // mode & 1: value is a module id, require it
    // mode & 2: merge all properties of value into the ns
    // mode & 4: return value when already ns object
    // mode & 8|1: behave like require
    __webpack_require__.t = function(value, mode) {}

    // getDefaultExport function for compatibility with non-harmony modules
    __webpack_require__.n = function(module) {}

    // Object.prototype.hasOwnProperty.call
    __webpack_require__.o = function(object, property) {}

    // __webpack_public_path__
    __webpack_require__.p = "";

    // Load entry module and return exports
    return __webpack_require__(__webpack_require__.s = "./index.js");
})
```

