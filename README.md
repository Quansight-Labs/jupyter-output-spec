Currently there are many different rendering engines for Jupyter notebook outputs. Off the top of my head...


1. JupyterLab
2. nteract
3. voila
4. nbconvert
5. Jupyter sphinx
6. Colab
7. VS Code

What this means is that if you have a custom way of rendering some widget (plotly, vega, etc), you maybe have to build
integration for ALL of these separately! It's insane!

Jupyter Widgets gets around this by providing a common base and [uses require.js](https://ipywidgets.readthedocs.io/en/stable/examples/Widget%20Custom.html) (in Classic Notebook and HTML pages) to let you provide custom renderers and it [uses a CDN](https://github.com/jupyter-widgets/ipywidgets/issues/1627) to grab the JS files.

Another way to get around this is to use a more general output, like html, and just embed a bunch of JS in there that creates a DOM node and does what you want.


However, a limitation of this is that you don't have a way to get access to the kernel, so you can't connect to comms. 


So here is a list of requirements I would have for any cross-implementation MIME rendering thingy:

1. Supports comms (including changing kernels, no kernel available)
2. Support multiple instances of one output (happens in JupyterLab when you open an output in a new tab. Now you have it mounted in two nodes.)
3. Support data update (it should be able to pick up on updates to the mime data, without a full re-render)


And in terms of hosting it should be able to be run in both:

1. prebuilt mode, like JupyterLab is today, where you already have the renderer loaded
2. Dynamic mode, like Jupyter Widgets, to pull new renderers from a CDN.

So what I see here is at the core, for a mime render extension author, you should write a function that looks like this:


```typescript

type Comm = {
    send(data: object): Promise<void>;
    close(data: object): Promise<void>;
    // Iterator ends on close
    msgs: AsyncIterable<object>
}

/**
 * Renders the `data` to the `nodes` using the `kernel`.
 * 
 * Returns an iterator that has a new item after finishing rendering
 * each new data/node combination.
 */
type RenderFn<T extends object> = (options: {
    // the actual mime data
    data: T,
    /// any updates of the data from update display
    dataUpdates: AsyncIterable<T>,
    // The initial node for rendering
    node: Element,
    // Any changes of the node
    nodeUpdates: AsyncIterable<{add: Element} | {remove: Element}>
    // will resolve to error if not connected to kernel
    createComm(targetName: string, data: object): Promise<Comm>;
}) => AsyncIterable<{data: T, node: Element}>
```

Cool, so we could make this "interface" and make a way that, given one of these,
we could add a renderer to JupyterLab using a regular extension. However, that doesn't help address all the other platforms. 

So I propose a new `mimeType` called `application/vnd.jupyter.extensible+json`:

```typescript
// data for mimetype `application/vnd.jupyter.extensible+json`
type JupyterExtensibleData = {
  // reference to a package that shold return a an ES6 module with a default export of the function
  url: string;
  data: object;
}
```

In JupyterLab, we can build the render of this mime type to also allow extension, so that if you want to build JupyterLab with this renderer "pre-built" you can,
in which case it won't fetch the package. We can do this by allowing you to register some function that corresponds to a URL to say use this function instead of fetching that es6 module URL.
