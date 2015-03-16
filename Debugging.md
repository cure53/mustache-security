# Introduction #

It's not always easy to debug a website using JavaScript MVC and client side templating engines. Especially when you don't have access to the uncompressed and editable sources. And, what a coincidence, that usually tends to happen during black-box penetration tests.

The following paragraphs will describe a bunch of small and trivial yet maybe helpful tricks to make pentesting and "remote debugging" JS MVC apps a bit easier - and maybe help spotting the one of the other XSS bug, that would otherwise have left unseen.

## Overwrite, break, call stack ##

```
 #1 window.alert = function(a){
 #2
B#3     return confirm(a)
 #4 }
```

## Load JavaScript from raw.github.com ##


Won't work - annoying:

```
<script 
   onerror="alert('Nay, won\'t load')" 
   src="https://raw.github.com/creationix/haml-js/master/lib/haml.js" 
   type="text/javascript"
></script>
```

Using view-source. Works perfectly fine:

```
<script 
    onload="alert('Yay - here we are!')" 
    src="view-source:https://raw.github.com/creationix/haml-js/master/lib/haml.js" 
    type="text/javascript"
></script>
```

And also this works - thanks [@mathias](https://twitter.com/mathias) (be aware that this domain is maintained by some dude out there and might or might not work as you hope):

```
<script 
    onload="alert('Yay - here we are!')" 
    src="https://rawgithub.com/creationix/haml-js/master/lib/haml.js" 
    type="text/javascript"
></script>
```

## Where is my Code? ##

Pentesting a JS MVC application often involves dealing with compressed, minimized and otherwise obfuscated JavaScript. There is no way to find out what happens where, where to set the right breakpoints, how the call stack looks and why the darn thing throws errors instead of showing an alert.

First of all, one wants to find out, what libraries are being used. Checking the website source doesn't always emit that information - especially if all the JavaScript files are served from one single compressed ball-of-script. The first step is then to inspect the DOM, look for global variables that tell about the frameworks and plugins in use. Once that is known, it's recommended to grab an uncompressed version of that exact script. Ideally the same minor version that is hopefully still around on some CDN or GitHub repo.

### Chrome Developer Tools ###

Over several months of working with these frameworks and websites, Chrome developer tools turned out to be the most handy and powerful helpers. Most importantly testers can beautify the JavaScript by using the tiny "{}" button in the sources-tab.

### JSBeautifier ###

Sometimes even Chrome's JavaScript beautifier cannot help and struggles with compressed, one-line JavaScript. Bummer. Luckily, the website http://jsbeautifier.com can help. So far, even the crunchiest and most wickedly compressed HTML and JavaScript could be turned into a nice looking, neatly indented representation.

### Closure Compiler ###

TDB