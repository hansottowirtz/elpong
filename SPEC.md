# Elpong
##### v0.4.0-beta

### This is a draft, please contribute!

Elpong is meant to be a simple, intuitive way for creating an API for a website
or app, and using that API on the client side.

It is a RESTful, JSON-based API protocol that intents to standardize

- the way API elements are sent from server to client and vice versa.
- the representation of these elements on the client.

It was created to make handling Elements on the client side, with their
relations other Collections and Elements, easier.

It does this by implementing a Scheme.

There are 3 simple concepts on which the protocol is built: Schemes, Collections and Elements.

### Schemes

A Scheme is just a large object where all Collections are defined.

e.g.
```json
{
  "name": "animal-farm",
  "selector": "id",
  "collections": {
    "pigs": {
      "fields": {
        "id": {},
        "name": {},
        "boss_id": { "reference": true }
      },
      "relations": {
        "belongs_to": {
          "boss": {
            "collection": "humans"
          },
        }
      },
      "actions": {
        "oink": {
          "method": "PUT"
        }
      }
    },
    "humans": {
      "fields": {
        "id": {},
        "name": {}
      },
      "relations": {
        "has_many": {
          "pigs": {
            "as": "boss"
          }
        }
      }
    }
  }
}
```

Quick look:

`name`: A unique name for the Scheme, here it is our app name.

`collections`: The names of the Collections of this scheme.

`fields`: A pig has an `id`, a `name`, and a `boss_id`, which refers to its human boss.

`selector`: We look for pigs by their `id`, which must be unique.

`reference`: Just makes clear that `boss_id` is a reference to something.

`relations`: An Element can have many Elements or belong to an Element.

`actions`: When the oink action is called on pig 8, it will trigger a PUT to
something like `api/dogs/8/oink`

### Collections

A Scheme has multiple Collections, and a Collection has multiple Elements.

### Elements

An Element has multiple fields, and relations to other Elements.

For example, there are two pigs:
```javascript
{id: 1, name: 'Napoleon', boss_id: null}
{id: 2, name: 'Snowball', boss_id: 9}
```
And one human:
```javascript
{id: 9, name: 'Mr. Jones'}
```
As Napoleon is his own boss, he does not relate to a human.
Snowball does have a boss, however.

A method like `snowball.relations.boss()` must therefore return
Mr. Jones.

`jones.relations.pigs()` must return a list of his pigs, in this case
only Snowball.

`napoleon.relations.boss()` must return nothing.

#### Selector
A collection can only have one selector field, and it must be unique. It is
typically something like `id`, `uid` or `uuid`. If an element doesn't have
a selector, it is new, and not yet saved on the server.

### API

#### Urls

There are a few types of urls, based on [CRUD][crud].
Let's say our base url is `/api`:

`GET /api/pigs` - to get all pigs - returns an array of Pre Elements.

`GET /api/pigs/8` - to get pig 8 - returns a Pre Element.

`POST /api/pigs` - to save a new pig - returns the Pre Element with a selector value.

`PUT /api/pigs/8` - to update pig 8 - returns the updated Pre Element.

`DELETE /api/pigs/8` - to remove pig 8 - returns `null`.

#### Pre Element
A Pre Element is the simple JSON object returned by the API.
An Element is a class that acts a wrapper around the Pre Element, and it
provides functions like relations and actions.

### Other

#### Embedded Elements and Collections

One of the strengths of HTTPong is the ability to embed Elements and Collections
in other Elements. This is done with the `embedded_element` and the
`embedded_collection` keys. When a Pre Element with a `embedded_element` field
is added to a Collection, the embedded Pre Elements are added to their Collection,
or merged with existing Elements with the same selector value. Then, its
selector value are set to their associated field.

e.g.
```json
{
  "name": "animal-farm",
  "selector": "id",
  "collections": {
    "apples": {
      "fields": {
        "id": {},
        "kind": {},
        "apple_stem": {"embedded_element": true},
        "apple_stem_id": {"no_send": true, "reference": true}
      },
      "relations": {
        "belongs_to": {
          "apple_stem": {}
        }
      }
    },
    "apple_stems": {
      "fields": {
        "id": {}
      },
      "relations": {
        "has_one": {
          "apple": {}
        }
      }
    }
  }
}
```
We make a GET request to `/api/apples`:
```json
[
  {
    "id": 5,
    "kind": "Granny Smith",
    "apple_stem": {
      "id": 3
    }
  }
]
```
This Pre Element is converted:
```javascript
{
  id: 5,
  kind: 'Granny Smith',
  apple_stem_id: 3
}
```
The `apple_stem_id` field can be changed with key `field` key.
The apple can be retrieved with `stem.relations.apple()`, and the
stem with `apple.relations.stem()`

#### Singular

```json
"geese": {
  "singular": "goose"
}
```

#### Merging Elements

When an Element is merged with a Pre Element, for example when it is updated
with a do action, it should overwrite all fields that are in the Pre Element,
except when it's a embedded Element or embedded Collection. If a field is
not present (not defined) on the Pre Element, then it is not overwritten.

### Relations

Relations are totally distinct from fields,
and two related types of relations are also distinct.<br/>
That means a `has_many` relation will not check the `belongs_to` relation
on the other collection, their keys must be defined seperately.

There are three types of relations:

##### Belongs to

A pig belongs to a human: the pig has a `human_id` field.

##### Has one

A pig has one human: the human has a `pig_id` field.<br/>
This is actually just has many, but it takes the first from the list.

##### Has many

A human has many pigs: a pig has a `human_id` field.

---

You can use other fields by using the `as`, `collection`, and `field` keys.
Remember: the name of the relation, is also the name of the function.

e.g.

`Pigs`:
```json
{
  "belongs_to": {
    "boss": {
      "collection": "humans"
    }
  }
}
```
This will result in a `boss()` function.
```json
{
  "belongs_to": {
    "human": {
      "field": "boss_id"
    }
  }
}
```
This will result in a `human()` function.

`Humans`:
```javascript
"has_many": {
  "pigs": {
    "as": "boss"
  }
}
```
```javascript
"has_many": {
  "pigs": {
    "field": "boss_id"
  }
}
```
These two have the same result, a `pigs()` function.

#### Polymorphism

When the `polymorphic` key is true on a belongs_to relation,
there must be a `{name}_collection` field, and a `{name}_{selector_name}`.

e.g.
```javascript
{
  "name": "pulser",
  "selector": "id",
  "collections": {
    "controls": {
      "fields": {
        "id": {},
        "name": {}
      },
      "relations": {
        "has_many": {
          "plugs": {"polymorphic": true, "as": "block"}
        }
      }
    },
    "devices": {
      "fields": {
        "id": {},
        "name": {}
      },
      "relations": {
        "has_many": {
          "plugs": {"polymorphic": true, "as": "block"}
        }
      }
    },
    "plugs": {
      "fields": {
        "id": {},
        "block_collection": {},
        "block_id": {"reference": true}
      },
      "relations": {
        "belongs_to": {
          "block": {"polymorphic": true}
        }
      }
    }
  }
}
```

Let's say there are two controls:
```javascript
{id: 1, name: 'Button'}
{id: 2, name: 'Slider'}
```
One device:
```javascript
{id: 1, name: 'Lamp'}
```

Four plugs:
```javascript
{id: 1, block_collection: 'controls', block_id: 1}
{id: 2, block_collection: 'controls', block_id: 2}
{id: 3, block_collection: 'devices', block_id: 1}
{id: 4, block_collection: 'devices', block_id: 1}
```

`button.relations.plugs()` must return a list with `plug1`.
`slider.relations.plugs()` must return a list with `plug2`.
`lamp.relations.plugs()` must return a list with `plug3` and `plug4`.
`plug1.relations.block()` must return `button`.
`plug2.relations.block()` must return `slider`.
`plug3.relations.block()` must return `lamp`.
`plug4.relations.block()` must return `lamp`.

The collection_field can be
changed with `collection_field`.

e.g.
```javascript
"belongs_to": {
  "block": {
    "polymorphic": true,
    "field": "block_id",
    "collection_field": "block_collection"
  }
}
```

### More API
#### Actions & collection actions

Built in actions:

`GET` is meant to get, refresh or reload the fields of an Element.<br/>
`POST` is meant to save, and thereby assign a selector value to an Element.<br/>
`PUT` is meant to push updates to the server.<br/>
`DELETE` is meant to remove the Element from the server database.<br/>

Custom actions:

With `no_data` the data is set to null.<br/>
With `no_selector` the url is like a collection action url. Use it if you
want to send along the Element, because a Collection action sends nothing.<br/>
With `path` you can set another path if the action name has another name.<br/>
With `returns_other`, the returned data will not be merged with the Element data.
This only applies to Elements.

Difference between actions and collection actions:

- Collection actions send null by default, actions send data when `exclude_data`
isn't true.
- Collection actions never have a selector value, actions do when
`no_selector` isn't true.
- Actions send element updates back, except when `returns_other` is true.

`Users`:
```javascript
"actions": {
  "login": {
    "method": "POST",
    "no_selector": true
  },
  "register": {
    "method": "POST",
    "no_selector": true
  }
},
"collection_actions": {
  "get_me": {
    "method": "GET",
    "path": "me"
  },
  "logout": {
    "method": "DELETE"
  }
}
```
The urls are:

`/api/users/login`: Sends the user data to the server.<br/>
`/api/users/register`: Sends the user data to the server.<br/>
`/api/users/me`: You need to handle the response yourself.<br/>
`/api/users/logout`: You can choose between collection actions and normal
actions here, they both send null.<br/>

#### Sending and receiving data

When an Element is sent, it is converted to JSON again.
The embedded Elements and Collections are not included, the associated field
for the embedded Element is included. The elements in the embedded Collection
should be updated separately.

#### Sending and receiving extras

Extra data can be put in the headers.

[crud]: https://en.wikipedia.org/wiki/Create,_read,_update_and_delete
