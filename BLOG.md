# Scheduled Rendering and Pipelining in Latency Sensitive Web Applications - [Mojito Pipeline](https://github.com/yahoo/mojito-pipeline)

Rendering views for a web app that has less than trivial backends can quickly become problematic as the app may end up holding onto resources while waiting for the full response from all backends. This can become a strategic pain point that monopolizes memory and blocks progress, resulting in seconds of nothing else than a blank page and an idle connection.
At Yahoo Search, our dependence on several backends has forced us to become creative to step up end-to-end performance. To decrease perceived latency, even-out bandwith usage, and free up frontend memory faster, we've adopted an approach similar to what Facebook detailed in [this post](https://www.facebook.com/note.php?note_id=389414033919).

## Web page pipelining: the basics
Our goal is to be able to render sections of the page as soon as the corresponding data becomes available and concurently flush rendered sections to the client, so the total time between transmiting the first to last byte of the response is significantly shorter.
The process can be roughly decomposed as follows:

**Step 1.** When a request arrives, the page is divided into small, coherent sections (called [_mojits_](http://developer.yahoo.com/cocktails/mojito/docs/intro/mojito_apps.html#mojits)) and data is requested from the backends if necessary.

**Step 2.** At the same time, the frontend sends the client a 'skeleton' of the page containing empty placeholders to be filled by sections as they get flushed. Something like this:

```html
<html>
    <head><!-- static assets, etc.. --></head>
    <body>
        <script type="text/javascript">
            /**
             * pipeline client that knows how to insert incoming markup.
             */
            var pipeline = ...
            pipeline.push = function (section) { ...
        </script>
        <div id="section1"><!-- this is an empty slot--></div>
        <div id="section2"><!-- this is an empty slot--></div>
```

> Notice how the `<body>` tag is not closed.

**Step 3.** The backends start responding with the requested data. Once the data that a section needs arrives, the section is rendered and serialized within a `<script>` block, which is flushed to the client as soon as possible. Something like:

```html
    <script>
        pipeline.push({
            id: "section1"
            markup: "<div>Hello, this is rendered section 1!!</div>"
        });
    </script>
```

**Step 4.** The client receives the script block containing the serialized section, and executes the script, which inserts the section into its corresponding placeholder on the page. Below is the simplified pipeline.push function that is called by the executed script.

```javascript
    pipeline.push = function (section) {
        window.getElementById(section.id).innerHTML = section.markup;
    }
```

**Step 5.** Once the frontend is done rendering all the sections and there are no more `<script>` blocks to send, it sends the closing tags to the client and closes the connection:

```html
    </body>
</html>

```

## So what do you get for free (almost)?
This approach has several advantages.
* It decreases the user perceived latency: now sections such as the logo, the search box and all the static stuff that appears on the page all the time can be sent immediately and the user can start typing instead of staring into cold nothingness.
* It maximizes bandwidth usage: by flushing sections as soon as they are ready, we ensure that the available bandwidth is fully utilized when there is data to be sent. 
* It optimizes last-byte latency: periodic flushing also means that the last flush is a lot smaller and faster to transmit since it contains only the last section.
* It optimizes memory usage: by rendering and flushing as soon as possible, we don't have to hold onto the data of faster sections, and so less memory is used at any given point.

![graphical representation of pipelining]()

## Okay, I'm pretty sure this is awesome - how do I get it?
You're in luck: we made this stuff open-source and free (as in beer and as in speech) for our favorite Node.js app-framework at Search: [mojito](http://developer.yahoo.com/cocktails/mojito/). Mojito makes it easy to divide your page into sections or "mojits" (which is also useful for reusability, maintainability, and overall spiritual wellness). We made it a package you can find [here](https://github.com/yahoo/mojito-pipeline), and you will be able to see how to get ninja powers with complex scheduling, dependencies between sections, conditional rules and more!
