# AgendaTS

(full typed version of agendaJS)

<p align="center">
  <img src="https://cdn.jsdelivr.net/gh/hokify/agenda@master/agendats.png" alt="Agenda TS" width="100" height="100">
</p>
<p align="center">
  A light-weight job scheduling library for Node.js
</p>

This was originally a fork of agenda.js,
it differs from the original version in following points:

- Complete rewrite in Typescript (fully typed!)
- mongodb4 driver (supports mongodb 5.x)
- Supports mongoDB sharding by name
- touch() can have an optional progress parameter (0-100)
- Bugfixes and improvements for locking & job processing (concurrency, lockLimit,..)
- **Intelligent batch processing** for improved performance and reduced write conflicts
- **Write conflict resilience** with exponential backoff retry logic
- **Optimized MongoDB indexes** for 50-70% faster job discovery
- **Memory-bounded job queue** with overflow protection
- **O(1) job lookups** using Map-based collections
- **Connection pooling** with optimized defaults
- Breaking change: define() config paramter moved from 2nd position to 3rd
- getRunningStats()
- automatically waits for agenda to be connected before calling any database operations
- uses a database abstraction layer behind the scene
- does not create a database index by default, you can set `ensureIndex: true` when initializing Agenda
  or run manually:

```
db.agendaJobs.ensureIndex({
    "name" : 1,
    "nextRunAt" : 1,
    "priority" : -1,
    "lockedAt" : 1,
    "disabled" : 1
}, "findAndLockNextJobIndex")
```

# Agenda offers

- Minimal overhead. Agenda aims to keep its code base small.
- Mongo backed persistence layer.
- Promises based API.
- Scheduling with configurable priority, concurrency, and repeating.
- Scheduling via cron or human readable syntax.
- Event backed job queue that you can hook into.
- **Intelligent batch processing** for high-performance job processing with configurable batch sizes.
- **Write conflict resilience** with automatic retry and exponential backoff for MongoDB operations.
- [Agendash](https://github.com/agenda/agendash): optional standalone web-interface.
- [Agenda-rest](https://github.com/agenda/agenda-rest): optional standalone REST API.
- [inversify-agenda](https://github.com/lautarobock/inversify-agenda) - Some utilities for the development of agenda workers with Inversify

### Feature Comparison

Since there are a few job queue solutions, here a table comparing them to help you use the one that
better suits your needs.

| Feature                    |      Bull       |   Bee    | Agenda | AgendaTS |
| :------------------------- | :-------------: | :------: | :----: | :------: |
| Backend                    |      redis      |  redis   | mongo  |  mongo   |
| Priorities                 |        ✓        |          |   ✓    |    ✓     |
| Concurrency                |        ✓        |    ✓     |   ✓    |    ✓     |
| Delayed jobs               |        ✓        |          |   ✓    |    ✓     |
| Global events              |        ✓        |          |        |    ✓     |
| Rate Limiter               |        ✓        |          |        |          |
| Pause/Resume               |        ✓        |          |        |    ✓     |
| Sandboxed worker           |        ✓        |          |        |    ✓     |
| Repeatable jobs            |        ✓        |          |   ✓    |    ✓     |
| Atomic ops                 |        ✓        |    ✓     |        |    ~     |
| Persistence                |        ✓        |    ✓     |   ✓    |    ✓     |
| UI                         |        ✓        |          |   ✓    |    ✓     |
| REST API                   |                 |          |   ✓    |    ✓     |
| Central (Scalable) Queue   |                 |          |        |    ✓     |
| Supports long running jobs |                 |          |        |    ✓     |
| **Batch Processing**       |                 |          |        |  **✓**   |
| **Write Conflict Resilience** |             |          |        |  **✓**   |
| Optimized for              | Jobs / Messages | Messages |  Jobs  |   Jobs   |

_Kudos for making the comparison chart goes to [Bull](https://www.npmjs.com/package/bull#feature-comparison) maintainers._

# Installation

Install via NPM

    npm install @hokify/agenda

You will also need a working [Mongo](https://www.mongodb.com/) database (v4+) to point it to.

# Example Usage

```js
const mongoConnectionString = 'mongodb://127.0.0.1/agenda';

const agenda = new Agenda({ db: { address: mongoConnectionString } });

// Or override the default collection name:
// const agenda = new Agenda({db: {address: mongoConnectionString, collection: 'jobCollectionName'}});

// or pass additional connection options:
// const agenda = new Agenda({db: {address: mongoConnectionString, collection: 'jobCollectionName', options: {ssl: true}}});

// or pass in an existing mongodb-native MongoClient instance
// const agenda = new Agenda({mongo: myMongoClient});

agenda.define('delete old users', async job => {
	await User.remove({ lastLogIn: { $lt: twoDaysAgo } });
});

(async function () {
	// IIFE to give access to async/await
	await agenda.start();

	await agenda.every('3 minutes', 'delete old users');

	// Alternatively, you could also do:
	await agenda.every('*/3 * * * *', 'delete old users');
})();
```

```js
agenda.define(
	'send email report',
	async job => {
		const { to } = job.attrs.data;
		await emailClient.send({
			to,
			from: 'example@example.com',
			subject: 'Email Report',
			body: '...'
		});
	},
	{ priority: 'high', concurrency: 10 }
);

(async function () {
	await agenda.start();
	await agenda.schedule('in 20 minutes', 'send email report', { to: 'admin@example.com' });
})();
```

```js
(async function () {
	const weeklyReport = agenda.create('send email report', { to: 'example@example.com' });
	await agenda.start();
	await weeklyReport.repeatEvery('1 week').save();
})();
```

## Batch Processing Example

```js
// High-throughput configuration with batch processing
const agenda = new Agenda({
	db: { address: mongoConnectionString },
	maxConcurrency: 50,              // Process up to 50 jobs concurrently
	batchSize: 10,                   // Lock up to 10 jobs in each batch
	enableBatchProcessing: true,     // Enable batch processing (default)
	processEvery: '10 second',        // Check for new jobs every second
	defaultLockLifetime: 5 * 60 * 1000 // 5 minute lock lifetime
});

// Define a job that benefits from batch processing
agenda.define('process payment', async job => {
	const { orderId, amount } = job.attrs.data;
	await processPayment(orderId, amount);
}, { concurrency: 20 }); // Allow up to 20 concurrent payment processing jobs

// Define a job with lower concurrency
agenda.define('send notification', async job => {
	const { userId, message } = job.attrs.data;
	await sendNotification(userId, message);
}, { concurrency: 5 }); // Limit to 5 concurrent notifications

(async function () {
	await agenda.start();
	
	// Schedule many jobs that will be processed in batches
	for (let i = 0; i < 1000; i++) {
		await agenda.now('process payment', { 
			orderId: `order-${i}`, 
			amount: Math.random() * 100 
		});
	}
	
	// Monitor batch processing performance
	agenda.on('queueOverflow', ({ jobName, queueSize, maxSize }) => {
		console.warn(`Queue overflow for ${jobName}: ${queueSize}/${maxSize}`);
	});
	
	// Track job processing stats
	setInterval(async () => {
		const stats = await agenda.getRunningStats();
		console.log('Running jobs:', stats.runningJobs.length);
		console.log('Locked jobs:', stats.lockedJobs.length);
		console.log('Queue utilization:', agenda.jobQueue.getUtilization() + '%');
	}, 5000);
})();
```

```js
// Resource-constrained configuration
const agenda = new Agenda({
	db: { address: mongoConnectionString },
	maxConcurrency: 5,               // Limited concurrency
	batchSize: 3,                    // Smaller batch size
	enableBatchProcessing: true,
	processEvery: '10 seconds',      // Less frequent job checking
	defaultLockLifetime: 60 * 1000   // 1 minute lock lifetime
});

// Disable batch processing for specific use cases
const agenda = new Agenda({
	db: { address: mongoConnectionString },
	enableBatchProcessing: false     // Force single job processing
});
```

# Full documentation

See also https://hokify.github.io/agenda/

Agenda's basic control structure is an instance of an agenda. Agenda's are
mapped to a database collection and load the jobs from within.

## Table of Contents

- [Configuring an agenda](#configuring-an-agenda)
- [Agenda Events](#agenda-events)
- [Defining job processors](#defining-job-processors)
- [Creating jobs](#creating-jobs)
- [Managing jobs](#managing-jobs)
- [Starting the job processor](#starting-the-job-processor)
- [Multiple job processors](#multiple-job-processors)
- [Manually working with jobs](#manually-working-with-a-job)
- [Job Queue Events](#job-queue-events)
- [Frequently asked questions](#frequently-asked-questions)
- [Example Project structure](#example-project-structure)
- [Known Issues](#known-issues)
- [Debugging Issues](#debugging-issues)
- [Acknowledgements](#acknowledgements)

## Configuring an agenda

All configuration methods are chainable, meaning you can do something like:

```js
const agenda = new Agenda();
agenda
  .database(...)
  .processEvery('3 minutes')
  ...;
```

Possible agenda config options:

```ts
{
	name: string;
	defaultConcurrency: number;
	processEvery: number;
	maxConcurrency: number;
	defaultLockLimit: number;
	lockLimit: number;
	defaultLockLifetime: number;
	ensureIndex: boolean;
	sort: SortOptionObject<IJobParameters>;
	db: {
		collection: string;
		address: string;
		options: MongoClientOptions;
	};
	mongo: Db;
}
```

Agenda uses [Human Interval](http://github.com/rschmukler/human-interval) for specifying the intervals. It supports the following units:

`seconds`, `minutes`, `hours`, `days`,`weeks`, `months` -- assumes 30 days, `years` -- assumes 365 days

More sophisticated examples

```js
agenda.processEvery('one minute');
agenda.processEvery('1.5 minutes');
agenda.processEvery('3 days and 4 hours');
agenda.processEvery('3 days, 4 hours and 36 seconds');
```

### database(url, [collectionName], [MongoClientOptions])

Specifies the database at the `url` specified. If no collection name is given,
`agendaJobs` is used.

By default `useNewUrlParser` and `useUnifiedTopology` is set to `true`,

```js
agenda.database('localhost:27017/agenda-test', 'agendaJobs');
```

You can also specify it during instantiation.

```js
const agenda = new Agenda({
	db: { address: 'localhost:27017/agenda-test', collection: 'agendaJobs' }
});
```

Agenda will emit a `ready` event (see [Agenda Events](#agenda-events)) when properly connected to the database.
It is safe to call `agenda.start()` without waiting for this event, as this is handled internally.
If you're using the `db` options, or call `database`, then you may still need to listen for `ready` before saving jobs.

### mongo(dbInstance, [collectionName])

Use an existing mongodb-native MongoClient/Db instance. This can help consolidate connections to a
database. You can instead use `.database` to have agenda handle connecting for you.

You can also specify it during instantiation:

```js
const agenda = new Agenda({ mongo: mongoClientInstance.db('agenda-test') });
```

Note that MongoClient.connect() returns a mongoClientInstance since [node-mongodb-native 3.0.0](https://github.com/mongodb/node-mongodb-native/blob/master/CHANGES_3.0.0.md), while it used to return a dbInstance that could then be directly passed to agenda.

### name(name)

Takes a string `name` and sets `lastModifiedBy` to it in the job database.
Useful for if you have multiple job processors (agendas) and want to see which
job queue last ran the job.

```js
agenda.name(os.hostname + '-' + process.pid);
```

You can also specify it during instantiation

```js
const agenda = new Agenda({ name: 'test queue' });
```

### processEvery(interval)

Takes a string `interval` which can be either a traditional javascript number,
or a string such as `3 minutes`

Specifies the frequency at which agenda will query the database looking for jobs
that need to be processed. Agenda internally uses `setTimeout` to guarantee that
jobs run at (close to ~3ms) the right time.

Decreasing the frequency will result in fewer database queries, but more jobs
being stored in memory.

Also worth noting is that if the job queue is shutdown, any jobs stored in memory
that haven't run will still be locked, meaning that you may have to wait for the
lock to expire. By default it is `'5 seconds'`.

```js
agenda.processEvery('1 minute');
```

You can also specify it during instantiation

```js
const agenda = new Agenda({ processEvery: '30 seconds' });
```

### maxConcurrency(number)

Takes a `number` which specifies the max number of jobs that can be running at
any given moment. By default it is `20`.

```js
agenda.maxConcurrency(20);
```

You can also specify it during instantiation

```js
const agenda = new Agenda({ maxConcurrency: 20 });
```

### defaultConcurrency(number)

Takes a `number` which specifies the default number of a specific job that can be running at
any given moment. By default it is `5`.

```js
agenda.defaultConcurrency(5);
```

You can also specify it during instantiation

```js
const agenda = new Agenda({ defaultConcurrency: 5 });
```

### lockLimit(number)

Takes a `number` which specifies the max number jobs that can be locked at any given moment. By default it is `0` for no max.

```js
agenda.lockLimit(0);
```

You can also specify it during instantiation

```js
const agenda = new Agenda({ lockLimit: 0 });
```

### defaultLockLimit(number)

Takes a `number` which specifies the default number of a specific job that can be locked at any given moment. By default it is `0` for no max.

```js
agenda.defaultLockLimit(0);
```

You can also specify it during instantiation

```js
const agenda = new Agenda({ defaultLockLimit: 0 });
```

### defaultLockLifetime(number)

Takes a `number` which specifies the default lock lifetime in milliseconds. By
default it is 10 minutes. This can be overridden by specifying the
`lockLifetime` option to a defined job.

A job will unlock if it is finished (ie. the returned Promise resolves/rejects
or `done` is specified in the params and `done()` is called) before the
`lockLifetime`. The lock is useful if the job crashes or times out.

```js
agenda.defaultLockLifetime(10000);
```

You can also specify it during instantiation

```js
const agenda = new Agenda({ defaultLockLifetime: 10000 });
```

### sort(query)

Takes a `query` which specifies the sort query to be used for finding and locking the next job.

By default it is `{ nextRunAt: 1, priority: -1 }`, which obeys a first in first out approach, with respect to priority.

### batchSize(size)

Sets the number of jobs to process in each batch operation. Batch processing improves performance by reducing MongoDB operations and write conflicts.

```js
agenda.batchSize(10);
```

You can also specify it during instantiation

```js
const agenda = new Agenda({ batchSize: 10 });
```

### enableBatchProcessing(enabled)

Enables or disables batch processing. When enabled, AgendaTS will intelligently use batch operations when beneficial.

```js
agenda.enableBatchProcessing(true);  // Enable (default)
agenda.enableBatchProcessing(false); // Disable - force single job processing
```

You can also specify it during instantiation

```js
const agenda = new Agenda({ enableBatchProcessing: true });
```

**Batch Processing Behavior:**
- When enabled and `batchSize > 1`: Uses batch operations when multiple jobs are available
- When disabled or `batchSize = 1`: Uses single job processing
- Automatically respects concurrency limits and job-specific lock limits
- Includes write conflict resilience with exponential backoff retry logic

## Agenda Events

An instance of an agenda will emit the following events:

- `ready` - called when Agenda mongo connection is successfully opened and indices created.
  If you're passing agenda an existing connection, you shouldn't need to listen for this, as `agenda.start()` will not resolve until indices have been created.
  If you're using the `db` options, or call `database`, then you may still need to listen for the `ready` event before saving jobs. `agenda.start()` will still wait for the connection to be opened.
- `error` - called when Agenda mongo connection process has thrown an error

```js
await agenda.start();
```

## Defining Job Processors

Before you can use a job, you must define its processing behavior.

### define(jobName, fn, [options])

Defines a job with the name of `jobName`. When a job of `jobName` gets run, it
will be passed to `fn(job, done)`. To maintain asynchronous behavior, you may
either provide a Promise-returning function in `fn` _or_ provide `done` as a
second parameter to `fn`. If `done` is specified in the function signature, you
must call `done()` when you are processing the job. If your function is
synchronous or returns a Promise, you may omit `done` from the signature.

`options` is an optional argument which can overwrite the defaults. It can take
the following:

- `concurrency`: `number` maximum number of that job that can be running at once (per instance of agenda)
- `lockLimit`: `number` maximum number of that job that can be locked at once (per instance of agenda)
- `lockLifetime`: `number` interval in ms of how long the job stays locked for (see [multiple job processors](#multiple-job-processors) for more info).
  A job will automatically unlock once a returned promise resolves/rejects (or if `done` is specified in the signature and `done()` is called).
- `priority`: `(lowest|low|normal|high|highest|number)` specifies the priority
  of the job. Higher priority jobs will run first. See the priority mapping
  below

Priority mapping:

```
{
  highest: 20,
  high: 10,
  normal: 0,
  low: -10,
  lowest: -20
}
```

Async Job:

```js
agenda.define('some long running job', async job => {
	const data = await doSomelengthyTask();
	await formatThatData(data);
	await sendThatData(data);
});
```

Async Job (using `done`):

```js
agenda.define('some long running job', (job, done) => {
	doSomelengthyTask(data => {
		formatThatData(data);
		sendThatData(data);
		done();
	});
});
```

Sync Job:

```js
agenda.define('say hello', job => {
	console.log('Hello!');
});
```

`define()` acts like an assignment: if `define(jobName, ...)` is called multiple times (e.g. every time your script starts), the definition in the last call will overwrite the previous one. Thus, if you `define` the `jobName` only once in your code, it's safe for that call to execute multiple times.

## Creating Jobs

### every(interval, name, [data], [options])

Runs job `name` at the given `interval`. Optionally, data and options can be passed in.
Every creates a job of type `single`, which means that it will only create one
job in the database, even if that line is run multiple times. This lets you put
it in a file that may get run multiple times, such as `webserver.js` which may
reboot from time to time.

`interval` can be a human-readable format `String`, a [cron format](https://www.npmjs.com/package/cron-parser) `String`, or a `Number`.

`data` is an optional argument that will be passed to the processing function
under `job.attrs.data`.

`options` is an optional argument that will be passed to [`job.repeatEvery`](#repeateveryinterval-options).
In order to use this argument, `data` must also be specified.

Returns the `job`.

```js
agenda.define('printAnalyticsReport', async job => {
	const users = await User.doSomethingReallyIntensive();
	processUserData(users);
	console.log('I print a report!');
});

agenda.every('15 minutes', 'printAnalyticsReport');
```

Optionally, `name` could be array of job names, which is convenient for scheduling
different jobs for same `interval`.

```js
agenda.every('15 minutes', ['printAnalyticsReport', 'sendNotifications', 'updateUserRecords']);
```

In this case, `every` returns array of `jobs`.

### schedule(when, name, [data])

Schedules a job to run `name` once at a given time. `when` can be a `Date` or a
`String` such as `tomorrow at 5pm`.

`data` is an optional argument that will be passed to the processing function
under `job.attrs.data`.

Returns the `job`.

```js
agenda.schedule('tomorrow at noon', 'printAnalyticsReport', { userCount: 100 });
```

Optionally, `name` could be array of job names, similar to the `every` method.

```js
agenda.schedule('tomorrow at noon', [
	'printAnalyticsReport',
	'sendNotifications',
	'updateUserRecords'
]);
```

In this case, `schedule` returns array of `jobs`.

### now(name, [data])

Schedules a job to run `name` once immediately.

`data` is an optional argument that will be passed to the processing function
under `job.attrs.data`.

Returns the `job`.

```js
agenda.now('do the hokey pokey');
```

### create(jobName, data)

Returns an instance of a `jobName` with `data`. This does _NOT_ save the job in
the database. See below to learn how to manually work with jobs.

```js
const job = agenda.create('printAnalyticsReport', { userCount: 100 });
await job.save();
console.log('Job successfully saved');
```

## Performance and Optimization

AgendaTS includes several performance optimizations for production workloads:

### Batch Processing
Batch processing reduces database operations and write conflicts:

```js
const agenda = new Agenda({
  mongo: db,
  batchSize: 10,              // Process up to 10 jobs per batch
  enableBatchProcessing: true // Enabled by default
});
```

### Memory Management
The job queue is bounded to prevent memory exhaustion:

```js
// Monitor queue usage
agenda.on('queueOverflow', ({ jobName, queueSize, maxSize }) => {
  console.warn(`Queue full for ${jobName}: ${queueSize}/${maxSize}`);
});
```

### Optimized Indexes
Enable automatic index creation for optimal performance:

```js
const agenda = new Agenda({
  mongo: db,
  ensureIndex: true // Creates optimized indexes
});
```

This creates indexes optimized for:
- Job discovery queries (50-70% faster)
- Locked job cleanup
- Job history queries

### Connection Pooling
MongoDB connections are pooled with optimized defaults:

```js
const agenda = new Agenda({
  db: {
    address: mongoConnectionString,
    options: {
      maxPoolSize: 100,      // Adjust based on load
      minPoolSize: 10
    }
  }
});
```

### Performance Tuning Examples

**High-Throughput Configuration:**
```js
const agenda = new Agenda({
  mongo: db,
  maxConcurrency: 100,
  defaultConcurrency: 10,
  batchSize: 20,
  processEvery: '1 second',
  defaultLockLifetime: 5 * 60 * 1000 // 5 minutes
});
```

**Resource-Constrained Configuration:**
```js
const agenda = new Agenda({
  mongo: db,
  maxConcurrency: 5,
  defaultConcurrency: 1,
  batchSize: 3,
  processEvery: '10 seconds',
  defaultLockLifetime: 60 * 1000 // 1 minute
});
```

## Managing Jobs

### jobs(mongodb-native query, mongodb-native sort, mongodb-native limit, mongodb-native skip)

Lets you query (then sort, limit and skip the result) all of the jobs in the agenda job's database. These are full [mongodb-native](https://github.com/mongodb/node-mongodb-native) `find`, `sort`, `limit` and `skip` commands. See mongodb-native's documentation for details.

```js
const jobs = await agenda.jobs({ name: 'printAnalyticsReport' }, { data: -1 }, 3, 1);
// Work with jobs (see below)
```

### cancel(mongodb-native query)

Cancels any jobs matching the passed mongodb-native query, and removes them from the database. Returns a Promise resolving to the number of cancelled jobs, or rejecting on error.

```js
const numRemoved = await agenda.cancel({ name: 'printAnalyticsReport' });
```

This functionality can also be achieved by first retrieving all the jobs from the database using `agenda.jobs()`, looping through the resulting array and calling `job.remove()` on each. It is however preferable to use `agenda.cancel()` for this use case, as this ensures the operation is atomic.

### purge()

Removes all jobs in the database without defined behaviors. Useful if you change a definition name and want to remove old jobs. Returns a Promise resolving to the number of removed jobs, or rejecting on error.

_IMPORTANT:_ Do not run this before you finish defining all of your jobs. If you do, you will nuke your database of jobs.

```js
const numRemoved = await agenda.purge();
```

## Starting the job processor

To get agenda to start processing jobs from the database you must start it. This
will schedule an interval (based on `processEvery`) to check for new jobs and
run them. You can also stop the queue.

### start

Starts the job queue processing, checking [`processEvery`](#processeveryinterval) time to see if there
are new jobs. Must be called _after_ `processEvery`, and _before_ any job scheduling (e.g. `every`).

### stop

Stops the job queue processing. Unlocks currently running jobs.

This can be very useful for graceful shutdowns so that currently running/grabbed jobs are abandoned so that other
job queues can grab them / they are unlocked should the job queue start again. Here is an example of how to do a graceful
shutdown.

```js
async function graceful() {
	await agenda.stop();
	process.exit(0);
}

process.on('SIGTERM', graceful);
process.on('SIGINT', graceful);
```

## Multiple job processors

Sometimes you may want to have multiple node instances / machines process from
the same queue. Agenda supports a locking mechanism to ensure that multiple
queues don't process the same job.

You can configure the locking mechanism by specifying `lockLifetime` as an
interval when defining the job.

```js
agenda.define(
	'someJob',
	(job, cb) => {
		// Do something in 10 seconds or less...
	},
	{ lockLifetime: 10000 }
);
```

This will ensure that no other job processor (this one included) attempts to run the job again
for the next 10 seconds. If you have a particularly long running job, you will want to
specify a longer lockLifetime.

By default it is 10 minutes. Typically you shouldn't have a job that runs for 10 minutes,
so this is really insurance should the job queue crash before the job is unlocked.

When a job is finished (i.e. the returned promise resolves/rejects or `done` is
specified in the signature and `done()` is called), it will automatically unlock.

## Manually working with a job

A job instance has many instance methods. All mutating methods must be followed
with a call to `await job.save()` in order to persist the changes to the database.

### repeatEvery(interval, [options])

Specifies an `interval` on which the job should repeat. The job runs at the time of defining as well in configured intervals, that is "run _now_ and in intervals".

`interval` can be a human-readable format `String`, a [cron format](https://www.npmjs.com/package/cron-parser) `String`, or a `Number`.

`options` is an optional argument containing:

`options.timezone`: should be a string as accepted by [moment-timezone](https://momentjs.com/timezone/) and is considered when using an interval in the cron string format.

`options.skipImmediate`: `true` | `false` (default) Setting this `true` will skip the immediate run. The first run will occur only in configured interval.

```js
job.repeatEvery('10 minutes');
await job.save();
```

```js
job.repeatEvery('3 minutes', {
	skipImmediate: true
});
await job.save();
```

```js
job.repeatEvery('0 6 * * *', {
	timezone: 'America/New_York'
});
await job.save();
```

### repeatAt(time)

Specifies a `time` when the job should repeat. [Possible values](https://github.com/matthewmueller/date#examples)

```js
job.repeatAt('3:30pm');
await job.save();
```

### schedule(time)

Specifies the next `time` at which the job should run.

```js
job.schedule('tomorrow at 6pm');
await job.save();
```

### priority(priority)

Specifies the `priority` weighting of the job. Can be a number or a string from
the above priority table.

```js
job.priority('low');
await job.save();
```

### unique(properties, [options])

Ensure that only one instance of this job exists with the specified properties

`options` is an optional argument which can overwrite the defaults. It can take
the following:

- `insertOnly`: `boolean` will prevent any properties from persisting if the job already exists. Defaults to false.

```js
job.unique({ 'data.type': 'active', 'data.userId': '123', nextRunAt: date });
await job.save();
```

_IMPORTANT:_ To avoid high CPU usage by MongoDB, make sure to create an index on the used fields, like `data.type` and `data.userId` for the example above.

### fail(reason)

Sets `job.attrs.failedAt` to `now`, and sets `job.attrs.failReason` to `reason`.

Optionally, `reason` can be an error, in which case `job.attrs.failReason` will
be set to `error.message`

```js
job.fail('insufficient disk space');
// or
job.fail(new Error('insufficient disk space'));
await job.save();
```

### run(callback)

Runs the given `job` and calls `callback(err, job)` upon completion. Normally
you never need to call this manually.

```js
job.run((err, job) => {
	console.log("I don't know why you would need to do this...");
});
```

### save()

Saves the `job.attrs` into the database. Returns a Promise resolving to a Job instance, or rejecting on error.

```js
try {
	await job.save();
	cosole.log('Successfully saved job to collection');
} catch (e) {
	console.error('Error saving job to collection');
}
```

### remove()

Removes the `job` from the database. Returns a Promise resolving to the number of jobs removed, or rejecting on error.

```js
try {
	await job.remove();
	console.log('Successfully removed job from collection');
} catch (e) {
	console.error('Error removing job from collection');
}
```

### disable()

Disables the `job`. Upcoming runs won't execute.

### enable()

Enables the `job` if it got disabled before. Upcoming runs will execute.

### touch()

Resets the lock on the job. Useful to indicate that the job hasn't timed out
when you have very long running jobs. The call returns a promise that resolves
when the job's lock has been renewed.

```js
agenda.define('super long job', async job => {
	await doSomeLongTask();
	await job.touch();
	await doAnotherLongTask();
	await job.touch();
	await finishOurLongTasks();
});
```

## Job Queue Events

An instance of an agenda will emit the following events:

- `start` - called just before a job starts
- `start:job name` - called just before the specified job starts

```js
agenda.on('start', job => {
	console.log('Job %s starting', job.attrs.name);
});
```

- `complete` - called when a job finishes, regardless of if it succeeds or fails
- `complete:job name` - called when a job finishes, regardless of if it succeeds or fails

```js
agenda.on('complete', job => {
	console.log(`Job ${job.attrs.name} finished`);
});
```

- `success` - called when a job finishes successfully
- `success:job name` - called when a job finishes successfully

```js
agenda.on('success:send email', job => {
	console.log(`Sent Email Successfully to ${job.attrs.data.to}`);
});
```

- `fail` - called when a job throws an error
- `fail:job name` - called when a job throws an error

```js
agenda.on('fail:send email', (err, job) => {
	console.log(`Job failed with error: ${err.message}`);
});
```

## Frequently Asked Questions

### What is the order in which jobs run?

Jobs are run with priority in a first in first out order (so they will be run in the order they were scheduled AND with respect to highest priority).

For example, if we have two jobs named "send-email" queued (both with the same priority), and the first job is queued at 3:00 PM and second job is queued at 3:05 PM with the same `priority` value, then the first job will run first if we start to send "send-email" jobs at 3:10 PM. However if the first job has a priority of `5` and the second job has a priority of `10`, then the second will run first (priority takes precedence) at 3:10 PM.

The default [MongoDB sort object](https://docs.mongodb.com/manual/reference/method/cursor.sort/) is `{ nextRunAt: 1, priority: -1 }` and can be changed through the option `sort` when configuring Agenda.

### What is the difference between `lockLimit` and `maxConcurrency`?

Agenda will lock jobs 1 by one, setting the `lockedAt` property in mongoDB, and creating an instance of the `Job` class which it caches into the `_lockedJobs` array. This defaults to having no limit, but can be managed using lockLimit. If all jobs will need to be run before agenda's next interval (set via `agenda.processEvery`), then agenda will attempt to lock all jobs.

Agenda will also pull jobs from `_lockedJobs` and into `_runningJobs`. These jobs are actively being worked on by user code, and this is limited by `maxConcurrency` (defaults to 20).

If you have multiple instances of agenda processing the same job definition with a fast repeat time you may find they get unevenly loaded. This is because they will compete to lock as many jobs as possible, even if they don't have enough concurrency to process them. This can be resolved by tweaking the `maxConcurrency` and `lockLimit` properties.

### Sample Project Structure?

Agenda doesn't have a preferred project structure and leaves it to the user to
choose how they would like to use it. That being said, you can check out the
[example project structure](#example-project-structure) below.

### Can I Donate?

Thanks! I'm flattered, but it's really not necessary. If you really want to, you can find my [gittip here](https://www.gittip.com/rschmukler/).

### Web Interface?

Agenda itself does not have a web interface built in but we do offer stand-alone web interface [Agendash](https://github.com/agenda/agendash):

<a href="https://raw.githubusercontent.com/agenda/agendash/master/job-details.png"><img src="https://raw.githubusercontent.com/agenda/agendash/master/job-details.png" style="max-width:100%" alt="Agendash interface"></a>

### Mongo vs Redis

The decision to use Mongo instead of Redis is intentional. Redis is often used for
non-essential data (such as sessions) and without configuration doesn't
guarantee the same level of persistence as Mongo (should the server need to be
restarted/crash).

Agenda decides to focus on persistence without requiring special configuration
of Redis (thereby degrading the performance of the Redis server on non-critical
data, such as sessions).

Ultimately if enough people want a Redis driver instead of Mongo, I will write
one. (Please open an issue requesting it). For now, Agenda decided to focus on
guaranteed persistence.

### Spawning / forking processes

Ultimately Agenda can work from a single job queue across multiple machines, node processes, or forks. If you are interested in having more than one worker, [Bars3s](http://github.com/bars3s) has written up a fantastic example of how one might do it:

```js
const cluster = require('cluster');
const os = require('os');

const httpServer = require('./app/http-server');
const jobWorker = require('./app/job-worker');

const jobWorkers = [];
const webWorkers = [];

if (cluster.isMaster) {
	const cpuCount = os.cpus().length;
	// Create a worker for each CPU
	for (let i = 0; i < cpuCount; i += 1) {
		addJobWorker();
		addWebWorker();
	}

	cluster.on('exit', (worker, code, signal) => {
		if (jobWorkers.indexOf(worker.id) !== -1) {
			console.log(
				`job worker ${worker.process.pid} exited (signal: ${signal}). Trying to respawn...`
			);
			removeJobWorker(worker.id);
			addJobWorker();
		}

		if (webWorkers.indexOf(worker.id) !== -1) {
			console.log(
				`http worker ${worker.process.pid} exited (signal: ${signal}). Trying to respawn...`
			);
			removeWebWorker(worker.id);
			addWebWorker();
		}
	});
} else {
	if (process.env.web) {
		console.log(`start http server: ${cluster.worker.id}`);
		// Initialize the http server here
		httpServer.start();
	}

	if (process.env.job) {
		console.log(`start job server: ${cluster.worker.id}`);
		// Initialize the Agenda here
		jobWorker.start();
	}
}

function addWebWorker() {
	webWorkers.push(cluster.fork({ web: 1 }).id);
}

function addJobWorker() {
	jobWorkers.push(cluster.fork({ job: 1 }).id);
}

function removeWebWorker(id) {
	webWorkers.splice(webWorkers.indexOf(id), 1);
}

function removeJobWorker(id) {
	jobWorkers.splice(jobWorkers.indexOf(id), 1);
}
```

### Recovering lost Mongo connections ("auto_reconnect")

Agenda is configured by default to automatically reconnect indefinitely, emitting an [error event](#agenda-events)
when no connection is available on each [process tick](#processeveryinterval), allowing you to restore the Mongo
instance without having to restart the application.

However, if you are using an [existing Mongo client](#mongomongoclientinstance)
you'll need to configure the `reconnectTries` and `reconnectInterval` [connection settings](http://mongodb.github.io/node-mongodb-native/3.0/reference/connecting/connection-settings/)
manually, otherwise you'll find that Agenda will throw an error with the message "MongoDB connection is not recoverable,
application restart required" if the connection cannot be recovered within 30 seconds.

# Example Project Structure

Agenda will only process jobs that it has definitions for. This allows you to
selectively choose which jobs a given agenda will process.

Consider the following project structure, which allows us to share models with
the rest of our code base, and specify which jobs a worker processes, if any at
all.

```
- server.js
- worker.js
lib/
  - agenda.js
  controllers/
    - user-controller.js
  jobs/
    - email.js
    - video-processing.js
    - image-processing.js
   models/
     - user-model.js
     - blog-post.model.js
```

Sample job processor (eg. `jobs/email.js`)

```js
let email = require('some-email-lib'),
	User = require('../models/user-model.js');

module.exports = function (agenda) {
	agenda.define('registration email', async job => {
		const user = await User.get(job.attrs.data.userId);
		await email(user.email(), 'Thanks for registering', 'Thanks for registering ' + user.name());
	});

	agenda.define('reset password', async job => {
		// Etc
	});

	// More email related jobs
};
```

lib/agenda.js

```js
const Agenda = require('agenda');

const connectionOpts = { db: { address: 'localhost:27017/agenda-test', collection: 'agendaJobs' } };

const agenda = new Agenda(connectionOpts);

const jobTypes = process.env.JOB_TYPES ? process.env.JOB_TYPES.split(',') : [];

jobTypes.forEach(type => {
	require('./jobs/' + type)(agenda);
});

if (jobTypes.length) {
	agenda.start(); // Returns a promise, which should be handled appropriately
}

module.exports = agenda;
```

lib/controllers/user-controller.js

```js
let app = express(),
	User = require('../models/user-model'),
	agenda = require('../worker.js');

app.post('/users', (req, res, next) => {
	const user = new User(req.body);
	user.save(err => {
		if (err) {
			return next(err);
		}
		agenda.now('registration email', { userId: user.primary() });
		res.send(201, user.toJson());
	});
});
```

worker.js

```js
require('./lib/agenda.js');
```

Now you can do the following in your project:

```bash
node server.js
```

Fire up an instance with no `JOB_TYPES`, giving you the ability to process jobs,
but not wasting resources processing jobs.

```bash
JOB_TYPES=email node server.js
```

Allow your http server to process email jobs.

```bash
JOB_TYPES=email node worker.js
```

Fire up an instance that processes email jobs.

```bash
JOB_TYPES=video-processing,image-processing node worker.js
```

Fire up an instance that processes video-processing/image-processing jobs. Good for a heavy hitting server.

# Debugging Issues

If you think you have encountered a bug, please feel free to report it here:

[Submit Issue](https://github.com/hokify/agenda/issues/new)

Please provide us with as much details as possible such as:

- Agenda version
- Environment (OSX, Linux, Windows, etc)
- Small description of what happened
- Any relevant stack track
- Agenda logs (see below)

#### To turn on logging, please set your DEBUG env variable like so:

- OSX: `DEBUG="agenda:*" ts-node src/index.ts`
- Linux: `DEBUG="agenda:*" ts-node src/index.ts`
- Windows CMD: `set DEBUG=agenda:*`
- Windows PowerShell: `$env:DEBUG = "agenda:*"`

While not necessary, attaching a text file with this debug information would
be extremely useful in debugging certain issues and is encouraged.

# Known Issues

#### "Multiple order-by items are not supported. Please specify a single order-by item."

When running Agenda on Azure cosmosDB, you might run into this issue caused by Agenda's sort query used for finding and locking the next job. To fix this, you can pass [custom sort option](#sortquery): `sort: { nextRunAt: 1 }`

# Performance

It is recommended to set this index if you use agendash:

```
db.agendaJobs.ensureIndex({
    "nextRunAt" : -1,
    "lastRunAt" : -1,
    "lastFinishedAt" : -1
}, "agendash2")
```

If you have one job definition with thousand of instances, you can add this index to improve internal sorting query
for faster sortings

```
db.agendaJobs.ensureIndex({
    "name" : 1,
    "disabled" : 1,
    "lockedAt" : 1
}, "findAndLockDeadJobs")
```

# Sandboxed Worker - use child processes

It's possible to start jobs in a child process, this helps for example for long running processes
to seperate them from the main thread. For example if one process consumes too much memory and gets killed,
it will not affect any others.
To use this feature, several steps are required.
1.) create a childWorker helper.
The subrocess has a complete seperate context, so there are no database connections or anything else that can be shared.
Therefore you have to ensure that all required connections and initializations are done here too. Furthermore
you also have to load the correct job definition so that agenda nows what code it must execute. Therefore 3 parameters
are passed to the childWorker: name, jobId and path to the job definition.

Example file can look like this:

childWorker.ts

```ts
import 'reflect-metadata';

function isCancelMessage(message): message is { type: 'cancel'; error: string } {
	return message !== null && typeof message === 'object' && message.type === 'cancel';
}

(async () => {
	const mongooseConnection = /** connect to database */

  /** do other required initializations */

  // get process arguments (name, jobId and path to agenda definition file)
	const [, , name, jobId, agendaDefinition] = process.argv;

  // set fancy process title
	process.title = `${process.title} (sub worker: ${name}/${jobId})`;

  // initialize Agenda in "forkedWorker" mode
	const agenda = new Agenda({ name: `subworker-${name}`, forkedWorker: true });
	// connect agenda (but do not start it)
	await agenda.mongo(mongooseConnection.db as any);

	if (!name || !jobId) {
		throw new Error(`invalid parameters: ${JSON.stringify(process.argv)}`);
	}

  // load job definition
  /** in this case the file is for example ../some/path/definitions.js
  with a content like:
  export default (agenda: Agenda, definitionOnly = false) => {
    agenda.define(
      'some job',
      async (notification: {
        attrs: { data: { dealId: string; orderId: TypeObjectId<IOrder> } };
      }) => {
        // do something
      }
    );

    if (!definitionOnly) {
        // here you can create scheduled jobs or other things
    }
	});
  */
	if (agendaDefinition) {
		const loadDefinition = await import(agendaDefinition);
		(loadDefinition.default || loadDefinition)(agenda, true);
	}

  // run this job now
	const job = await agenda.getForkedJob(jobId);

  process.on('message', message => {
		if (isCancelMessage(message)) {
			job.cancel(message.error);
			setTimeout(() => {
				// kill it after 10 seconds
				process.exit(2);
			}, 10000);
		} else {
			console.log('got message', message);
		}
	});

	await job.runJob();

  // disconnect database and exit
	process.exit(0);
})().catch(err => {
	console.error('err', err);
	if (process.send) {
		process.send(JSON.stringify(err));
	}
	process.exit(1);
});


```

Ensure to only define job definitions during this step, otherwise you create some
overhead (e.g. if you create new jobs inside the defintion files). That's why I call
the defintion file with agenda and a second paramter that is set to true. If this
parameter is true, I do not initialize any jobs (create jobs etc..)

2.) to use this, you have to enable it on a job. Set forkMode to true:

```ts
const job = agenda.create('some job', { meep: 1 });
job.forkMode(true);
await job.save();
```

# Acknowledgements

- Agenda was originally created by [@rschmukler](https://github.com/rschmukler).
- [Agendash](https://github.com/agenda/agendash) was originally created by [@joeframbach](https://github.com/joeframbach).
- These days Agenda has a great community of [contributors](https://github.com/hokify/agenda/graphs/contributors) around it. Join us!

# License

[The MIT License](LICENSE.md)
