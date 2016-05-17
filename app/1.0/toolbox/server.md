---
title: Serve your app
---

<!-- toc -->

You can  serve an App Toolbox app using any server technology you want, but App Toolbox includes a
_reference server_, a sample server implementation built on top of Google App Engine. The reference
server takes advantage of HTTP/2 and HTTP/2 server push to deliver the resources the browser needs
for a fast first paint while optimizing caching.

## PRPL strategy

To optimize delivery, the reference server uses the [PRPL strategy](link-to-blog-yet-to-be), which
stands for:

*  Push critical resources for the initial route.
*  Render initial route.
*  Pre-cache remaining routes.
*  Lazy-load and create remaining routes on demand.

To do this, the server needs to be able to identify the resources required by each of the app's
routes. Instead of bundling the resources into a single unit for download, it uses HTTP2 push to
deliver the individual resources needed to render the requested route.

The server and service worker together work to precache the resources for the inactive routes.

When the user switches routes, the app lazy-loads any required resources that haven't been cached
yet, and creates the required views.

## App structure

Currently, the Polymer CLI and reference server support a single-page app (SPA) with the
following structure:

-   The main _entrypoint_ of the application which is served from every valid route. This
    file should be very small, since it will be served from different URLs therefore be cached
    multiple times. All resource URLs in the entrypoint need to be absolute, since it may be served
    from non-top-level URLs.

-   The _shell_ or app-shell, which includes the top-level app logic, router, and so on.

-   Lazily loaded _fragments_ of the app. A fragment can represent the code for a particular
    view, or other code that can be loaded lazily (for example, parts of the main app not required
    for first paint, like menus that aren't displayed until a user interacts with the app). The
    shell is responsible for dynamically importing the fragments as needed.

The diagram below shows the components of a simple app:

![](/images/1.0/toolbox/app-build-components.png)

In this diagram, the solid lines represent _static dependencies_, external resources identified
in the files using `<link>` and `<script>` tags. Dotted lines represent _dynamic_ or _demand-loaded
dependencies_: files loaded as needed by the shell.

The build process builds a graph of all of these dependencies, and the server uses this information
to serve the files efficiently. It also builds a set of vulcanized bundles, for browsers that don't
support HTTP2 push.

### App entrypoint

The entrypoint must import and instantiate the shell, as well as conditionally load any
required polyfills.

The main considerations for the entrypoint are:

-   Has minimal static dependencies—not much beyond the app-shell itself.
-   Conditionally loads required polyfills.
-   Uses absolute paths for all dependencies.

When you generate a [[[fill in the blank]]] project using Polymer CLI, the new
project contains an entrypoint in `index.html`. For most projects, you shouldn't
need to update this file.

### App shell

The shell is responsible for routing and usually includes the main navigation UI for the app.

The app should call `importHref` to lazy-load fragments as they're required. For example, when the
user changes to a new route, it imports the fragment(s) associated with that route. This may
initiate a new request to the server, or simply load the resource from the cache.

    importHref('list-view.html');

The shell (including its static dependencies) should contain everything needed for first paint.


### Serving bundled resources

For browsers that don't handle HTTP2 Push, the build process produces a set of vulcanized bundles:
one bundle for the shell, and one bundle for each fragment. The diagram below shows how a simple
app would be bundled:

![](/images/1.0/toolbox/app-build-bundles.png)

Any dependency shared by two or more fragments is bundled in with the shell and its static
dependencies.

Each fragment and its _unshared_ static dependencies are bundled into a single bundle. The server
returns the appropriate version of the fragment (bundled or unbundled), depending on the browser.
This means that the shell code can lazy-load `detail-view.html` _without having to know whether
it is bundled or unbundled_. It relies on the server and browser to load the dependencies in the
most efficient way.

## Deploy with the reference server

To deploy with the reference server, you'll need to install the

## Background: HTTP/2 and HTTP/2 server push

HTTP/2 allows _multiplexed_ downloads over a single connection, so that multiple small files can be
downloaded more efficiently.

HTTP/2 server push allows the server to preemptively send resources to the browser.

For an example of how HTTP/2 server push speeds up downloads, consider how the browser retrieves an
HTML file with a linked stylesheet.

In HTTP/1:
*   The browser requests the HTML file.
*   The server returns the HTML file and the browser starts parsing it.
*   The browser encounters the `<link rel="stylesheet">` tag, and starts a new request for the
    stylesheet.
*   The browser receives the stylesheet.

With HTTP/2 push:
*   The browser requests the HTML file.
*   The server returns the HTML file, and pushes the stylesheet at the same time.
*   The browser starts parsing the HTML. By the time it encounters the `<link rel="stylesheet">`,
the stylesheet is already in the cache.

In this simplest case, HTTP/2 server push eliminates a single HTTP request-response.

With HTTP/1, developers bundle resources together to reduce the number of HTTP requests required to
render a page. However, bundling can reduce the efficiency of the browser's cache. if resources for
each page are combined into a single bundle, each page gets its own bundle, and the browser can't
identify shared resources.

The combination of HTTP/2 and HTTP/2 server push can provide the _benefits_ of bundling (reduced
latency) without needing to bundle resources. Keeping resources separate means they can be cached
efficiently and be shared between pages.