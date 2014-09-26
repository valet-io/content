# Valet.io Realtime API Documentation

*__Disclaimer__: Our realtime API is still an early prerelease. Its structure may change as we move towards a stable release. We will always try to provide at least 48 hours notice of upcoming breaking changes, circumstances permitting. Non-breaking changes may happen at any time without notice.*

## Overview
Our realtime API is based on [Firebase](https://www.firebase.com/). Firebase supplies client libraries for [JavaScript](https://www.firebase.com/docs/web/) (browser and Node.js), [Objective-C/Swift](https://www.firebase.com/docs/ios/) (iOS and OS X) and [Java](https://www.firebase.com/docs/android/) (Android, desktop, server). This document primarily outlines the schema for campaign and pledge data. While code examples are provided in JavaScript where appropriate, you'll want to refer to [Firebase's own documentation](https://www.firebase.com/docs/) for a detailed description of API methods.

## Authentication and Setup

*__WIP__: This segment of the API is not quite finished yet. We've documented our plans here. We expect to have this available by October 24, 2014.*

You will need to authenticate before connecting to Firebase. While the realtime API's data is read-only for the time being, it still contains sensitive donor information, including names, emails, phone numbers, and street addresses.

**Never** include your authentication token on any publicly accessible application that is not protected by a secure password.

------------------

### Step 1: Obtain a Firebase Token

To authenticate, you will send a `POST` request to `https://auth.valet.io` with an empty body and your token in a header named `X-Valet-Token`.

```js
function authorizeValet (token) {
  return $.ajax({
    url: 'https://auth.valet.io',
    type: 'POST',
    headers: {
      'X-Valet-Token': token
    }
  });
}

authorizeValet('mytoken')
  .done(function (data) {
    // ready to authenticate with firebase
  });
```

The authentication server will respond with the following as JSON:

* `firebase_token` *(String)*: A token you will use to authenticate with Firebase before receiving data
* `firebase_endpoint` *(String)*: The root Firebase reference where your campaigns are located
* `expires_at` *(String)*: An [ISO 8601](http://en.wikipedia.org/wiki/ISO_8601) timestamp indicating when the `firebase_token` will expire. You can cache the Firebase token until then, but make sure to fetch a new token before the old one expires.

----------------------

### Step 2: Connect to Firebase

Once you have a Firebase token, you can connect and begin receiving events. You'll need your `campaign_id` for this. It's a uuid—a 36 character random string unique to your event. We'll provide it to you ahead of your event.

```js
var ref;
authorizeValet('mytoken')
  .done(function (data) {
    ref = new Firebase(data.firebase_endpoint).child(campaign_id)
    ref.auth(data.firebase_token, function (err) {
      if (!err) {
        // ready to attach events
        attachEvents(ref);
      }
      else {
        // something went wrong
        handleError(err);
      }
    });
  });
```

### Step 3: Attach Event Listeners

Firebase takes care of notifying you when there's new data. You have the flexibility to get as much or as little data as you'd like. You can subscribe to an event for every new pledge or just get updates when the fundraising total changes.

This first example will log the pledge total every time it changes:

```js
function attachEvents (ref) {
  ref.child('aggregates').child('total').on('value', function (snapshot) {
    console.log(snapshot.val());
  });
}
```

Next we'll try logging the donor name and pledge amount for each new pledge:

```js
function attachEvents (ref) {
  ref.child('pledges').on('child_added', function (snapshot) {
    var pledge = snapshot.val();
    console.log(pledge.donor.name, ' pledged $', pledge.amount);
  });
}
```

## Schema
Each campaign has the following keys:

* `aggregates` *(Object)*: Aggregate metrics for the campaign
  * `count` *(Number)*: The number of pledges made to the campaign
  * `total` *(Number)*: The sum of the amount for all pledges made to the campaign
* `options` *(Object)*: Configuration for the campaign
  * `starting_value` *(Number)*: A manually set starting point for the campaign. Normally you will add this to `aggregates.total` to generate a number to display on screen.
* `pledges` *(Object)*: An object containing the pledges to the campaign. The keys are the uuids of the pledges.

### Pledge Schema

* `id` *(String)*: uuid
* `amount` *(Number)*
* `anonymous` *(Boolean)*: If `true`, do not display the donor name on screen
* `created_at` *(Number)*: Timestamp
* `donor_id` *(String)*: uuid for the donor
* `donor` *(Object)*
  * `id` *(String)*: uuid, same as `donor_id`
  * `name` *(String)*: Donor name, as entered
* `.priority` *(Number)*: See [Queries and Data Order](#queries-and-data-order)

### Queries and Data Order

Keys in Firebase objects ordered by priority, a special property of all snapshots. Keys with assigned priorities appear in order from the lowest priority to the highest. Each pledge is assigned a priority equal to its timestamp (`pledge.created_at`). Pledges are therefore ordered starting with the earliest and ending with the latest. 

### Support

Please send questions about the Realtime API as well as question and feedback about the documentation to [Ben Drucker](mailto:ben@valet.io).
