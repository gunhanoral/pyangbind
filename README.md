[![#PyangBind][img-pyangbind]][pyangbind-docs]

[![PyPI][img-pypi]][pypi-project]
[![PyPI - License][img-license]][license]
[![PyPI - Python Version][img-pyversion]][pypi-project]


**PyangBind** is a plugin for [Pyang][pyang] that generates a Python class hierarchy from a YANG data model. The resulting classes can be directly interacted with in Python. Particularly, **PyangBind** will allow you to:

 * Create new data instances - through setting values in the Python class hierarchy.
 * Load data instances from external sources - taking input data from an external source and allowing it to be addressed through the Python classes.
 * Serialise populated objects into formats that can be stored, or sent to another system (e.g., a network element).

Development of **PyangBind** has been motivated by consuming the  [OpenConfig][openconfig] data models; and is intended to be well-tested against these models. The Python classes generated, and serialisation methods are intended to provide network operators with a starting point for loading data instances from network elements, manipulating them, and sending them to a network device. **PyangBind** classes also have functionality which allows additional methods to be associated with the classes, such that it can be used for the foundation of a NMS.

## Contents

* [Getting Started](#getting-started)
	* [Generating a Set of Classes](#generating-classes)
	* [Using the Classes in a Python Program](#using-in-python)
	* [Creating a Data Instance](#create-instance)
	* [Serialising a Data Instance](#serialising)
	* [Deserialising a Data Instance to Classes](#deserialising)
  * [Example Code](#example-code)
* [Further Documentation](#documentation)
* [Licensing](#licensing)
* [Acknowledgements](#acks)
* [Test Status](#tests)


## Getting Started <a name="getting-started"></a>

**PyangBind** is distributed through [PyPI][pypi], it can be installed by simply running:

```
$ pip install pyangbind
```

The `pyangbind` module installs both the Pyang plugin (`pyangbind.plugin.*`), as well as a set of library modules (`pyangbind.lib.*`) that are used to provide the Python representation of YANG types.

### Generating a Set of Classes <a name="generating-classes"></a>

To generate your first set of classes, you will need a YANG module, and its dependencies. A number of simple modules can be found in the `tests` directory (e.g., `tests/base-test.yang`).

To generate a set of Python classes, Pyang needs to be provided a pointer to where PyangBind's plugin is installed. This location can be found by running:

```
$ export PYBINDPLUGIN=`/usr/bin/env python -c \
'import pyangbind; import os; print ("{}/plugin".format(os.path.dirname(pyangbind.__file__)))'`
$ echo $PYBINDPLUGIN
```

Once this path is known, it can be provided to the `--plugin-dir` argument to Pyang. In the simplest form the command used is:

```
$ pyang --plugindir $PYBINDPLUGIN -f pybind -o binding.py tests/base-test.yang
```
where:

* `$PYBINDPLUGIN` is the location that was exported from the above command.
* `binding.py` is the desired output file.
* `doc/base-test.yang` is the path to the YANG module that bindings are to be generated for.

There are a number of other options for **PyangBind**, which are discussed further in the `docs/` directory.

### Using the Classes in a Python Program <a name="using-in-python"></a>

**PyangBind** generates a (set of) Python modules. The top level module is named after the YANG module - with the name made Python safe. In general this appends underscores to reserved names, and replaces hyphens with underscores - such that `openconfig-local-routing.yang` becomes `openconfig_local_routing` as a module name.

Primarily, we need to generate a set of classes for the model(s) that we are interested in. The OpenConfig local routing module will be used as an example for this walkthrough.

The bindings can be simply generated by running the `docs/example/oc-local-routing/generate_bindings.sh` script. This script simply uses `curl` to retrieve the modules, and subsequently, builds the bindings to a `binding.py` file as expressed above.

The simplest program using a PyangBind set of classes will look like:

```python
# Using the binding file generated by the `generate_bindings.sh` script
# Note that CWD is the file containing the binding.py file.
# Alternatively, you can use sys.path.append to add the CWD to the PYTHONPATH
from binding import openconfig_local_routing

oclr = openconfig_local_routing()
```

### Creating a Data Instance <a name="create-instance"></a>

At this point, the `oclr` object can be used to manipulate the YANG data tree that is expressed by the module.

A subset of `openconfig-local-routing` looks like the following tree:

```
module: openconfig-local-routing
   +--rw local-routes
   ...
      +--rw static-routes
      |  +--rw static* [prefix]
      |     +--rw prefix    -> ../config/prefix
      |     +--rw config
      |     |  +--rw prefix?     inet:ip-prefix
      |     |  +--rw next-hop*   union
```

To add an entry to the `static` list the `add` method of the `static` object is used:

```python
rt = oclr.local_routes.static_routes.static.add("192.0.2.1/32")
```

The `static` list is addressed exactly as per the path that it has within the YANG module - such that it is a member of the `static-routes` container (whose name has been made Python-safe), which itself is a member of the `local-routes` container.

The `add` method returns a reference to the newly created list object - such that we can use the `rt` object to change characteristics of the newly created list entry. For example, a tag can be set on the route:

```python
rt.config.set_tag = 42
```

The tag value can then be accessed directly via the `rt` object, or via the original `oclr` object (which both refer to the same object in memory):

```python
# Retrieve the tag value
print(rt.config.set_tag)
# output: 42

# Retrieve the tag value through the original object
print(oclr.local_routes.static_routes.static["192.0.2.1/32"].config.set_tag)
# output: 42
```

In addition, PyangBind classes which represent `container` or `list` objects have a special `get()` method. This dumps a dictionary representation of the object for printing or debug purposes (see the sections on serialisation for outputting data instances for interaction with other devices). For example:

```python
print(oclr.local_routes.static_routes.static["192.0.2.1/32"].get(filter=True))
# returns {'prefix': u'192.0.2.1/32', 'config': {'set-tag': 42}}
```

The `filter` keyword allows only the elements within the class that have changed (are not empty or their default) to be output - rather than all possible elements.

The `next-hops` element in this model is another list. This keyed data structure acts like a Python dictionary, and has the special method `add` to add items to it. YANG `leaf-list` types use the standard Python list `append` method to add items to it. Equally, a `list` can be iterated through using the same methods as a dictionary, for example, using `iteritems()`:

```python
# Add a set of next_hops
for nhop in [(0, "192.168.0.1"), (1, "10.0.0.1")]:
  nh = rt.next_hops.next_hop.add(nhop[0])
  nh.config.next_hop = nhop[1]

# Iterate through the next-hops added
for index, nh in rt.next_hops.next_hop.iteritems():
    print("{}: {}".format(index, nh.config.next_hop))
```

Where (type or value) restrictions exist. PyangBind generated classes will result in a Python `ValueError` being raised. For example, if we attempt to set the `set_tag` leaf to an invalid value:

```python
# Try and set an invalid tag type
try:
  rt.config.set_tag = "INVALID-TAG"
except ValueError as m:
  print("Cannot set tag: {}".format(m))
```

### Serialising a Data Instance <a name="serialising"></a>

Clearly, populated PyangBind classes are not useful in and of themselves - the common use case is that they are sent to a external system (e.g., a router, or other NMS component). To achieve this the class hierarchy needs to be serialised into a format that can be sent to the remote entity. There are currently multiple ways to do this:

 * **XML** - the rules for this mapping are defined in [RFC 7950][rfc7950] - supported.
 * **OpenConfig-suggested JSON** - the rules for this mapping are currently being written into a formal specification. This is the standard (`default`) format used by PyangBind. Some network equipment vendors utilise this serialisation format.
 * **IETF JSON** - the rules for this mapping are defined in [RFC 7951][rfc7951] - some network equipment vendors use this format.

Any PyangBind class can be serialised into any of the supported formats. Using the static route example above, the entire `local-routing` module can be serialised into OC-JSON using the following code:

```python
from pyangbind.lib.serialise import pybindIETFXMLEncoder
# Dump the entire instance as RFC 7950 XML
print(pybindIETFXMLEncoder.serialise(oclr))
```

This outputs the following XML fragment:

```xml
<openconfig-local-routing xmlns="http://openconfig.net/yang/local-routing">
  <local-routes>
    <static-routes>
      <static>
        <prefix>192.0.2.1/32</prefix>
        <config>
          <set-tag>42</set-tag>
        </config>
        <next-hops>
          <next-hop>
            <index>0</index>
            <config>
              <next-hop>192.168.0.1</next-hop>
            </config>
          </next-hop>
          <next-hop>
            <index>1</index>
            <config>
              <next-hop>10.0.0.1</next-hop>
            </config>
          </next-hop>
        </next-hops>
      </static>
    </static-routes>
  </local-routes>
</openconfig-local-routing>
```

Or, similarly, using OpenConfig-suggested JSON:

```python
import pyangbind.lib.pybindJSON as pybindJSON
print(pybindJSON.dumps(oclr))
```

This outputs the following JSON structured text:

```json
{
    "local-routes": {
        "static-routes": {
            "static": {
                "192.0.2.1/32": {
                    "next-hops": {
                        "next-hop": {
                            "0": {
                                "index": "0", 
                                "config": {
                                    "next-hop": "192.168.0.1"
                                }
                            }, 
                            "1": {
                                "index": "1", 
                                "config": {
                                    "next-hop": "10.0.0.1"
                                }
                            }
                        }
                    }, 
                    "prefix": "192.0.2.1/32", 
                    "config": {
                        "set-tag": 42
                    }
                }
            }
        }
    }
}
```

Note here that the `static` list is represented as a JSON object (such that if this JSON is loaded elsewhere, a prefix can be referenced using `obj["local-routes"]["static-routes"]["static"]["192.0.2.1/32"]`).

It is also possible to serialise a subset of the data, e.g., only one `list` or `container` within the class hierarchy. This is done as follows (into IETF-JSON):

```
# Dump the static routes instance as JSON in IETF format
print(pybindJSON.dumps(oclr.local_routes, mode="ietf"))
```

And the corresponding output:

```json
{
    "openconfig-local-routing:static-routes": {
        "static": [
            {
                "next-hops": {
                    "next-hop": [
                        {
                            "index": "0", 
                            "config": {
                                "next-hop": "192.168.0.1"
                            }
                        }, 
                        {
                            "index": "1", 
                            "config": {
                                "next-hop": "10.0.0.1"
                            }
                        }
                    ]
                }, 
                "prefix": "192.0.2.1/32", 
                "config": {
                    "set-tag": 42
                }
            }
        ]
    }
}
```

Here, note that the list is represented as a JSON array, as per the IETF specification; and that only the `static-routes` children of the object have been serialised.

### Deserialising a Data Instance <a name="deserialising"></a>

PyangBind also supports taking data instances from a remote system (or locally saved documents) and loading them into either a new, or existing set of classes. This is useful for when a remote system sends a data instance in response to a query - and the programmer wishes to injest this response such that further logic can be performed based on it.

Instances can be deserialised from any of the supported serialisation formats (see above) into the classes.

To de-serialise into a new object, the `load` method of the serialise module can be used:

```python
new_oclr = pybindJSON.load(os.path.join("json", "oc-lr.json"), binding, "openconfig_local_routing")
```

This creates a new instance of the `openconfig_local_routing` class that is within the `binding` module, and loads the data from `json/oc-lr.json` into it. The `new_oclr` object can then be manipulated as per any other class:

```python
# Manipulate the data loaded
print("Current tag: %d" % new_oclr.local_routes.static_routes.static[u"192.0.2.1/32"].config.set_tag)
# Outputs: 'Current tag: 42'

new_oclr.local_routes.static_routes.static[u"192.0.2.1/32"].config.set_tag += 1
print("New tag: %d" % new_oclr.local_routes.static_routes.static[u"192.0.2.1/32"].config.set_tag)
# Outputs: 'Current tag: 43'
```

Equally, a JSON instance can be loaded into an existing set of classes - this is done by directly calling the relevant deserialisation class -- in this case `pybindJSONDecoder`:

```python
# Load JSON into an existing class structure
from pyangbind.lib.serialise import pybindJSONDecoder
import json

ietf_json = json.load(open(os.path.join("json", "oc-lr_ietf.json"), 'r'))
pybindJSONDecoder.load_ietf_json(ietf_json, None, None, obj=new_oclr.local_routes)
```

The direct `load_ietf_json` method is handed a JSON object - and no longer requires the arguments for the module and class name (hence they are both set to `None`), rather the optional `obj=` argument is used to specify the object that corresponds to the JSON that is being loaded.

Following this load, the classes can be iterated through - showing both the original loaded route (`192.0.2.1/32`) and the one in the IETF JSON encoded file (`192.0.2.2/32`) exist in the data instance:

```python
# Iterate through the classes - both the 192.0.2.1/32 prefix and 192.0.2.2/32
# prefix are now in the objects
for prefix, route in new_oclr.local_routes.static_routes.static.iteritems():
  print("Prefix: {}, tag: {}".format(prefix, route.config.set_tag))

# Output:
#		Prefix: 192.0.2.2/32, tag: 256
#		Prefix: 192.0.2.1/32, tag: 43
```

### Example Code <a name="example-code"></a>
This worked example can be found in the `docs/example/oc-local-routing` directory.

## Further Documentation <a name="documentation"></a>

Further information as to the implementation and usage of PyangBind can be found in the `docs/` directory -- the [README](docs/README.md) provides a list of documents and examples container therein.

## Licensing <a name="licensing"></a>
```
Copyright 2015, Rob Shakir (rjs@rob.sh)
Modifications copyright, the Pyangbind contributors.

This project has been supported by:
          * Jive Communications, Inc.
          * BT plc.
          * Google, Inc.
          * GoDaddy, LLC.

Licensed under the Apache License, Version 2.0 (the "License");
you may not use this file except in compliance with the License.
You may obtain a copy of the License at

    http://www.apache.org/licenses/LICENSE-2.0

Unless required by applicable law or agreed to in writing, software
distributed under the License is distributed on an "AS IS" BASIS,
WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
See the License for the specific language governing permissions and
limitations under the License.
```

## Acknowledgements <a name="acks"></a>
* This project was initiated as part of BT plc. Network Architecture 'future network management' projects.
* Additional development efforts were supported by [Jive Communications, Inc][jive].
* Current maintenance is supported by [Google][google].
* Key contributions have been made to this project by the following developers, and companies. Many thanks
  are extended to them:
  * [GoDaddy][godaddy], particularly Joey Wilhelm's herculean efforts to refactor test code to use the `unittest` framework.
  * David Barroso, who initiated efforts to address Python 3 compatibility, and a number of other enhancements.
* Design, debugging, example code, and ideas have been contributed by:
  * Members of the [OpenConfig][openconfig] working group.
  * The managability team at Juniper Networks.


[img-pyangbind]: https://cdn.rob.sh/img/pyblogo_gh.png
[img-travis]: https://img.shields.io/travis/robshakir/pyangbind.svg
[img-codecov]: https://img.shields.io/codecov/c/github/robshakir/pyangbind.svg
[img-pypi]: https://img.shields.io/pypi/v/pyangbind.svg
[img-license]: https://img.shields.io/pypi/l/pyangbind.svg
[img-pyversion]: https://img.shields.io/pypi/pyversions/pyangbind.svg

[pyangbind-docs]: https://github.com/robshakir/pyangbind/tree/master/docs
[travis]: https://travis-ci.org/robshakir/pyangbind
[codecov]: https://codecov.io/gh/robshakir/pyangbind
[pypi-project]: https://pypi.org/project/pyangbind/
[license]: http://www.apache.org/licenses/LICENSE-2.0
[pyang]: https://github.com/mbj4668/pyang
[openconfig]: http://www.openconfig.net
[pypi]: https://pypi.org/
[rfc7950]: https://tools.ietf.org/html/rfc7950
[rfc7951]: https://tools.ietf.org/html/rfc7951
[jive]: https://www.jive.com/
[google]: https://www.google.com/
[godaddy]: https://www.godaddy.com/
