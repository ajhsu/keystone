Keystone
========

## About

Keystone is designed to be used in a web application built on Express and Mongoose.

Keystone provides:
*	A simple way to create an Express web app with custom routes, templates and models
*	Out of the box session management and authentication
*	Enhanced `models` with additional field types and functionality, building on those
	natively supported by Mongoose
*	An updates framework for managing data updates or initialisation
*	An auto-generated Admin UI based on the defined `models`
*	Integration with Coudinary for image uploading, storage and resizing
*	Integration with Mandrill for sending emails easily

Keystone is *not* designed to execute as a standalone application.


## Installation & Requirements

Specific congurations are required in your main application script for Keystone to work,
and assumptions are made in the code that this has been done correctly.

1.	Express ~3.2.6 and Mongoose ~3.6.13 must be included by your application, and must
	be `require`d *before* Keystone
	
2.	Keystone assumes that you have correctly configured, and successfully connected to,
	a Mongo database with Mongoose's default connection

3.	Connect-Flash ~0.1.1 must be included and configured in your Express app instance
	*before* you call `keystone.routes(app)`. This also requires the configuration of
	`express.session()` in your Express app.


### Usage

When you first `require` Keystone, it creates a single instance of itself. Do this somewhere
near the top of your app.js (or web.js, etc) file. Any subsequent `require('keystone')`
statements will return the same instance of Keystone.

You must provide a `mongoose` instance to Keystone's `connect` function before defining
any lists. `connect` returns `this` so you can do this in the `require` call.

Configuration variables can be set at any time, and include:

*	auth (callback function to authenticate a request, or 'native' to use native session management)
*	user model (list key for users if using native session management)
*	brand (label displayed in the top left of the UI)
*	cloudinary config `{cloud_name: '', api_key: '', api_secret: ''}` - alternatively set `process.env.CLOUDINARY_URL`
*	cloudinary prefix (prefix for all native tags added to uploaded images)
*	signout (href for the signout link in the top right of the UI)

Keystone can be locked down with the auth config. This must be a function matching the
express middleware pattern `fn(req,res,next)`. It will be called before any Keystone
routes are matched. If the user fails the validation check they should be redirected to
a signin or access-denied page implemented in the application.

`keystone.static(app)` adds Keystone's static route-handling middleware to the Express
app. It's a good idea to do this after your application's other static assets, before
any dynamic logic (e.g. cookie parsing, session authentication, body parsing, etc)

`keystone.routes(app);` adds Keystone's dynamic routes to the Express app router. This
can be done before or after your application's routes are defined, although if they come
after, you can explicitly lock down or replace Keystone routes with your own.

The `NODE_ENV` environment variable is used to control template caching and html formatting,
and should be set to `production` for production environments.


## Examples

### Application script (web.js) - basic

If you want, Keystone can take care of everything required to set up your express app and
then start it for you.

	var keystone = require('keystone');

	keystone.init({
		
		'name': 'My Project',
		'brand': 'Project Admin',
		
		'favicon': 'public/favicon.ico',
		'less': 'public',
		'static': 'public',
		
		'views': 'views',
		'view engine': 'jade',
		
		'auto update': true,
		'mongo': process.env.MONGOLAB_URI || ['localhost', 'my-project'],
		
		'auth': true,
		'user model': 'User',
		'cookie secret': '--- your secret ---',
		
		'emails': 'emails',
		'mandrill api key': '--- your api key ---',
		'email rules': { find: '/images/', replace: (keystone.get('env') != 'production') ? 'http://localhost:3000/images/' : 'http://www.team9.com.au/images/' },
		
		'cloudinary config': { cloud_name: '--- your cloud name ---', api_key: '--- your api key ---', api_secret: '--- your api secret ---' }
		
	});

	require('./models');

	keystone.set('routes', require('./routes'));
		
	keystone.start();


### Application script (web.js) - advanced

For full control over the express application, you can bind keystone by calling its
integration methods as part of the application configuration.

	var express = require('express'),
		app = express(),
		http = require('http'),
		path = require('path'),
		flash = require('connect-flash'),
		mongoose = require('mongoose'),
		keystone = require('keystone').connect(mongoose, app);

	require('./models');
	
	var session = require('./session');

	keystone.set('auth', session.keystoneAuth); // session.keystoneAuth is responsible for redirect visitors who shouldn't have access
	keystone.set('brand', 'Team 9'); // the brand is displayed in the top left hand corner of keystone

	// Initialise Express Application

	app.configure(function() {
		
		// Setup
		app.set('port', process.env.PORT || 3000);
		app.set('views', __dirname + '/views');
		app.set('view engine', 'jade');
		
		// Serve static assets
		app.use(express.compress());
		app.use(express.favicon(__dirname + '/public/favicon.ico'));
		app.use(require('less-middleware')({ src: __dirname + '/public' }));
		app.use(express.static(path.join(__dirname, 'public')));
		keystone.static(app);
		
		// Handle dynamic requests
		app.use(express.logger('dev'));
		app.use(express.bodyParser());
		app.use(express.methodOverride());
		app.use(express.cookieParser('-- your secret here --'));
		app.use(express.session());
		app.use(flash());
		app.use(session.persist);
		
		// Route requests
		app.use(app.router);

		// Handle 404s
		app.use(function(req, res, next) {
			res.status(404).send("Sorry, no page could be found at this address.");
		});

	});

	// Use Express error handler in the development environment
	app.configure('development', function() {
		app.use(express.errorHandler());
	});

	// Configure keystone routes
	keystone.routes(app);

	// Configure application routes
	require('./routes')(app);

	// Connect to the database and start the webserver
	mongoose.connect('localhost', '-- your database --');

	mongoose.connection.on('error', function() {
		console.error('Website failed to launch: mongo connection error', arguments);
	}).on('open', function() {
		http.createServer(app).listen(app.get('port'), function() {
			console.log("Website is ready on port " + app.get('port'));
		});
	});


### Routes.js

Keystone makes it easy to recursively import all routes in a directory, and provides
additional methods to the `request` and `response` express objects for APIs.

	var keystone = require('keystone'),
		importRoutes = keystone.importer(__dirname);

	// Load Routes
	var routes = {
		site: importRoutes('./site'),
		api: importRoutes('./api')
	};

	keystone.set('404', routes.site['404']);

	exports = module.exports = function(app) {

		// Site
		app.all('/', routes.site.index);
		
		// API
		app.all('/api*', keystone.initAPI);
		app.all('/api/contact', routes.api.contact);

	}

All route files are expected to export a single function like this:

	exports = module.exports = function(req, res) {
		res.render('site/index');
	}


## Thanks

A massive thanks to the people & projects that have been the foundation of 
Keystone or helped during its development, including

* Node.js, obviously :)
* ExpressJS (*the* webserver for node.js)
* MongoDB (for the great database)
* Mongoose (for the ODB that makes this easier)
* Bootstrap (for the great css framework, you guys make clean, responsive UI easy)
* Cloudinary (for the amazing image service)
* Google (for the maps)
* Heroku (for the servers)
* jQuery (of course)
* Underscore.js (for making javascript better)
* [Yusuke Kamiyamane](http://p.yusukekamiyamane.com/) (for some of the icons)


## License

(The MIT License)

Copyright (c) 2013 Jed Watson

Permission is hereby granted, free of charge, to any person obtaining
a copy of this software and associated documentation files (the
'Software'), to deal in the Software without restriction, including
without limitation the rights to use, copy, modify, merge, publish,
distribute, sublicense, and/or sell copies of the Software, and to
permit persons to whom the Software is furnished to do so, subject to
the following conditions:

The above copyright notice and this permission notice shall be
included in all copies or substantial portions of the Software.

THE SOFTWARE IS PROVIDED 'AS IS', WITHOUT WARRANTY OF ANY KIND,
EXPRESS OR IMPLIED, INCLUDING BUT NOT LIMITED TO THE WARRANTIES OF
MERCHANTABILITY, FITNESS FOR A PARTICULAR PURPOSE AND NONINFRINGEMENT.
IN NO EVENT SHALL THE AUTHORS OR COPYRIGHT HOLDERS BE LIABLE FOR ANY
CLAIM, DAMAGES OR OTHER LIABILITY, WHETHER IN AN ACTION OF CONTRACT,
TORT OR OTHERWISE, ARISING FROM, OUT OF OR IN CONNECTION WITH THE
SOFTWARE OR THE USE OR OTHER DEALINGS IN THE SOFTWARE.