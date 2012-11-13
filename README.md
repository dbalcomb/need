# need

need is a script loader for node.js. It acts as both a replacement and companion to the built-in *require* function. It takes the fundamental ideas of *require* and adapts them to have a tighter integration with your node.js application.

*require* is a great tool for including external node modules in to an application. However it falls short on internal node modules. *need* aims to fix this without forcing major structural changes within an application.

## Basic Usage

### require

To use need, simply run the following command in the root of your application:

	npm install need
	
Then include the following line in your node.js application:

	var need = require('need');
	
*need* has its own *require* function that acts independantly to the built-in function. The following line demonstrates how at first glance *need* is almost identical to *require*. The function returns the exports from the required file just as *require* does.

	var exported = need('./a');
	
This function will first attempt to look for *./a.js* and if a matching file is not found then it will attempt to find a file at *./a/index.js*. It is possible to override this behavior by specifying an alternate fall-back path however should only be used if absolutely required.

### scope

./

### file extensions

You should not include the file extension when requiring a file. *need* will always search for the file name before attempting the file extension and so including a file extension will first search for a file with a double-extension. For example:

	# example.js
	module.exports = function() { console.log('This one') }
	
	# example.js.js
	module.exports = function() { console.log('Not this one') }
	
	# app.js
	need('example.js')()
	
	# console:
	Not this one
	
If the file *example.js.js* did not exist then it would fall back to *example.js*.

### need many

*need* offers the ability to require multiple files at the same time. For example:

	need(['./a', './b']);

The output here is a key-value pair of paths and exports. Multiple keys may point to the same exports. For example:

	# models/index.js
	module.exports = function() { console.log('Models') }
	
	# controllers.js
	module.exports = function() { console.log('Controllers') }
	
	# app.js
	var needs = need(['./models', './controllers']);
	
	needs.get('./models')();
	needs.get('./models/index')();
	
	needs.get('./controllers')();
	needs.get('./controllers/index')();
	
	# console:
	Models
	Models
	Controllers
	[ Error ]
	
### need anything

*need* allows the use of wildcards to require multiple files without being specific. For example:

	need([
		'a/b/', // require b/index.js
		'a/b',  // require b.js, fall back to b/index.js
		'a/*/,  // recursive require in sub-directories of a
		'a/*',  // recursive require in a
		'*'     // recursive require in current directory
	]);
	
If you require a single path that includes a wildcard then the output will always be a key-value pair. If there is any uncertainty in whether a path fill include a wildcard, for example when the path is generated from another function, then simply enclose the input as though it was an array.

### need function

If *need* encounters a function then it will evaluate the function and add the results to be required. For example:

	need(function() { return ['a/b', 'd/e'] });

### need object

if *need* encounters an object instead of a string or array then it will loop through the object and add the keys to be required. For example:

	need({
		'a/b': function(){},
		'd/e': function(){}
	});
	
Note that the functions here do not do anything. The purpose of these are explained below.

### need nesting

*need* allows you to nest the various different inputs that it accepts. For example:

	need([
		['a/b', 'c/d'],
		function() {
			return 'e/f'
		},
		{
			'g/h': function(){},
			'i/j': function(){}
		}
	]);

## Advanced Usage

### need namespaces

*need* allows you to define directory namespaces to split code up in to a more logical format. For example:

	# example.js
	var models = need.set('models', './models');

This namespace can then be accessed from anywhere in an application. The best use of this functionality is when trying to require a file that is several levels away or when requiring multiple files in the same directory. The idea is to make the code cleaner and easier to understand.

	# a/b/c/d/e/example.js
	var models = need.get('models');

The result of getting or setting will be a subclass of the *need* function with the root set to the specified directory in the set function. You can then use the returned function to require files. For example:

	models(['a/b', 'c/d']);
	
This code may not be very clear as there is no longer any mention of *need*. While the above still works it may be beneficial to use the following syntax:

	models.need(['a/b', 'c/d']);

Or alternatively:

	models.require(['a/b', 'c/d']);
	
Both functions work identically to that of the subclassed *need* function. If you wish, these functions are also exposed on the root *need* function so you can use the following syntax:

	need.need(['a/b', 'c/d']);
	need.require(['a/b', 'c/d']);
	
These namespaces can go as deep as you like and can only be accessed from the parent namespace. For example:

	# file1.js
	var models = need.set('models', './models'),
		people = models.set('people', 'people');
	
	# file2.js
	var people = need.get('people'); // Error
		
### callback parameter

*need* accepts a callback function as a second parameter. This function is passed three parameters.

	need(['a/b', 'c/d'], function(send, each, pair) {
		// ...
	});
	
#### send

This is one of the core features of *need* and is the reason why it was developed. The idea was to make it easy to send *app* and other variables to multiple files. *send* is a function that can be called to evaluate the exports from each required file. This only applies to exports that are functions:

	# a/b.js
	exports.something = function(input) {
		console.log('a/b says: ' + input);
	}
	
	# c/d.js
	module.exports = function(input) {
		console.log('c/d says: ' + input);
	}
	
	# app.js
	need(['a/b', 'c/d'], function(send, each, pair) {
		send('Hello World!');
	});
	
	# console:
	c/d says: Hello World!
	
In the above example only *c/d.js* actually exported a function via *module.exports*. *send* then made it easy to pass values on to other files.

Of course there may be situations where you do not wish all required files to be evaluated and sent parameters. In these cases it would be advised to use a seperate *need* function or alternatively use *each*. *send* is also exposed in a more general form on the *need* variable.

#### each

*each* is a function that allows you to perform actions on each file individually. *each* is also exposed in a more general form on the *need* variable.

	# a/b.js
	module.exports = function(input) {
		console.log('a/b says: ' + input);
	}
	
	# c/d.js
	module.exports = function(input) {
		console.log('c/d says: ' + input);
	}

	# app.js
	need(['a/b', 'c/d'], function(s, each, p) {
		each(function(send, match, key, val) {
			send(key);
		});
	});
	
	# console:
	a/b says: a/b
	c/d says: c/d

In the above example each of the required functions is sent its own path.
	
#### match

*match* is only available within *each*. It evaluates a given path to see if it matches another path. *match* is also exposed on the *need* variable.

	# a/b.js
	module.exports = function(input) {
		console.log('a/b says: ' + input);
	}
	
	# c/d.js
	module.exports = function(input) {
		console.log('c/d says: ' + input);
	}
	
	# app.js
	need(['a/b', 'c/d'], function(s, each, p) {
		each(function(send, match, key, val) {
			if (match(key, 'a/*')) {
				send('Matched!');
			} else {
				send('Not matched!');
			}
		});
	});
	
	# console:
	a/b says: Matched!
	c/d says: Not matched!
	
In the above example, *match* is used to check whether the required file is within the specified directory and sends different commands depending on whether the result is true or false.

#### pair

This is the key-value object that would normally be returned from the *need* function.

#### return

The callback function is allowed to return a value. By default if nothing is returned then *need* will return the key-value pair. If the callback returns a value then this value will be returned from *need*.

### config parameter

*need* accepts a config object as a second parameter.

	need(['a/b', 'c/d'], {
		ordered: false, // default true. Set to false if required files do not need to be synchronously required in a specific order.
		logging: true, // default false. Set to true to log warnings, errors
		recurse: false, // default true. Set to false to stop recursive require
		pattern: ['*Controller'], // only require files matching pattern
		ignores: ['a/b/*'], // ignore files matching pattern/path
		success: function(){} // The callback function
	});

### logging

.

## Something

…

modules/ index.js only, not models.js
	
	var application = need.set('application', './'),
		controllers = need.set('controllers', './controllers'),
		models      = need.set('models',      './models'),
		routes      = need.set('routes',      './routes'),
		config      = need.set('config',      './config');
		
		
	need('a/{dir}/*'); // a/b/c -> c: b
	need('a/{dir}/x'); // a/b/c/x -> x: b/c
	
sending

need.send
need.each
need.match
	
future improvements:
support loading modules
support loading json
watch directory for moving files
warning levels 0 == false == off, 1 == error, 2 == warn+error, 3 == warn+error+log