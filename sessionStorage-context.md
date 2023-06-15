# SessionStorage

## What's sessionStorage

The read-only sessionStorage property accesses a session Storage object for the current origin.


## Difference with localStorage

The difference is that while data in localStorage doesn't expire, data in sessionStorage is cleared when the page session ends.


# Q&A

## Does data stored in same sessionStorage object between in HTTP and HTTPS?

No. Data stored in sessionStorage is specific to the protocol of the page. In particular, data stored by a script on a site accessed with HTTP (e.g., http://example.com) is put in a different sessionStorage object from the same site accessed with HTTPS (e.g., https://example.com).

## Could copy a sessionStorage object into a new tab via JavaScript?

Generally, it can't.

BUT, we find it in MDN that opening a page in a new tab or window creates a new session with the value of the top-level browsing context, which differs from how session cookies work.


For example,
```js
sessionStorage.setItem('test', 'a');
window.open('http://localhost:3000/1');
```

```js
sessionStorage.getItem('test','a'); // get a
```


## How to disabled?

Add `noopener` for `window.open` or `<a>`

The noopener keyword for the rel attribute of the <a>, <area>, and <form> elements instructs the browser to navigate to the target resource without granting the new browsing context access to the document that opened it.



# Refer

[session-storage-demo](https://github.com/mortalYoung/session-storage-demo)