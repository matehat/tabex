tabex
=====

[![Build Status](https://travis-ci.org/nodeca/tabex.svg?branch=master)](https://travis-ci.org/nodeca/tabex)
[![NPM version](https://img.shields.io/npm/v/tabex.svg?style=flat)](https://www.npmjs.org/package/tabex)

> Cross-tab message bus for browsers.
>
> Awesome things to do with this library:
>
> - Send messages between browser tabs and windows.
> - Share single websocket connection when multiple tabs open (save server
>   resources).
> - Shared locks (run live sound notification only in single tab on server event).

[Demo](http://nodeca.github.io/tabex/)

__Supported browser:__

- All modern and IE9+.
- Ancient browsers will fallback to legacy mode:
  - Each tab will not know about neighbours.
  - Clients in the same window will be ok.

__Known issues:__

- Safari in private mode will fallback to legacy, because it prohibits
  `localStorage` write.
- Chrome on iOS will fallback to legacy, because it does not send
  `localStorage` events between tabs.
- Cross-domain messaging is not recommended due serious unfixable problems
  in browsers:
  - will not work in Safari by default, because of
  [security settings](http://stackoverflow.com/questions/20401751/).
  - will not work in IE11 due multiple bugs.


Install
-------

__node.js (for use with `browserify`):__

```bash
$ npm install tabex
```

__bower:__

```bash
$ bower install tabex
```


API
---

In 99% of use cases your tabs are on the same domain. Then `tabex` use is very
simple:


```js
// Do it in every browser tab, and clients become magically connected
// to common broadcast bus :)
var live = window.tabex.client();

live.on('channel.name', function handler(message) {
  // Do something
});

// Broadcast message to subscribed clients in all tabs, except self.
live.emit('channel.name2', message);

// Broadcast message to subscribed clients in all tabs, including self.
live.emit('channel.name2', message, true);
```

### Client


#### tabex.client(options)

Fabric to create messaging interface. For single domain you don't need any
options, and everything will be initialized automatically. For cross-domain
communication you will have to create html file with router for iframe, loaded
from shared domain.

Options:

- __iframe__ (String) - url of iframe source with router (on shared domain).
  Optional.
- __namespace__ (String) - optional, for mad case when you need muiltiple
  isolated `tabex` instances in parallel.


#### client.on(channel, handler)

Subscribe to channel. Chainable.

- __channel__ (String) - name of channel to subscribe.
- __handler__ (Function) - `function (data, channel) { ... }`.


#### client.off(channel [, handler])

Unsubscribe handler from channel. Chainable. If handler not specified -
unsubscribe all handlers from given channel.


#### client.emit(channel, message [, toSelf])

Send message to all tabs in specified channel. By default messages are
broadcasted to all clients except current one. To include existing client -
set `toSelf` to `true`.


#### client.lock(id, [timeout, ] fn):

- __id__ - lock identifier
- __timeout__ - optional, lock lifetime in ms, default 5000
- __fn(unlock)__ - handler will be executed if lock is acquired
  - __unlock__ - function to release acquired lock

__Note__. Getting lock takes 30ms when localStorage used as events transport.


#### client.filterIn(fn), client.filterOut(fn)

Add transformers for incoming (handled) and outgoing (emitted) messages.
Chainable. This is a very powerful feature, allowing `tabex` customization. You
can morph data as you wish when it pass client pipeline from one end to another.

- __fn(channel, message, callback)__ - filter function.
  - __channel__ - channel name.
  - __message__ - wrapped event data (see below).
  - __callback(channel, message)__ - function to return output data.

Filter message structure:

- __id__ - unique message id, `<node_id>_<msg_counter>`. Consists or unique
  client instance id (random) and message ccounter (inremented for each
  new message)
- __node_id__ - id message emitter source (client or router), random string.
- __data__ - message data, passed to `client.emit()`.

Use case examples:

- `faye` does not allow `.` in channel names. You can add filters
  for transparent replace of `.` with `!!`.
- you can drop and generate internal events to implement new features like locks
  or remote function calls.


### Router

Direct access to router is needed only for cross-domain communication (with
iframe). For single domain `tabex` client will create local router automatically.


#### tabex.router(options)

Fabric to create router in iframe. Options:

- __origin__ (Array|String) - list of valid origins, allowed to communicate with
  this iframe. If nothing set, then iframe domain will be used.
- __namespace__ - the same as in `tabex.client`.

__Example.__

In your code:

```js
var live = window.tabex.client({
  iframe: 'https://shared.yourdomain.com/tabex_iframe.html'
});
```

In iframe:

```js
window.tabex.router({
  origin: [
    'http://yourdomain.com',
    'https://yourdomain.com',
    'http://forum.yourdomain.com',
    'https://forum.yourdomain.com'
  ]
});
```

__Warning! Never set `*` to allowed origins value.__ That's not secure.


## Advanced use


### System events

Channels `!sys.*` are reserved for internal needs and extensions. Also `tabex`
already has built-in events, emitted on some state changes:

- __!sys.channels.refresh__ - emitted when list of subscribed channels changed
  (or in specific case, when master tab switched). Message data:
  - `channels` - array of all channels subscribed in all tabs
- __!sys.channels.add__ - emitted by `tabex.client` to notify router about new
  subscribed channel. Message data:
  - `channel` - channel name
- __!sys.channels.remove__ - emitted by `tabex.client` to notify router that all
  subscriptions to channel gone.  Message data:
  - `channel` - channel name
- __!sys.lock.request__ - emitted by `tabex.client` to try acquire lock. Message data:
  - __id__ - lock identifier
  - __timeout__ - lock lifetime in ms
- __!sys.lock.acquired__ -  emitted when router acquire lock for client
  - __request_id__ - request message id
- __!sys.lock.release__ - emitted by `tabex.client` to release already acquired
  lock. Message data:
  - __id__ - lock identifier
- __!sys.error__ - emitted on internal errors, for debug.
- __!sys.master__ - sepecific for localStorage-based router. Message data:
  - `node_id` - id of "local" router node
  - `master_id` - id of node that become master

__Note.__ `!sys.master` event is broadcasted only when `localStorage` router
used. You should NOT rely on it in your general application logic. Use locks
instead to filter single handler on broadcasts.


### Sharing single server connection (faye)

Example below shows how to extend `tabex` to share single `faye` connection
between all open tab. We will create `faye` instances it all tabs, but activate
only one in "master" tab.

User can close tab with active server connection. When this happens, new master
will be elected and new faye instance will be activated.

We also do "transparent" subscribe to faye channels when user subscribes with
tabex client. Since user can wish to do local broadcasts too, strict separation
required for "local" and "remote". We do it with addind "remote.*" prefix for
channels which require server subscribtions.

__Note.__ If you don't need cross-domain features - drop iframe-related options
and code.


In iframe:

```js
window.tabex.router({
  origin: [ '*://*.yourdomain.com', '*://yourdomain.com' ]
});
```


In client:

```js
// This one is for your application.
//
var live = window.tabex({
  iframe: 'https://shared.yourdomain.com/tabex_iframe.html'
});


// Faye will work via separate interface. Second interface instance will
// reuse the same router automatically. Always use separate interface for
// different bus build blocks to properly track message sources in filters.
//
// Note, you can attach faye in iframe, but it's more convenient to keep
// iframe source simple. `tabex` is isomorphic, and faye code location does
// not make sense.
//
var flive = window.tabex({
  iframe: 'https://shared.yourdomain.com/tabex_iframe.html'
});


var fayeClient = null;
var trackedChannels = {};


// Connect to messaging server when become master and
// kill connection if master changed
//
flive.on('!sys.master', function (data) {

  // If new master is in our tab - connect
  if (data.node_id === data.master_id) {
    if (!fayeClient) {
      fayeClient = new window.faye.Client('/faye-server');
    }
    return;
  }

  // If new master is in another tab - make sure to destroy zombie connection.
  if (fayeClient) {
    fayeClient.disconnect();
    fayeClient = null;
    trackedChannels = {};
  }
});


// If list of active channels changed - subscribe to new channels and
// remove outdated ones.
//
flive.on('!sys.channels.refresh', function (data) {

  if (!fayeClient) {
    return;
  }

  // Filter channels by prefix `local.` and system channels (starts with `!sys.`)
  var channels = data.channels.filter(function (channel) {
    return channel.indexOf('local.') !== 0 && channel.indexOf('!sys.') !== 0;
  });

  // Unsubscribe removed channels
  //
  Object.keys(trackedChannels).forEach(function (channel) {
    if (data.channels.indexOf(channel) === -1) {
      trackedChannels[channel].cancel();
      delete trackedChannels[channel];
    }
  });

  // Subscribe to new channels
  //
  data.channels.forEach(function (channel) {
    if (!trackedChannels.hasOwnProperty(channel)) {
      trackedChannels[channel] = fayeClient.subscribe('/' + channel.replace(/\./g, '!!'), function (message) {
        flive.emit(channel, message.data);
      });
    }
  });
});


// Resend events without prefix `local.` and prefix `!sys` to server, convert channel
// names to faye-compatible format: add '/' at start of channel name and replace '.' with '!!'
//
flive.filterIn(function (channel, message, callback) {
  if (fayeClient && channel.indexOf('local.') !== 0 && channel.indexOf('!sys.') !== 0) {
    fayeClient.publish('/' + channel.replace(/\./g, '!!'), message);
    return;
  }

  callback(channel, message);
});


// Convert channel name back from faye compatible format: remove '/'
// at start of channel name and replace '!!' with '.'
//
flive.filterOut(function (channel, message, callback) {
  if (channel[0] === '/') {
    callback(channel.slice(1).replace(/!!/g, '.'), message);
    return;
  }

  callback(channel, message);
});
```


## License

[MIT](https://github.com/nodeca/tabex/blob/master/LICENSE)
