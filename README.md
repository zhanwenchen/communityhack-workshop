# Ship a to-do list web app using PostgreSQL and NodeJS

A basic single page application built with Node, Express, Angular, and PostgreSQL.

This tutorial is based on Michael Herman's [blog post](http://mherman.org/blog/2015/02/12/postgresql-and-nodejs). Improved the
pedagogical coherence to suit beginners and programmers new to JS.

## 1. Setup

1. [Install Node](https://nodejs.org/en/download/)
1. [Install PostgreSQL](https://www.postgresql.org/download/)
1. [Install Git](https://git-scm.com/downloads)
1. Get the html and css files by cloning the repo `git clone https://github.com/zhanwenchen/communityhack-workshop`

## 2. Create the todo database

1. Start your PostgreSQL server app/service
1. Create a database called "todo". To do this,

    1. Mac:

      Bring up a terminal/command prompt with command &#8984; + space to bring up Spotlight Search

      Search for "terminal" or "te"/"ter"/"term" etc and hit return

      Enter `psql`. If there's an error, edit your ~/.bash_profile file by

      ```sh
      vim ~/.bash_profile
      ```

      Hit `G` and `a` to go to the bottom of the file and switch to Insert Mode.

      Copy and paste

      ```sh
      PATH="/Applications/Postgres.app/Contents/Versions/latest/bin:$PATH"
      ```

      Quit vim by `Esc` followed by `:wq`.

      Refresh your terminal by `source ~/.bash_profile`.

      You are now in the PostgreSQL command line interface (CLI).

      Type `create database todo;` (include the semicolon) then hit return.

      You just created a database called 'todo'

      Exit the PostgreSQL CLI by entering `\q`


## 2. Initialize project
1. Under project folder, start a new npm project by

```sh
npm init
```

Hit enter all the way through. This creates a package.json file that tells Node what to do when deploying this project. Now go to the generated package.json and get rid of the line

```json
"main": "index.js",
```

Then add this line to the 'scripts' key between the curly brackets:

```json
"start": "node ./bin/www",
```

So that the package.json file becomes

```json
{
  "name": "communityhack-workshop",
  "version": "1.0.0",
  "description": "A basic single page application built with Node, Express, Angular, and PostgreSQL.",
  "version": "0.0.0",
  "scripts": {
    "start": "node ./bin/www"
  },
  "author": "",
  "license": "ISC"
}
```

## 3. Create the database tables

1. Start a new file under project folder "server/models" called "database.js."
1. Install the "pg" Node Module to enable JavaScript-PostgreSQL interaction

  ```sh
  npm install pg --save
  ```

  The "--save" flag saves the "pg" module as a "dependency" written into package.json. This is useful when you or anyone else clone the module and use `npm install` before deploying on a different machine.

1. Back to the newly created "database.js", add the following lines.

  ```javascript
  const pg = require('pg');
  const connectionString = process.env.DATABASE_URL || 'postgres://localhost:5432/todo';
  ```

  The above two lines imports the "pg" module and creates a static variable for the address of your database. If your system environment has a database address specified already (only applies when you are deploying to a server, which you aren't), that's used instead of your local PostgreSQL's default address: 'localhost' at port 5432. Then add

  ```javascript
  const client = new pg.Client(connectionString);
  client.connect();
  ```

  to the same file. Here a new PostgreSQL "client" is created and connected to your database address. But how would we create our todo items table? Insert

  ```javascript
  const query = client.query(
    'CREATE TABLE items(id SERIAL PRIMARY KEY, text VARCHAR(40) not null, complete BOOLEAN)');
  ```

  This is a SQL statement that creates a table called "items" that has an automatic "id" field, a "text" string field, and a "complete" boolean field. With a query in process, we need to tell the compiler what to do after the query finishes.

  ```javascript
  query.on('end', () => { client.end(); });
  ```

  This statement is very JavaScript in that it uses an asynchronous event listener and passes in a "callback" function - a function that's ready to be called when something else happens. In this case, the client completes with a mystical end() function when the event 'end' happens. This is "pg" specific and we won't dig deeper into it.

  To summarize, we should now have a file called "database.js" that contains:

  ```javascript
  // server/models/database.js
  // Database schema for creating "items" table

  const pg = require('pg');
  const connectionString = process.env.DATABASE_URL || 'postgres://localhost:5432/todo';

  const client = new pg.Client(connectionString);
  client.connect();
  const query = client.query(
    'CREATE TABLE items(id SERIAL PRIMARY KEY, text VARCHAR(40) not null, complete BOOLEAN)');
  query.on('end', () => { client.end(); });
  ```

1. Now that we have the table creation file, we use it by calling it in the terminal using the command line JavaScript engine. In your terminal, enter
```sh
node server/models/database.js
```

(assuming your terminal window is under the project root directory. Adjust accordingly).

## 4. Create the backend server

1. Install the Express router.

  The meat of the Node.js backend is the Express server. To install it, like you did "pg", do

  ```sh
  npm install express --save
  ```

2. Create the server init script

  Under the "server" folder, create a "binary" folder and put this new file in it and name it "www" (NOT www.js).

  ```javascript
  // server/bin/www

  #!/usr/bin/env node

  /**
   * Module dependencies.
   */

  var app = require('../server/app');
  var debug = require('debug')('node-postgres-todo:server');
  var http = require('http');

  /**
   * Get port from environment and store in Express.
   */

  var port = normalizePort(process.env.PORT || '3000');
  app.set('port', port);

  /**
   * Create HTTP server.
   */

  var server = http.createServer(app);

  /**
   * Listen on provided port, on all network interfaces.
   */

  server.listen(port);
  server.on('error', onError);
  server.on('listening', onListening);

  /**
   * Normalize a port into a number, string, or false.
   */

  function normalizePort(val) {
    var port = parseInt(val, 10);

    if (isNaN(port)) {
      // named pipe
      return val;
    }

    if (port >= 0) {
      // port number
      return port;
    }

    return false;
  }

  /**
   * Event listener for HTTP server "error" event.
   */

  function onError(error) {
    if (error.syscall !== 'listen') {
      throw error;
    }

    var bind = typeof port === 'string'
      ? 'Pipe ' + port
      : 'Port ' + port;

    // handle specific listen errors with friendly messages
    switch (error.code) {
      case 'EACCES':
        console.error(bind + ' requires elevated privileges');
        process.exit(1);
        break;
      case 'EADDRINUSE':
        console.error(bind + ' is already in use');
        process.exit(1);
        break;
      default:
        throw error;
    }
  }

  /**
   * Event listener for HTTP server "listening" event.
   */

  function onListening() {
    var addr = server.address();
    var bind = typeof addr === 'string'
      ? 'pipe ' + addr
      : 'port ' + addr.port;
    debug('Listening on ' + bind);
  }
  ```

  This is the entry point of the project. This script officially starts the backend server at a selected port (here it is 3000, but it can be anything: 5000, 8000, 8080, etc). You can reuse this in your future projects, as long as `var app = require('../server/app');` points to the "app.js" file you want.

1. Create the router init script

  After taking care of the low-level server init, there's one more init: the router. Immediately under the server directory, copy the code below and paste into a new file called "app.js".

  ```javascript
  const express = require('express');
  const path = require('path');
  const cookieParser = require('cookie-parser');
  const bodyParser = require('body-parser');

  const routes = require('./routes/index');
  // var users = require('./routes/users');

  const app = express();

  // view engine setup
  // app.set('views', path.join(__dirname, 'views'));
  // app.set('view engine', 'html');

  // uncomment after placing your favicon in /public
  //app.use(favicon(path.join(__dirname, 'public', 'favicon.ico')));
  app.use(logger('dev'));
  app.use(bodyParser.json());
  app.use(bodyParser.urlencoded({ extended: false }));
  app.use(cookieParser());
  app.use(express.static(path.join(__dirname, 'client')));

  app.use('/', routes);
  // app.use('/users', users);

  // catch 404 and forward to error handler
  app.use((req, res, next) => {
    var err = new Error('Not Found');
    err.status = 404;
    next(err);
  });

  // error handlers

  // development error handler
  // will print stacktrace
  if (app.get('env') === 'development') {
    app.use((err, req, res, next) => {
      res.status(err.status || 500);
      res.json({
        message: err.message,
        error: err
      });
    });
  }

  // production error handler
  // no stacktraces leaked to user
  app.use((err, req, res, next) => {
    res.status(err.status || 500);
    res.json({
      message: err.message,
      error: {}
    });
  });


  module.exports = app;
  ```

  This file tells the express router where to look when serving static files (such as HTML, CSS, and client-side JavaScript files) and how to process routes, which are urls such as /about, /todos, etc. We are not teaching you static web dev such as /about.html, which is useless in a modern setting.

  But we still have some admin stuff to take care of. Install miscellaneous Node modules required in this script, such as body-parser and cookie-parser, etc. You can do them all in one line by

  ```sh
  npm install cookie-parser body-parser --save
  ```

1. Create routes!

  Now comes the interesting stuff, which is the the routes.js file required by the last script. Create a new folder called "routes," under which a new file called "index.js". First, importing and setting up:

  ```javascript
  const express = require('express');
  const router = express.Router();
  const pg = require('pg');
  const path = require('path');
  const connectionString = process.env.DATABASE_URL || 'postgres://localhost:5432/todo';
  ```

  most of which we have seen before. Now let's define root page ('/') behavior:

  ```javascript
  router.get('/', (req, res, next) => {
    res.sendFile(path.join(
      __dirname, '..', '..', 'client', 'views', 'index.html'));
  });
  ```

  What the above says is that when the user requests '/' (root, such as amazon.com, facebook.com which are really amazon.com/ and facebook.com/), we serve them the content of client/views/index.html, an HTML file provided for you.

  But for a to-do list, we want to be able to Create (POST), Read (GET), Update (PUT), and Delete (DELETE) a to-do item. Thus comes the "CRUD" acronym for basic database application requirements, and POST, GET, PUT, and DELETE are the standard HTTP request methods. Networking 101 here.

  So let's define these behavior in the router file. First, creating a to-do:

  ```javascript
  router.post('/api/v1/todos', (req, res, next) => {
    const results = [];
    // Grab data from http request
    const data = {text: req.body.text, complete: false};
    // Get a Postgres client from the connection pool
    pg.connect(connectionString, (err, client, done) => {
      // Handle connection errors
      if(err) {
        done();
        console.log(err);
        return res.status(500).json({success: false, data: err});
      }
      // SQL Query > Insert Data
      client.query('INSERT INTO items(text, complete) values($1, $2)',
      [data.text, data.complete]);
      // SQL Query > Select Data
      const query = client.query('SELECT * FROM items ORDER BY id ASC');
      // Stream results back one row at a time
      query.on('row', (row) => {
        results.push(row);
      });
      // After all data is returned, close connection and return results
      query.on('end', () => {
        done();
        return res.json(results);
      });
    });
  });
  ```

  Here we INSERT a row of data into the items table, and query all todos after the insertion. For DELETE, we have

  ```javascript
  router.delete('/api/v1/todos/:todo_id', (req, res, next) => {
    const results = [];
    // Grab data from the URL parameters
    const id = req.params.todo_id;
    // Get a Postgres client from the connection pool
    pg.connect(connectionString, (err, client, done) => {
      // Handle connection errors
      if(err) {
        done();
        console.log(err);
        return res.status(500).json({success: false, data: err});
      }
      // SQL Query > Delete Data
      client.query('DELETE FROM items WHERE id=($1)', [todo_id]);
      // SQL Query > Select Data
      var query = client.query('SELECT * FROM items ORDER BY id ASC');
      // Stream results back one row at a time
      query.on('row', (row) => {
        results.push(row);
      });
      // After all data is returned, close connection and return results
      query.on('end', () => {
        done();
        return res.json(results);
      });
    });
  });
  ```

  Similar to POST, the DELETE method deletes a row and then refreshes the query table. Notice the ':todo_id' in the url, which becomes a variable. What's left are GET and PUT:

  ```javascript
  router.get('/api/v1/todos', (req, res, next) => {
    const results = [];
    // Get a Postgres client from the connection pool
    pg.connect(connectionString, (err, client, done) => {
      // Handle connection errors
      if(err) {
        done();
        console.log(err);
        return res.status(500).json({success: false, data: err});
      }
      // SQL Query > Select Data
      const query = client.query('SELECT * FROM items ORDER BY id ASC;');
      // Stream results back one row at a time
      query.on('row', (row) => {
        results.push(row);
      });
      // After all data is returned, close connection and return results
      query.on('end', () => {
        done();
        return res.json(results);
      });
    });
  });

  router.put('/api/v1/todos/:todo_id', (req, res, next) => {
    const results = [];
    // Grab data from the URL parameters
    const id = req.params.todo_id;
    // Grab data from http request
    const data = {text: req.body.text, complete: req.body.complete};
    // Get a Postgres client from the connection pool
    pg.connect(connectionString, (err, client, done) => {
      // Handle connection errors
      if(err) {
        done();
        console.log(err);
        return res.status(500).json({success: false, data: err});
      }
      // SQL Query > Update Data
      client.query('UPDATE items SET text=($1), complete=($2) WHERE id=($3)',
      [data.text, data.complete, id]);
      // SQL Query > Select Data
      const query = client.query("SELECT * FROM items ORDER BY id ASC");
      // Stream results back one row at a time
      query.on('row', (row) => {
        results.push(row);
      });
      // After all data is returned, close connection and return results
      query.on('end', function() {
        done();
        return res.json(results);
      });
    });
  });
  ```

  Lastly, because we are calling this script from another script, we need to export our definitions in order to call 'require()' on it.

  ```javascript
  module.exports = router;
  ```

## 5. Create the front-end JavaScript functions

1. Under the "client" folder, create a "javascripts" folder and a new file called "app.js".

  ```javascript
  angular.module('nodeTodo', [])
  .controller('mainController', ($scope, $http) => {
    $scope.formData = {};
    $scope.todoData = {};
    // Get all todos
    $http.get('/api/v1/todos')
    .success((data) => {
      $scope.todoData = data;
      console.log(data);
    })
    .error((error) => {
      console.log('Error: ' + error);
    });
    // Create a new todo
    $scope.createTodo = () => {
      $http.post('/api/v1/todos', $scope.formData)
      .success((data) => {
        $scope.formData = {};
        $scope.todoData = data;
        console.log(data);
      })
      .error((error) => {
        console.log('Error: ' + error);
      });
    };
    // Delete a todo
    $scope.deleteTodo = (todoID) => {
      $http.delete('/api/v1/todos/' + todoID)
      .success((data) => {
        $scope.todoData = data;
        console.log(data);
      })
      .error((data) => {
        console.log('Error: ' + data);
      });
    };
  });
  ```

  These are Angular.js functions that interacts with the web forms we have in client/views/index.html. There you can see these functions called:

  ```html
  <div class="todo-form">
    <form>
      <div class="form-group">
        <input type="text" class="form-control input-lg" placeholder="Enter text..." ng-model="formData.text">
      </div>
      <button type="submit" class="btn btn-primary btn-lg btn-block" ng-click="createTodo()">Add Todo</button>
    </form>
  </div>
  ```

  The `ng-click="createTodo()"` here assigns the createTodo() function to the button. And in

  ```html
  <div class="todo-list">
    <ul ng-repeat="todo in todoData">
      <li><h3><input class="lead" type="checkbox" ng-click="deleteTodo(todo.id)">&nbsp;{{ todo.text }}</h3></li><hr>
    </ul>
  </div>
  ```

  The deleteTodo(todo.id) function gets assigned to the checkbox for todo completion, so that when you checks off a to-do item, it gets deleted from the database. `ng-repeat="todo in todoData"` iterates over all todos we got from the Angular.js app.js script earlier:

  ```javascript
  $scope.todoData = {};
  // Get all todos
  $http.get('/api/v1/todos')
  .success((data) => {
    $scope.todoData = data;
    console.log(data);
  })
  ```

## 6. Deploy

Now you can see this project in action! In your terminal, under the project root directory, do

```sh
npm start
```

If there are no errors, you can fire up your web browser and in a new tab, go to

```
http://localhost:3000/
```
