**This place will host a collection of security tips and tricks for JavaScript MVC frameworks and templating libraries.**

Our focus will on shedding light on the numerous novel ways to abuse common MVC frameworks to execute arbitrary JavaScript in unexpected situations. We further aim to be able to find a metric for the security of JS MVC frameworks and allow penetration testers as well as developers to save time on attacking and hardening JS MVC-based applications and apps.

A nice set of slides has been created too, you can find it here: [JSMVCOMFG - To sternly look at JavaScript MVC and Templating Frameworks](http://www.slideshare.net/x00mario/jsmvcomfg-to-sternly-look-at-javascript-mvc-and-templating-frameworks)

Currently, the following qualifiers are used to estimate a framework's security level:

  * **{}SEC-A** Are template expressions executed without using `eval` or `Function`? (yes = pass)
  * **{}SEC-B** Is the the execution scope well isolated or sand-boxed? (yes = pass)
  * **{}SEC-C** Can only script elements serve as template containers? (yes = pass)
  * **{}SEC-D** Does the framework allow, encourage or even enforce separation of code and content? (yes = pass)
  * **{}SEC-E** Does the framework maintainer have a security response program? (yes = pass)
  * **{}SEC-F** Does the Framework allow or encourage safe CSP rules to be used (yes = pass)

The project is in the earliest of possible alpha stages - don't expect anything useful before late 2013, early 2014 - a lot of research-in-progress.

<font color='red'><b>Note:</b></font> **We try to maintain this project as good as we can in our spare time. We might (and will) make mistakes - if you spot one let us know please! We'll fix it then. Projects like this cannot live without active participation - don't be a grump, tell us what we did wrong if you feel we did.**

## Now show me some bugs! ##

Alright! Here's what we found so far. Some bug reports were already sent, some are still pending - others will probably never be sent successfully as **{}SEC-E** wasn't considered to be useful. We're working on it!


  * [VueJS](https://code.google.com/p/mustache-security/wiki/VueJS)
  * [AngularJS](https://code.google.com/p/mustache-security/wiki/AngularJS)
  * [CanJS](https://code.google.com/p/mustache-security/wiki/CanJS)
  * [Underscore.js](https://code.google.com/p/mustache-security/wiki/UnderscoreJS)
  * [KnockoutJS](https://code.google.com/p/mustache-security/wiki/KnockoutJS)
  * [Ember.js](https://code.google.com/p/mustache-security/wiki/EmberJS)
  * [Polymer](https://code.google.com/p/mustache-security/wiki/Polymer)
  * [Ractive.js](https://code.google.com/p/mustache-security/wiki/RactiveJS)
  * [jQuery](https://code.google.com/p/mustache-security/wiki/jQuery)
  * [JsRender](https://code.google.com/p/mustache-security/wiki/JsRender)
  * [Kendo UI](https://code.google.com/p/mustache-security/wiki/KendoUI)

## Security Matrix ##

Spot a mistake? Let us know! We go for fail if unclear - rather too harsh than too lax.

| **Framework** | **{}SEC-A** | **{}SEC-B** | **{}SEC-C** | **{}SEC-D** | **{}SEC-E** | **{}SEC-F** |
|:--------------|:------------|:------------|:------------|:------------|:------------|:------------|
| [VueJS](https://code.google.com/p/mustache-security/wiki/VueJS) | <font color='red'><b>Fail</b></font> | <font color='red'><b>Fail</b></font> | <font color='red'><b>Fail</b></font> | <font color='red'><b>Fail</b></font> | <font color='red'><b>Fail</b></font> | <font color='red'><b>Fail</b></font> |
| [AngularJS 1.0.8](https://code.google.com/p/mustache-security/wiki/AngularJS) | <font color='orange' title='RC-versions have this covered and fixed'><b>Fail</b></font> | <font color='red'><b>Fail</b></font> | <font color='red'><b>Fail</b></font> | <font color='red'><b>Fail</b></font> | <font color='green'><b>PASS</b></font> | <font color='orange' title="CSP-mode in AngularJS re-enables injection attacks despite CSP. Thus it's a FAIL, not really making apps more secure as supposed to."><b>Fail</b></font> |
| [AngularJS 1.2.0](https://code.google.com/p/mustache-security/wiki/AngularJS) | <font color='orange' title='Some template directives still allow for arbitrary JavaScript execution'><b>Fail</b></font> | <font color='green'><b>PASS</b></font> | <font color='orange' title='Any element with the ng-app attribute can execute expressions. Injections might cause damage.'><b>Fail</b></font> | <font color='red'><b>Fail</b></font> | <font color='green'><b>PASS</b></font> | <font color='green'><b>PASS</b></font> |
| [CanJS](https://code.google.com/p/mustache-security/wiki/CanJS) | <font color='red'><b>Fail</b></font> | <font color='red'><b>Fail</b></font> | <font color='green'><b>PASS</b></font> | <font color='red'><b>Fail</b></font> | <font color='red'><b>Fail</b></font> | <font color='red'><b>Fail</b></font> |
| [Underscore.js](https://code.google.com/p/mustache-security/wiki/UnderscoreJS) | <font color='red'><b>Fail</b></font> | <font color='red'><b>Fail</b></font> | <font color='green'><b>PASS</b></font> | <font color='red'><b>Fail</b></font> | <font color='red'><b>Fail</b></font> | <font color='red'><b>Fail</b></font> |
| [KnockoutJS](https://code.google.com/p/mustache-security/wiki/KnockoutJS) | <font color='red'><b>Fail</b></font> | <font color='red'><b>Fail</b></font> | <font color='red'><b>Fail</b></font> | <font color='red'><b>Fail</b></font> | <font color='red'><b>Fail</b></font> | <font color='red'><b>Fail</b></font> |
| [Ember.js](https://code.google.com/p/mustache-security/wiki/EmberJS) | <font color='red'><b>Fail</b></font> | <font color='green'><b>PASS</b></font> | <font color='green'><b>PASS</b></font> | <font color='red'><b>Fail</b></font> | <font color='green'><b>PASS</b></font> | <font color='grey'><b>TBD</b></font> |
| [Polymer](https://code.google.com/p/mustache-security/wiki/Polymer) | <font color='grey'><b>TBD</b></font>| <font color='grey'><b>TBD</b></font>| <font color='grey'><b>TBD</b></font>| <font color='grey'><b>TBD</b></font>| <font color='grey'><b>TBD</b></font>| <font color='grey'><b>TBD</b></font> |
| [Ractive.js](https://code.google.com/p/mustache-security/wiki/RactiveJS) | <font color='red'><b>Fail</b></font>| <font color='red'><b>Fail</b></font>| <font color='red'><b>Fail</b></font>| <font color='red'><b>Fail</b></font>| <font color='red'><b>Fail</b></font>| <font color='red'><b>Fail</b></font> |
| [jQuery](https://code.google.com/p/mustache-security/wiki/jQuery) | <font color='grey'><b>TBD</b></font>| <font color='grey'><b>TBD</b></font>| <font color='grey'><b>TBD</b></font>| <font color='grey'><b>TBD</b></font>| <font color='green'><b>PASS</b></font>| <font color='grey'><b>TBD</b></font> |
| [JsRender](https://code.google.com/p/mustache-security/wiki/JsRender) | <font color='red'><b>Fail</b></font> | <font color='red'><b>Fail</b></font> | <font color='red'><b>Fail</b></font> | <font color='red'><b>Fail</b></font> | <font color='red'><b>Fail</b></font> | <font color='red'><b>Fail</b></font> |
| [Kendo UI](https://code.google.com/p/mustache-security/wiki/KendoUI) | <font color='red'><b>Fail</b></font> | <font color='red'><b>Fail</b></font> | <font color='red'><b>Fail</b></font> | <font color='red'><b>Fail</b></font> | <font color='red'><b>Fail</b></font> | <font color='red'><b>Fail</b></font> |

## Useful Things ##

  * [Debugging JavaScript MVC](https://code.google.com/p/mustache-security/wiki/Debugging)
  * [Useful Resources](https://code.google.com/p/mustache-security/wiki/Resources)