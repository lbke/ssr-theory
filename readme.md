# Theoretical foundations for server-side rendering and static-rendering

And why they are actually the same thing.

## High-level overview

A renderer is a function that takes an HTTP request as input, and outputs a web page.

$$
renderer(req)=res
$$

A request is a set of attributes. It's the HTTP request, but also the information that can be derived from it (current user, the slug of the article the user wants to read...).

A result is a rendered template. Like a text with blanks, whose blanks have been filled. The template could be a React component, or a template written in more classical language, like EJS, PUG, PHP... 

Therefore, the rendered page is entirely defined by the choice of the template and by the filling values.

We do not really care about the final rendered value here. That's a matter of implementation. So instead, we will call the template choice + the values that fills the blanks the $props$. And we will focus only on the sub-part of the $renderer$ function that computes the props, the $propsGetter$.

Therefore, we will focus on the following function:
$$
propsGetter(req) = props\\
\left\{
        \begin{array}{ll}
req = (attr_i)\\
props = (prop_j)\\
    \end{array}
\right.
$$
Attributes of a request may be of different nature. We can represent them as a flat dictionary or a vector.

- Discrete and finite : the user id, some url parameters
- Infinite, or continuous : the current date, some other url parameters

So, there is an infinite number of requests if you consider all parameters, but you can build finite subsets if you consider only certain attributes of the request.

### Takeaways

- server-side rendering is just normal request processing. We don't really care about the technology, any app with server-rendering is just a function that processes requests and outputs some results. If the result happens to be some HTML, CSS and JS, we call that rendering, but there is no strong difference with any other kind of API. 
- what matters are the value you will use to fill the blanks your template: the props.
- the difference between build-time rendering ("static") and request-time rendering ("ssr") is mostly based on the nature of the request attributes you'll want to consider to compute the result.

## Rendering at build time

At build time, there is no HTTP request happening. However, to keep our definition consistent, we can suppose that the render function is still using a request as input, except that it is limited to parameters you can know at build-time.
$$
\forall req; propsGetter(req)=propsGetter(req_{build})
$$
But what is this $req_{build}$ object? Let's define it in terms of ensemble of possible requests instead. To be eligible for build-time rendering, our set of requests must have following properties:

- It's a subset of the set of all possible requests, which we'll call $R$
- It must be a finite set, otherwise build time would be infinite
- Values should be known at build time and stay constant afterward

Let's call $R_{getter}$ the set of requests that a given $propsGetter$ takes into account. It's equivalent to picking a few attributes that your renderer function actually uses.
$$
R_{getter} = R_{\{attr_1, attr_4\}} \iff \forall req \in R; propsGetter(req) = propsGetter({\{attr_1, attr_4\}})\\
$$

### The 3 static rules of build-eligibility

We note $R$ the set of all possible requests. A build-eligible set of requests would have to respect those 3 conditions:

$$
R_{build} \subset R\\
|R_{build}| < \infty\\
R_{build}(t) = R_{build}
$$

Rule 1 is that obviously the request should make sense and be a "possible" request. Rule 2 is the "rule of finiteness", rule 3 is the "rule of staticness".

So, a valid build time request is any set of attributes that belongs to any valid $R_{build}$ ensemble. 

Let's call $RB$ the set of all build-eligible sets. Picture it as any valid combination of attributes that can be used to compute the props, and still respect the 3 rules of build-eligibility. This is not useful to our reasonning but facilitates notation a lot :
if $R_{getter}$ respects all 3 conditions, it's a build-eligible renderer, and it belongs to $RB$. 

Which can be written like this: $R_{getter} \subset RB$.

Congratulations, you can enjoy static rendering from an ensemblist point of view.

### Implementation

Build-time rendering is applying $renderer$ (and thus $propsGetter$) to all $req$ included in $R_{getter}$, supposing that $R_{getter} \subset RB$.

But how do you actually compute this set? Here are the typings and the final build-time rendering function:

```ts
type Req = Object
type Props = Object
type HTML = string
type RBuildComputer = () => Array<Req> // must respect the build-eligbility conditions
type PropsGetter = (req: Req) => Props
type Renderer = (props: Props) => HTML

// compute the props for each possible request
function buildTimeProps(RGetter: RBuildComputer, propsGetter: PropsGetter): Array<Props> => {
    return RGetter().map(req => propsGetter(req))
}

// finally compute the HTML based on those props. But this is not the interesting part here.
function buildTimeRender = (computedBuildTimeProps: Array<Props>, render: Renderer): Array<HTML> => {
    return computedBuildTimeProps.map(render)
}
```

This doesn't feel very natural. That's because requests are supposed to be random events, triggered by the user, so we are not used to list all the possible requests for a given endpoint.

Also, when actually serving the pages, this means you still need some request processing logic. The final HTML/CSS/JS result may be cached, the result of the $render$ function, but you still need to run the $propsGetter$ function again for each request. 

We'll describe how Next.js solves a simplified version of this problem later-on.

### Takeaways

- By selecting the attributes you consider as input of your renderer function, you define an $R_{getter}$ set. If it respects the 3 build rules, you can enjoy static rendering.
- Example: rendering a finite number of blog articles. $R_{getter}$ is the list of all your articles URL, it's finite, doesn't change. If you write a new article, you can of course rebuild to get fresh data (we'll dig that problem later).
- This is theoretical, in real life computing the set of all possible requests feels unnatural.
- There is no such thing as a website without a server. Static hosts are just extremely basic server that can only process the URL part of the request, but they still do process the request. They act as $propsGetter$ functions in our model.

## Rendering at request time

As soon as $R_{getter}$ violates one of the 3 conditions to be a build-eligible set... you cannot build-time render anymore :( You are good to do some request-time server-side rendering, meaning you'll need to compute the render function for each incoming request (so both the $propsGetter$ and the $renderer$).

Examples :

- One attribute is infinite. Say, your page is displaying the current date. You cannot prebuild for all dates.
- One attribute is time-dependent. Request time is the obvious example here, but it also violates the finiteness property so it's not that interesting. A better example would be any kind of dynamic attribute. For example, the user id. The list of users evolves each time someone sign up on your website, so you cannot rebuild every time someone signs up.

Let's focus on this "you cannot rebuild every time". Actually, this definition is a bit trickier.


### The 4th dynamic rule of build-eligibility

Suppose that rendering time is constant for all requests, for the sake of simplicity. Let's call it $t_{render}$.

Let's also define the time between 2 evolutions of the set of possible values of an attributes. Basically, the time before you add a new article for your awesome blog, or someone signs up to your equally awesome SaaS product. 

Let's note this time $t_{\Delta R_{getter}}$ (time for a variation of $R_{getter}$ to happen).

$$
t_{render} > t_{\Delta R_{getter}} \implies R_{getter} \not\subset RB  \\
$$

It means that if you can rebuild faster than the list of possible values, for all attributes, the build-eligibility still holds. But if you rebuild slower, it doesn't not, you need request-time rendering.

Examples:

- the list of articles on your personal blog only evolves daily/weekly. A rebuild takes a few minutes => articles are eligible for build-time rendering
- the list of users on your brilliant SaaS product evolves a lot, you can get 10 new users a day (or more, who knows) => you are not eligible

### Takeaways

- there are actually 4 rules for build-eligibility, one of them depends on how fast the possible values evolves for each input attributes. Build-time rendering makes sense only for slowly changing values (a few seconds being a minimum, a few minutes more appropriate).
- request-time server rendering is more an "exception" than the default. Build-time rendering should be preferred whenever possible.

## Next.js vision of rendering

Let's apply our model to Next.js.

### A page = a template + a props getter function

In Next.js, a "page" is a React component tied to props computing functions, either build time or request time.

So, in our terminology, a Next.js "page" is a template + a props getter function.

### Static Site Generation

Static Site Generation is a form build-time rendering. However, it does accept only one attribute as input: the request URL (and thus, in our definition, anything that can be derived from this URL). 

This includes the URL parameters as well. So, a page always have a base path, for instance, `/blog/articles/:id `. In Next.js, this corresponds specifically to its position in the `pages` folder. But it can also have "parameterized paths", which are called dynamic routes : that's just the base path + some route parameters. For instance, `/blog/articles/12`.

So, Next implicitly defines an additional build-time eligibility rule like follow:
$$
R_{getter} \subset RB \implies \forall req \in R_{getter}; propsGetter(req)= propsGetter(req_{url})
$$
The props must only depends on the considered route or URL. Otherwise you can't use the static props computation functions provided by Next.

If we consider that all URLs for a page are known at build-time, we can render the page once for each possible URL. At runtime, all subsequent request on the same URL will get the same page.

#### Implementation

The big advantage over our generic version? This is easier to understand for Next.js end users. The request processing is limited to a bare minimum, since you only need to look at the base path to find the template, and the route parameters to compute the props. 

Intuitively, it's easy to precompute all the possible paths for an app. Even if the values are dynamic, you simply need a few requests, for instance to get the list of articles from your CRM.

The limitation is however that in order to get different props, you need a different URL. So if you want to prerender a light and dark mode for the same route, you would need an ugly route parameter just for that. You cannot get different props based on a cookie or a query parameter. 

Also, a lot of valid build-time attributes cannot be obtained just by looking at the URL. You would also need to process the request cookies for instance, to tell if the user is authenticated or not.

### Server-side rendering

Nothing fancy here, it's just classical SSR. Take a deep breath, the last part is the interesting one.

### Incremental Static Regeneration

Formally, Incremental Static Rendering alleviates both the rule of finiteness and the rule of staticness for build-eligibility. Instead of rendering all the pages at build-time, you render them only on-demand. You can also rerender the pages more frequently, without rebuilding everything.

Since the number of requests that can hit your website is finite in a finite amount of time, it allows you to stretch the build time infinitely. 

Note: there is still a very minor hypothesis, that people don't spam your website with so many different requests that it saturates your ability to render the pages. More formally, that they don't map $R_{getter}$ faster than $|R_{getter}|*t_{render}$. 

And also, that the possible values still evolves relatively slowly, otherwise user will still get stale data. If the cache time to live is too short, you end up with usual server-side rendering, which defeats the purpose of ISR.

There is still a strong limitation: props are still entirely defined by the URL. You cannot process the request with a custom function in the current implementation.

## A more generic server-rendering?

Based on this model, here is what could be a unified view of SSR and SSG, for a Next.js page:

```ts
// the template
export const MyPage = (props: Props) => { return <>...</>}

// the requests you'd like to precompute. In Next.js, this is currently limited to a list of URLs
export async function computePossibleRequests = (): Array<Request> => {
    // do your thing
    return [...]
}
// for both request-time and build-time rendering
/* 
/!\ this function will run for each request, even when static rendering 
(in order to get the right cache key). 
In Next.js, for static pages, that corresponds to the router of 
the "hidden" Node.js server provided by Next + your custom getStaticProps. 
For SSR, that's getServerSideProps.
*/
export async function propsGetter(req: Request): Props {
    /// do your thing
    return {...}
}

// for Incremental Static Regeneration and request-time rendering
// this affects the renderer function = the React component rendering, which is the slow part
export const TTL = 5
```

Yes, build-time static rendering is just server-side rendering with a cache + precomputed requests. Tada.

- $TTL = \infty$ => this is static rendering. You must define `computePossibleRequests` as well, to get the list of pages to render.
- $TTL = 0$ => this is server-side rendering.
- $TTL = X; 0 < X < \infty$ => this is incremental static regeneration. You may want to prerender some pages as well.
- If `propsGetter` always return a new value (say it includes current time for instance), TTL should be set at zero. Otherwise memory will explode because of useless caching.
- You can always define `computePossibleRequests` to precompute some pages at build-time, for an hybridation between static render and server render (that's the point of ISR).
<!--stackedit_data:
eyJoaXN0b3J5IjpbLTExNTg5MjAzODEsMTkzMzA1MzUzMiwtMT
c4NDM1MDE5OF19
-->