# Simple LDAP Get

[![Travis Build](https://img.shields.io/travis/geekydatamonkey/simple-ldap-get.svg?style=flat)](https://travis-ci.org/geekydatamonkey/simple-ldap-get)
[![NPM Version](https://img.shields.io/npm/v/simple-ldap-get.svg)](https://www.npmjs.com/package/simple-ldap-get)

Get data from a LDAP. Nothing fancy.

Returns either a promise or a stream.

## Usage

```js
import SimpleLDAP from 'simple-ldap-get';

const settings = {
  url: 'ldap://0.0.0.0:1389',
  base: 'dc=users,dc=localhost',
  bind: { dn: 'cn=root', password: 'secret' }
}

// create a new client
const ldap = new SimpleLDAP(settings);

// setup a filter and attributes for your LDAP query
const filter = '(uid=artvandelay)';
const attributes = [
  'idNumber',
  'uid',
  'givenName',
  'sn',
  'telephoneNumber',
];

// Promise API
// if ldap.get() is followed by a then, a promise is returned
ldap
  .get(filter, attributes)
  .then(console.log);

// [{
//   dn: 'uid=artvandelay, dc=users, dc=localhost',
//   idNumber: 1234567,
//   uid: 'artvandelay',
//   givenName: 'Art',
//   sn: 'Vandelay',
//   telephoneNumber: '555-123-4567',
// }]

// Streams API
// if ldap.get() is followed by a pipe, a stream is returned.
ldap
  .get(filter, attributes)
  .pipe(process.stdout)

  // [{
  //   dn: 'uid=artvandelay, dc=users, dc=localhost',
  //   idNumber: 1234567,
  //   uid: 'artvandelay',
  //   givenName: 'Art',
  //   sn: 'Vandelay',
  //   telephoneNumber: '555-123-4567',
  // }]
```

## Streams Example

For large amounts of data, I recommend using the streams api. Streams can be piped directly to an http response. (If you use promises, all the data will load an buffer before the promise resolves).

In the example below, we use streams to remap some properties of LDAP data:

```js
import SimpleLDAP from 'simple-ldap-get';
import through from 'through2' // useful for streams

const settings = {
  url: 'ldap://0.0.0.0:1389',
  base: 'dc=users,dc=localhost',
  bind: { dn: 'cn=root', password: 'secret' }
}

// create a new client
const ldap = new SimpleLDAP(settings);

ldap.get()
  .pipe(through.obj(function transformIt(obj, _, done) {
    if (!obj) return done();

    // Transform the data, by relabeling `idNumber` as `id`,
    // `uid` as`username`, and create a fullName property.
    // Ditch the rest.
    this.push({
      id: obj.idNumber,
      username: obj.uid,
      fullName: `${obj.givenName} ${obj.sn}`,
    });
    return done();
  }))
  .pipe(process.stdout)
  .on('error', err => console.error(err))
  .on('finish', () => {
    // all done. Don't need the client anymore
    ldap.destroy();
  });

  // [{
  //   id: 1234567,
  //   username: 'artvandelay',
  //   fullName: 'Art Vandelay',
  // }, {
  //   id: 765432,
  //   username: 'ebenes',
  //   fullName: 'Elaine Benes',
  // }]
```

## API

### `ldap.get(filter, attributes)`

Parameters
  - `filter`: filters results.
  - `attributes`: a list of attributes to return

Returns
  - A promise if `ldap.get()` is followed by a `.then()`
  - A stream if `ldap.get()` is followed by a `.pipe()`

Technically, `ldap.get` returns an object with two methods:
`then()` and `pipe()`. `then()` invokes `ldap.getPromise()`
and `pipe()` invokes `ldap.getStream()`.

### `ldap.getPromise(filter, attributes)`
Returns a promise for the data.

### `ldap.getStream(filter, attributes)`
Returns a readable stream of data.

### `ldap.destroy()`
Destroys the connection to the LDAP server. Use when all done with LDAP client.

```js
  let ldap;

  // Using Promise
  ldap = new SimpleLDAP(settings);
  ldap.get('(uid=artvandelay)')
    .then(console.log)
    .catch(console.error)
    .finally(() => {
      // nuke it
      ldap.destroy();
    });

  // Using Stream
  ldap = new SimpleLDAP(settings);
  ldap.get()
      .pipe(process.stdout)
      .on('error', console.error);
      .on('finish', () => {
        ldap.destroy();
      });
```