[Source](https://github.com/OAI/OpenAPI-Specification/issues/1722)
----

# OpenAPI Overlays

In recent months we have been discussing various use cases for overlays and various solutions.  The following proposal takes a somewhat more radical approach to the problem.  It is a more ambitious proposal than the others we have seen before but the additional complexity does allow for supporting many of the scenarios that have been discussed to date.


#### <a name="overlayDocument"></a>Overlay Document

An overlay document contains a list of [Update Objects](#overlayUpdates) that are to be applied to the target document.  Each [Update Object](#updateObject) has a `target` property and a `value` property.  The `target` property is a [JMESPath](http://jmespath.org/specification.html) query that identifies what part of the target document is to be updated and the `value` property contains an object with the properties to be overlayed.


#### <a name="overlayObject"></a>Overlay Object

This is the root object of the [OpenAPI Overlay document](#oasDocument).

##### Fixed Fields

Field Name | Type | Description
---|:---:|---
<a name="overlayVersion"></a>overlay | `string` | Version of the Overlay specification that this document conforms to. 
<a name="overlayInfo"></a>info | [[Info Object](#overlayInfoObject)] | Identifying information about the overlay.
<a name="overlayExtends"></a>extends | `url` | URL to an OpenAPI document this overlay applies to. 
<a name="overlayUpdates"></a>updates | [[Update Object](#updateObject)] | A list of update objects to be applied to the target document.

The list of update objects MUST be applied in sequential order to ensure a consistent outcome.  This enables objects to be deleted in one update and then re-created in a subsequent update.

#### <a name="overlayInfoObject"></a>Info Object

This object contains identifying information about the [OpenAPI Overlay document](#oasDocument).

##### Fixed Fields

Field Name | Type | Description
---|:---:|---
<a name="overlayTitle"></a>title | `string` | A human readable description of the purpose of the overlay.
<a name="overlayVersion"></a>version | `string` | A version identifer for indicating changes to an overlay document.

#### <a name="updateObject"></a>Update Object

This object represents one or more changes to be applied to the target document at the location defined by the target JMESPath.

##### Fixed Fields

Field Name | Type | Description
---|:---:|---
<a name="updateTarget"></a>target | `string` | A JMESPath expression referencing the target objects in the target document.
<a name="updateValue"></a>value | [Any](#valueObject) | An object with the properties and values to be updated in the target document.

The properties of the `Value Object` MUST be compatible with the target object referenced by the JMESPath key.  When the Overlay document is applied, the properties in the `Value Object` replace properties in the target object with the same name and new properties are appended to the target object.


##### Structured Overlays Example

When updating properties throughout the target document it may be more efficient to create a single `Update Object` that mirrors the structure of the target document. e.g.

```yaml
overlay: 1.0.0
info:
  title: Structured Overlay
  version: 1.0.0
updates:
- target: "@"
    value:
      info:
        x-overlay-applied: structured-overlay
      paths:
        "/":
          summary: "The root resource"
          get:
            summary: "Retrieve the root resource"
            x-rate-limit: 100
        "/pets":
          get:
            summary: "Retrieve a list of pets"
            x-rate-limit: 100
      components:
      tags:
```

##### Targeted Overlays

Alternatively, where only a small number of updates need to be applied to a large document, each [Update Object](#updateObject) can be more targeted.

```yaml
overlay: 1.0.0
info:
  title: Structured Overlay
  version: 1.0.0
updates:
- target: paths."/foo".get
    value:
        description: This is the new description
- target: paths."/bar".get
    value:
        description: This is the updated description
- target: paths."/bar"
    value:
        post:
            description: This is an updated description of a child object
            x-safe: false
```

##### Wildcard Overlays Examples

One significant advantage of using the JMESPath syntax that it allows referencing multiple nodes in the target document.  This would allow a single update object to be applied to multiple target objects using wildcards.

```yaml
overlay: 1.0.0
info:
  title: Update many objects at once
  version: 1.0.0
updates:
- target: paths.*.get
    value:
      x-safe: true
- target: paths.*.get.parameters[?name=='filter' && in=='query']
    value:
      schema:
        $ref: "/components/schemas/filterSchema"
```

##### Array Modification Examples

Due to the fact that we can now reference specific elements of the parameter array, it does open the possibilty to being able to add parameters and potentially remove them using a `null` value.

```yaml
overlay: 1.0.0
info:
  title: Add an array element
  version: 1.0.0
updates:
- target: paths.*.get.parameters[length(@)]
    value: 
      name: newParam
      in: query
```

```yaml
overlay: 1.0.0
info:
  title: Remove a array element
  version: 1.0.0
updates:
- target: $.paths[*].get.parameters[? name == 'dummy']
    value: null
```


## Proposal Summary

### Benefits

- This approach addresses the two distinct approaches of structured overlay vs targeted overlay which suits distinct but equally valid scenarios.
- Addresses the problem of modifying the parameters array and removes the need to replace the entire array when a small change is required.
- Allows sets of related overlays to be stored in a same file.
- Enables updating a set of objects based on a pattern. This might be an effective way of apply common behaviour across many operations in an API.

### Challenges
- Tooling will need a JMESPath implementation.
- Large overlays may be slow to process.
- Multiple complex pattern based overlays may cause overlapping updates causing confusing outcomes.