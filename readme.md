# Theoretical foundations for server-rendering

And why they are the same thing.

## High-level view

A renderer is a function that takes an HTTP request as input, and outputs a web page.
$$
renderer(req)=res
$$
A request is a set of attributes.

A result is a rendered template. Like a text with holes, whose holes have been filled. The template could be a React component, or a template written in more classical language, like EJS, PUG, PHP... 

Therefore, the page is entirely defined by the choice of the templates, and by the filling values.

We do not really care about the final rendered value. That's a matter of implementation. So instead, we we call the template choice + the values that fills the holes the "props". And we will focus only on the sub-part of the $renderer$ function that computes the props.

Therefore, we will focus on the following function:
$$
propsGetter(req) = props\\
req = (attr_i)\\
props = (prop_j)\\
$$
Attributes of a request may be of different nature. We can represent them as a flat dictionary or a vector.

- Finite : the user id, some url parameters
- Infinite : the current date, some url parameters

So, there is an infinite number of requests if you consider all parameters, but you can build finite subsets if you consider only certain attributes of the request.

### Takeaways

- server-side rendering is just normal request processing. We don't really care about the technology, any app with server-rendering is just a function that processes requests and output some results. If the result happens to be some HTML, CSS and JS, we call that rendering, but there is no strong difference with any other kind of API. What matters are the value you will use to fill your template.
- the difference between build-time rendering ("static") and request-time rendering ("ssr") is mostly based on the nature of the request attributes you'll want to consider to compute the result.

## Rendering at build time

At build time, there is no HTTP request happening. However, to keep our definition consistent, we can suppose that the render function is still using a request as input, except that it is limited to parameters you can know at build-time.
$$
\forall req; propsGetter(req)=propsGetter(req_{build})
$$
But what is this $req_{build}$ object? Let's define it in terms of ensemble of possible requests instead. To be eligible for build-time rendering, our set of requests must have following properties:

- It's a subset of the set of all possible requests
- It must be a finite set, otherwise build time would be infinite
- Values should be known at build time and stay constant

Let's call $R_{renderer}$ the set of requests that a renderer takes into account. It's equivalent to picking a few attributes that your renderer function actually uses.
$$
R_{getter} = R_{\{req_1, req_4\}} \iff \forall req \in R; propsGetter(req) = propsGetter(req_{\{req_1, req,4\}})\\
$$

### The 3 static-rules of build-eligibility

Formally, if we note $R$ the set of all possible requests. A build-eligible set of request would have to respect those 3 conditions:
$$
\begin{equation}
R_{build} \subset R\\
|R_{build}| < \infty\\
R_{build}(t) = R_{build}
\end{equation}
$$
Rule 1 is that obviously the request should be actually possible. Rule 2 is the "rule of finiteness", rule 3 is the "rule of staticness".

So, a valid build time request is any set of attributes that belongs to any valid R_{build} ensemble. Let's call $RB$ the set of all build-eligible sets. Picture it as any valid combination of attributes that can be used

If $R_{renderer}$ respects all 3 conditions, it's a build-eligible renderer. Congratulations, you can enjoy static rendering.

### Implementation

Build-time rendering is applying $renderer$ to all $req$ included in $R_renderer$.

But how do you actually compute this set? Here are the typings and the final build-time rendering function:

```ts
type Req = Object
type Props = Object
type HTML = string
type RBuildComputer = () => Array<Props> // must respect the build-eligbility conditions
type PropsGetter = (req: Req) => Props
type Renderer = (props: Props) => HTML
function buildTimeProps(RGetter: RBuildComputer, propsGetter: Renderer): Array<Res> => (
    RGetter().map(req => propsGetter(req))
)
// finally compute the HTML. But this is not the interesting part here.
function buildTimeRender = (buildTimeProps: Array<Props>, render: Renderer) => buildTimeProps.map(render)

```

This doesn't feel very natural. That's because requests are supposed to be random events, triggered by the user, so we are not used to list all the possible requests for a given endpoint.

Also, when actually serving the pages, this means you still need some request processing logic. The final HTML/CSS/JS result may be cached, the result of the $render$ function, but you still need to run the $propsGetter$ function again for each request. 

We'll describe how Next.js solves a simplified version of this problem later-on.

### Takeaways

- By selecting the attributes you consider as input of your renderer function, you define an $R_renderer$ set. If it respects the 3 build rules, you can enjoy static rendering.
- Example: rendering a finite number of blog article. $R_{renderer}$ is the list of all your articles URL, it's finite, doesn't change. If you write a new article, you can of course rebuild to get fresh data (we'll dig that problem later).
- This is theoretical, in real life computing the set of possible requests feels unnatural.
- There is no such thing as a website without a server. Static hosts are just extremely basic server that can only process the URL part of the request, but they still do process the request.

## Rendering at request time

As soon as $R_{renderer}$ violates one of the 3 conditions to be a build-eligible set... you cannot static render anymore :( You are good to do some server-side rendering, meaning you'll need to compute the render function for each incoming request.

Examples :

- One attribute is infinite. Say, your page is displaying the current date. You cannot prebuild for all dates.
- One attribute is time-dependent. Request time is the obvious example here, but it also violates the finiteness property so it's not that interesting. A better example would be any kind of dynamic attribute. For example, the user id. The list of users evolves each time someone sign up on your website, so you cannot rebuild every time.

Let's focus on this "you cannot rebuild every time". Actually, this definition is a bit trickier.

Suppose that rendering time is constant for all requests, for the sake of simplicity. Let's call it $t__{render}$._

### The 4th dynamic rule of build-eligibility

Let's also define the time between 2 evolutions of the set of possible values of an attributes. Basically, the time before you add a new article for your awesome blog. Let's note it $t__{\DeltaR_{getter}$.
$$
\exists req_i, t_{render} > t_{\Delta R_{req_i}} \implies R_{req_i} \not\subset RB  \\
$$
It means that if you can rebuild faster than the list of possible values, for all attributes, the build-eligibility still holds. But if you rebuild slower, it doesn't not, you need request-time rendering.

Examples:

- the list of articles on your personal blog only evolves daily/weekly. A rebuild takes a few minutes => articles are eligible for build-time rendering
- the list of users on your brilliant SaaS product evolves a lot, you can get 10 new users a day => you are not eligible

### Takeaways

- there are actually 4 rules for build-eligibility, one of them depends on how fast the possible values evolves for each input attributes. Build-time rendering makes sense only for slowly changing values (a few seconds being a minimum, a few minutes more appropriate).
- Request-time server rendering is more an "exception" than the default. Build-time rendering should be preferred whenever possible.

## Next.js vision of rendering

Let's apply our model to Next.js.

### A page = a template + a props getter function

In Next.js, a "page" is a React component tied to props computing functions, either build time or request time.

So, in our terminology, a Next.js "page" is a template + a props getter function.

### Static Site Generation

Static Site Generation is a form build-time rendering. However, it does accept only one attribute as input: the request URL. 

This includes the URL parameters as well. So, a page always have a base path, for instance, `/blog/articles/:id `. In Next.js, this corresponds specifically to its position. But it can also have "parameterized paths", which are called dynamic routes : that's just the base path + some route parameters. For instance, `/blog/articles/12`.

So, Next implicitly defines an additional build-time eligibility rule like follow:
$$
\R_{getter} \subset RB \implies \forall req \in R_{getter}; propsGetter(req)= propsGetter(req_{url})
$$
The props must only depends on the considered route or URL.

If we consider that all URLs for a page are known at build-time, we can render the page once for each possible URL. At runtime, all subsequent request on the same URL will get the same page.

#### Implementation

The big advantage over our generic version? This is dead easy to implement. The request processing is limited to a bare minimum, since you only need to look at the base path to find the template, and the route parameters to compute the props. 

Intuitively, it's easy to precompute all the possible paths for an app. Even if the value are dynamic, you simply need a few requests, for instance to get the list of articles from your CRM.

The limitation is however that in order to get different props, you need a different URL. So if you want to prerender a light and dark mode for the same route, you would need an ugly route parameter. 

Also, a lot of valid build-time attributes cannot be obtained just by looking at the URL. You would also need to process the request cookies for instance, to tell if the user is authenticated or not.

### Server-side rendering

Nothing fancy here, it's just classical SSR.

### Incremental Static Regeneration

Formally, Incremental Static Rendering alleviates both the rule of finitness and the rule of staticness for build-eligibility. Instead of rendering all the pages at build-time, you render them only on-demand. You can also rerender the pages more frequently, without rebuilding everything.

Since the number of requests that can hit your website is finite in a finite amount of time, it allows you to stretch the build time. Note: there is still a very minor hypothesis, that people don't spam your website with so many different requests that it saturates your ability to render the pages. More formally, that they don't map $R_getter$ faster than $|R_getter|*t_render$. 

And also, that the possible values still evolves relatively slowly, otherwise user will still get stale data. If the cache time to live is to short, you end up with usual server-side rendering, which defeats the purpose of ISR.

There is still a strong limitation: props are still entirely defined by the URL. You cannot process the request with current implementation.

#### A better ISR

```ts
export async function propsGetter(req: Request): Props {
    /// do your thing
    return {...}
}
export async TTL = 5
```

Yes, this is server-side rendering with a cache.

- TTL = Infinity => this is static rendering
- TTL = 0 => this is server-side rendering
- TTL = X, 0 < X < Infinity => this is incremental static regeneration