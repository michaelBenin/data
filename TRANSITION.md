Ember Data 1.0 has a number of changes from Ember Data 0.13. The changes
are pretty significant, so you should assume that you will need to
dedicate some time to the upgrade.

# Elimination of Model.find and Model.createRecord

This change future-proofs the API for modules and makes it less
error-prone to clean up state between tests.

Ember Data 0.13:

```js
App.PostsRoute = Ember.Route.extend({
  model: function() {
    return App.Post.find()
  }
});

App.PostRoute = Ember.Route.extend({
  model: function(params) {
    return App.Post.find(params.post_id)
  }
});
```

Ember Data 1.0.beta.1:

```js
App.PostsRoute = Ember.Route.extend({
  model: function() {
    return this.store.find('post');
  }
});

App.PostRoute = Ember.Route.extend({
  model: function(params) {
    return this.store.find('post', params.post_id);
  }
});
```

Ember Data 0.13:

```js
App.NewPostRoute = Ember.Route.extend({
  model: function() {
    return App.Post.createRecord();
  }
});
```

Ember Data 1.0.beta.1:

```js
App.NewPostRoute = Ember.Route.extend({
  model: function() {
    return this.store.createRecord('post');
  }
});
```

Note that the store is automatically injected into all routes and
controllers. If you were looking up records in views or components, you
may wish to inject the store into those as well:

```js
// inject the store into all components
App.inject('component', 'store', 'store:main');
```

In general, looking up models directly in a component is an
anti-pattern, and you should prefer to pass in any model you need in the
template that included the component.

```js
{{my-popup person=user}}
```

# All Finders Return Promises

In Ember Data 0.13, finders returned empty models or record arrays that
would get filled in with the data once the server responded.

In Ember Data 1.0.beta.1, finders always return promises.

Ember Data 0.13:

```js
var person = App.Post.find(1);
person.one('didLoad', function() {
  // do something with the post
});
```

Ember Data 1.0.beta.1:

```js
this.store.find('post', 1).then(function(post) {
  // do something with post
});
```

If you need to wait for several `find` operations to return, you can use
`RSVP.all` to join them together.

Ember Data 0.13:

```js
var person1 = App.Person.find(1);
var person2 = App.Person.find(2);

var counter = 2;

person1.one('didLoad', bothLoaded);
person2.one('didLoad', bothLoaded);

function bothLoaded() {
  if (--counter === 0) {
    // work with person1 and person2
  }
}
```

Ember Data 1.0.beta.1:

```js
var promise1 = this.store.find('person', 1);
var promise2 = this.store.find('person', 2);

Ember.RSVP.all([ promise1, promise2 ]).then(function(people) {
  // people is an array of Person records
  // work with people here
});
```

Note that the promises returned from finders have special support for
Ember data-binding. They can be used in templates and templates will
automatically update once the promise has resolved.

# Transaction is Gone: Save Individual Records

You no longer need to push records into a transaction in order to save
them.

Ember Data 0.13:

```js
App.NewPostRoute = Ember.Route.extend({
  model: function() {
    return App.Post.createRecord();
  },

  setupController: function(controller, model) {
    var transaction = this.store.transaction()
    controller.set('transaction', this.store.transaction());
    transaction.add(model);
  },

  actions: {
    save: function() {
      this.controllerFor('newPost').get('transaction').commit();
    }
  }
});
```

Ember Data 1.0.beta.1:

```js
App.NewPostRoute = Ember.Route.extend({
  model: function() {
    return this.store.createRecord('post');
  }

  actions: {
    save: function() {
      this.modelFor('newPost').save();
    }
  }
});
```

If you want to batch up a bunch of records to save and save them all at
once, you can just put them in an Array and call `.invoke('save')` when
you're ready.

We plan to support batch saving with a single HTTP request through a
dedicated API in the future.

# Save Returns a Promise

Calling `save` on a record returns a promise that will be resolved when
the server returns successfully or rejected if the server returns with
an error code.

Ember Data 0.13:

```js
person.save();

person.one('didCommit', function() {
  // work with saved person
  // some bugs existed regarding newly created records and ID timing
});

person.one('didError', function() {
  // work with errored person
});
```

Ember Data 1.0.beta.1:

```js
person.save().then(function() {
  // work with saved person
  // newly created records are guaranteed to have IDs assigned
}, function() {
  // work with person that failed to save
});
```

This also means that retrying should be simple in Ember Data 1.0.beta.1:

```js
person.save().then(null, function() {
  return person.save();
}).then(function() {
  // person was retried once
});
```

You can of course implement whatever retry strategy you want
(incremental backoff, try N times, etc.) on top of the basic promise
primitive.

```js
function retry(promise, retryCallback, nTimes) {
  // if the promise fails,
  return promise.then(null, function(reason) {
    // if we haven't hit the retry limit
    if (nTimes-- > 0) {
      // retry again with the result of calling the retry callback
      // and the new retry limit
      return retry(retryCallback(), retryCallback, nTimes);
    }

    // otherwise, if we hit the retry limit, rethrow the error
    throw reason;
  });
}

// try to save the person up to 5 times 
retry(person.save(), function() {
  return person.save();
}, 5);
```

Because all async operations in Ember Data 1.0.beta.1 are promises, you
can combine them together using normal promise primitives.

```js
this.store.find('person').then(function(people) {
  people.forEach(function(person) {
    person.set('isPaidUp', true);
  });

  return Ember.RSVP.all(people.invoke('save'));
}).then(function(people) {
  // people we successfully saved
});
```

In Ember 0.13, the semantics for how to determine that a `find` had
finished loading were unclear and confusing.

# Adapters

The adapter API has undergone a significant change.

Existing adapters will likely need to be rebuilt. The good news is that
the new adapter API is somewhat simpler.

## Promises

Adapter hooks no longer call directly into the store to notify the store
that the backend has returned. Instead, all adapter hooks return a
promise that the store will automatically handle as appropriate for the
type of query that kicked off the request.

Ember Data 0.13:

```js
App.MyAdapter = DS.Adapter.extend({
  find: function(store, type, id) {
    $.getJSON("/" + this.pluralize(type) + "/" + id, function(payload) {
      store.load(payload);
    }
  }
});
```

Ember Data 1.0.beta.1:

```js
App.MyAdapter = DS.Adapter.extend({
  find: function(store, type, id) {
    return $.getJSON("/" + this.pluralize(type) + "/" + id);
  }
});
```

The short version is that adapter hooks don't need to know how to update
the store, records, or record arrays once the server has returned a
payload. They just need to return a promise, and Ember Data will take
care of interpreting it correctly.

## Per Type Adapters

Ember Data 0.13:

```js
App.Post = DS.Model.extend({
  // ...
});

App.PostAdapter = DS.RESTAdapter.extend({
  // ...
});

App.Store = DS.Store.extend();

App.Store.registerAdapter(App.Post, App.PostAdapter);
```

Ember Data 1.0.beta.1:

```js
App.Post = DS.Model.extend({
  // ...
});

App.PostAdapter = DS.RESTAdapter.extend({
  // ...
});

// No store is needed
// Adapters are wired up by name
```

## Per Type Serializers

Ember Data 0.13:

```js
App.Post = DS.Model.extend({
  // ...
});

App.PostSerializer = DS.JSONSerializer.extend({
  // ...
});

App.PostAdapter = DS.RESTAdapter.extend({
  serializer: 'App.PostSerializer',
});

App.Store = DS.Store.extend();

App.Store.registerAdapter(App.Post, App.PostAdapter);
```

Ember Data 1.0.beta.1:

```js
App.Post = DS.Model.extend({
  // ...
});

App.PostSerializer = DS.JSONSerializer.extend({
  // ...
});

// That's it. Everything is wired up by naming
```

## REST Adapter and Serializer Configuration

There are significant improvements and simplifications to configuring
the REST Adapter and Serializer.

The short version is that once an Ajax request has completed, the
resulting payload is sent through the following hooks:

1. The payload is sent to `extractSingle` if the original request was
   for a single record (like `find`/`save`) or `extractArray` if the
   original request was for an Array of records (like
   `findAll`/`findQuery`)
2. The default behavior of those methods is to pull apart the top-level
   of the payload into multiple smaller records.
3. Each of those smaller records is sent to `normalize`, which can do
   normalization a record at a time.
4. Finally, specific types of records can be specially normalized.

Here's how it works.

Imagine you start with a payload like this for request for Post 1.

```js
{
  "id": "1",
  "title": "Rails is omakase",
  "_links": {
    "user": [{
      "href": "/people/dhh"
    }]
  },
  "_embedded": {
    "comment": [{
      "ID_": "1",
      "CMT_BODY": "Rails is unagi"
    }, {
      "ID_": "2",
      "CMT_BODY": "Omakase O_o"
    }]
  }
}
```

Ember Data 1.0.beta.1 is expecting a payload like this:

```js
{
  "post": {
    "id": 1
    "title": "Rails is omakase",
    "comments": ["1", "2"],
    "_links": {
      "user": "/people/dhh"
    },
  },

  "comments": [{
    "id": "1",
    "body": "Rails is unagi"
  }, {
    "id": "2",
    "body": "Omakase O_o"
  }]
}
```

The simplest way to approach this would be to override the
`extractSingle` method and do all the munging in one place. This would
totally work.

```js
App.PostSerializer = DS.JSONSerializer.extend({
  extractSingle: function(store, type, payload, id, requestType) {
    var post = {}, commentIds = [];

    post.id = payload.id;
    post.title = payload.title;
    post._links = { user: payload._links.user.href };

    // Leave the original un-normalized comments alone, but put them
    // in the right place in the payload. We'll normalize the comments
    // below in `normalizeHash`
    var comments = payload._embedded.comments.map(function(comment) {
      commentIds.push(comment.ID_);
      return { id: comment.ID_, body: comment.CMT_BODY };
    });

    post.comments = commentIds;

    return { post: post, comments: comments };
  }
});
```

However, Ember Data also allows you to break things into into more
sensible chunks. This is especially useful if you have a lot of custom
normalization to do on different pieces of the JSON.

```js
App.PostSerializer = DS.JSONSerializer.extend({
  extractSingle: function(store, type, payload, id, requestType) {
    var post = {}, commentIds = [];

    post.id = payload.id;
    post.title = payload.title;
    post._links = { user: payload._links.user.href };

    // Leave the original un-normalized comments alone, but put them
    // in the right place in the payload. We'll normalize the comments
    // below in `normalizeHash`
    var comments = payload._embedded.comments;
    post.comments = comments.mapProperty('ID_');

    return { post: post, comments: comments };
  },

  normalizeHash: {
    comments: function(hash) {
      return { id: hash.ID_, body: hash.CMT_BODY };
    }
  }
});
```

You can also implement the `normalize` method to do generic things like
camelize all of the keys in a hash. The `normalize` method gets called
once for every sub-hash.

Imagine that we have a payload like this, which is totally fine except
for the use of underscores rather than camelization and `"_id"` instead
of `"id"`:

```js
{
  "post": {
    "_id": "1",
    "author_name": "DHH",
    "title": "Rails is omakase",
    "comments": [ "1", "2" ]  
  },
  "comments": [{
    "_id": "1",
    "author_name": "unagisan",
    "like_count": 12,
    "body": "Unagi!"
  }, {
    "_id": "2",
    "author_name": "tlo",
    "like_count": 100,
    "body": "Omakase is delicious"
  }]
}
```

We don't need to implement `extractSingle`, because the top-level is
already organized perfectly.

```js
App.PostSerializer, = DS.Serializer.extend({
  // This method will be called 3 times: once for the post, and once
  // for each of the comments
  normalize: function(type, property, hash) {
    // property will be "post" for the post and "comments" for the
    // comments (the name in the payload)

    // normalize the `_id`
    var json = { id: hash._id };
    delete hash._id;

    // normalize the underscored properties
    for (var prop in hash) {
      json[prop.camelize()] = hash[prop]; 
    }

    // delegate to any type-specific normalizations
    return this._super(type, property, json);
  }
});
```

So to sum up, you should:

* use `extractSingle` and `extractArray` when the top-level of your
  payload is organized differently than Ember Data expects
* use `normalize` to normalize sub-hashes if all sub-hashes in the
  payload can be normalized in the same way.
* use `normalizeHash` to normalize specific sub-hashes.
* make sure to call super if you override `extractSingle`,
  `extractArray` or `normalize` so the rest of the chain will get
  called.
* beta.1 expects `comments` key now instead of `comments_ids`.
  This is likely to be configurable in beta.2.

### Embedded Records

Explicit support for embedded records is gone for now.

You can handle embedded records yourself by implementing `extractSingle`
and reorganizing the payload.

Consider this payload:

```js
{
  "post": {
    "id": "1",
    "title": "Rails is omakase",
    "comments": [{
      "id": "1",
      "body": "I like omakase"
    }, {
      "id": "2",
      "body": "I prefer not to rely on elitist chefs"
    }]
  }
}
```

You could handle embedded records like this:

```js
App.PostSerializer = DS.Serializer.extend({
  extractSingle: function(store, type, payload, id, requestType) {
    var comments = payload.post.comments,
        commentIds = comments.mapProperty('id');

    payload.comments = comments;
    payload.post.comments = commentIds;

    return this._super.apply(this, arguments);
  }
});
```
