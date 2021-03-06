<p align="center"><img width="465" height="170" src="https://raw.github.com/onury/accesscontrol/master/ac-logo.png" /></p>

[![build-status](https://img.shields.io/travis/onury/accesscontrol.svg?branch=master)](https://travis-ci.org/onury/accesscontrol)
[![npm](http://img.shields.io/npm/v/accesscontrol.svg)](https://www.npmjs.com/package/accesscontrol)
[![release](https://img.shields.io/github/release/onury/accesscontrol.svg)](https://github.com/onury/accesscontrol)
[![license](http://img.shields.io/npm/l/accesscontrol.svg)](https://github.com/onury/accesscontrol/blob/master/LICENSE)
[![dependencies](https://david-dm.org/onury/accesscontrol.svg)](https://david-dm.org/onury/accesscontrol)
[![maintained](https://img.shields.io/maintenance/yes/2017.svg)](https://github.com/onury/accesscontrol/graphs/commit-activity)
[![TypeScript](https://img.shields.io/badge/written%20in-%20TypeScript%20-6575ff.svg)](https://www.typescriptlang.org)    

> © 2017, Onur Yıldırım ([@onury](https://github.com/onury)). MIT License.

Role and Attribute based Access Control for Node.js  

`npm i accesscontrol --save`  

Many [RBAC][rbac] (Role-Based Access Control) implementations differ, but the basics is widely adopted since it simulates real life role (job) assignments. But while data is getting more and more complex; you need to define policies on resources, subjects or even environments. This is called [ABAC][abac] (Attribute-Based Access Control).

With the idea of merging the best features of the two (see this [NIST paper][nist-paper]); this library implements RBAC basics and also focuses on *resource* and *action* attributes.

### Core Features

- Chainable, friendly API.  
e.g. `ac.can(role).createOwn(resource)`
- Role hierarchical inheritance.
- Define grants at once (e.g. from database result) or one by one.
- Grant/deny permissions by attributes defined by glob notation (with nested object support).
- Ability to filter data (model) instance by allowed attributes.
- Ability to control access on "own" or "any" resources.
- Fast. (Grants are stored in memory, no database queries.)
- TypeScript support.

_In order to build on more solid foundations, this library (v1.5.0+) is completely re-written in TypeScript._

## Guide

```js
const AccessControl = require('accesscontrol');
// or:
// import { AccessControl } from 'accesscontrol';
```

### Basic Example

Define roles and grants one by one.
```js
const ac = new AccessControl();
ac.grant('user')                    // define new or modify existing role. also takes an array.
    .createOwn('video')             // equivalent to .createOwn('video', ['*'])
    .deleteOwn('video')
    .readAny('video')
  .grant('admin')                   // switch to another role without breaking the chain
    .extend('user')                 // inherit role capabilities. also takes an array
    .updateAny('video', ['title'])  // explicitly defined attributes
    .deleteAny('video');

const permission = ac.can('user').createOwn('video');
console.log(permission.granted);    // —> true
console.log(permission.attributes); // —> ['*'] (all attributes)

permission = ac.can('admin').updateAny('video');
console.log(permission.granted);    // —> true
console.log(permission.attributes); // —> ['title']
```

### Express.js Example

Check role permissions for the requested resource and action, if granted; respond with filtered attributes.

```js
const ac = new AccessControl(grants);
// ...
router.get('/videos/:title', function (req, res, next) {
    const permission = ac.can(req.user.role).readAny('video');
    if (permission.granted) {
        Video.find(req.params.title, function (err, data) {
            if (err || !data) return res.status(404).end();
            // filter data by permission attributes and send.
            res.json(permission.filter(data));
        });
    } else {
        // resource is forbidden for this user/role
        res.status(403).end();
    }
});
```

### Roles

You can create/define roles simply by calling `.grant(<role>)` or `.deny(<role>)` methods on an `AccessControl` instance.  

Roles can extend other roles.

```js
// user role inherits viewer role permissions
ac.grant('user').extend('viewer');
// admin role inherits both user and editor role permissions
ac.grant('admin').extend(['user', 'editor']);
// both admin and superadmin roles inherit moderator permissions
ac.grant(['admin', 'superadmin']).extend('moderator');
```

### Actions and Action-Attributes

[CRUD][crud] operations are the actions you can perform on a resource. There are two action-attributes which define the **possession** of the resource: *own* and *any*.

For example, an `admin` role can `create`, `read`, `update` or `delete` (CRUD) **any** `account` resource. But a `user` role might only `read` or `update` its **own** `account` resource.

<table>
    <thead>
        <tr>
            <th>Action</th>
            <th>Possession</th>
        </tr>
    </thead>
    <tbody>
        <tr>
            <td rowspan="2">
            <b>C</b>reate<br />
            <b>R</b>ead<br />
            <b>U</b>pdate<br />
            <b>D</b>elete<br />
            </td>
            <td>Own</td>
            <td>The C|R|U|D action is (or not) to be performed on own resource(s) of the current subject.</td>
        </tr>
        <tr>
            <td>Any</td>
            <td>The C|R|U|D action is (or not) to be performed on any resource(s); including own.</td>
        </tr>   
    </tbody>
</table>

```js
ac.grant('role').readOwn('resource');
ac.deny('role').deleteAny('resource');
```

_Note that **own** requires you to also check for the actual possession. See [this](https://github.com/onury/accesscontrol/issues/14#issuecomment-328316670) for more._

### Resources and Resource-Attributes

Multiple roles can have access to a specific resource. But depending on the context, you may need to limit the contents of the resource for specific roles.  

This is possible by resource attributes. You can use Glob notation to define allowed or denied attributes.

For example, we have a `video` resource that has the following attributes: `id`, `title` and `runtime`.
All attributes of *any* `video` resource can be read by an `admin` role:
```js
ac.grant('admin').readAny('video', ['*']);
// equivalent to:
// ac.grant('admin').readAny('video');
```
But the `id` attribute should not be read by a `user` role.  
```js
ac.grant('user').readOwn('video', ['*', '!id']);
// equivalent to:
// ac.grant('user').readOwn('video', ['title', 'runtime']);
```

You can also use nested objects (attributes).
```js
ac.grant('user').readOwn('account', ['*', '!record.id']);
```

### Checking Permissions and Filtering Attributes

You can call `.can(<role>).<action>(<resource>)` on an `AccessControl` instance to check for granted permissions for a specific resource and action.

```js
const permission = ac.can('user').readOwn('account');
permission.granted;       // true
permission.attributes;    // ['*', '!record.id']
permission.filter(data);  // filtered data (without record.id)
```
See [express.js example](#expressjs-example).

### Defining All Grants at Once

You can pass the grants directly to the `AccessControl` constructor.
It accepts either an `Object`:

```js
// This is actually how the grants are maintained internally.
let grantsObject = {
    admin: {
        video: {
            'create:any': ['*'],
            'read:any': ['*'],
            'update:any': ['*'],
            'delete:any': ['*']
        }
    },
    user: {
        video: {
            'create:own': ['*'],
            'read:own': ['*'],
            'update:own': ['*'],
            'delete:own': ['*']
        }
    }
};
const ac = new AccessControl(grantsObject);
```
... or an `Array` (useful when fetched from a database):
```js
// grant list fetched from DB (to be converted to a valid grants object, internally)
let grantList = [
    { role: 'admin', resource: 'video', action: 'create:any', attributes: ['*'] },
    { role: 'admin', resource: 'video', action: 'read:any', attributes: ['*'] },
    { role: 'admin', resource: 'video', action: 'update:any', attributes: ['*'] },
    { role: 'admin', resource: 'video', action: 'delete:any', attributes: ['*'] },

    { role: 'user', resource: 'video', action: 'create:own', attributes: ['*'] },
    { role: 'user', resource: 'video', action: 'read:any', attributes: ['*'] },
    { role: 'user', resource: 'video', action: 'update:own', attributes: ['*'] },
    { role: 'user', resource: 'video', action: 'delete:own', attributes: ['*'] }
];
const ac = new AccessControl(grantList);
```
You can set/get grants any time:
```js
const ac = new AccessControl();
ac.setGrants(grantsObject);
console.log(ac.getGrants());
```

## Documentation

You can read the full [**API reference**][docs] with lots of details, features and examples.  
And more at the [F.A.Q. section][faq].

## Change-Log

See [CHANGELOG][changelog].

## License

[MIT][license].

[docs]:http://onury.io/accesscontrol/?api=ac
[faq]:http://onury.io/accesscontrol/?content=faq
[rbac]:https://en.wikipedia.org/wiki/Role-based_access_control
[abac]:https://en.wikipedia.org/wiki/Attribute-Based_Access_Control
[crud]:https://en.wikipedia.org/wiki/Create,_read,_update_and_delete
[nist-paper]:http://csrc.nist.gov/groups/SNS/rbac/documents/kuhn-coyne-weil-10.pdf
[changelog]:https://github.com/onury/accesscontrol/blob/master/CHANGELOG.md
[license]:https://github.com/onury/accesscontrol/blob/master/LICENSE
