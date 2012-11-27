# Using secure client-side sessions to build simple and scalable Node.JS applications

Static websites are easy to scale. You can cache the heck out of them and
you don't have state to propagate between the various servers that deliver
this content to end-users.

Unfortunately, most web applications need to carry some state in order to
offer a personalized experience to users. If users can log into your site,
then you need to keep sessions for them. The typical way that this is done
is by setting a cookie with a random session identifier and storing session
details on the server under this identifier.

# Scaling a stateful service

Now, if you want to scale that service, you essentially have to:

1. replicate that session data across all of the web servers,
2. use a central store that each web server connects to, or
3. ensure that a given user always hits the same web server

These all have downsides:

* Replication has a performance cost and increases complexity.
* A central store will limit scaling and increase latency.
* Confining users to a specific server leads to problems when that
server needs to come down.

However, if you flip the problem around, you can find a fourth option: storing
the session data on the client.

# Client-side sessions

Pushing the session data to the browser has some obvious advantages:

1. the data is always available, regardless of which machine is serving a user
2. there is no state to lose on the server
3. new web servers can be added instantly
4. nothing needs to be replicated between the web servers

There is one key problem though: you cannot trust the client not to tamper
with the session data.

For example, if you store the user ID for the user's account in a cookie,
it would be easy for that user to change that ID and then gain access to
someone else's account.

While this sounds like a deal breaker, there is a clever solution to
work around this trust problem: store the session data in a
tamper-proof package. That way, there is no need to trust that the
user hasn't modified the session data. It can be verified by the
server.

What that means in practice is that you encrypt and sign the cookie
using a server key to keep users from reading or modifying the session
data. This is what
[client-sessions](https://github.com/benadida/node-client-sessions)
does.

# node-client-sessions

If you use Node.JS, there's a library available that makes getting
started with client-side sessions trivial:
[node-client-sessions](https://github.com/benadida/node-client-sessions). It
replaces [Connect](http://www.senchalabs.org/connect/)'s built-in
[session](http://www.senchalabs.org/connect/session.html) and
[cookieParser](http://www.senchalabs.org/connect/cookieParser.html)
middlewares.

This is how you can add it to a [simple Express application](https://github.com/fmarier/node-client-sessions-sample):

    const clientSessions = require("client-sessions");

    app.use(clientSessions({
      secret: '0GBlJZ9EKBt2Zbi2flRPvztczCewBxXK' // set this to a long random string!
    }));

Then, you can set properties on the `req.session` object like this:

    app.get('/login', function (req, res){
      req.session.username = 'JohnDoe';
    });

and read them back:

    app.get('/', function (req, res){
      res.send('Welcome ' + req.session.username);
    });

To terminate the session, use the reset function:

    app.get('/logout', function (req, res) {
      req.session.reset();
    });

# Immediate revocation of Persona sessions

One of the main downsides of client-side sessions as compared to server-side
ones is that the server no longer has the ability to destroy sessions.

Using a server-side scheme, it's enough to delete the session data
that's stored on the server because any cookies that remain on clients
will now point to a non-existent session. With a client-side scheme
though, the session data is not on the server, so the server cannot be
sure that it has been deleted on every client. In other words, we
can't easily synchronize the server state (user logged out) with the
state that's stored on the client (user logged in).

To compensate for this limitation, client-sessions adds an expiry to the
cookies. Before unpacking the session data stored in the encrypted cookie,
the server will check that it hasn't expired. If it has, it will simply
refuse to honour it and consider the user as logged out.

While the expiry scheme works fine in most applications (especially when it's set to a
relatively low value), in the case of [Persona](https://login.persona.org),
we needed a way for users to immediately revoke their sessions as soon as they learn that they password has been compromised.

This meant keeping a little bit of state on the backend. The way we
[made this instant revocation possible](https://github.com/mozilla/browserid/commit/1b0444d85700a951edc74a0bf7ad5581b2cbfedd)
was by adding a new token in the user table as well as in
the session cookie.

Every API call that looks at the cookie now also reads the current token
value from the database and compares it with the token from the cookie. Unless they are the same, an error is returned and the user is logged out.

The downside of this solution, of course, is the extra database read for each
API call, but fortunately we already read from the user table in most of
these calls, so the new token can be pulled in at the same time.

# Learn more

If you want to give [client-sessions](https://github.com/benadida/node-client-sessions) a go, have a look at this [simple demo application](https://github.com/fmarier/node-client-sessions-sample). Then if you find any bugs, let us know via [our bug tracker](https://github.com/benadida/node-client-sessions/issues).
