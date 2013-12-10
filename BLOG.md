# Scheduled Rendering and Pipelining in Latency Sensitive Web Applications - [Mojito Pipeline](https://github.com/yahoo/mojito-pipeline)

Rendering views for a web app that has less than trivial backends can quickly become problematic as the app most often ends up holding onto resources waiting for the full response from all backend. This is acceptable when all backends are reasonably quick, but otherwise it can become a strategic painpoint that monopolizes memory, hangs the user's connection for seconds with nothing else than a blank page and an idle connection.
At Yahoo Search, we depend on several backends that forced us to become creative if we wanted to step up end-to-end performance. To decrease perceived latency, even-out bandwith usage and free-up front-end memory faster, we decided to adopt an approach that facebook detailed in [this post](https://www.facebook.com/note.php?note_id=389414033919).

## Web page pipelining: the basics
Our goal is to be able to send sections of the page to the client as soon as they are ready on the front end, so the total time to transmit the last byte of the response would be significantly lower.
The process can be roughly decomposed as follow:

1. A request arrives, the page is split into small - coherent sections (called _mojits_) and we request information for all of them to the backends.
2. In the meantime, we start sending a "skeleton" of page to the client that will place rendered mojits in their empty spots when they arrive. Something like this:

```html
<html>
    <head><!-- static assets, etc.. --></head>
    <body>
        <script type="text/javascript">
            /**
             * pipeline client that knows how to insert incoming markup.
             */
            var pipeline = ...
            pipeline.push = function (sectionObject) { ...
        </script>
        <div id="section1"><!-- this is an empty slot--></div>
        <div id="section2"><!-- this is an empty slot--></div>
```

> Notice how the `<body>` tag is not closed.

3. The backends start responding with the requested data. As soon as a mojit gets the data it needs from the backend, it is rendered with it on the front-end and a script tag is _flushed as soon as possible_ to the client, something like:

```html
    <script>
        pipeline.push({
            id: "section47"
            markup: "<div>Hello, this is rendered section 1!!</div>"
        });
    </script>
```

4. The skeleton receives the markup, inserts and displays it in the right spot on the page. Remember the definition of `pipeline.push` in the skeleton of step 1? It does something like this:

```javascript
    pipeline.push = function (sectionObject) {
        window.getelementbyid(sectionObject.id).insertBefore(sectionObject.markup);
    }
```

> Remember, that definition of the push method was sent in the skeleton, now simply flushing a line with `pipeline.push()` in a `<script>` tag makes the client execute the `pipeline.push` function.

## So what do you get for free (almost)?
This approach has several advantages.
* It decreases the user perceived latency: now the logo, the search box and all the static stuff that will be there all the time can be sent immediately and the user can start typing instead of staring into cold nothingness.
* It optimizes bandwidth usage: by flushing early our faster independant sections, we anticipate by clearing out the pipe early for when that last slow section arrives - we increase throughput.
* It optimizes last-byte latency: "clearing out the pipe" also means that last packet will be a lot smaller and faster to transmit if it contains only the slowest, last section.
* It optimizes memory usage: by flushing early, we don't have to hold onto the data of the faster sections and we can use that memory do do something more useful.

![graphical representation of pipelining]()

## Okay, I'm pretty sure this is awesome - how do I get it?
You're in luck: we made this stuff open-source and free (as in beer and as in speech) for our favorite Node.js app-framework at Search: [mojito](http://developer.yahoo.com/cocktails/mojito/). Mojito makes it easy to split your page into sections or "mojits" (which is also useful for reusability, maintainability, and overall spiritual wellness). We made it a package you can find [here](https://github.com/yahoo/mojito-pipeline), and you will be abe to see how to get ninja powers with complex scheduling, dependencies between sections, conditional rules and more!
