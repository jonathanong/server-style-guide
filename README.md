
# Server Style Guide

This is a style guide for creating web servers,
specifically with [koa](http://koajs.com).
This is an extension of the [JS Style Guide](https://github.com/jonathanong/js-style-guide)
as well as [Module Style Guide](https://github.com/jonathanong/module-style-guide).

## Modularize!

Don't just modularize your code,
modularize your problem.
Break your problem into tiny pieces and solve each piece as simply as possible.

### Model as a module

I'd recommend moving your entire "model",
i.e. all your business logic,
into a completely separate module with its own tests.

```js
var User = require('model').User;

app.get('/users/:userid', function () {
  this.user = yield User.get(this.session.userid);
});
```

This completely separates your model from your app,
and allows you to differentiate between business logic errors and app errors.

### Microservices!

If you have a CPU intensive process or complex modules that requires a non-trivial web server,
split your app into several microservices.
There are many benefits in doing this:

- Memory leaks become much easier to isolate.
- Developers do not need to install all the dependencies required to start your app.
- Developing on multiple environments is easier.
- Scale each microservice independently..

## File structure

This is how I split up files in my apps.
Remember, this is written specifically for Koa.
The main idea is that you define folders depending on the framework you use.
This makes it easy for developers to know where things are located.

A folder called `controller` is not specific enough.
If you modularize your model and your view from your app,
your entire app is essentially a controller.
Be more specific!

### `app.js`

First, initialize your app.
This allows you to `import` your app in other files.
This should be minimal and essentially only include configurations.

```js
import Framework from 'my-favorite-framework';

var app = new Framework();

// export the app
export default app;

// do settings, maybe from another file
app.set('key', 'value');
```

Now, almost every file can mutate `app` like so:

```js
import app from '../app';

app.get('/', async function () {
  this.render('home');
});
```

Writing code like this is absolutely horrendous:

```js
module.exports = function (app) {
  app.get('/', async function () {
    this.render('home');
  });
}
```

If you have a function that is used by multiple routes,
define it in `middleware/` and require it in the routes.
Don't wrap it in a function above.

### `routes/`

Routes are where you define all the routes and their perspective controllers.
These route definitions should be minimal and high level.

- Any validations and business logic should be refactored out of this folder.
- There should be at most 1 nested folders within this folder.
- Name your files well.

```js
import app from '../app';
import { User, Post } from 'model';
import parse from 'body-parser';

app.post('/posts', async =>
  // grab the user
  var user = await User.get(this.session.userid);
  // authenticate
  this.assert(user, 401, 'You must be logged in to create a post.');
  // authorize
  this.assert(user.permissions.post, 403, 'You are not allowed to create a post.');
  // parse the request body
  var body = await parse(this, '100kb');
  body.creator = user;
  // actually execute the business logic
  this.response.body = await Post.create(body);
)
```

A possible option for you is to move all those asserts into your model.
I like it in the controller as it makes it easier for others to understand.

Note that all the routes and their middleware are written in the same file.
Splitting these up only make your life more difficult.
Each middleware pertaining to a route should be only a few lines.
When viewing each middleware, I should be able to understand what it does at a high level.

### `middleware/`

The middleware folder is for any "global" middleware or middleware
that's used in multiple routes.
For example:

- Routers
- Checking whether the HTTP protocol is supported
- Checking whether SSL is being used
- Rate limiting
- Sessions
- Compression
- Logging

You should keep this folder to a minimal.
Many features that you think are middleware should be in fact functions.

- Body parsing
- CSRF validation
- Authorization (anything with database calls, really)

### Context Helpers

A feature of `koa` is the ability to add methods and properties to `this`.
It's not really documented, however, as we don't want people abusing it.
But if you're writing your own app, go for it!

I generally create 3 files or folders:

- `context.js` - for `this`
- `request.js` - for `this.request`
- `response.js` - for `this.response`

An example of using this is to create a function to get the user (as opposed to middleware):

```js
import app from 'app';
import { User } from 'model';

app.context.user = async function () {
  if (this._user !== undefined) return this._user;
  return this._user = await User.get(this.session.userid) || false;
}
```

This allows to do the following in any middleware:

```js
app.get('/', async function () {
  var user = await this.user();
});
```

Saves you from doing a lot of imports.

### `index.js`

`index.js` is where you import all the modules.
Because all the modules mutate the app,
__ordering is important!__

```js
import app from 'app';
export default app;

// import middleware
import * as middleware from 'middleware';
app.use(middleware.session);

// import routes
import 'routes/users';
import 'routes/posts';
```

### `templates/`

These folders correspond to server-side templates.
As we move towards client-side templating, this folder will eventually go away.

If a template is route specific, it's a good idea to put it in the same folder as the route.

```
routes/
  users/
    index.js
    index.jade
```

This makes it easier to understand how each route is built.
There's no need to make other developers search for corresponding templates.

## Where do I put...

### Validations

Put your validations in the model and test your model with bad values!
This makes your app more robust in general.
Your app developers should deliver data to your model the way the model wants.

Remember, the web server is just another interface to your model.
You should be able to make executables, chron jobs, and external modules with your model,
and it's preferable to always have validation.

### Authorization

This completely depends on your business logic.

For example, if a user is __always__ required to create a post,
then you may want to put authorization in your business logic.
But if you might programatically create a few posts without a user,
then perhaps you would move authorization to your app.
