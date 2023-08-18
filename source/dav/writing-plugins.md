---
title: Writing Plugins
layout: default
versions:
    "1.8": /dav/1.8/writing-plugins/
    "2.x": /dav/2.x/writing-plugins/
    "3.x": /dav/writing-plugins/
thisversion: 3.x
---

sabre/dav uses an event system for plugin development. There's a big list of
supported events. Every single [plugin](/dav/plugins) uses these events to
add their new features and functionality.

An example: adding support for a new HTTP method
------------------------------------------------

Say if we wanted to implement a new HTTP method, named `BREW`, we'd use the
following syntax:

    use Sabre\HTTP\RequestInterface;
    use Sabre\HTTP\ResponseInterface;

    $server->on('method:BREW', function(RequestInterface $request, ResponseInterface $response) {

        $response->setStatus(200);
        $response->setBody('Coffee is under way!');

        return false;

    });


A few things to note in the previous example:

1. This specific event gets two arguments.
2. The arguments represent the HTTP request and HTTP response.
3. These two objects are documented [here](/http/api).
4. We _must_ return `false` because this tells the server 'we handled this method'.
5. Returning true will allow another plugin to handle the method.

Wrapping this in a plugin
-------------------------

If you have a few event handlers, and you would like to combine them into a
single class, you can turn this into a plugin class.

What follows, is an example that wraps the previous event handler into a
coffeepot plugin:

    use Sabre\DAV\Server;
    use Sabre\DAV\ServerPlugin;
    use Sabre\HTTP\RequestInterface;
    use Sabre\HTTP\ResponseInterface;

    class CoffeePlugin extends ServerPlugin {

        protected $server;

        function getName() {

            return 'coffee';

        }

        function initialize(Server $server){

            $this->server = $server;
            $server->on('method:BREW', [$this, 'brewHandler']);

        }

        function getHTTPMethods($uri) {

            return ['BREW'];

        }

        function brewHandler(RequestInterface $request, ResponseInterface $response) {

            $response->setStatus(200);
            $response->setBody('Coffee is under way!');
            return false;

        }

    }

Then if we want to use this plugin:

    $coffeePlugin = new CoffeePlugin();
    $server->addPlugin($coffeePlugin);

Events
------

The following events are supported in the server

### `method`

The `method` event is typically used to implement new methods.
An example can be seen above.

This event can also be used to override behavior of other methods. If you want
to for example _always_ want to override `GET` for certain urls, the following
works:

    use Sabre\HTTP\RequestInterface;
    use Sabre\HTTP\ResponseInterface;

    $server->on('method:GET', function(RequestInterface $request, ResponseInterface $response) {

        if ($request->getPath() !== 'contact.html') {
            return;
        }

        // Handles the 'contact.html' file.
        return false;

    }, 90);

Note that we specified `90` as the last argument. This is a priority number
which ensures that this event handler is called before the default. The
default priority is `100`.

You are not required to specify a method name in the event, it's also possible
to intercept this event for *any* http method:

    use Sabre\HTTP\RequestInterface;
    use Sabre\HTTP\ResponseInterface;

    $server->on('method', function(RequestInterface $request, ResponseInterface $response) {

        // Always triggered for any method.

    });


### `beforeMethod`

The `beforeMethod` handler is identical to the `method` handler. Using the
`beforeMethod` handler you could do some pre-processing to the request.

The authentication plugin uses `beforeMethod` to force users to be
authenticated.

The `beforeMethod` event can also be used with a method, e.g.:
`beforeMethod:POST`.

If you return false from your event handler the process is completely cancelled.

### `afterMethod`

After a method is completely handled, it's possible to do post-processing with
`afterMethod`.

The CalDAV plugin uses this event to transform iCalendar into jCal objects, if
this was requested by the client.

The `afterMethod` event can also be used with a method, e.g.:
`afterMethod:GET`.

If you return false from `afterMethod`, you prevent other `afterMethod` events
from processing. Generally this is a bad idea.

### `beforeCreateFile`

This event is triggered before new files are created. Using this event it is, for
example, possible to modify the new file before storage.

Example:

    function myHandler($path, &$data, \Sabre\DAV\ICollection $parent, &$modified) {

        // if the filename contains the word 'upper' we uppercase the entire
        // file. Normally this is a pretty bad idea.
        if (strpos($path, 'upper')) {
            if (is_resource($data)) {
                $data = stream_get_contents($data);
            }
            $data = strtoupper($data);
            $modified = true;
        }

    }

    $server->on('beforeCreateFile', 'myHandler');

A few notes:

* `$data` is passed by reference.
* `$data` may either be a string or a stream resource. Make sure you handle
  both scenarios.
* If you did modify `$data`, make sure you set `$modified` to true. sabre/dav
  needs to do different things depending on if you stored the data as-is, or
  if you made modifications.
* If you return `false` from your event handler, the file will not be stored
  and the process is effectively cancelled.

This is used by the CalDAV plugin to transform jCal to iCalendar.
This is also used by the TemporaryFileFilter plugin to intercept creation of
garbage files.

### `afterCreateFile`

This event is triggered, only if the creation of the file succeeded.

Example:

    function afterCreateFile($path, \Sabre\DAV\ICollection $parent) {

        // Do some logging here

    }

    $server->on('afterCreateFile','afterCreateFile');

### `beforeWriteContent`

This event is triggered before files are updated. Using this event, it is
possible to intercept these actions, validate content or block them.

Otherwise this works identical to `beforeCreateFile`, except this is only
called when updating existing files, and never when creating new files.

Example:

    function beforeWriteContent($path, \Sabre\DAV\IFile $node, &$data, &$modified) {

        file_put_contents('/tmp/davlog', $path . " just got updated!\n",FILE_APPEND);

    }

    $server->on('beforeWriteContent','beforeWriteContent');

All the caveats from `beforeCreateFile` apply here too. Note that the arguments
are slightly different for historical reasons.

### `afterWriteContent`

This event is triggered, only if updating of an existing file succeeded.

Example:

    function afterWriteContent($path, \Sabre\DAV\IFile $node) {

        // Do some logging here

    }

    $server->on('afterWriteContent','afterWriteContent');

### `beforeBind`

This event is triggered whenever a new node is about to be created in the tree.
This is for example triggered by `PUT`, `MKCOL`, `COPY` and `MOVE`.

Example:

    function beforeBind($path) {

        if (permitted()) return true;
        else return false;

    }

    $server->on('beforeBind','beforeBind');

### `beforeUnbind`

This event is triggered whenever a node is about to be deleted. If an entire
tree of nodes is deleted, the event will only trigger once, for the top-level
node. The event is triggered by for example `DELETE`, and `COPY` and `MOVE` in
case the target resource was being overwritten.

Example:

    function beforeUnbind($path) {

        if (permitted()) return true;
        else return false;

    }

    $server->on('beforeUnbind','beforeUnbind');

### `afterUnbind`

This event is triggered after a node is deleted. If an entire tree of nodes is
deleted, the event will only trigger once, for the top-level node. The event is
triggered by for example `DELETE`, and `COPY` and `MOVE` in case the target
resource was being overwritten.

Example:

    function afterUnbind($path) {

        // Some logging could go here	

    }

    $server->on('afterUnbind','afterUnbind');

### `beforeLock`

The beforeLock event is triggered right before a resource is locked.
Intercepting this event allows you to block a users' lock, or change information
regarding the lock, such as the lock owner or lock timeout.

Example:

    function beforeLock($path, \Sabre\DAV\Locks\LockInfo $lock) {

        if (!permitted()) return false;

    }

    $server->on('beforeLock','beforeLock');

### `beforeUnlock`

The beforeUnlock event is triggered right before a resource is unlocked.
Intercepting this event allows you to block the unlock.

Example:

    function beforeUnlock($path, \Sabre\DAV\Locks\LockInfo $lock) {

        if (!permitted()) return false;

    }

    $server->on('beforeUnlock','beforeUnlock');

### `propFind`

The `propFind` event is called during updating of properties. The event is
very versatile and used for a _lot_ of property-related operations.

Super basic example:

    use Sabre\DAV\PropFind;
    use Sabre\DAV\INode;

    function addProperty(PropFind $propfind, INode $node) {

        // This gives _every_ node a `{DAV:}displayname` of 'foo'.
        $propFind->set('{DAV:}displayname', 'foo');

    }

    $server->on('propFind', 'addProperty');

Lets zoom in a bit on our `addProperty` event handler. The `set` method
basically sets the `{DAV:}displayname` property to 'foo'.

We can change this method to only do this, if there wasn't already a
`{DAV:}displayname`. For that we use the `handle` method:

    function addProperty(PropFind $propfind, INode $node) {

        // This gives nodes without a displayname the value 'foo'.
        $propFind->handle('{DAV:}displayname', 'foo');

    }

The string `foo` is pretty simple... but there are also cases where the value
is some calculated value from the database. The simplest way to make sure that
you're only doing the heavy calculation when the property is actually
requested, you can put the value in a callback.

    function addProperty(PropFind $propfind, INode $node) {

        // This gives nodes without a displayname the value 'foo'.
        $propFind->handle('{DAV:}displayname', function() {
            return 'foo';
        });

    }

If a property was set earlier, but you want to block access to this property,
the easiest is to set its status to 403:

    use Sabre\DAV\PropFind;
    use Sabre\DAV\INode;

    function blockProperty(PropFind $propfind, INode $node) {

        // This blocks access to the displayname property.
        $propFind->set('{DAV:}displayname', null, 403);

    }

    $server->on('propFind', 'blockProperty');

Note that the PropFind object will automatically ignore any `handle` or set
that modifies properties that were never requested.

### `propPatch`

The `propPatch` event is called when updating properties on a node.

This event works similarly to `propFind`. Usually you should always use the
`handle` method for updating:

    use Sabre\DAV\PropPatch;
    use Sabre\DAV\INode;

    function saveDisplayName($path, PropPatch $propfind) {

        // This tells sabredav that we are responsible for handling storing
        // the `{DAV:}displayname` property.
        $propPatch->handle('{DAV:}displayname', function($value) {

            // Make sure you save $value somewhere.
            return true;

        });

    }

    $server->on('propPatch', 'saveDisplayName');

The callback you send to `handle` _must_ return `true` or `false`.

Take a look at the source for `Sabre\DAV\PropPatch` to learn more about its
features.

### `exception`

The server will trigger an 'exception' event whenever it's about to return an XML error document.

The actual exception is passed as the first argument.

### `validateTokens`

The `validateTokens` event is used internally by both the
[WebDAV Sync](/dav/sync) and the [Locking](/dav/locks) systems to work with
'tokens' that appear in the HTTP `If` header.

This header is quite advanced and allows you to do very complex conditional
requests.

By implementing this event you could potentially add new types of conditions
to any http request.

### `report`

If an HTTP `REPORT` method is invoked, all subscribers to this event get the
opportunity to handle a `REPORT` request.

Example:

    function myReport($reportName, $report, $uri) {

      // $reportName contains the report name in clark-notation.
      // For example:
      //
      // {DAV:}expand-properties

      return true;

    }

    $server->on('report','myReport');

The `$report` parameter contains a fully parsed XML report. Processed by
`sabre/xml`. Generally you will want to create a custom parser for these.
You can do so by adding it to the elementMap:

    $server->xml->elementMap['{http://example.org/ns}my-report'] = 'MyReportParserClass';

If you are reading this and are actually interested in this, drop us a line.
The documentation is extremely sparse here and we'd be happy to expand on
this.


### `calendarObjectChange`

This event is triggered by the CalDAV plugin, whenever a calendar object got
changed or created.

Example:

    use Sabre\HTTP\RequestInterface;
    use Sabre\HTTP\ResponseInterface;
    use Sabre\VObject\Document;

    function calendarObjectChange(RequestInterface $request, ResponseInterface $response, Document $calendar, $parentPath, &$modified, $isNew) {

    }

    $server->on('calendarObjectChange', 'calendarObjectChange');

It has quite a bit of arguments:

* `$request` - The HTTP request object.
* `$response` - The HTTP response object.
* `$calendar` - The fully parsed VCALENDAR object.
* `$parentPath` - The path to the object's parent calendar.
* `&$isModified` - Passed by reference. Set this to `true` to tell the system
   that you've modified `$calendar` so the modified version will end up
   getting stored, instead of the original. This also ensures that an `ETag`
   will not be sent back.
* `$isNew` - Whether the object was a newly created calendar object, or an
   update.


### `schedule`

This event is triggered for scheduling operations. A scheduling operation is
for example a invite to an event, a cancellation of an event or an
accept/decline response to an event.

This event is used by the [CalDAV scheduling plugin][13] to deliver these
messages to local calendar users, and it is used by the [iMip plugin][14],
to send invites via email.

Example:

    use Sabre\VObject\ITip\Message;

    function schedule(Message $iTipMessage) {

        // Send message via email
        $iTipMessage->scheduleStatus = '1.0;Delivered!';

    }

    $server->on('schedule', 'schedule');

Consult the source for `Sabre\VObject\ITip\Message` for more information.


### `afterMove`

This event is triggered after a successful `MOVE` request. This is used by
the system to copy WebDAV properties from the old to the new path, but could
be used for other purposes as well.

Example:

    function afterMove($sourcePath, $destinationPath) {

    }

    $server->on('afterMove', 'afterMove');


### `afterResponse`

This event is triggered after a HTTP response is sent back to the DAV client.
At this point it's no longer possible to influence the HTTP response, but it
could be used for logging or clean-up operations.

This would also be a good moment to call
[`fastcgi_finish_request`](https://php.net/manual/en/function.fastcgi-finish-request.php), if
you are on a fastcgi PHP sapi.

    use Sabre\HTTP\RequestInterface;
    use Sabre\HTTP\ResponseInterface;

    function afterResponse(RequestInterface $request, ResponseInterface $response) {

        fastcgi_finish_request();

    }

    $server->on('afterResponse', 'afterResponse');


Other examples
--------------

Check out `Sabre\DAV\CorePlugin`. This plugin is actually responsible for
_all_ core HTTP methods and most default features.

Priorities
----------

It may be needed to ensure that your event gets triggered early or late in the
process. This can be achieved by specifying the priority argument to the `on`
method.

This should be used with care. The Authentication plugin uses this for example
to make sure authentication happens before any other process. The default
priority number is 100. Lower numbers for priorities will be fired off earlier.

This is a comprehensive list of event handlers within sabredav that override
the default priority of 100. This can be useful to figure out the desired
timing of an event.

You may for instance want to make sure that your `beforeMethod` handler
happens after authentication, but before the acl system kicks in. In that case
`15` would be an appropriate priority.

| Event          | Implemented by               | Priority | Why ? |
| -------------- | ---------------------------- | -------- | ----- |
| `beforeBind`   | [ACL plugin][12]             |       20 | Block creation of new files |
| `beforeMethod` | [Auth plugin][7]             |       10 | To authenticate before anything else happens. |
| `beforeMethod` | [ACL plugin][12]             |       20 | Block certain requests if the user did not have permission. |
| `beforeUnbind` | [ACL plugin][12]             |       20 | Block deletion of files. |
| `beforeUnlock` | [ACL plugin][12]             |       20 | Block unlocking of files if the user is not the owner of the lock. |
| `method:GET`   | [ICSExport Plugin][1]        |       90 | To be before the `GET` handler of the browser plugin. |
| `method:GET`   | [VCFExport Plugin][6]        |       90 | To be before the `GET` handler of the browser plugin. |
| `method:GET`   | [Mount Plugin][10]           |       90 | To be before the `GET` handler of the browser plugin. |
| `method:GET`   | [Notifications plugin][15]   |       90 | To correctly emit the notifications xml responses |
| `method:GET`   | [Browser plugin][9]          |      200 | Only generated directory indexes if nothing else can handle a `GET` request. |
| `propFind`     | [ACL plugin][12]             |       20 | Block fetching properties if user did not have permission. |
| `propFind`     | Core server                  |      120 | Ask properties implementing `IProperties` to return properties. |
| `propFind`     | Core server                  |      200 | Map the `getctag` property to `{DAV:}sync-token`. |
| `propFind`     | [PropertyStorage plugin][11] |      130 | Only fetch from propertystorage after everything else has had a chance. |
| `propFind`     | [Subscriptions Plugin][4]    |      150 | Transform properties after fetching. |
| `propFind`     | [CardDAV plugin][5]          |      150 | Transform properties after fetching. |
| `propFind`     | [CalDAV Sharing Plugin][3]   |      150 | Transform properties after fetching. |
| `propFind`     | [GuessContentType Plugin][8] |      200 | Only set a property as the ultimate fallback. |
| `propPatch`    | [CalDAV Sharing Plugin][3]   |       40 | To handle a non-standard update of `{DAV:}resourcetype` that would otherwise be rejected. |
| `propPatch`    | Core Server                  |       90 | Block updating protected properties. |
| `propPatch`    | Core Server                  |      200 | Allow nodes implementing `IProperties` to update properties last. |
| `propPatch`    | [PropertyStorage plugin][11] |      300 | Only handle storing properties that no other subsystem understood. |
| `schedule`     | [iMip plugin][14]            |      120 | Attempt to send an email invite _after_ the internal delivery system. |

Other event-system features
--------------------------

sabre/dav's event system is based on [sabre/event](/event/). Read the
documentation on that page for additional event-related features. You can
emit custom events and remove subscribers as well as a few other things.

[1]: /dav/ics-export-plugin/
[2]: /dav/caldav/
[3]: /dav/caldav-sharing/
[4]: /dav/caldav-subscriptions/
[5]: /dav/carddav/
[6]: /dav/vcf-export-plugin/
[7]: /dav/authentication/
[8]: /dav/guesscontenttype/
[9]: /dav/browser-plugin/
[10]: /dav/davmount/
[11]: /dav/property-storage/
[12]: /dav/acl/
[13]: /dav/scheduling/
[14]: /dav/scheduling/
[15]: /dav/notifications/
