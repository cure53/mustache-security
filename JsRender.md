

# Introduction #

"JsRender is a light-weight but powerful templating engine, highly extensible, and optimized for high-performance pure string-based rendering, without DOM or jQuery dependency.

JSRender and JsViews together provide the next-generation implementation of both JQuery Templates, and JQuery Data Link - and supersede those libraries."

From https://github.com/BorisMoore/jsrender

# Quick Facts #

  * [JsRender 1.0.0-beta (Uncompressed)](https://raw.github.com/BorisMoore/jsrender/master/jsrender.js)

  * **{}SEC-A** <font color='red'><b>FAIL</b></font> Template expressions for data paths/conditions are equivalent to `eval`
  * **{}SEC-B** <font color='red'><b>FAIL</b></font> No sandbox or isolated execution scope
  * **{}SEC-C** <font color='red'><b>FAIL</b></font> Any HTML element can server as template container
  * **{}SEC-D** <font color='red'><b>FAIL</b></font> No enforced separation, no obvious way to outsource templates to static files
  * **{}SEC-E** <font color='red'><b>FAIL</b></font> No dedicated security response program, no security@jsviews.com address

# Injection Attacks #

JsRender doesn't allow direct JavaScript execution via template expressions - but as soon as conditions or data path queries are being used, the curse of the constructor strikes again. Almost nothing new here, moving on...

```
<script src="http://code.jquery.com/jquery.js" type="text/javascript"></script>
<script src="https://rawgithub.com/BorisMoore/jsrender/master/jsrender.js" type="text/javascript"></script>
<div id="tmpl" type="text/x-jsrender">
    {{if constructor.constructor('alert(1)')() /}}
    {{:constructor.constructor('alert(2)')()}}
</div>
<script>
  $("#tmpl").render([{}])
</script>
```

# Eval via Function #

JsRender ships a massive method called `buildCode()` that effectively receives an AST node, walks over it and concatenates a string that will later be fed to `Function`.

```
    function buildCode(ast, tmpl, isLinkExpr) {
        // Build the template function code from the AST nodes, and set as property on the passed-in template object
        // Used for compiling templates, and also by JsViews to build functions for data link expressions
        var i, node, tagName, converter, params, hash, hasTag, hasEncoder, getsVal, hasCnvt, useCnvt, tmplBindings, pathBindings, 
        nestedTmpls, tmplName, nestedTmpl, tagAndElses, content, markup, nextIsElse, oldCode, isElse, isGetVal, prm, tagCtxFn, 
        tmplBindingKey = 0, 
        code = "", 
        noError = "", 
        tmplOptions = {}, 
        l = ast.length;
        
        if ("" + tmpl === tmpl) {
            tmplName = isLinkExpr ? 'data-link="' + tmpl.replace(rNewLine, " ").slice(1, -1) + '"' : tmpl;
            tmpl = 0;
        } else {
            tmplName = tmpl.tmplName || "unnamed";
            if (tmpl.allowCode) {
                tmplOptions.allowCode = true;
            }
            if (tmpl.debug) {
                tmplOptions.debug = true;
            }
            tmplBindings = tmpl.bnds;
            nestedTmpls = tmpl.tmpls;
        }
        for (i = 0; i < l; i++) {
            // AST nodes: [ tagName, converter, params, content, hash, noError, pathBindings, contentMarkup, link ]
            node = ast[i];

            // Add newline for each callout to t() c() etc. and each markup string
            if ("" + node === node) {
                // a markup string to be inserted
                code += '\nret+="' + node + '";';
            } else {
                // a compiled tag expression to be inserted
                tagName = node[0];
                if (tagName === "*") {
                    // Code tag: {{* }}
                    code += "" + node[1];
                } else {
                    converter = node[1];
                    params = node[2];
                    content = node[3];
                    hash = node[4];
                    noError = node[5];
                    markup = node[7];
                    
                    if (!(isElse = tagName === "else")) {
                        tmplBindingKey = 0;
                        if (tmplBindings && (pathBindings = node[6])) { // Array of paths, or false if not data-bound
                            tmplBindingKey = tmplBindings.push(pathBindings);
                        }
                    }
                    if (isGetVal = tagName === ":") {
                        if (converter) {
                            tagName = converter === "html" ? ">" : converter + tagName;
                        }
                        if (noError) {
                            // If the tag includes noerror=true, we will do a try catch around expressions for named or unnamed parameters
                            // passed to the tag, and return the empty string for each expression if it throws during evaluation
                            //TODO This does not work for general case - supporting noError on multiple expressions, e.g. tag args and properties.
                            //Consider replacing with try<a.b.c(p,q) + a.d, xxx> and return the value of the expression a.b.c(p,q) + a.d, or, if it throws, return xxx||'' (rather than always the empty string)
                            prm = "prm" + i;
                            noError = "try{var " + prm + "=[" + params + "][0];}catch(e){" + prm + '="";}\n';
                            params = prm;
                        }
                    } else {
                        if (content) {
                            // Create template object for nested template
                            nestedTmpl = TmplObject(markup, tmplOptions);
                            nestedTmpl.tmplName = tmplName + "/" + tagName;
                            // Compile to AST and then to compiled function
                            buildCode(content, nestedTmpl);
                            nestedTmpls.push(nestedTmpl);
                        }
                        
                        if (!isElse) {
                            // This is not an else tag.
                            tagAndElses = tagName;
                            // Switch to a new code string for this bound tag (and its elses, if it has any) - for returning the tagCtxs array
                            oldCode = code;
                            code = "";
                        }
                        nextIsElse = ast[i + 1];
                        nextIsElse = nextIsElse && nextIsElse[0] === "else";
                    }
                    
                    hash += ",args:[" + params + "]}";
                    
                    if (isGetVal && pathBindings || converter && tagName !== ">") {
                        // For convertVal we need a compiled function to return the new tagCtx(s)
                        tagCtxFn = new Function("data,view,j,u", " // " 
                        + tmplName + " " + tmplBindingKey + " " + tagName + "\n" + noError + "return {" + hash + ";");
                        tagCtxFn.paths = pathBindings;
                        tagCtxFn._ctxs = tagName;
                        if (isLinkExpr) {
                            return tagCtxFn;
                        }
                        useCnvt = 1;
                    }
                    
                    code += (isGetVal 
                    ? "\n" + (pathBindings ? "" : noError) + (isLinkExpr ? "return " : "ret+=") + (useCnvt  // Call _cnvt if there is a converter: {{cnvt: ... }} or {^{cnvt: ... }}
                    ? (useCnvt = 0, hasCnvt = true, 'c("' + converter + '",view,' + (pathBindings 
                    ? ((tmplBindings[tmplBindingKey - 1] = tagCtxFn), tmplBindingKey)  // Store the compiled tagCtxFn in tmpl.bnds, and pass the key to convertVal()
                    : "{" + hash) + ");") 
                    : tagName === ">" 
                    ? (hasEncoder = true, "h(" + params + ");") 
                    : (getsVal = true, "(v=" + params + ")!=" + (isLinkExpr ? "=" : "") + 'u?v:"";') // Strict equality just for data-link="title{:expr}" so expr=null will remove title attribute 
                    ) 
                    : (hasTag = true, "{tmpl:"  // Add this tagCtx to the compiled code for the tagCtxs to be passed to renderTag()
                    + (content ? nestedTmpls.length : "0") + ","  // For block tags, pass in the key (nestedTmpls.length) to the nested content template
                    + hash + ","));
                    
                    if (tagAndElses && !nextIsElse) {
                        code = "[" + code.slice(0, -1) + "]"; // This is a data-link expression or the last {{else}} of an inline bound tag. We complete the code for returning the tagCtxs array
                        if (isLinkExpr || pathBindings) {
                            // This is a bound tag (data-link expression or inline bound tag {^{tag ...}}) so we store a compiled tagCtxs function in tmp.bnds
                            code = new Function("data,view,j,u", " // " + tmplName + " " + tmplBindingKey + " " + tagAndElses + "\nreturn " + code + ";");
                            if (pathBindings) {
                                (tmplBindings[tmplBindingKey - 1] = code).paths = pathBindings;
                            }
                            code._ctxs = tagName;
                            if (isLinkExpr) {
                                return code; // For a data-link expression we return the compiled tagCtxs function
                            }
                        }

                        // This is the last {{else}} for an inline tag.
                        // For a bound tag, pass the tagCtxs fn lookup key to renderTag.
                        // For an unbound tag, include the code directly for evaluating tagCtxs array
                        code = oldCode + '\nret+=t("' + tagAndElses + '",view,this,' + (tmplBindingKey || code) + ");";
                        pathBindings = 0;
                        tagAndElses = 0;
                    }
                }
            }
        }
        // Include only the var references that are needed in the code
        code = "// " + tmplName 
        + "\nvar j=j||" + (jQuery ? "jQuery." : "js") + "views" 
        + (getsVal ? ",v" : "")  // gets value
        + (hasTag ? ",t=j._tag" : "")  // has tag
        + (hasCnvt ? ",c=j._cnvt" : "")  // converter
        + (hasEncoder ? ",h=j.converters.html" : "")  // html converter
        + (isLinkExpr ? ";\n" : ',ret="";\n') 
        + ($viewsSettings.tryCatch ? "try{\n" : "") 
        + (tmplOptions.debug ? "debugger;" : "") 
        + code + (isLinkExpr ? "\n" : "\nreturn ret;\n") 
        + ($viewsSettings.tryCatch ? "\n}catch(e){return j._err(e);}" : "");
        try {
            code = new Function("data,view,j,u", code);
        } catch (e) {
            syntaxError("Compiled template code:\n\n" + code, e);
        }
        if (tmpl) {
            tmpl.fn = code;
        }
        return code;
    }
```