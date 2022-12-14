# warc-embed 🏛️
Experimental proxy and wrapper boilerplate for safely and efficiently embedding Web Archives (`.warc.gz`, `.wacz`) into web pages. 

This implementation:
- Wraps [Webrecorder's `<replay-web-page>`](https://replayweb.page/docs/embedding) client-side playback technology.
- Serves, proxies and [caches](https://www.nginx.com/blog/smart-efficient-byte-range-caching-nginx/) web archive files using [NGINX](https://www.nginx.com/). 
- Allows for two-way communication between the embedding website and the embedded archive using post messages.

🖼️ [Live Demo](https://warcembed-demo.lil.tools)

📖 See also: [Blog post](https://lil.law.harvard.edu/blog/2022/09/15/opportunities-and-challenges-of-client-side-playback/ "Blog post on lil.law.harvard.edu - Web Archiving: Opportunities and Challenges of Client-Side Playback")

---

## Summary
- [Concept](#concept)
- [Routes](#routes)
- [Deployment](#deployment)
- [Local development](#local-development)
- [Communicating with the embedded archive](#communicating-with-the-embedded-archive)
- [Changelog](/CHANGELOG.md)

---

## Concept

### "It's a wrapper"
`warc-embed` serves an HTML document containing a pre-configured instance of [`<replay-web-page>`](https://replayweb.page/), [webrecorder's front-end archive playback system](https://webrecorder.net/), pointing at a proxied version of the requested archive. 

The playback will only start when said document is embedded in a cross-origin `<iframe>` for security reasons _(XSS prevention in the context of an `<iframe>` needing both `allow-script` and `allow-same-origin`)_.  

### "It's a proxy"
`warc-embed` pulls and serves the requested archive file in the format required by `<replay-web-page>` _(right [`Content-Type`](https://developer.mozilla.org/en-US/docs/Web/HTTP/Headers/Content-Type), support for [range requests](https://developer.mozilla.org/en-US/docs/Web/HTTP/Range_requests), CORS resolution)_.  

**The requested archive can be sourced from either:**
- The local [`/archives/` folder](/html/archives/). This is where the server will look first.
- A remote location the server will proxy from, defined in `nginx.conf`.

### Example 
```html
<!-- On https://*.domain.ext: -->
<iframe
  src="https://warcembed.domain.ext/?source=archive.warc.gz&url=https://what-was-archived.ext/path"
  allow="allow-scripts allow-forms allow-same-origin"
>
</iframe>
```

[☝️ Back to summary](#summary)

---

## Routes

### /?source=X&url=Y

#### Role
Serves [an HTML document containing an instance of `<replay-web-page>`](/html/embed/index.html), pointing at a proxied archive file. 

Must be embedded in a cross-origin `<iframe>`, preferably on the same parent domain to avoid thrid-party cookie limitations:
```
warcembed.example.com: Hosts warc-embed
www.example.com: Has iframes pointing at warcembed.example.com
```

#### Methods
`GET`, `HEAD`

#### Query parameters
| Name | Required ? | Description |
| --- | --- | --- |
| `source` | Yes | Path + filename of the `.warc.gz` or `.wacz`. Can contain a path. <br>Must either be present in the [`/archives/` folder](/html/archives/) or on the remote server defined in `nginx.conf`. |
| `url` | No | Url of a page within the archive to display. If not set, will try to open the first page available. | 
| `ts`| No | Timestamp of the page to retrieve. Can be either a YYYYMMDDHHMMSS-formatted string or a millisecond timestamp or a. |
| `embed` | No | `<replay-web-page>`'s [embed mode](https://replayweb.page/docs/embedding). Can be set to `replayonly` to hide its UI. |
| `deepLink` | No | `<replay-web-page>`'s [`deepLink` mode](https://replayweb.page/docs/embedding). |

#### Examples
```html
<!-- On https://*.domain.ext: -->
<iframe
  src="https://warcembed.domain.ext/?source=archive.warc.gz&url=https://what-was-archived.ext/path"
  allow="allow-scripts allow-forms allow-same-origin"
>
</iframe>
```

### /*.[wacz|warc.gz]

### Role
Pulls, caches and serves a given `.warc.gz` or `.wacz` file, with full support for range requests.

Will first look for the path + file given in the local [`/archives/` folder](/html/archives/), and try to proxy it from the remote server defined in `nginx.conf`.

[☝️ Back to summary](#summary)

---

## Deployment
This project consists of a single `Dockerfile` derived from [the official NGINX Docker image](https://hub.docker.com/_/nginx), which can be deployed on any docker-compatible machine. 

### Example
The following example describes the process of deploying `warc-embed` on [fly.io](https://fly.io), a platform-as-a-service provider. 
1. `nginx.conf` needs to be edited. See comments starting with `EDIT:` in the document for instructions.
2. Install the [`flyctl`](https://fly.io/docs/hands-on/install-flyctl/) client and [sign-in](https://fly.io/docs/hands-on/sign-in/), if not already done.
3. Initialize and deploy the project by running the `flyctl launch` command _(use `flyctl deploy` for subsequent deploys)_. 
4. `warc-embed` is now live and visible on the [`fly.io` dashboard](https://fly.io/dashboard). 
5. We highly recommend setting up a **custom domain and SSL certificate**. This can be done directly from the `fly.io` dashboard. Ideally, the target domain should be a subdomain of the website on which `warc-embed` iframes are going to be embedded: for example, `www.domain.ext` embedding an `<iframe>` from `warcembed.domain.ext`.

[☝️ Back to summary](#summary)

---

## Local development

### Example: Running `warc-embed` locally using docker
```bash
docker build . -t warc-embed-local
docker run --rm -p 8080:8080 warc-embed-local
# warc-embed is now accessible on http://localhost:8080
```

[☝️ Back to summary](#summary)

---

## Communicating with the embedded archive

`warc-embed` allows the embedding website to communicate with the embedded archive playback using [post messages](https://developer.mozilla.org/en-US/docs/Web/API/Window/postMessage). 
All messages coming _from_ a `warc-embed` `<iframe>` come with a `warcEmbedHref` property, helping identify the sender.  

### Messages interpreted by the `warc-embed` `<iframe>`
`warc-embed` will look for the following properties in messages coming from the embedding website and react accordingly:

| Property name | Expected value | Description |
| --- | --- | --- |
| `updateUrl` | String | If provided, will replace the current `url` parameter of `<replay-web-page>`. |
| `updateTs` | Number | If provided, will replace the current `ts` parameter of `<replay-web-page>`. |
| `getCollInfo` | Boolean | If provided, will send a post message back with `<replay-web-page>`'s `collInfo` object, containing meta information about the currently-loaded archive. |
| `getInited` | Boolean | If provided, will send a post message back with the current value of `<replay-web-page>`s `inited` property, indicating whether or not the service worker is ready. | 

### Messages hoisted from `<replay-web-page>`
`warc-embed` will forward to the embedding website every post message sent by `<replay-web-page>`'s service worker. 

The most common example is the following, which is sent during navigation within an archive:

```json
{
  "warcEmbedHref": "https://warcembed.domain.ext/?source=archive.warc.gz&url=https://what-was-archived.ext/path",
  "url": "https://what-was-archived.ext/new-path/",
  "view": "pages",
  "ts": "20220816162527"
}
```

### Example: Intercepting messages from a `warc-embed` `<iframe>`
```javascript
// Assuming: there's only 1 <iframe class="warc-embed">  
const playback = document.querySelector("iframe.warc-embed");

window.addEventListener("message", (e) => {
  // This message bears data and comes from the `warc-embed` <iframe>
  if (event?.data && event.source === playback.contentWindow) {
    console.log(event);
  }
});
```

### Example: Sending a message to a `warc-embed` `<iframe>`
```javascript
// Assuming: there's only 1 <iframe class="warc-embed">  
const playback = document.querySelector("iframe.warc-embed");
const playbackOrigin = new URL(playback.src).origin;

playback.contentWindow.postMessage(
  {"setUrl": "https://what-was-archived.ext/new-path"},
  playbackOrigin
);
```

[☝️ Back to summary](#summary)
