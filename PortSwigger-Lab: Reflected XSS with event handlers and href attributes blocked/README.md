[Lab Link.](https://portswigger.net/web-security/cross-site-scripting/contexts/lab-event-handlers-and-href-attributes-blocked)

The goal of this lab is to inject an XSS payload but with most of HTML tags, all events event handlers and href attribute are blocked.

First we should determine which tags can be used, using Turbo intruder with a list of all tags from XSS cheat sheet of portswigger, the results are:

![Image](https://github.com/user-attachments/assets/06f20f2a-aca9-4480-9109-9262e0de7e32)

Looking at the available tags, we can conclude that we need to use SVG features.

The `<a>` SVG element has the `href` attribute, but it is blocked by the WAF, what we can do is to use the `<animate>` element, which can modify the attributes of an element like `<a>`. The payload would be:

```html
<svg width="300" height="100">
  <a>
    <animate attributeName="href" values="javascript:alert();"></animate>
    <text x="50" y="90" text-anchor="middle">CLICK ME</text>
  </a>
</svg>
```

The `<a>` wont have the `href` attribute when making the initial request, so the WAF won’t block it, but when the code gets executed it will be a valid XSS.