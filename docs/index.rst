.. Flask-SocketIO documentation master file, created by
   sphinx-quickstart on Sun Feb  9 12:36:23 2014.
   You can adapt this file completely to your liking, but it should at least
   contain the root `toctree` directive.

Welcome to Flask-SocketIO's documentation!
==========================================

**Flask-SocketIO** gives Flask applications access to low latency
bi-directional communications between the clients and the server. The
client-side application can use the `SocketIO <http://socket.io>`_ Javascript
library or any compatible client to establish a permanent connection to the
server.

Installation
------------

You can install this package in the usual way using ``pip``::

    pip install flask-socketio

Requirements
------------

Since version 1.0, this extension is compatible with both Python 2.7 and
Python 3.3+. The asynchronous services that this package relies on can be
selected among three choices:

- `eventlet <http://eventlet.net/>`_ is the best performant option, with
  support for long-polling and WebSocket transports.
- `gevent <http://www.gevent.org/>`_ is the framework used in previous
  releases of this extension. The long-polling transport is fully supported.
  To add support for WebSocket, the `gevent-websocket <https://pypi.python.org/pypi/gevent-websocket/>`_
  package must be installed as well. The use of gevent and gevent-websocket
  is also a performant option, but slightly lower than eventlet.
- The Flask development server based on Werkzeug can be used as well, with the
  caveat that it lacks the performance of the other two options, so it should
  only be used to simplify the development workflow. This option only supports
  the long-polling transport only.

The extension automatically detects which asynchronous framework to use based
on what is installed. Preference is given to eventlet, followed by gevent. If
neither one is installed, then the Flask development server is used.

On the client-side, the official Socket.IO Javascript client library can be
used to establish a connection to the server. There are also official clients
written in Swift and C++. Unofficial clients may also work, as long as they
implement the `Socket.IO protocol <https://github.com/socketio/socket.io-protocol>`_.

Current Limitations
~~~~~~~~~~~~~~~~~~~

- Flask-SocketIO can only run in a single worker process at this time. Work is
  currently in progress to eliminate this limitation.

Differences With Flask-SocketIO Versions 0.x
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

Older versions of Flask-SocketIO had a completely different set of
requirements. Those versions had a dependency on
`gevent-socketio <https://gevent-socketio.readthedocs.org/en/latest/>`_ and
`gevent-websocket <https://pypi.python.org/pypi/gevent-websocket/>`_, which
are not required in release 1.0.

In spite of the change in dependencies, there aren't many significant
changes introduced in version 1.0. Below is a detailed list of
the actual differences:

- Release 1.0 drops support for Python 2.6, and adds support for Python 3.3,
  Python 3.4, and pypy.
- Releases 0.x required an old version of the Socket.IO Javascript client.
  Starting with release 1.0, the current releases of Socket.IO and Engine.IO
  are supported. Releases of the Socket.IO client prior to 1.0 are no
  supported. The Swift and C++ official Socket.IO clients are now supported
  as well.
- The 0.x releases depended on gevent, gevent-socketio and gevent-websocket.
  In release 1.0 gevent-socketio is not used anymore, and gevent is one of
  three options for backend web server, with eventlet and any regular
  multi-threaded WSGI server, including Flask's development web server.
- The Socket.IO server options have changed in release 1.0. They can be
  provided in the SocketIO constructor, or in the ``run()`` call. The options
  provided in these two are merged before they are used.
- The 0.x releases exposed the gevent-socketio connection as
  ``request.namespace``. In release 1.0 this is not available anymore. The
  request object defines ``request.namespace`` as the name of the namespace
  being handled, and adds ``request.sid``, defined as the unique session ID
  for the client connection, and ``request.event``, which contains the event
  name and arguments.
- To get the list of rooms a client was in the 0.x release required the
  application to use a private structure of gevent-socketio, with the
  expression ``request.namespace.rooms``. This is not available in release
  1.0, which includes a proper ``rooms()`` function.
- The recommended "trick" to send a message to an individual client was to
  put each client in a separate room, then address messages to the desired
  room. This was formalized in release 1.0, where clients are assigned a room
  automatically when they connect.
- The ``'connect'`` event for the global namespace did not fire on releases
  prior to 1.0. This has been fixed and now this event fires as expected.
- Support for client-side callbacks was introduced in release 1.0.

Upgrading to Flask-SocketIO 1.x from older releases
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

On the client side, you need to upgrade your Socket.IO Javascript client from
the 0.9.x releases to the 1.3.x or newer releases.

On the server side, there are a few points to consider:

- If you wish to continue using gevent, then uninstall gevent-socketio from
  your virtual environment, as this package is not used anymore and may
  collide with its replacement, python-socketio.
- If you want to have slightly better performance and stability, then it is
  recommended that you switch to eventlet. To do this, uninstall gevent,
  gevent-socketio and gevent-websocket, and install eventlet.
- If your application uses monkey patching and you switched to eventlet, call
  `eventlet.monkey_patch()` instead of gevent's `monkey.patch_all()`. Also,
  any calls to gevent must be replaced with equivalent calls to eventlet.
- Any uses of `request.namespace` must be replaced with direct calls into the
  Flask-SocketIO functions. For example, `request.namespace.rooms` must be
  replaced with the `rooms()` function.
- Any uses of internal gevent-socketio objects must be removed, as this
  package is not a dependency anymore.

Initialization
--------------

The following code example shows how to add Flask-SocketIO to a Flask
application::

    from flask import Flask, render_template
    from flask_socketio import SocketIO

    app = Flask(__name__)
    app.config['SECRET_KEY'] = 'secret!'
    socketio = SocketIO(app)

    if __name__ == '__main__':
        socketio.run(app)

The ``init_app()`` style of initialization is also supported. Note the way the
web server is started. The ``socketio.run()`` function encapsulates the start
up of the web server and replaces the standard ``app.run()`` standard Flask
development server start up. When the application is in debug mode the
Werkzeug development server is still used and configured properly inside
``socketio.run()``. In production mode the eventlet web server is used if
available, else the gevent web server is used. If eventlet and gevent are not
installed, the Werkzeug development web server is used.

The application must serve a page to the client that loads the Socket.IO
library and establishes a connection::

    <script type="text/javascript" src="//cdnjs.cloudflare.com/ajax/libs/socket.io/1.3.6/socket.io.min.js"></script>
    <script type="text/javascript" charset="utf-8">
        var socket = io.connect('http://' + document.domain + ':' + location.port);
        socket.on('connect', function() {
            socket.emit('my event', {data: 'I\'m connected!'});
        });
    </script>

Receiving Messages
------------------

When using SocketIO, messages are received by both parties as events. On the
client side Javascript callbacks are used. With Flask-SocketIO the server
needs to register handlers for these events, similarly to how routes are
handled by view functions.

The following example creates a server-side event handler for an unnamed
event::

    @socketio.on('message')
    def handle_message(message):
        print('received message: ' + message)

The above example uses string messages. Another type of unnamed events use
JSON data::

    @socketio.on('json')
    def handle_json(json):
        print('received json: ' + str(json))

The most flexible type of event uses custom event names::

    @socketio.on('my event')
    def handle_my_custom_event(json):
        print('received json: ' + str(json))

Named events are the most flexible, as they eliminate the need to include
additional metadata to describe the message type.

Flask-SocketIO also supports SocketIO namespaces, which allow the client to
multiplex several independent connections on the same physical socket::

    @socketio.on('my event', namespace='/test')
    def handle_my_custom_namespace_event(json):
        print('received json: ' + str(json))

When a namespace is not specified a default global namespace with the name
``'/'`` is used.

Clients may request an acknowledgement callback that confirms receipt of a
message. Any values returned from the handler function will be passed to the
client as arguments in the callback function::

    @socketio.on('my event')
    def handle_my_custom_event(json):
        print('received json: ' + str(json))
        return 'one', 2

In the above example, the client callback function will be invoked with
two arguments, ``'one'`` and ``2``. If a handler function does not return any
vallues, the client callback function will be invoked without arguments.

Sending Messages
----------------

SocketIO event handlers defined as shown in the previous section can send
reply messages to the connected client using the ``send()`` and ``emit()``
functions.

The following examples bounce received events back to the client that sent
them::

    from flask_socketio import send, emit

    @socketio.on('message')
    def handle_message(message):
        send(message)

    @socketio.on('json')
    def handle_json(json):
        send(json, json=True)

    @socketio.on('my event')
    def handle_my_custom_event(json):
        emit('my response', json)

Note how ``send()`` and ``emit()`` are used for unnamed and named events
respectively.

When working with namespaces, ``send()`` and ``emit()`` use the namespace of
the incoming message by default. A different namespace can be specified with
the optional ``namespace`` argument::

    @socketio.on('message')
    def handle_message(message):
        send(message, namespace='/chat')

    @socketio.on('my event')
    def handle_my_custom_event(json):
        emit('my response', json, namespace='/chat')

SocketIO supports acknowledgement callbacks that confirm that a message was
received by the client::

    def ack():
        print 'message was received!'

    @socketio.on('my event')
    def handle_my_custom_event(json):
        emit('my response', json, callback=ack)

When using callbacks the Javascript client receives a callback function to
invoke upon receipt of the message. After the client application invokes the
callback function the server invokes the corresponding server-side callback.
If the client-side callback returns any values, these are provided as
arguments to the server-side callback.

The client application can also request an acknoledgement callback for an
event sent to the server. If the server wants to provide arguments for this
callback, it must return them from the event handler function::

    @socketio.on('my event')
    def handle_my_custom_event(json):
        # ... handle the event

        return 'foo', 'bar', 123  # client callback will receive these 3 arguments

Broadcasting
------------

Another very useful feature of SocketIO is the broadcasting of messages.
Flask-SocketIO supports this feature with the ``broadcast=True`` optional
argument to ``send()`` and ``emit()``::

    @socketio.on('my event')
    def handle_my_custom_event(data):
        emit('my response', data, broadcast=True)

When a message is sent with the broadcast option enabled, all clients
connected to the namespace receive it, including the sender. When namespaces
are not used, the clients connected to the global namespace receive the
message. Note that callbacks are not invoked for broadcast messages.

In all the examples shown until this point the server responds to an event
sent by the client. But for some applications, the server needs to be the
originator of a message. This can be useful to send notifications to clients
of events that originated in the server, for example in a background thread.
The ``socketio.send()`` and ``socketio.emit()`` methods can be used to
broadcast to all connected clients::

    def some_function():
        socketio.emit('some event', {'data': 42})

Note that ``socketio.send()`` and ``socketio.emit()`` are not the same
functions as the context-aware ``send()`` and ``emit()``. Also note that in the
above usage there is no client context, so ``broadcast=True`` is assumed and
does not need to be specified.

Rooms
-----

For many applications it is necessary to group users into subsets that can be
addressed together. The best example is a chat application with multiple rooms,
where users receive messages from the room or rooms they are in, but not from
other rooms where other users are. Flask-SocketIO supports this concept of
rooms through the ``join_room()`` and ``leave_room()`` functions::

    from flask_socketio import join_room, leave_room

    @socketio.on('join')
    def on_join(data):
        username = data['username']
        room = data['room']
        join_room(room)
        send(username + ' has entered the room.', room=room)

    @socketio.on('leave')
    def on_leave(data):
        username = data['username']
        room = data['room']
        leave_room(room)
        send(username + ' has left the room.', room=room)

The ``send()`` and ``emit()`` functions accept an optional ``room`` argument
that cause the message to be sent to all the clients that are in the given
room.

All clients are assigned a room when they connect, named with the session ID
of the connection, which can be obtained from ``request.sid``. A given client
can join any rooms, which can be given any names. When a client disconnects it
is removed from all the rooms it was in. The context-free ``socketio.send()``
and ``socketio.emit()`` functions also accept a ``room`` argument to broadcast
to all clients in a room.

Since all clients are assigned a personal room, to address a message to a
single client, the session ID of the client can be used as the room argument.

Connection Events
-----------------

Flask-SocketIO also dispatches connection and disconnection events. The
following example shows how to register handlers for them::

    @socketio.on('connect', namespace='/chat')
    def test_connect():
        emit('my response', {'data': 'Connected'})

    @socketio.on('disconnect', namespace='/chat')
    def test_disconnect():
        print('Client disconnected')

The connection event handler can optionally return ``False`` to reject the
connection. This is so that the client can be authenticated at this point.

Note that connection and disconnection events are sent individually on each
namespace used.

Error Handling
--------------

Flask-SocketIO can also deal with exceptions::

    @socketio.on_error()        # Handles the default namespace
    def error_handler(e):
        pass

    @socketio.on_error('/chat') # handles the '/chat' namespace
    def error_handler_chat(e):
        pass

    @socketio.on_error_default  # handles all namespaces without an explicit error handler
    def default_error_handler(e):
        pass

Error handler functions take the exception object as an argument.

The message and data arguments of the current request can also be inspected
with the ``request.event`` variable, which is useful for error logging and
debugging outside the event handler::

    from flask import request

    @socketio.on("my error event")
    def on_my_event(data):
        raise RuntimeError()

    @socketio.on_error_default
    def default_error_handler(e):
        print request.event["message"] # "my error event"
        print request.event["args"]    # (data,)

Access to Flask's Context Globals
---------------------------------

Handlers for SocketIO events are different than handlers for routes and that
introduces a lot of confusion around what can and cannot be done in a SocketIO
handler. The main difference is that all the SocketIO events generated for a
client occur in the context of a single long running request.

In spite of the differences, Flask-SocketIO attempts to make working with
SocketIO event handlers easier by making the environment similar to that of a
regular HTTP request. The following list describes what works and what doesn't:

- An application context is pushed before invoking an event handler making
  ``current_app`` and ``g`` available to the handler.
- A request context is also pushed before invoking a handler, also making
  ``request`` and ``session`` available. But note that WebSocket events do not
  have individual requests associated with them, so the request context that
  started the connection is pushed for all the events that are dispatched
  during the life of the connection.
- The ``request`` context global is enhanced with a ``sid`` member that is set
  to a unique session ID for the connection. This value is used as an initial
  room where the client is added.
- The ``request`` context global is enhanced with ``namespace`` and ``event``
  members that contain the currently handled namespace and event arguments.
  The ``event`` member is a dictionary with ``message`` and ``args`` keys.
- The ``session`` context global behaves in a different way than in regular
  requests. A copy of the user session at the time the SocketIO connection is
  established is made available to handlers invoked in the context of that
  connection. If a SocketIO handler modifies the session, the modified session
  will be preserved for future SocketIO handlers, but regular HTTP route
  handlers will not see these changes. Effectively, when a SocketIO handler
  modifies the session, a "fork" of the session is created exclusively for
  these handlers. The technical reason for this limitation is that to save the
  user session a cookie needs to be sent to the client, and that requires HTTP
  request and response, which do not exist in a SocketIO connection. When
  using server-side sessions such as those provided by the Flask-Session or
  Flask-KVSession extensions, changes made to the session in HTTP route
  handlers can be seen by SocketIO handlers, as long as the session is not
  modified in the SocketIO handlers.
- The ``before_request`` and ``after_request`` hooks are not invoked for
  SocketIO event handlers.
- SocketIO handlers can take custom decorators, but most Flask decorators will
  not be appropriate to use for a SocketIO handler, given that there is no
  concept of a ``Response`` object during a SocketIO connection.

Authentication
--------------

A common need of applications is to validate the identify of their users. The
traditional mechanisms based on web forms and HTTP requests cannot be used in
a SocketIO connection, since there is no place to send HTTP requests and
responses. If necessary, an application can implement a customized login form
that sends credentials to the server as a SocketIO message when the submit
button is pressed by the user.

However, in most cases it is more convenient to perform the traditional
authentication process before the SocketIO connection is established. The
user's identify can then be recorded in the user session or in a cookie, and
later when the SocketIO connection is established that information will be
accessible to SocketIO event handlers.

Using Flask-Login with Flask-SocketIO
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

Flask-SocketIO can access login information maintained by
`Flask-Login <https://flask-login.readthedocs.org/en/latest/>`_. After a
regular Flask-Login authentication is performed and the ``login_user()``
function is called to record the user in the user session, any SocketIO
connections will have access to the ``current_user`` context variable::

    @socketio.on('connect')
    def connect_handler():
        if current_user.is_authenticated:
            emit('my response',
                 {'message': '{0} has joined'.format(current_user.name)},
                 broadcast=True)
        else:
            return False  # not allowed here

Note that the ``login_required`` decorator cannot be used with SocketIO event
handlers, but a custom decorator that disconnects non-authenticated users can
be created as follows::

    import functools
    from flask import request
    from flask_login import current_user
    from flask_socketio import disconnect

    def authenticated_only(f):
        @functools.wraps(f)
        def wrapped(*args, **kwargs):
            if not current_user.is_authenticated:
                disconnect()
            else:
                return f(*args, **kwargs)
        return wrapped

    @socketio.on('my event')
    @authenticated_only
    def handle_my_custom_event(data):
        emit('my response', {'message': '{0} has joined'.format(current_user.name)},
             broadcast=True)

Deployment
----------

The simplest deployment strategy is to have eventlet or gevent installed, and
start the web server by calling ``socketio.run(app)`` as shown above. This
will run the application on the eventlet or gevent web servers, in that order
of preference.

An alternative is to use `gunicorn <http://gunicorn.org/>`_ as web server,
using the eventlet or gevent workers. The command line that starts the
eventlet server is shown below::

    gunicorn --worker-class eventlet module:app

If you prefer to use gevent, the command to start the server is::

    gunicorn --worker-class gevent module:app

In these commands ``module`` is the Python module or package that defines the
application instance, and ``app`` is the application instance itself.

WebSocket support comes standard with eventlet, but when using gevent, to
enable WebSocket it is necessary to also install package gevent-websocket. If
this package is not installed the gevent server will only use the long-polling
transport mechanism.

When using gunicorn with the gevent worker and WebSocket, the command that
starts the server must be changed to select a custom gevent web server that
supports WebSocket. The modified command is the following::

    gunicorn --worker-class geventwebsocket.gunicorn.workers.GeventWebSocketWorker module:app

Gunicorn release 18.0 is the recommended release to use with Flask-SocketIO.
The 19.x releases are known to have incompatibilities in certain deployment
scenarios.

Using nginx as a WebSocket Reverse Proxy
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

It is possible to use nginx as a front-end reverse proxy that passes requests
to the application. However, it is important to note that only releases of
nginx 1.4 and newer support proxying of the WebSocket protocol. Below is an
example nginx configuration that proxies regular and WebSocket requests::

    server {
        listen 80;
        server_name localhost;
        access_log /var/log/nginx/example.log;

        location / {
            proxy_pass http://127.0.0.1:5000;
            proxy_redirect off;

            proxy_set_header Host $host;
            proxy_set_header X-Real-IP $remote_addr;
            proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        }

        location /socket.io {
            proxy_pass http://127.0.0.1:5000/socket.io;
            proxy_redirect off;
            proxy_buffering off;

            proxy_set_header Host $host;
            proxy_set_header X-Real-IP $remote_addr;
            proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;

            proxy_http_version 1.1;
            proxy_set_header Upgrade $http_upgrade;
            proxy_set_header Connection "Upgrade";
        }
    }

API Reference
-------------

.. module:: flask_socketio
.. autoclass:: SocketIO
   :members:
.. autofunction:: emit
.. autofunction:: send
.. autofunction:: join_room
.. autofunction:: leave_room
.. autofunction:: close_room
.. autofunction:: rooms
.. autofunction:: disconnect
