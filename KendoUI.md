

# Introduction #

"Telerik's Kendo UI is everything professional developers need to build HTML5 sites and mobile apps. Today, productivity of an average HTML/jQuery developer is hampered by assembling a Frankenstein framework of disparate JavaScript libraries
and plug-ins.

Kendo UI has it all: rich jQuery-based widgets, a simple and consistent programming interface, a rock-solid DataSource, validation, internationalization, a MVVM framework, themes, templates and the list goes on.

You don't have to take our word for it, or that of our customers. Go ahead and check it out for yourself: get your Kendo UI free trial now! (4 free products to choose from!)"

From http://www.kendoui.com/

# Quick Facts #

  * [Kendo UI Complete v2013.2.918 (Minimized)](http://cdn.kendostatic.com/2013.2.918/js/kendo.all.min.js)
  * Maintained by Telerik
  * Usage Stats http://trends.builtwith.com/framework/Kendo-UI

  * **{}SEC-A** <font color='red'><b>FAIL</b></font> Template expressions are equivalent to `eval`
  * **{}SEC-B** <font color='red'><b>FAIL</b></font> No isolated execution scope seems to exist
  * **{}SEC-C** <font color='red'><b>FAIL</b></font> JavaScript can be executed from any element fetched by the Kendo UI application logic
  * **{}SEC-D** <font color='red'><b>FAIL</b></font> No enforced separation
  * **{}SEC-E** <font color='red'><b>FAIL</b></font> No security@kendoui.com address seems to be in place
  * **{}SEC-F** <font color='red'><b>FAIL</b></font> Kendo UI uses Function, CSP will have a hard time

# Injection Attacks #

Kendo UI doesn't really claim to provide security in case of an injection event - but rather goes the way of claiming flexibility and the "power of JavaScript". This might lead to surprises when an attacker can control content of arbitrary elements applied with an ID that is later being used by Kendo UI.

```
<script src="http://code.jquery.com/jquery-1.7.1.min.js"></script>
<script src="http://cdn.kendostatic.com/2012.3.1114/js/kendo.all.min.js"></script>
<div id="x"># alert(1) #</div>
<script>
  var template = kendo.template($("#x").html());
  var tasks = [{ id: 1}];
  var dataSource = new kendo.data.DataSource({ data: tasks });
  dataSource.bind("change", function(e) { 
    var html = kendo.render(template, this.view());
  });
  dataSource.read();
</script>
```

## Eval via Function ##

Not unlike many other frameworks, Kendo makes use of the oh-so-comfy Function constructor.

```

    Template = {paramName: "data",useWithBlock: !0,render: function(e, t) {
            var n, i, r = "";
            for (n = 0, i = t.length; i > n; n++)
                r += e(t[n]);
            return r
        },compile: function(e, t) {
            var n, i, r = extend({}, this, t), o = r.paramName, a = o.match(argumentNameRegExp)[0], s = r.useWithBlock, l = "var o,e=kendo.htmlEncode;";
            if (isFunction(e))
                return 2 === e.length ? function(t) {
                    return e($, {data: t}).join("")
                } : e;
            for (l += s ? "with(" + o + "){" : "", l += "o=", n = e.replace(escapedCurlyRegExp, "__CURLY__").replace(encodeRegExp, "#=e($1)#").replace(curlyRegExp, "}").replace(escapedSharpRegExp, "__SHARP__").split("#"), i = 0; n.length > i; i++)
                l += compilePart(n[i], 0 === i % 2);
            l += s ? ";}" : ";", l += "return o;", l = l.replace(sharpRegExp, "#");
            try {
                return Function(a, l)
            } catch (d) {
                throw Error(kendo.format("Invalid template:'{0}' Generated code:'{1}'", e, l))
            }
        }}
```