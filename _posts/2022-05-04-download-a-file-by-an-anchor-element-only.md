---
title:  "Download a file by an anchor-element only. No more Content-Disposition!"
date:   2022-05-04 08:00:00 +0200
excerpt: "The anchor-element has a download attribute which instructs a browser to download a file instead of linking to it."
header:
  og_image: /assets/images/2022-05-04/download-a-file-by-an-anchor-element-only.png
---

In the past I used the HTTP header `Content-Disposition` to instruct the browser to download a file. I ended up doing this because I knew of no other way for this.

Until today.

I discovered, that the `anchor`-element has a `download` attribute.

```html
<a href="35567e83.pdf" download="report-2022.pdf">
  Download Report
</a>
```

By clicking this link, the browser will download the file instead of linking to it and saves it as `report-2022.pdf`.

The [browser support](https://caniuse.com/download) for this attribute is pretty good with support by all major desktop and mobile browsers.

More details can be found at the [MDN docs](https://developer.mozilla.org/en-US/docs/Web/HTML/Element/a#attr-download).

Comments are welcome on [Twitter](https://twitter.com/TheThomasPr/status/1521745346937991169) or [LinkedIn](https://www.linkedin.com/posts/thomas-preissler_download-a-file-by-an-anchor-element-only-activity-6927511231180234752-D-Vz).
