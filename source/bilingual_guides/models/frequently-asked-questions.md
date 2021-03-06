英文原文：[http://emberjs.com/guides/models/frequently-asked-questions/](http://emberjs.com/guides/models/frequently-asked-questions/)

#### Should I use a query or a filter to search records?

#### 应该使用查询还是过滤条件来查询记录？

It depends on how many records you want to search and whether they have
been loaded into the store.

这取决于想要查询多少记录和记录是否已经加载到仓库。

_Queries_ are useful for doing searches of hundreds, thousands, or even
millions of records. You just hand the search options to your server,
and it is responsible for handing you back the list of records that
match. Because the response from the server includes the ID of all of
the records that matched, it doesn't matter if the store hadn't loaded
them previously; it sees that they are not in the cache and can request
the records by ID if necessary.

_查询_比较适合于搜索成百上千甚至百万级的记录。需要做的只是将搜索条件发送给服务器端，然后搜索匹配的记录的责任就交由服务器来处理。因为服务器返回的数据包含匹配的记录的ID，所以仓库即使之前没有加载这些记录也不会有影响；仓库发现这些记录并没有被缓存会通过ID来发送请求获取记录。

The downside of queries is that they do not live update, they are
slower, and they require that your server support the kind of queries
that you wish to perform.

使用查询的缺点是它们无法得到自动更新，效率较低，并且要求服务器端支持希望执行的查询类型。

Because the server decides which records match the query, not the store,
queries do not live update. If you want to update them, you must
manually call `reload()` and wait for the server to respond. If you
create a new record on the client, it will not show up in the results
until you both save the new record to the server and reload the query
results.

因为服务器端决定与查询匹配的记录，而不是仓库，查询无法自动更新。如果需要更新查询结果，需要手动的调用`reload()`，并等待服务器端返回。如果在客户端创建了一个新的记录，该记录不会自动在结果中显示，除非服务端已经包含了该记录，并且重新加载了查询结果。

Because the store must confer with your server to determine the results
of a query, it necessitates a network request. This can feel slow to
users, especially if they are on a slow connection or your server is
slow to respond. The typical speed of JavaScript web applications can
heighten the perceived slowness when the server must be consulted.

因为仓库必须让服务器端来决定查询的结果，因此需要一个网络请求。这会让用户觉得缓慢，特别是在用于与服务器间的连接速度较慢的情况下。当需要访问服务器的话，Javascript Web应用会明显的感觉到慢。

Lastly, performing queries requires collaboration between the store and
your server. By default, Ember Data will send the search options that
you pass as the body of an HTTP request to your server. If your server
does not support requests in this format, you will need to either change
your server to do so, or customize how queries are sent by creating a
custom adapter.

最后，执行查询需要仓库和服务器端协同工作。在默认情况下，Ember
Data将搜索条件作为HTTP请求的`body`发送到服务器端。如果服务器端并不支持查询的格式，那么需要修改服务器以便支持查询，或者通过自定义一个适配器来定义需要发送的查询。

_Filters_, on the other hand, perform a live search of all of the records
in the store's cache. As soon as a new record is loaded into the store,
the filter will check to see if the record matches, and if so, add it to
the array of search results. If that array is displayed in a template,
it will update automatically.

_过滤器_是对仓库中缓存的对象执行实时的查询。一旦新的记录被加载到仓库，过滤器会自动检查记录是否匹配条件，如果匹配，会将其加入到搜索结果的数组中。如果这个数组显示在模板中，模板也会自动更新。

Filters also take into account newly created records that have not been
saved, and records that have been modified but not yet saved. If you
want records to show up in search results as soon as they are created or
modified on the client, you should use a filter.

过滤器还会顾及新创建的还未保存的记录，以及被修改未保存的记录。如果希望记录在客户端一创建或者修改就在搜索结果中显示，那么就应该是用过滤器。

Keep in mind that records will not show up in a filter if the store
doesn't know about them. You can ensure that a record is in the store by
using the store's `push()` method.

请记住，如果仓库不知道记录的存在，记录不会包含在过滤器中。通过仓库的`push()`方法可以确保仓库知道记录的存在。

There is also a limit to how many records you can reasonably keep in
memory and search before you start hitting performance issues.

此外也有一个制约条件，就是有多少记录可以保存在内存中进行过滤，直到出现性能问题。

Finally, keep in mind that you can combine queries and filters to take
advantage of their respective strengths and weaknesses. Remember that
records returned by a query to the server are cached in the store. You
can use this fact to perform a filter, passing it a query that starts
matching records into the store, and a filter function that matches the 
same records.

最后，应该联合使用查询和过滤器来扬长避短。记住通过查询服务器返回的记录都是缓存在仓库中的。可以利用这一点来做过滤，通过查询仓库来将记录加载到仓库，然后使用一个过滤函数来匹配相同的记录。

This will offload searching all of the possible records to the server,
while still creating a live updating list that includes records created
and modified on the client.

这可以避免所有的记录都通过服务器来查询，同时也创建了一个能包含客户端创建和修改的记录的动态更新列表。

```js
App.PostsFavoritedRoute = Ember.Route.extend({
  model: function() {
    var store = this.store;

    // Create a filter for all favorited posts that will be displayed in
    // the template. Any favorited posts that are already in the store
    // will be displayed immediately;
    // Kick off a query to the server for all posts that
    // the user has favorited. As results from the query are
    // returned from the server, they will also begin to appear.
    return store.filter('post', { favorited: true }, function(post) {
      return post.get('isFavorited');
    });
  }
});
```

#### How do I inform Ember Data about new records created on the backend?

#### 如何将后台创建的记录通知Ember Data

When you request a record using Ember Data's `store.find` method, Ember
will automatically load the data into the store. This allows Ember to
avoid the latency of making a round trip to the backend next time
that record is requested. Additionally, loading a record into the
store will update any `RecordArray`s (e.g. the result of
`store.filter` or `store.all`) that should include that record. This
means any data bindings or computed properties that depend on the
`RecordArray` will automatically be synced to include the new or
updated record values.

当通过Ember
Data的`store.find`方法来请求一条记录时，Ember会自动将数据加载到`store`中。这样Ember就避免了在之后在发起一个请求来获取已经获取到的记录。此外，加载一条记录到`store`时，所有的包含该条记录的`RecordArray`都会被更新（例如`store.filter`或者`store.all`构造的）。这就意味着所有依赖与`RecordArray`的数据绑定或者计算属性都会在添加新的或者更新记录值的时候自动进行同步。

Some applications may want to add or update records in the store
without requesting the record via `store.find`. To accomplish this you
can use the `DS.Store`'s `push`, `pushPayload`, or `update`
methods. This is useful for web applications that have a channel
(such as [SSE](http://dev.w3.org/html5/eventsource/) or
[Web Sockets](http://www.w3.org/TR/2009/WD-websockets-20091222/)) to
notify it of new or updated records on the backend.

而一些应用可能希望能不通过`store.find`请求记录来添加或者更新`store`中得记录。为了实现这种需求，可以通过使用`DS.Store`的`push`，`pushPayload`，或者`update`方法。这对于那些有一个通道（例如[SSE](http://dev.w3.org/html5/eventsource/)或者[Web Socket](http://www.w3.org/TR/2009/WD-websockets-20091222/)）通知应用后台有新记录创建或者更新非常有用。

[push](http://emberjs.com/api/data/classes/DS.Store.html#method_push)
is the simplest way to load records to Ember Data's store. When using
`push` it is important to remember to deserialize the JSON object
before pushing it into the store. `push` only accepts one record at a
time. If you would like to load an array of records to the store you
can call
[pushMany](http://emberjs.com/api/data/classes/DS.Store.html#method_pushMany).

[push](http://emberjs.com/api/data/classes/DS.Store.html#method_push)是加载记录到Ember
Data的`store`的最简单方法。当使用`push`时，一定要记住将JSON对象推入`store`之前将其反序列化。`push`一次只接受一条记录。如果希望一次加载一组记录到`store`那么可以调用[pushMany](http://emberjs.com/api/data/classes/DS.Store.html#method_pushMany).

```js
socket.on('message', function (message) {
  var type = store.modelFor(message.model);
  var serializer = store.serializerFor(type.typeKey);
  var record = serializer.extractSingle(store, type, message.data);
  store.push(message.model, record);
});
```

[pushPayload](http://emberjs.com/api/data/classes/DS.Store.html#method_pushPayload)
is a convenience wrapper for `store#push` that will deserialize
payloads if the model's Serializer implements a `pushPayload`
method. It is important to note this method will not work with the
`JSONSerializer` because it does not implement a `pushPayload`
method.

[pushPayload](http://emberjs.com/api/data/classes/DS.Store.html#method_pushPayload)是一个`store#push`方法的便利封装，它将使用模型实现了`pushPayload`方法的序列化对象来反序列化有效载荷。需要注意的是这个方法并不能与`JSONSerializer`一同使用，因为其并没有实现`pushPayload`方法。

```js
socket.on('message', function (message) {
  store.pushPayload(message.model, message.data);
});
```

[update](http://emberjs.com/api/data/classes/DS.Store.html#method_update)
works likea `push` except it can handle partial attributes without
overwriting the existing record properties. This method is useful if
your web application only receives notifications of the changed
attributes on a model. Like `push` it is important to remember to
deserialize the JSON object before calling `update`.

[update](http://emberjs.com/api/data/classes/DS.Store.html#method_update)与`push`方法类似，不同的是其可以处理部分属性，而不需要覆盖整个记录的属性。这个方法对于只接收到记录改变的属性的通知的应用尤为有用。与`push`方法一样，`update`需要在调用之前将JSON对象反序列化。

```js
socket.on('message', function (message) {
  var hash = message.data;
  var type = store.modelFor(message.model);
  var fields = Ember.get(type, 'fields');
  fields.forEach(function(field) {
    var payloadField = Ember.String.underscore(field);
    if (field === payloadField) { return; }
      hash[field] = hash[payloadField];
      delete hash[payloadField];
  });
  store.push(message.model, hash);
});
```
