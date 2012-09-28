# The Asynchronous Plumber
As internal HTTP REST services, popularised by Amazon, have been gaining a lot of attention - newly emerging HTTP/1.1 extensions like Server Sent Messages and WebSockets are the perfect accompaniment to asynchronous tech stacks like Tornado, Twisted, Gevent and Node.js, not just for powering 10 minute chat demos.

## Who? -- SLIDE
 Wes Mason

  -	 @1stvamp (twitter/github)
  -	 Product Engineer at ServerDensity
  -	 I like Python and evented code 

## The web is our API -- SLIDE
  -	 Everything does HTTP
  -	 REST is *great* - let's do more of this 
  -	 Almost everything can do HTTP *asynchronously *

## HTTP All The Way Down 

  -	 HTTP is asynchronous because we can issue requests in a none blocking way - Tornado's AsyncHTTPClient, Gevent monkey patched urllib2, Twisted's HTTP client handler, and requests in Node.js for the coffee lovers in the audience,  are just a few examples. 
  -	 HTTP services have a common API
  -	 *Everything* has some way of sending a HTTP request - even if only synchronously 
  -	 Pretty much everything we care about getting an async response back to is itself enabled to act as a HTTP server  

## Our Internal Services -- SLIDE
  -	 RESTful
  -	 Stateless
  -	 Mostly thinlayers
  -	 Common documented API
  -	 Callable from frontend 

## Brave New (Async) World -- SLIDE
  -	 Longpolling, WebSockets and Server Sent Events
  -	 UIs are **responsive** or GTFO
  -	 Libraries like socket.io make this pretty easy for devs  

## Come Back To Me -- SLIDE
  -	 Jobs running in parallel using task systems like Celery
  -	 Async responses to the browser using servers like Tornado/Twisted/Gevent
 
## Come Back To Me (cont.) -- SLIDE
  -	 HTTP or WebSocket request kicks off a task
  -	 Task runs asynchronous to requests
  -	 Task completed and returns data...
  -	 ...how? 

## A stick of Celery
  -	 Celery is great at providing a framework for distributing async tasks
  -	 It's not so great at giving you a way of making tasks singletons, idempotent or letting you retrieve task results for more than one user if that task is a rerun singleton. 
  -	 And by it's not so great, I mean it's terrible.  

## Publish and subscribe -- SLIDE
  -	 We needed a way of getting information back out of Celery tasks keyed by any account ID
  -	 Pubsub is ideal for this
  -	 Originally we used a custom distributed task manager built on top of ZeroMQ and Gevent - like all the cool kids are doing
  -	 ZeroMQ has an inbuilt pubsub pattern - but it only let's you do 1 publisher 

## Many to many with pubsub -- SLIDE
  -	 Redis let's you do this,  and has a nice python API,  but we don't know Redis as a company
  -	 We did know RabbitMQ with production experience
  -	 In theory AMQP let's you do M2M pubsub 
  -	 Celery is a great distributed task manager that runs primarily on AMQP (usually Rabbit) queues 
  -	 So switched,  only to discover that while in theory AMQP will let you do this,  no one does,  so it wasn't supported 

## ...well, kind of -- SLIDE
  -	 You can obviously have multiple publishers and multiple subscribers,  otherwise celery wouldn't work very well
  -	 *but* you'll only ever get one result returned to one subscriber because tasks are meant to be single function calls,  not idempotent objects in themselves 
  -	 e.g.

```
>>> result = task.delay(foo) 
AsyncResult(id="some-unique-id") 
>>> result.get()
>>> # will get back a single result while other async calls to .get()
>>> # will get different to subsequent results 
```

## Cue the haxx, lots of 'em -- SLIDE
  -	 Running the AsyncResult inits in threads so they don't block tornado. Nope they're still definitely 1-result-per-request. (**TODO** link to library)
  -	 Polling MongoDB - great, but no realtime responses to actions. *Note*: MongoDB actually does have a way of facilitating a *long poll* request, but not supported by bit.ly' s asyncmongo library for Tornado, and does have other performance costs. 

## You Had Me Hooked -- SLIDE
  -	 The stack's just web APIs why not webhooks to return data?
  -	 They're async, just like handling the frontend
  -	 They can be tested the same way as your other APIs 

## Webhooks are great - let's do more of this 
  -	 Arming Ronacher of flask and jinja2 fame posted a a few interesting ideas on async recently (link) 
  -	 Effectively everything in our current stacks are already asynchronous. That is if it talks stateless HTTP and has some way of returning data to somewhere - It's async. Huzzah! 
  -	 Taking the ideas from *Skyhooks* we don't just have to post data back to a backend process and back out over a socket, we can fake out WebSockets (and faked sockets like those created by socket.io and SockJS) from regular HTTP services.  

## The Webhook Registry -- SLIDE
  -	 Register/unregister methods 
  -	 Add a callback to be called on return
  -	 Updates URIs in a MongoDB collection with current base URI + Webhook API URI and account ID for connected user

## Skyhoooks to reach the Tornado -- SLIDE
  -	 A registry of callbacks assigned to your application instance (register / unregister / call) 
  -	 Each registry call for a callback also registers an idempotent (per URI and account or user ID) entry in MongoDB with the "hook"  URI of the current Tornado instance 
  -	 On post back (HTTP POST) to a hook URI a Tornado webhandler calls the registry  with the data and given ID (account or user)
  -	 The registry in turn calls each registered callback for that ID with the data
  -	 **NB** this technique was copied and modified slightly from the excellent *Introduction to Tornado* by Dory & Michael 

## Meanwhile, inside a tasklet
  -	 Task completes async work (e.g. polling data from external APIs, running more queries etc.)
  -	 Data serialised as JSON payload
  -	 HTTP POST(s) back to URIs in MongoDB collections matching account ID for data

## Meanwhile,  back on the [server] farm -- SLIDE
  -	 You've got a lots of Celery tasks running in some way,  possibly via single processes using the greenlet or Gevent handlers, and/or multiple processes,  or multiple threads, and perhaps clustered over multiple hosts
  -	 One or more of these tasks needs to get data back to one or more web clients connected to one or more server instances 
  -	 Using webhooks created the way we do with *Skyhooks*: 
  -	 Data return can be lazy

```python
hooks = webhook_collection.find({"account_id": id}) 
if data and len(data) > 0:
    for hook in hooks:
	requests.post(hook["url"], data) 
```

  -	 Job done.  

## Aaand back on the Tornado ranch
  -	 Web handler receives data POST
  -	 Web handler passes data internally to Webhook registry and registry calls any callbacks registered for that account ID
  -	 WebSocket handlers with registered callbacks received data in turn and send this back upstream as socket messages  

## Some things to note
  -	 This can happen across as many tornado instances as you have running in your cluster and for any number of asynchronously connected users
  -	 The handlers (web or socket) don't care where the data comes from or when, this happens without polling or blocking the tornado event loop.  

## But what about..? 
  -	 Sometimes Tornado instances die
  -	 Upstream issues aren't a problem as when our longpoll http request or WebSocket dies it calls a handler method and we can call unregister in the hooks registry. 
  -	 If the instance itself dies it may leave a phantom webhook in mongo:
  -	     We can remove hooks that repeatedly fail to POST
  -	     We can also give hooks a TTL (MongoDB 2.2 has native TTLs)  and renew it in thr registry as a timed Tornado callback from the interested handler  

## Cleanup can be bothersome 
  -	 Each handler must unregister itself (it's callback) from the Webhook registry on connect ion close (socket drop, tornado instance shutdown etc.)
  -	 Webhook MongoDB collection can still collect stale URLs, primarily due to unclean tornado instance shutdowns but perhaps due to unforeseen bugs 

## For whom the bell tolls
  -	 TTLs can help deal with this
  -	 Each handler pings the registry every *X* seconds and the registry updates the MongoDB collection
  -	  A cleanup process can catch stale URLs or as of MongoDB 2.2 these can be handled automatically with inbuilt TTL fields (other DBs may have similar native management of stale entries) 


## The Reverse Webhook Proxy 
  -	 HTTP gateway/proxy takes requests (http, longpoll, WebSockets, flash socket etc) and proxies to backend service. As part of the request it exposes a Webhook URL somehow. e.g. via request parameter or convention.
  -	 The service returns a standard return format that the proxy understands (meaning either **here is my response** or **got ya but I need to think about it**) 
  -	 In the case of the latter response the service either continues processing its task or triggers an asynchronous task in some way,  e.g. in a job queue like Celery or even crontab triggered batch processing. 
  -	 Responses are posted back to the proxy Webhook address which in turn funnels the response back over the correct return channel to the client that made the initial request. Which if it is going over a socket of some variety will appear to have happened over a posh async channel like a WebSocket,  and not the plain ghetto HTTP requests that actually took place.  

## Is this the only way? -- SLIDE
  -	  Well no,  if we'd been able to get bidirectional pubsub out of ZeroMQ we probably would never have ended up down this rabbit hole. 
  -	 -  ZeroMQ offers some control over messaging,  e.g. Target a device so you could build a similar system partially over ZeroMQ 
  -	 -  Crossroads I/O supports a bidirectional pubsub pattern
  - -  redis
  -	 -  MongoDB tailing cursors (link to 10gen blog post) 
  -	 -  If you're not scaling to many simultaneously connected users then polling with timed callbacks will probably actually work well for you, especially with the new query speeds in Mongo 2.2 (if that's the way you swing) 
