# Theoretical foundations for server-rendering

And why they are the same thing.

## Definitions

A renderer is a function that takes an HTTP request as input, and outputs a page.
$$
renderer(req)=res
$$
A request is a set of attributes.

A result is a rendered template. Like a text with holes, whose holes have been filled. The template could be a React component, or a template written in more classical language, like EJS, PUG, PHP... Therefore, the page is entirely defined by the choice of the templates, and by the filling values.

Therefore, we can also consider a result as a set of attributes.
$$
renderer(req) = res\\
req = (req_i)\\
res = (res_j)\\
$$
Attributes of a request may be of different nature. We can represent them as a flat dictionary or a vector.

- Finite : the user id, some url parameters
- Infinite : the current date, some url parameters

So, there is an infinite number of requests if you consider all parameters, but you can build finite subsets if you consider only certain attributes of the request.

### Takeaways

- server-side rendering is just normal request processing. We don't really care about the technology, any app with server-rendering is just a function that processes requests and output some results. If the result happens to be some HTML, CSS and JS, we call that rendering, but there is no strong difference with any other kind of API.
- the difference between build-time rendering ("static") and request-time rendering ("ssr") is mostly based on the nature of the request attributes you'll want to consider to compute the result.

## Rendering at build time

At build time, there is no HTTP request happening. However, to keep our definition consistent, we can suppose that the render function is still using a request as input, except that it is limited to parameters you can know at build-time.
$$
\forall req; renderer(req)=renderer(req_{build})
$$
But what is this "req_build" object? Let's define it in terms of ensemble of possible requests instead. To be eligible for build-time rendering, our set of requests must have following properties:

- It's a subset of the set of all possible requests
- It must be a finite set, otherwise build time would be infinite
- Values should be known at build time and stay constant

Let's call $R_{renderer}$ the set of requests that a renderer takes into account. It's equivalent to picking a few attributes that your renderer function actually uses.
$$
\forall req \in R; renderer(req) = renderer(req_{req_1, req,4})\\
\implies R_{renderer} = R_{req_1, req_4}
$$

### The 3 static-rules of build-eligibility

Formally, if we note $R$ the set of all possible requests. A build-eligible set of request would have to respect those 3 conditions:
$$
\begin{equation}
R_{build} \subset R
|R_{build}| < \infty\\
R_{build}(t) = R_{build}
\end{equation}
$$
So, a valid build time request is any set of attributes that belongs to any valid R_{build} ensemble. Let's call $RB$ the set of all build-eligible sets. Picture it as any valid combination of attributes that can be used

If $R_{renderer}$ respects all 3 conditions, it's a build-eligible renderer. Congratulations, you can enjoy static rendering.

### Takeaways

- By selecting the attributes you consider as input of your renderer function, you define an $R_renderer$ set. If it respects the 3 build rules, you can enjoy static rendering.
- Example: rendering a finite number of blog article. $R_{renderer}$ is the list of all your articles URL, it's finite, doesn't change. If you write a new article, you can of course rebuild to get fresh data (we'll dig that problem later).

## Rendering at request time

As soon as $R_{renderer} violates one of the 3 conditions to be a build-eligible set... you cannot static render anymore :( You are good to do some server-side rendering, meaning you'll need to compute the render function for each incoming request.

Examples :

- One attribute is infinite. Say, your page is displaying the current date. You cannot prebuild for all dates.
- One attribute is time-dependent. Request time is the obvious example here, but it also violates the finiteness property so it's not that interesting. A better example would be any kind of dynamic attribute. For example, the user id. The list of users evolves each time someone sign up on your website, so you cannot rebuild every time.

Let's focus on this "you cannot rebuild every time". Actually, this definition is a bit trickier.

Suppose that rendering time is constant for all requests, for the sake of simplicity. Let's call it $t_{render}$._

### The 4th dynamic rule of build-eligibility

Let's also define the time between 2 evolutions of the set of possible values of an attributes. Basically, the time before you add a new article for your awesome blog. Let's note it $t_{\DeltaR_{renderer}$.
$$
\exists req_i, t_{render} < t_{\Delta R_{req_i}} \\
\implies R_{req_i} 
$$
It means that if you can rebuild faster than the list of possible values, the build-eligibility still holds. But if you rebuild slower, it doesn't not, you need request-time rendering.

Examples:

- the list of articles on your personal blog only evolves daily/weekly. A rebuild takes a few minutes => articles are eligible for build-time rendering
- the list of users on your brilliant SaaS product evolves a lot, you can get 10 new users a day => you are not eligible

### Takeaways

- there are actually 4 rules for build-eligibility.

## The question of picking the right template

Need some



## Next.js vision of rendering TODO

### One rendering function per "page"

In Next.js, a "page" is a React component tied to props computing functions, either build time or request time.

So, in our terminology, a Next.js "page" is a template associated to it's own rendering function.



### Static Site Generation

Static Site Generation is a form of build-time rendering. However, it does accept only one attribute as input: the request URL.
$$
\forall req; R(req)=R(req_{url})
$$
If we consider that all URL are known at build-time, we can render the page once for each possible URL. At runtime, all subsequent request on the same URL will get the same page.

t could be rephrased in terms of finitness. An equivalent phrasing would be to say that if the set of possible URLs is finite, you can render all of them at build time.



### Incremental Static Rendering

A problem happens when the number of possible URLs is starting to grow big, or to become dynamic. Technically, there can still be a finite number of valid URL at a certain point in time. Think of an e-commerce, when you log in, there is is a finite, known, number of items.

However, if this number changes often or is too big, it is more relevant to consider the set of possible attributes as infinite.





What happens when the possible URLs are actually infinite?

## 