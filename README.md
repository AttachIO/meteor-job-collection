# jobCollection

**NOTE:** This Package remains experimental until v0.1.0 is released, and while the API methods described here are maturing, they may still change.

##Intro

`jobCollection` is a powerful and easy to use job manager designed and built for Meteor.js

It solves the following problems (and more):

*    Schedule jobs to run (and repeat) in the future, persisting across server restarts
*    Move work out of the Meteor's single threaded event-loop
*    Permit work on computationally expensive jobs to run anywhere
*    Track jobs and their progress, and automatically retry failed jobs
*    Easily build an admin UI to manage all of the above using Meteor's reactivity and UI goodness

### Quick example

The code snippets below show a Meteor server that creates a `jobCollection`, Meteor client code that subscribes to it and creates a new job, and a pure node.js program that can run *anywhere* and work on such jobs.

```js
///////////////////
// Server
if (Meteor.isServer) {

   myJobs = JobCollection('myJobQueue');
   myJobs.allow({
    // Grant full permission to any authenticated user
    admin: function (userId, method, params) { return (userId ? true : false); }
   });

   Meteor.startup(function () {
      // Normal Meteor publish call, the server always
      // controls what each client can see
      Meteor.publish('allJobs', function () {
         myJobs.find({});
      });

      // Start the myJobs queue running
      myJobs.startJobs();
   }
}
```

Alright, the server is set-up and running, now let's add some client code to create/manage a job.

```js
///////////////////
// Client
if (Meteor.isClient) {

   myJobs = JobCollection('myJobQueue');
   Meteor.subscribe('allJobs');

   // Because of the server settings, the code below will only work
   // if the client is authenticated.
   // On the server all of it would run unconditionally

   // Create a job:
   job = myJobs.createJob('sendEmail', // type of job
      // Job data, defined by you for type of job
      // whatever info is needed to complete it.
      // May contain links to files, etc...
      {
         address: 'bozo@clowns.com',
         subject: 'Critical rainbow hair shortage'
         message: 'LOL; JK, KThxBye.'
      }
   );

   // Set some proerties of the job and then submit it
   job.priority('normal')
      .retry({ retries: 5,
               wait: 15*60*1000 })  // 15 minutes between attempts
      .delay(60*60*1000)            // Wait an hour before first try
      .save();                      // Commit it to the server

   // Now that it's saved, this job will appear as a document
   // in the myJobs Collection, and will reactively update as
   // its status changes, etc.

   // Any job document from myJobs can be turned into a Job object
   job = myJobs.makeJob(myJobs.findOne({}));

   // Or a job can be fetched from the server by _id
   myJobs.getJob(_id, function (err, job) {
      // If successful, job is a Job object corresponding to _id
      // With a job object, you can remotely control the
      // job's status (subject to server allow/deny rules)
      // Here are some examples:
      job.pause();
      job.cancel();
      job.remove();
      // etc...
   });
}
```

**Q:** Okay, that's cool, but where does the actual work get done?

**A:** Anywhere you want!

Below is a pure node.js program that can obtain jobs from the server above and "get 'em done."
Powerfully, this can be run ***anywhere*** that has node.js and can connect to the server.

```js
///////////////////
// node.js Worker
var DDP = require('ddp');
var DDPlogin = require('ddp-login');
var Job = require('meteor-job')

// Job here has essentially the same API as jobCollection on Meteor
// In fact, Meteor jobCollection is built on top of the 'node-job' npm package!

// Setup the DDP connection
var ddp = new DDP({
   host: "meteor.mydomain.com",
   port: 3000,
   use_ejson: true
});

// Connect Job with this DDP session
Job.setDDP(ddp);

// Open the DDP connection
ddp.connect(function (err) {
   if (err) throw err;
   // Call below will prompt for email/password if an
   // authToken isn't available in the process environment
   DDPlogin(ddp, function (err, token) {
      if (err) throw err;
      // We're in!
      // Create a worker to get sendMail jobs from 'myJobQueue'
      // This will keep running indefinitely, obtaining new work from the
      // server whenever it is available.
      workers = Job.processJobs('myJobQueue', 'sendEmail', function (job, cb) {
         // This will only be called if a 'sendEmail' job is obtained
         email = job.data.email // Only one email per job
         sendEmail(email.address, email.subject, email.message, function(err) {
            if (err) {
               job.log("Sending failed with error" + err, {level: 'warning'});
               job.fail("" + err);
            } else {
               job.done();
            }
            cb(); // Be sure to invoke the callback when work on this job has finished
         });
      });
   });
});
```

Worker code very similar to the above (without all of the DDP setup) can run on the Meteor server or even a Meteor client.

### Design

The design of jobCollection is heavily influenced by [Kue](https://github.com/LearnBoost/kue) and to a lesser extent by the [Maui Cluster Scheduler](https://en.wikipedia.org/wiki/Maui_Cluster_Scheduler). However, unlike Kue's use of Redis Pub/Sub and an HTTP API, `jobCollection` uses MongoDB, Meteor, and Meteor's DDP protocol to provide persistence, reactivity, and secure remote access.

As the name implies, a `JobCollection` looks and acts like a Meteor Collection because under the hood it actually is one. However, other than `.find()` and `.findOne()`, all accesses to a `JobCollection` happen via the easy to use API on `Job` objects. Most `Job` API calls are transformed internally to Meteor [Method](http://docs.meteor.com/#methods_header) calls. This is cool because the underlying `Job` class is implemented as pure Javascript that can run in both the Meteor server and client environments, and most significantly as pure node.js code running independently from Meteor (as shown in the example code above).

## Installation

I've only tested with Meteor v0.8. It may run on Meteor v0.7 as well, I don't know.

Requires [meteorite](https://atmospherejs.com/docs/installing). To add to your project, run:

    mrt add jobCollection

The package exposes a global object `JobCollection` on both client and server.

**NOTE!** Sample app and tests mentioned below are not implemented yet!

If you'd like to try out the sample app, you can clone the repo from github:

```
git clone --recursive \
    https://github.com/vsivsi/meteor-job-collection.git \
    jobCollection
```

Then go to the `sampleApp` subdirectory and run meteorite to launch:

```
cd fileCollection/sampleApp/
mrt
```

You should now be able to point your browser to `http://localhost:3000/` and play with the sample app.

To run tests (using Meteor tiny-test) run from within the `jobCollection` subdirectory:

    meteor test-packages ./

Load `http://localhost:3000/` and the tests should run in your browser and on the server.

## Use

### Security

## JobCollection API

### `jc = new JobCollection([name], [options])` - Server and Client
#### Creates a new JobCollection

Creating a new `JobCollection` is similar to creating a new Meteor Collection. You simply specify a name (which defaults to `"queue"`. There currently are no valid `options`, but the parameter is included for possible future use. On the server there are some additional methods you will probably want to invoke on the returned object to configure it further.

For security and simplicity the traditional client allow/deny rules for Meteor collections are preset to deny all direct client `insert`, `update` and `remove` type operations on a `JobCollection`. This effectively channels all remote activity through the `JobCollection` DDP methods, which may be secured using allow/deny rules specific to `JobCollection`. See the documentation for `js.allow()` and `js.deny()` for more information.

```js
// the "new" is optional
jc = JobCollection('defaultJobCollection');
```

### `jc.setLogStream(writeStream)` - Server only
#### Sets where the jobCollection method invocation log will be written

You can log everything that happens to a jobCollection on the server by providing any valid writable stream. You may only call this once, unless you first call `jc.shutdown()`, which will automatically close the existing `logStream`.

```js
// Log everything to stdout
jc.setLogStream(process.stdout);
```

### `jc.logConsole` - Client only
#### Member variable that turns on DDP method call logging to the console

```js
jc.logConsole = false  // Default. Do not log method calls to the client console
```

### `jc.promote([milliseconds])` - Server only
#### Sets time between checks for delayed jobs that are now ready to run

`jc.promote()` may be called at any time to change the polling rate. jobCollection must poll for this operation because it is time that is changing, not the contents of the database, so there are no database updates to listen for.

```js
jc.promote(15*1000);  // Default: 15 seconds
```

### `jc.allow(options)` - Server only
#### Allow remote execution of specific jobCollection methods

Compared to vanilla Meteor collections, `jobCollection` has very a different set of remote methods with specific security implications. Where the `.allow()` method on a Meteor collection takes functions to grant permission for `insert`, `update` and `remove`, `jobCollection` has more functionality to secure and configure.

By default no remote operations are allowed, and in this configuration, jobCollection exists only as a server-side service, with the creation, management and execution of all jobs dependent on the server.

The opposite extreme is to allow any remote client to perform any action. Obviously this is totally insecure, but is perhaps valuable for early development stages on a local firewalled network.

```js
// Allow any remote client (Meteor client or node.js application) to perform any action
jc.allow({
  // The "admin" below represents the grouping of all remote methods
  admin: function (userId, method, params) { return true; };
});
```

If this seems a little reckless (and it should), then here is how you can grant admin rights specifically to an single authenticated Meteor userId:

```js
// Allow any remote client (Meteor client or node.js application) to perform any action
jc.allow({
  // Assume "adminUserId" contains the Meteor userId string of an admin super-user.
  // The array below is assumed to be an array of userIds
  admin: [ adminUserId ]
});

// The array notation in the above code is a shortcut for:
var adminUsers = [ adminUserId ];
jc.allow({
  // Assume "adminUserId" contains the Meteor userId string of an admin super-user.
  admin: function (userId, method, params) { return (userId in adminUsers); };
});
```

In addition to the all-encompassing `admin` method group, there are three others:

*    `manager` -- Managers can remotely manage the jobCollection (e.g. cancelling jobs).
*    `creator` -- Creators can remotely make new jobs to run.
*    `worker` -- Workers can get Jobs to work on and can update their status as work proceeds.

All remote methods affecting the jobCollection fall into at least one of the four groups, and for each client-capable API method below, the group(s) it belongs to will be noted.

In addition to the above groups, it is possible to write allow/deny rules specific to each `jobCollection` DDP method. This is a more advanced feature and the intent is that the four permission groups described above should be adequate for many applications. The DDP methods are generally lower-level than the methods available on `Job` and they do not necessarily have a one-to-one relationship. Here's an example of how to given permission to create new "email" jobs to a single userId:

```js
// Assumes emailCreator contains a Meteor userId
jc.allow({
  jobSave: function (userId, method, params) {
              if ((userId === emailCreator) &&
                  (params[0].type === 'email')) { // params[0] is the new job doc
                  return true;
              }
              return false;
           };
});
```

### `jc.deny(options)` - Server only
#### Override allow rules

This call has the same semantic relationship with `allow()` as it does in Meteor collections. If any deny rule is true, then permission for a remote method call will be denied, regardless of the status of any other allow/deny rules. This is powerful and far reaching. For example, the following code will turn off all remote access to a jobCollection (regardless of any other rules that may be in force):

```js
jc.deny({
  // The "admin" below represents the grouping of all remote methods
  admin: function (userId, method, params) { return false; };
});
```

See the `allow` method above for more details.

### `jc.forever` - Server or Client
#### Constant value used to indicate that something should repeat forever

```js
job = jc.createJob('jobType', { work: "to", be: "done" })
   .retry({ retries: jc.forever })    // Default for .retry()
   .repeat({ repeats: jc.forever });  // Default for .repeat()
```

### `jc.jobPriorities` - Server or Client
#### Valid non-numeric job priorities

```js
jc.jobPriorities = {
  low: 10
  normal: 0
  medium: -5
  high: -10
  critical: -15
};
```

### `jc.jobStatuses` - Server or Client
#### Possible states for the status of a job in the job collection

```js
jc.jobStatuses = [
    'waiting'
    'paused'
    'ready'
    'running'
    'failed'
    'cancelled'
    'completed'
];
```

### `jc.jobLogLevels` - Server or Client
#### Valid log levels

If these look familiar, it's because they correspond to some the Bootstrap [context](http://getbootstrap.com/css/#helper-classes) and [alert](http://getbootstrap.com/components/#alerts) classes.

```js
jc.jobLogLevels: [
    'info'
    'success'
    'warning'
    'danger'
];
```

### `jc.jobStatusCancellable` - Server or Client
#### Job status states that can be cancelled

```js
jc.jobStatusCancellable = [ 'running', 'ready', 'waiting', 'paused' ];
```

### `jc.jobStatusPausable` - Server or Client
#### Job status states that can be paused

```js
jc.jobStatusPausable = [ 'ready', 'waiting' ];
```

### `jc.jobStatusRemovable` - Server or Client
#### Job status states that can be removed

```js
jc.jobStatusRemovable = [ 'cancelled', 'completed', 'failed' ];
```

### `jc.jobStatusRestartable` - Server or Client
#### Job status states that can be restarted

```js
jc.jobStatusRestartable = [ 'cancelled', 'failed' ];
```

### `jc.ddpMethods` - Server or Client
#### Array of the names of all DDP methods used by `jobCollection`

```js
jc.ddpMethods = [
    'startJobs', 'stopJobs', 'jobRemove', 'jobPause', 'jobResume'
    'jobCancel', 'jobRestart', 'jobSave', 'jobRerun', 'getWork'
    'getJob', 'jobLog', 'jobProgress', 'jobDone', 'jobFail'
    ];
```

### `jc.ddpPermissionLevels` - Server or Client
#### Array of the predefined DDP method permission levels

```js
jc.ddpPermissionLevels = [ 'admin', 'manager', 'creator', 'worker' ];
```

### `jc.ddpMethodPermissions` - Server or Client
#### Object mapping permission levels to DDP method names

```js
jc.ddpMethodPermissions = {
    'startJobs': ['startJobs', 'admin'],
    'stopJobs': ['stopJobs', 'admin'],
    'jobRemove': ['jobRemove', 'admin', 'manager'],
    'jobPause': ['jobPause', 'admin', 'manager'],
    'jobResume': ['jobResume', 'admin', 'manager'],
    'jobCancel': ['jobCancel', 'admin', 'manager'],
    'jobRestart': ['jobRestart', 'admin', 'manager'],
    'jobSave': ['jobSave', 'admin', 'creator'],
    'jobRerun': ['jobRerun', 'admin', 'creator'],
    'getWork': ['getWork', 'admin', 'worker'],
    'getJob': ['getJob', 'admin', 'worker'],
    'jobLog': [ 'jobLog', 'admin', 'worker'],
    'jobProgress': ['jobProgress', 'admin', 'worker'],
    'jobDone': ['jobDone', 'admin', 'worker'],
    'jobFail': ['jobFail', 'admin', 'worker']
};
```

### `jc.getJobs(ids, [options], [callback])` - Server or Client
#### Like `jc.getJob` except it takes an array of ids
This is much more efficient than calling `jc.getJob()` in a loop because it gets Jobs from the server in batches.

### `jc.pauseJobs(ids, [options], [callback])` - Server or Client
#### Like `job.pause()` except it pauses a list of jobs by id

### `jc.resumeJobs(ids, [options], [callback])` - Server or Client
####Like `job.resume()` except it resumes a list of jobs by id

### `jc.cancelJobs(ids, [options], [callback])` - Server or Client
#### Like `job.cancel()` except it cancels a list of jobs by id

### `jc.restartJobs(ids, [options], [callback])` - Server or Client
#### Like `job.restart()` except it restarts a list of jobs by id

### `jc.removeJobs(ids, [options], [callback])` - Server or Client
#### Like `job.remove()` except it removes a list of jobs by id

### `job = jc.createJob(type, data)` - Server or Client
#### Create a new `Job` object

Data should be reasonably small, if worker requires a lot of data (e.g. video, image or sound files), they should be included by reference (e.g. with a URL pointing to the data, and another to where the result should be saved).

See documentation below for `Job` object API

```js
job = jc.createJob(
  'jobType',    // type of the job
  { /* ... */ } // Data for the worker, any valid EJSON object
);
```

### `jc.makeJob(jobDoc)` - Server or Client
#### Make a Job object from a jobCollection document

See documentation below for `Job` object API

```js
doc = jc.findOne({});
if (doc) {
   job = jc.makeJob('jobQueue', doc);
}
```

### `jc.getJob(id, [options], [callback])` - Server or Client
#### Create a job object by id from the server job Collection

See documentation below for `Job` object API

Returns `undefined` if no such job exists.

`id`: -- The id of the job to get.

`options`:
* `getLog` -- If `true`, get the current log of the job. Default is `false` to save bandwidth since logs can be large.

`callback(error, result)` -- Optional only on Meteor Server with Fibers. `result` is a job object or `undefined`

```js
if (Meteor.isServer) {
  job = jc.getJob(  // Job will be undefined or contain a Job object
    id,          // job id of type Meteor.Collection.ObjectID
    {
      getLog: false  // Default, don't include the log information
    }
  );
  // Job may be undefined
} else {
  Job.getJob(
    id,            // job id of type Meteor.Collection.ObjectID
    {
      getLog: true  // include the log information
    },
    function (err, job) {
      if (job) {
        // Here's your job
      }
    }
  );
}
```

### `jc.getWork(type, [options], [callback])` - Server or Client
#### Get one or more jobs from the jobCollection, setting status to `'running'`

See documentation below for `Job` object API

`options`:
* `maxJobs` -- Maximum number of jobs to get. Default `1`  If `maxJobs > 1` the result will be an array of job objects, otherwise it is a single job object, or `undefined` if no jobs were available

`callback(error, result)` -- Optional only on Meteor Server with Fibers. Result will be an array or single value depending on `options.maxJobs`.

```js
if (Meteor.isServer) {
  job = jc.getWork(  // Job will be undefined or contain a Job object
    'jobType',   // type of job to request
    {
      maxJobs: 1 // Default, only get one job, returned as a single object
    }
  );
} else {
  jc.getWork(
    [ 'jobType1', 'jobType2' ]  // can request multiple types in array
    {
      maxJobs: 5 // If maxJobs > 1, result is an array of jobs
    },
    function (err, jobs) {
      // jobs contains between 0 and maxJobs jobs, depending on availability
      // job type is available as
      if (job[0].type === 'jobType1') {
        // Work on jobType1...
      } else if (job[0].type === 'jobType2') {
        // Work on jobType2...
      } else {
        // Sadness
      }
    }
  );
}
```

### `jq = jc.processJobs(type, [options], worker)` - Server or Client
#### Create a new jobQueue to work on jobs

See documentation below for `JobQueue` object API

## Job API

New jobs objects are created using the following JobCollection API calls:
* `jc.createJob()` -- Creates a new `Job` object
* `jc.makeJob()` -- Makes a `Job` object from a job document (as retrieved from MongoDB)
* `jc.getJob()` -- Get a `Job` object from the jobCollection by `id`
* `jc.getJobs()` -- Get multiple `Job` objects from a jobCollection using an array of Ids

The methods below may be performed on job objects regardless of their source. All `Job` methods may be run on the client or server.

### `j.depends([dependencies])` - Server or Client
#### Adds jobs that this job depends upon (antecedents)

This job will not run until these jobs have successfully completed. Defaults to an empty array (no dependencies). Returns `job`, so it is chainable.
Added jobs must have already had `.save()` run on them, so they will have the `_id` attribute that is used to form the dependency. Calling `j.depends()` with a falsy value will clear any existing dependencies for this job.

```js
job.depends([job1, job2]);  // job1 and job2 are Job objects, and must successfully complete before job will run
job.depends();  // Clear any dependencies previously added on this job
```

### `j.priority([priority])` - Server or Client
#### Sets the priority of this job

Can be integer numeric or one of `Job.jobPriorities`. Defaults to `'normal'` priority, which is priority `0`. Returns `job`, so it is chainable.

```js
job.priority('high');  // Maps to -10
job.priority(-10);     // Same as above
```

### `j.retry([options])` - Server or Client
#### Set how failing jobs are rescheduled and retried by the job Collection

Returns `job`, so it is chainable.

`options:`
* `retries` -- Number of times to retry a failing job. Default: `Job.forever`
* `wait`  -- How long to wait between attempts, in ms. Default: `300000` (5 minutes)

`[options]` may also be a non-negative integer, which is interpreted as `{ retries: [options] }`

Note that the above stated defaults are those when `.retry()` is explicitly called. When a new job is created, the default number of `retries` is `0`.

```js
job.retry({
  retries: 5,   // Retry 5 times,
  wait: 20000   // waiting 20 seconds between attempts
});
```

### `j.repeat([options])` - Server or Client
#### Set how many times this job will be automatically re-run by the job Collection

Each time it is re-run, a new job is created in the job collection. This is equivalent to running `job.rerun()`. Only `'completed'` jobs are repeated. Failing jobs that exhaust their retries will not repeat. By default, if an infinitely repeating job is added to the job Collection, any existing repeating jobs of the same type that are cancellable, will be cancelled.  See `option.cancelRepeats` for `job.save()` for more info. Returns `job`, so it is chainable.

`options:`
* `repeats` -- Number of times to rerun the job. Default: `Job.forever`
* `wait`  -- How long to wait between re-runs, in ms. Default: `300000` (5 minutes)

`[options]` may also be a non-negative integer, which is interpreted as `{ repeats: [options] }`

Note that the above stated defaults are those when `.repeat()` is explicitly called. When a new job is created, the default number of `repeats` is `0`.

```js
job.repeat({
  repeats: 5,   // Rerun this job 5 times,
  wait: 20000   // wait 20 seconds between each re-run.
});
```

### `j.delay([milliseconds])` - Server or Client
#### Sets how long to wait until this job can be run

Counts from when it is initially saved to the job Collection.
Returns `job`, so it is chainable.

```js
job.delay(0);   // Do not wait. This is the default.
```

### `j.after([time])` - Server or Client
#### Sets the time after which a job may be run

`time` is a date object.  It is not guaranteed to run "at" this time because there may be no workers available when it is reached. Returns `job`, so it is chainable.

```js
job.after(new Date());   // Run the job anytime after right now. This is the default.
```

### `j.log(message, [options], [callback])` - Server or Client
#### Add an entry to this job's log

May be called before a new job is saved. `message` must be a string.

`options:`
* `level`: One of `Jobs.jobLogLevels`: `'info'`, `'success'`, `'warning'`, or `'danger'`.  Default is `'info'`.
* `echo`: Echo this log entry to the console. `'danger'` and `'warning'` level messages are echoed using `console.error()` and `console.warn()` respectively. Others are echoed using `console.log()`. If echo is `true` all messages will be echoed. If `echo` is one of the `Job.jobLogLevels` levels, only messages of that level or higher will be echoed.

`callback(error, result)` -- Result is true if logging was successful. When running as `Meteor.isServer` with fibers, for a saved object the callback may be omitted and the return value is the result. If called on an unsaved object, the result is `job` and can be chained.

```js
job.log(
  "This is a message",
  {
    level: 'warning'
    echo: true   // Default is false
  },
  function (err, result) {
    if (result) {
      // The log method worked!
    }
  }
);

var verbosityLevel = 'warning';
job.log("Don't echo this", { level: 'info', echo: verbosityLevel } );
```

### `j.progress(completed, total, [options], [cb])` - Server or Client
#### Update the progress of a running job

May be called before a new job is saved. `completed` must be a number `>= 0` and `total` must be a number `> 0` with `total >= completed`.

`options:`
* `echo`: Echo this progress update to the console using `console.log()`.

`callback(error, result)` -- Result is true if progress update was successful. When running as `Meteor.isServer` with fibers, for a saved object the callback may be omitted and the return value is the result. If called on an unsaved object, the result is `job` and can be chained.

```js
job.progress(
  50,
  100,    // Half done!
  {
    echo: true   // Default is false
  },
  function (err, result) {
    if (result) {
      // The progress method worked!
    }
  }
);
```

### `j.save([options], [callback])` - Server or Client
#### Submits this job to the job Collection

Only valid if this is a new job, or if the job is currently paused in the job Collection. If the job is already saved and paused, then most properties of the job may change (but not all, e.g. the jobType may not be changed.)

`options:`
* `cancelRepeats`: If true and this job is an infinitely repeating job, will cancel any existing jobs of the same job type. Default is `true`. This is useful for background maintenance jobs that may get added on each server restart (potentially with new parameters).

`callback(error, result)` -- Result is true if save was successful. When running as `Meteor.isServer` with fibers, the callback may be omitted and the return value is the result.

```js
job.save(
  {
    cancelRepeats: false  // Do not cancel any jobs of the same type, even if this job repeats forever.  Default: true.
  }
);
```
### `j.refresh([options], [callback])` - Server or Client
#### Refreshes the current job object state with the state on the remote jobCollection

Note that if you subscribe to the job Collection, the job documents will stay in sync with the server automatically via Meteor reactivity.

`options:`
* `getLog` -- If true, also refresh the jobs log data (which may be large).  Default: `false`

`callback(error, result)` -- Result is true if refresh was successful. When running as `Meteor.isServer` with fibers, the callback may be omitted and the return value is the result.

```js
job.refresh(function (err, result) {
  if (result) {
    // Refreshed
  }
});
```

### `j.done(result, [options], [callback])` - Server or Client
#### Change the state of a running job to `'completed'`.

`result` is any EJSON object.  If this job is configured to repeat, a new job will automatically be cloned to rerun in the future.  Result will be saved as an object. If passed result is not an object, it will be wrapped in one.

`options:` -- None currently.

`callback(error, result)` -- Result is true if completion was successful. When running as `Meteor.isServer` with fibers, the callback may be omitted and the return value is the result.

```js
job.done(function (err, result) {
  if (result) {
    // Status updated
  }
});

// Pass a non-object result
job.done("Done!");
// This will be saved as:
// { "value": "Done!" }
```

### `j.fail(message, [options], [callback])` - Server or Client
#### Change the state of a running job to `'failed'`.

It's next state depends on how the job's `job.retry()` settings are configured. It will either become `'failed'` or go to `'waiting'` for the next retry. `message` is a string.

`options:`
* `fatal` -- If true, no additional retries will be attempted and this job will go to a `'failed'` state. Default: `false`

`callback(error, result)` -- Result is true if failure was successful (heh). When running as `Meteor.isServer` with fibers, the callback may be omitted and the return value is the result.

```js
job.fail(
  'This job has failed again!',
  {
    fatal: false  // Default case
  },
  function (err, result) {
    if (result) {
      // Status updated
    }
  }
});
```

### `j.pause([options], [callback])` - Server or Client
#### Change the state of a job to `'paused'`.

Only `'ready'` and `'waiting'` jobs may be paused. This specifically does nothing to affect running jobs. To stop a running job, you must use `job.cancel()`.

`options:` -- None currently.

`callback(error, result)` -- Result is true if pausing was successful. When running as `Meteor.isServer` with fibers, the callback may be omitted and the return value is the result.

```js
job.pause(function (err, result) {
  if (result) {
    // Status updated
  }
});
```

### `j.resume([options], [callback])` - Server or Client
#### Change the state of a job from `'paused'` to `'waiting'`

`options:` -- None currently.

`callback(error, result)` -- Result is true if resuming was successful. When running as `Meteor.isServer` with fibers, the callback may be omitted and the return value is the result.

```js
job.resume(function (err, result) {
  if (result) {
    // Status updated
  }
});
```

### `j.cancel([options], [callback])` - Server or Client
#### Change the state of a job to `'cancelled'`.

Any job that isn't `'completed'`, `'failed'` or already `'cancelled'` may be cancelled. Cancelled jobs retain any remaining retries and/or repeats if they are later restarted.

`options:`
* `antecedents` -- Also cancel all cancellable jobs that this job depends on.  Default: `false`
* `dependents` -- Also cancel all cancellable jobs that depend on this job.  Default: `true`

`callback(error, result)` -- Result is true if cancellation was successful. When running as `Meteor.isServer` with fibers, the callback may be omitted and the return value is the result.

```js
job.cancel(
  {
    antecedents: false,
    dependents: true    // Also cancel all jobs that will never run without this one.
  },
  function (err, result) {
    if (result) {
      // Status updated
    }
  }
);
```

### `j.restart([options], [callback])` - Server or Client
#### Change the state of a `'failed'` or `'cancelled'` job to `'waiting'` to be retried.

A restarted job will retain any repeat count state it had when it failed or was cancelled.

`options:`
* `retries` -- Number of additional retries to attempt before failing with `job.retry()`. Default: `0`. These retries add to any remaining retries already on the job (such as if it was cancelled).
* `antecedents` -- Also restart all `'cancelled'` or `'failed'` jobs that this job depends on.  Default: `true`
* `dependents` -- Also restart all `'cancelled'` or `'failed'` jobs that depend on this job.  Default: `false`

`callback(error, result)` -- Result is true if restart was successful. When running as `Meteor.isServer` with fibers, the callback may be omitted and the return value is the result.

```js
job.restart(
  {
    antecedents: true,  // Also restart all jobs that must complete before this job can run.
    dependents: false,
    retries: 0          // Only try one more time. This is the default.
  },
  function (err, result) {
    if (result) {
      // Status updated
    }
  }
);
```

### `j.rerun([options], [callback])` - Server or Client
#### Clone a completed job and run it again

`options:`
* `repeats` -- Number of times to repeat the job, as with `job.repeat()`.
* `wait` -- Time to wait between reruns. Default is the existing `job.repeat({ wait: ms }) setting for the job.

`callback(error, result)` -- Result is true if rerun was successful. When running as `Meteor.isServer` with fibers, the callback may be omitted and the return value is the result.

```js
job.rerun(
  {
    repeats: 0,         // Only repeat this once. This is the default.
    wait: 60000         // Wait a minute between repeats. Default is previous setting.
  },
  function (err, result) {
    if (result) {
      // Status updated
    }
  }
);
```

### `j.remove([options], [callback])` - Server or Client
#### Permanently remove this job from the job collection

The job must be `'completed'`, `'failed'`, or `'cancelled'` to be removed.

`options:` -- None currently.

`callback(error, result)` -- Result is true if removal was successful. When running as `Meteor.isServer` with fibers, the callback may be omitted and the return value is the result.

```js
job.remove(function (err, result) {
  if (result) {
    // Job removed from server.
  }
});
```

### `j.type` - Server or Client
#### Contains the type of a job

Useful for when `getWork` or `processJobs` are configured to accept multiple job types. This may not be changed after a job is created.

### `j.data` - Server or Client
#### Contains the job data needed by the worker to complete a job of a given type

Always an object. This may not be changed after a job is created.


## JobQueue API

JobQueue is similar in spirit to the [async.js](https://github.com/caolan/async) [queue](https://github.com/caolan/async#queue) and [cargo]([queue](https://github.com/caolan/async#cargo)) except that it gets its work from the Meteor jobCollection via calls to `Job.getWork()`

### `q = Job.processJobs()` - Server or Client

Create a `JobQueue` to automatically get work from the job Collection, and asynchronously call the worker function.

`options:`
* `concurrency` -- Maximum number of async calls to `worker` that can be outstanding at a time. Default: `1`
* `cargo` -- Maximum number of job objects to provide to each worker, Default: `1` If `cargo > 1` the first paramter to `worker` will be an array of job objects rather than a single job object.
* `pollInterval` -- How often to ask the remote job Collection for more work, in ms. Default: `5000` (5 seconds)
* `prefetch` -- How many extra jobs to request beyond the capacity of all workers (`concurrency * cargo`) to compensate for latency getting more work.

`worker(result, callback)`
* `result` -- either a single job object or an array of job objects depending on `options.cargo`.
* `callback` -- must be eventually called exactly once when `job.done()` or `job.fail()` has been called on all jobs in result.

```js
queue = Job.processJobs(
  'jobQueue',   // name of job Collection
  'jobType',    // type of job to request, can also be an array of job types
  {
    concurrency: 4,
    cargo: 1,
    pollInterval: 5000,
    prefetch: 1
  },
  function (job, callback) {
    // Only called when there is a valid job
    job.done();
    callback();
  }
);

// The job queue has methods... See JobQueue documentation for details.
queue.pause();
queue.resume();
queue.shutdown();
```
### `q.pause()` - Server or Client

Pause the JobQueue. This means that no more work will be requested from the job collection, and no new workers will be called with jobs that already exist in this local queue. Jobs that are already running locally will run to completion. Note that a JobQueue may be created in the paused state by running `q.pause()` immediately on the returned new jobQueue.

```js
q.pause()
```
### `q.resume()` - Server or Client

Undoes a `q.pause()`, returning the queue to the normal running state.

```js
q.resume()
```
### `q.shutdown([options], [callback])` - Server or Client

`options:`
* `level` -- May be 'hard' or 'soft'. Any other value will lead to a "normal" shutdown.
* `quiet` -- true or false. False by default, which leads to a "Shutting down..." message on stderr.

`callback()` -- Invoked once the requested shutdown conditions have been achieved.

Shutdown levels:
* `'soft'` -- Allow all local jobs in the queue to start and run to a finish, but do not request any more work. Normal program exit should be possible.
* `'normal'` -- Allow all running jobs to finish, but do not request any more work and fail any jobs that are in the local queue but haven't started to run. Normal program exit should be possible.
* `'hard'` -- Fail all local jobs, running or not. Return as soon as the server has been updated. Note: after a hard shutdown, there may still be outstanding work in the event loop. To exit immediately may require `process.exit()` depending on how often asynchronous workers invoke `'job.progress()'` and whether they die when it fails.

```js
q.shutdown({ quiet: true, level: 'soft' }, function () {
  // shutdown complete
});
```
### `q.length()` - Server or Client

Number of tasks ready to run.

### `q.full()` - Server or Client

`true` if all of the concurrent workers are currently running.

### `q.running()` - Server or Client

Number of concurrent workers currently running.

### `q.idle()` - Server or Client

`true` if no work is currently running.


## Job document data models

The definitions below use a slight shorthand of the Meteor [Match pattern](http://docs.meteor.com/#matchpatterns) syntax to describe the valid structure of a job document. As a user of `jobCollection` this is mostly for your information because jobs are automatically built and maintained by the package.

```js
validStatus = (
   Match.test(v, String) &&
   (v in [
      'waiting',
      'paused',
      'ready',
      'running',
      'failed',
      'cancelled',
      'completed'
   ])
);

validLogLevel = (
   Match.test(v, String) &&
   (v in [
      'info',
      'success',
      'warning',
      'danger'
   ])
);

validLog = [{
      time:    Date,
      runId:   Match.OneOf(
                  Meteor.Collection.ObjectID, null
               ),
      level:   Match.Where(validLogLevel),
      message: String
}];

validProgress = {
  completed: Match.Where(validNumGTEZero),
  total:     Match.Where(validNumGTEZero),
  percent:   Match.Where(validNumGTEZero)
};

validJobDoc = {
   _id:       Match.Optional(
                 Match.OneOf(
                    Meteor.Collection.ObjectID,
                    null
              )),
  runId:      Match.OneOf(
                 Meteor.Collection.ObjectID,
                 null
              ),
  type:       String,
  status:     Match.Where(validStatus),
  data:       Object
  result:     Match.Optional(Object),
  priority:   Match.Integer,
  depends:    [ Meteor.Collection.ObjectID ],
  resolved:   [ Meteor.Collection.ObjectID ],
  after:      Date,
  updated:    Date,
  log:        Match.Optional(validLog()),
  progress:   validProgress(),
  retries:    Match.Where(validNumGTEZero),
  retried:    Match.Where(validNumGTEZero),
  retryWait:  Match.Where(validNumGTEZero),
  repeats:    Match.Where(validNumGTEZero),
  repeated:   Match.Where(validNumGTEZero),
  repeatWait: Match.Where(validNumGTEZero)
};
```

## DDP Method reference

These are the underlying Meteor methods that are actually invoked when a method like `.save()` or `.getWork()` is called. In most cases you will not need to program to this interface because the `JobCollection` and `Job` APIs do this work for you. One exception to this general rule is if you need finer control over allow/deny rules than is provided by the predefined `admin`, `manager`, `creator`, and `worker` access categories.

Each `jobCollection` you create on a server causes a number of Meteor methods to be defined. The method names are prefaced with the name of the jobCollection (e.g. "myJobs_getWork") so that multiple jobCollections on a server will not interfere with one another. Below you will find the Method API reference.

### `startJobs(options)`
#### Start running the jobCollection

* `options` -- No options currently used

    `Match.Optional({})`

Returns: `Boolean` - Success or failure

### `stopJobs(options)`
#### Shut down the jobCollection

* `options` -- Supports the following options:

    * `timeout` -- Time in ms until all outstanding jobs will be marked as failed.

    `Match.Optional({
      timeout: Match.Optional(Match.Where(validIntGTEOne))
    })`

Returns: `Boolean` - Success or failure

### `getJob(ids, options)`
#### Returns a Job document corresponding to provided id

* `ids` -- an Id or array of Ids to get from server

    `ids: Match.OneOf(Meteor.Collection.ObjectID, [ Meteor.Collection.ObjectID ])`

* `options` -- Supports the following options:

    * `getLog` -- If true include the job log data in the returned job data. Default is false.

    `Match.Optional({
      getLog: Match.Optional(Boolean)
    })`

Returns: `validJobDoc()` or `[ validJobDoc() ]` depending on if `ids` is a single value or an array.

### `getWork(type, options)`
#### Returns jobs ready-to-run to a requesting worker

* `type` -- a string job type or an array of such types
    `type: Match.OneOf(String, [ String ])`

* `options` -- Supports the following options:
    * `maxJobs` -- The maximum number of jobs to return, Default: `1`

    `Match.Optional({
         maxJobs: Match.Optional(Match.Where(validIntGTEOne))
    })`

Returns: `validJobDoc()` or `[ validJobDoc() ]` depending on if maxJobs > 1.


### `jobRemove(ids, options)`
#### Permanently remove jobs from the jobCollection

* `ids` -- an Id or array of Ids to remove from server

    `ids: Match.OneOf(Meteor.Collection.ObjectID, [ Meteor.Collection.ObjectID ])`

* `options` -- No options currently used

    `Match.Optional({})`

Returns: `Boolean` - Success or failure

### `jobPause(ids, options)`
#### Pauses a job in the jobCollection, changes status to `paused` which prevents it from running

* `ids` -- an Id or array of Ids to remove from server

    `ids: Match.OneOf(Meteor.Collection.ObjectID, [ Meteor.Collection.ObjectID ])`

* `options` -- No options currently used

    `Match.Optional({})`

Returns: `Boolean` - Success or failure

### `jobResume(ids, options)`
#### Resumes (unpauses) a job in the jobCollection, returns it to the `waiting` state

* `ids` -- an Id or array of Ids to remove from server

    `ids: Match.OneOf(Meteor.Collection.ObjectID, [ Meteor.Collection.ObjectID ])`

* `options` -- No options currently used

    `Match.Optional({})`

Returns: `Boolean` - Success or failure

### `jobCancel(ids, options)`
#### Cancels a job in the jobCollection. Cancelled jobs will not run and will stop running if already running.

* `ids` -- an Id or array of Ids to remove from server

    `ids: Match.OneOf(Meteor.Collection.ObjectID, [ Meteor.Collection.ObjectID ])`

* `options` -- Supports the following options:
    * `antecedents` -- If true, all jobs that this one depends upon will also be cancelled. Default: `false`
    * `dependents` -- If true, all jobs that depend on this one will also be be cancelled. Default: `true`

    `Match.Optional({
        antecedents: Match.Optional(Boolean)
        dependents: Match.Optional(Boolean)
    })`

Returns: `Boolean` - Success or failure

### `jobRestart(ids, options)`
#### Restarts a cancelled or failed job.

* `ids` -- an Id or array of Ids to remove from server

    `ids: Match.OneOf(Meteor.Collection.ObjectID, [ Meteor.Collection.ObjectID ])`

* `options` -- Supports the following options:
    * `antecedents` -- If true, all jobs that this one depends upon will also be restarted. Default: `true`
    * `dependents` -- If true, all jobs that depend on this one will also be be restarted. Default: `false`

    `Match.Optional({
        antecedents: Match.Optional(Boolean)
        dependents: Match.Optional(Boolean)
    })`

Returns: `Boolean` - Success or failure

### `jobSave(doc, options)`
#### Adds a job to the jobCollection in the `waiting` or `paused` state

* `doc` -- Job document of job to save to the server jobCollection

    `validJobDoc()`

* `options` -- Supports the following options:
    * `cancelRepeats` --  If true and this job is an infinitely repeating job, will cancel any existing jobs of the same job type. Default is true.

    `Match.Optional({
      cancelRepeats: Match.Optional(Boolean)
    })`

Returns: `Meteor.Collection.ObjectID` of the added job.

### `jobRerun(id, options)`
#### Creates and saves a new job based on an existing job that has successfully completed.

* `id` -- The id of the job to rerun

    `Meteor.Collection.ObjectID`

* `options` -- Supports the following options:
    * `wait` -- Amount of time to wait until the new job runs in ms. Default: 0
    * `repeats` -- Number of times to repeat the new job. Default: 0

    `Match.Optional({
      repeats: Match.Optional(Match.Where(validIntGTEZero))
      wait: Match.Optional(Match.Where(validIntGTEZero))
    })`

Returns: `Meteor.Collection.ObjectID` of the added job.

### `jobProgress(id, runId, completed, total, options)`
#### Update the progress of a running job

* `id` -- The id of the job to update

    `Meteor.Collection.ObjectID`

* `runId` -- The runId of this worker

    `Meteor.Collection.ObjectID`

* `completed` -- The estimated amount of effort completed

    `Match.Where(validNumGTEZero)`

* `total` -- The estimated total effort

    `Match.Where(validNumGTZero)`

* `options` -- No options currently used

    `Match.Optional({})`

Returns: `Boolean` - Success or failure or `null` if jobCollection is shutting down

### `jobLog(id, runId, message, options)`
#### Add an entry in the job log of a running job

* `id` -- The id of the job to update

    `Meteor.Collection.ObjectID`

* `runId` -- The runId of this worker

    `Meteor.Collection.ObjectID`

* `message` -- The text of the message to add to the log

    `String`

* `options` -- Supports the following options:
    * `level` -- The information level of this log entry. Must be a valid log level. Default: `'info'`

    `Match.Optional({
        level: Match.Optional(Match.Where(validLogLevel))
    })`

Returns: `Boolean` - Success or failure

### `jobDone(id, runId, result, options)`
#### Change a job's status to `completed`

* `id` -- The id of the job to update

    `Meteor.Collection.ObjectID`

* `runId` -- The runId of this worker

    `Meteor.Collection.ObjectID`

* `result` -- A result object to store with the completed job.

    `Object`

* `options` -- No options currently used

    `Match.Optional({})`

Returns: `Boolean` - Success or failure

### `jobFail(id, runId, err, options)`
#### Change a job's status to `failed`

* `id` -- The id of the job to update

    `Meteor.Collection.ObjectID`

* `runId` -- The runId of this worker

    `Meteor.Collection.ObjectID`

* `err` -- An error string to store with the failed job.

    `String`

* `options` -- Supports the following options:
    * `fatal` -- If true, cancels any remaining repeat runs this job was scheduled to have. Default: false.

    `options: Match.Optional({
      fatal: Match.Optional(Boolean)
    })`

Returns: `Boolean` - Success or failure
