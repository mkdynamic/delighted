[![Build Status](https://travis-ci.org/delighted/delighted-node.svg?branch=master)](https://travis-ci.org/delighted/delighted-node)

# Delighted API Node Client

Official Node.js client for the [Delighted API](https://delighted.com/docs/api).

**Note:** This is intended for server-side use only and does not support client-side JavaScript.

## Installation

Run `npm install delighted --save` to install.

## Configuration

To get started, you need to require the client and configure it with your secret API key. You can require, configure and initialize the client in one go:

```javascript
var delighted = require('delighted')('YOUR_API_KEY');
```

For further options, read the [advanced configuration section](#advanced-configuration).

**Note:** Your API key is secret, and you should treat it like a password. You can find your API key in your Delighted account, under *Settings* > *API*.

## Usage

All resources can be accessed directly off of the `delighted` instance we created above. All actions immediately return a promise. In this initial example we'll create a person and log out their attributes when the promise resolves (finishes):

```javascript
var params = {
  email: 'jony@appleseed.com',
  name:  'Jony Appleseed',
  delay: 86400,
  properties: { plan: 'basic' } // an object to pass properties for segmentation
};

delighted.person.create(params).then(function(person) {
  console.log(person);
});
```

Previously subscribed people can be unsubscribed:

```javascript
delighted.unsubscribe.create({ person_email: 'jony@appleseed.com' });
```

Listing all people:

```javascript
// List all people, auto pagination
// Note: this method automatically handles rate limits asynchronously
delighted.person
  .list()
  .autoPagingEach((person) => {
    // Do something with `person`
  })
  .then(() => {
    console.log('Done iterating.');
  }, (error) => {});

// If you wish to stop the pagination at any point, just return false
var count = 0;
delighted.person
  .list()
  .autoPagingEach((person) => {
    // Do something with `person`
    count++;
    if (count >= 2) {
      // Autopagination will stop after 2 interations
      return false;
    }
  })
  .then(() => {
    // After stopping, this will still get executed
    console.log('Done iterating.');
  }, (error) => {});

// You can limit the numbers of max successive retries during auto rate limits handling
delighted.person
  .list()
  .autoPagingEach((person) => {
      // Do something with `person`
    },
    { auto_handle_rate_limits_max_retries: 12 }
  )
  .then(() => {
    console.log('Done iterating.');
  }, (error) => {});


// You can also handle rate limits and other errors yourself
delighted.person
  .list()
  .autoPagingEach((person) => {
    // Do something with `person`
  }, { auto_handle_rate_limits: false })
  .then(() => {
    console.log('Done iterating.');
  }, (error) => {
    if (error.type == 'TooManyRequestsError') {
      // Indicates how long to wait (in seconds) before making this request again
      console.log(error.retryAfter);
    } else if (error.type == 'PaginationError') {
      // General pagination error
      console.log(error.message);
    }
  });
```

Get a paginated list of people who have unsubscribed:

```javascript
delighted.unsubscribe.all({ page: 2}).then(function(responses) {
  responses.length; // => 20
});
```

Get a paginated list of people whose emails have bounced:

```javascript
delighted.bounce.all({ page: 2}).then(function(responses) {
  responses.length; // => 20
});
```

Deleting a person and all of the data associated with them:

```javascript
// Delete by person id
delighted.person.delete({ id: 42 })
// Delete by email address (must be E.164 format)
delighted.person.delete({ email: "test@example.com" })
// Delete by phone number
delighted.person.delete({ phone_number: "+14155551212" })
```

Pending survey requests can be deleted:

```javascript
delighted.surveyRequest.deletePending({ person_email: 'jony@appleseed.com' });
```

Responses can be created for somebody using their id. Note that the id is not the same as their email:

```javascript
delighted.surveyResponse.create({ person: person.id, score: 10 });
```

Get a paginated list of all responses:

```javascript
delighted.surveyResponse.all({ page: 2 }).then(function(responses) {
  responses.length; // => 20
});
```

Retrieve summary metrics of all responses:

```javascript
delighted.metrics.retrieve();
```

## Rate limits

If a request is rate limited, a `DelightedError` error is raised with a type of `TooManyRequestsError`. You can handle that error to implement exponential backoff or retry strategies. The error object provides a `.retry_after` property to tell you how many seconds you should wait before retrying. For example:

```js
instance.metrics.retrieve().then(
  function(metrics) { ... },
  function(error) {
    if (error.type === 'TooManyRequestsError') { // rate limited
      var retryAfterSeconds = error.retryAfter;
      // wait for retryAfterSeconds before retrying
      // add your retry strategy here ...
    } else {
      // some other type of error
    }
  }
);
```

## <a name="advanced-configuration"></a> Advanced configuration & testing

All of the connection details can be configured through the `delighted` constructor, primarily for testing purposes. The available configuration options are:

* `host` – defaults to `api.delighted.com`
* `port` – defaults to `443`
* `base` – defaults to `/v1`
* `headers` – defaults to specifying `application/json` for `Accept` and `Content-Type` and sets `User-Agent` to identify the Delighted API Node Client and version.
* `scheme` – defaults to `https`

Testing with real requests against a mock server is the easiest way to integration test your application. For convenience, and our own testing, a test server is provided with the `delighted` package. Below is an example of testing the `person` resource within an application:

```javascript
var delighted  = require('delighted');
var mockServer = require('delighted/server');
var instance   = delighted('DUMMY_API_KEY', {
  host:   'localhost',
  port:   5678,
  base:   '',
  scheme: 'http'
});

var mapping = {
  'POST /v1/people': {
    status: 201,
    body: { id: "1", email: 'foo@example.com', name: null, survey_scheduled_at: 1490298348 }
  }
};

var server = mockServer(5678, mapping);
```

Setting up the server only requires a port and a mapping. The mapping should match an exact endpoint and will send back a JSON body with the specified status code. With the server running you can then make a request:

```javascript
instance.person.create({ email: 'foo@example.com' }).then(
  function(person) {
    console.log(person.survey_scheduled_at); //=> 1490298348
  },
  function(error) {
    console.log(error.type); //=> ResourceValidationError
  }
);
```

When you are done reading responses from the server, be sure to close it.

```javascript
server.close();
```

## Contributing

1. Fork it
2. Create your feature branch (`git checkout -b my-new-feature`)
3. Run the tests (`npm run-script test`)
4. Commit your changes (`git commit -am 'Add some feature'`)
5. Push to the branch (`git push origin my-new-feature`)
6. Create new Pull Request

## Releasing

1. Decide on the new version number (let's call it `x`).
2. Update `CHANGELOG.md` with notes about the new version's changes.
3. Update the version number in `lib/Delighted.js` and `package.json` and create a git tag for the version number (use `git tag -l` to check the format).
4. Commit your changes to git and push the tag.
5. Run `npm publish --tag next` to upload a pre-release to the NPM package registry, use `npm publish --tag latest` to upload the final release

## Author

Originally by Sean McGary. Graciously transfered and now officially maintained by Delighted.
