# python-oced

A single-module Python library for recording Object-Centric Event Data (OCED)
in memory. OCED is an event data model in which one event can create, change or
refer to several objects, their relations and their attribute values, instead
of belonging to a single case. It was written for university work in process
mining.

## What it models

Everything lives in `OCED.py`.

- `OCED` is the model. It holds a set of pandas DataFrames (`event`,
  `event_type`, `event_time`, `event_attribute_name`, `event_attribute_value`,
  `object`, `object_type`, `object_attribute_name`, `object_attribute_value`,
  `object_relation`, `object_relation_type`, `event_x_object`,
  `event_x_object_relation`, `event_x_object_attribute_value`), a `log` of the
  qualifiers applied per event, a `current_state` dictionary and an
  `event_counter`. Events are added with `insert_event(event)`.
- `Event(event_time, event_type, qualifiers=[], event_attributes={})` is one
  event. `event_time` is an ISO 8601 string, `event_attributes` maps strings to
  strings, and `qualifiers` is the ordered list of changes the event makes.
- Qualifiers are the changes. Each kind exists for objects, object relations
  and object attribute values:
  - `create_object(object_id, object_type)`,
    `create_object_relation(object_relation_id, from_object_id, to_object_id, object_relation_type)`,
    `create_object_attribute_value(object_attribute_value_id, object_id, object_attribute_name, object_attribute_value)`
  - `modify_object(object_id, new_object_type)`,
    `modify_object_relation(object_relation_id, new_object_relation_type)`,
    `modify_object_attribute_value(object_attribute_value_id, new_object_attribute_value)`
  - `delete_object(object_id)`, `delete_object_relation(object_relation_id)`,
    `delete_object_attribute_value(object_attribute_value_id)`
  - `involve_object(object_id)`, `involve_object_relation(object_relation_id)`,
    `involve_object_attribute_value(object_attribute_value_id)`, which link an
    existing item to the event without changing it.

Behaviour, as implemented:

- All identifiers, types, names and values must be strings.
- Each event must be later than the previous one.
- The qualifiers of an event are first applied to a copy of the current state;
  the tables change only if all of them pass.
- Deletion is a flag: the item is marked as no longer existing and a new row
  with existency `False` is appended. Deleting an object also marks its
  relations and attribute values as deleted. A deleted id cannot be created
  again.
- Modify, delete and involve need an existing item. A modify must change the
  value, and a relation cannot point from an object to itself.
- The tables are append-only, so they keep the full history; `current_state`
  holds the latest state of every object, relation and attribute value.

`dump_json(file_name, model)` / `load_json(file_name)` and
`dump_xml(file_name, model)` / `load_xml(file_name)` save and restore a model.

## Use

There is no package and nothing on PyPI. Copy `OCED.py` next to your code, or
add its folder to `PYTHONPATH`, and import it. It needs `pandas` and
`typing_extensions`; `dump_xml` also needs `lxml`.

```bash
pip install pandas typing_extensions lxml
```

```python
from OCED import (OCED, Event, create_object, create_object_relation,
                  create_object_attribute_value, modify_object_attribute_value,
                  delete_object)

model = OCED()

model.insert_event(Event('2024-01-10T09:00:00', 'place_order', [
    create_object('c1', 'Customer'),
    create_object('o1', 'Order'),
    create_object_relation('r1', 'c1', 'o1', 'places'),
    create_object_attribute_value('a1', 'o1', 'status', 'open'),
], {'channel': 'web'}))

model.insert_event(Event('2024-01-11T14:30:00', 'pay_order', [
    modify_object_attribute_value('a1', 'paid'),
]))

model.insert_event(Event('2024-01-12T08:00:00', 'cancel_order', [
    delete_object('o1'),
]))

print(model.object_attribute_value.to_string())
print(model.current_state['object']['o1'])
```

Output:

```
  object_attribute_value_id object_id object_attribute_name object_attribute_value object_attribute_value_existency
0                        a1        o1                status                   open                             True
1                        a1        o1                status                   paid                             True
2                        a1        o1                status                   paid                            False
{'type': 'Order', 'existency': False, 'object_relation_ids': ['r1'], 'object_attribute_value_ids': ['a1']}
```

## Repository contents

- `OCED.py`: the library.
- `test.ipynb`: a notebook that walks through building a model step by step.
- `OCED_model.json`, `OCED_model.xml`: example exports.
- `NOTE.md`: design notes and open questions, in Italian.
- `LICENSE`: MIT.

## Status and limits

This is study code. There are no automated tests and no
packaging. Known limits:

- Attribute values are strings only.
- All data is held in memory.
- The properties of `OCED` return the internal DataFrames and dictionaries
  directly; changing them changes the model.
- Some validation errors raise `AttributeError` or `KeyError` instead of a
  clear `ValueError` (for example, creating an id that already exists, or an
  attribute value for an unknown object).
- A model loaded with `load_json` cannot take new events if one of its tables
  was empty when it was saved.

Project page: https://vincenzo.imperati.dev/projects/python-oced

## Licence

MIT. See `LICENSE`.
