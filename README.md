Currently there are many different rendering engines for Jupyter notebook outputs. Off the top of my head...


1. JupyterLab
2. nteract
3. voila
4. nbconvert
5. Jupyter sphinx
6. Colab
7. VS Code

What this means is that if you have a custom way of rendering some widget (plotly, vega, etc) you maybe have to build
integration for ALL of these separately! It's insane!

Jupyter Widgets get's around this by providing a common base and [uses require.js](https://ipywidgets.readthedocs.io/en/stable/examples/Widget%20Custom.html) to let you provide custom renderers and it [uses a CDN](https://github.com/jupyter-widgets/ipywidgets/issues/1627) to grab the JS files.

Another way to get around this is to use a more general output, like html, and just embed a bunch of JS in there that creates a dom node and does what you want.


However, a limitation of this is that you don't have a way to get access to the kernel, so you can't connect to comms. 


So here is a list of requirements I would have for any cross-implementation MIME rendering thingy:

1. Supports comms (including changing kernels, no kernel available)
2. Support multiple instance of one output (happens in jupyterlab when you open an output in a new tab. Now you have it mounted in two nodes.)
3. Support data update (it should be able to pick up on updates to the mime data, without a full re-render)


And in terms of hosting it should be able to be run in both:

1. prebuilt mode, like jupyterlab is today, where you already have the renderer loaded
2. Dynamic mode, how ipywidgets, to pull new renderers from a CDN.

So what I see here is at the core, for a mime render extension author, you should write a function that looks like this (using RXJS Observables for semantic
simplicity but we could change to not require them):


```typescript

type Comm = {
    send(data: object): Promise<void>;
    close(data: object): Promise<void>;
    msgs: Observable<object>
    // resolved on close
    close: Promise<object>
}

/**
 * Some subset of JupyterLab's IKernelConnection
 */
type Kernel = {
  createComm(targetName: string, data: object): Comm;
  // registers/unregisters on observable subscription
  registerCommTarget(targetName: string): Observable<{
    data: object,
    comm: Comm
  }>;
}

/**
 * Renders the `data` to the `nodes` using the `kernel`.
 * 
 * Returns an observable that has a new item after finishing rendering
 * each new data.
 */
type RenderFn<T extends object> = (options: {
    // the actual mime data
    data: Observable<T>,
    // the nodes we want to render on
    nodes: Observable<Array<Element>>
    // the current kernel or null if none connected
    kernel: Observable<Kernel | null>
}) => Observable<null>
```

Cool, so we could make this "interface" and make a way that, given one of these,
we could add a renderer to JupyterLab using a regular extension. However, that doesn't help address all the other platforms. 

So I propse a new `mimeType` called `application/vnd.jupyter.extensible+json` which should have two keys `package` and `data`. The package should be a NPM package name that we should be able to fetch using [`jsdelivr`](https://www.jsdelivr.com/features) that has a default ES6 export of this function.

In JupyterLab, we can build the render of this mime type to also allow extension, so that if you want to build JupyterLab with this renderer "pre-built" you can,
in which case it won't fetch the package. 
