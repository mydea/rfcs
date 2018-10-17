- Start Date: 2018-10-17
- RFC PR: (leave this empty)
- Ember Issue: (leave this empty)

# [Ember Data] Move record finding from store to adapter

## Summary

Currently, the functionality based on `coalesceFindRequest=true` is deeply entangled with the store. This means that it is next to impossible to properly change this behavior, and it also increases complexity of the store. The same is true for relationship loading - the decision when to use `links` and when to use `data` is currently not customizable.

## Motivation

The proposed changes would solve two issues. First, it would disentangle the store & the adapter, making changes & refactors much easier and more understandable.

In addition, it would make it possible to customize behavior that is currently being decided very intransparently by Ember Data itself, somewhere deep in the internals. This includes when to use `coalesceFindRequests` and when to use `links`/`data` for relationships.

## Detailed design

This RFC proposes to add two new (public) methods to the REST Adapter: `findResource` and `findRelationship`.

### findResource

This new method will be responsible for deciding if to use `coalesceFindRequests`. Depending on this, it will either call the existing `findRecord` or `findMany` methods on the adapter. It will also handle the normalization, and call a new, private method `_pushRecordIntoStore` that will actually handle the conversion to Ember Data records.

Below you can find a simplified implementation of the new `findResource` method. Note that this is not about exact implementation, but it should show the flow of data through the system.

```js
export default RESTAdapter.extend({

  async findResource(store, type, id, snapshot) {
    if (this.coalesceFindRequests) {
      return this._scheduleFindResource(store, type, id, snapshot);
    }
    
    let payload = await this.findRecord(store, type, id, snapshot);
    let serializer = store.serializerFor(type.modelName);
    let normalizedPayload = serializer.normalizeFindRecordResponse(type, payload);
    return this._pushRecordIntoStore(store, type, normalizedPayload, snapshot);
  },
  
  _scheduleFindResource(store, type, id, snapshot) {
    // keep a map of find requests per type
    // run this._flushCoalesceFind in the next runloop to resolve them together
    return new Promise(...);
  },
  
  async _flushCoalesceFind(type) {
    let pendingFinds = []; // get pending finds for the given type from the internal map
    
    // If just fetching one, use findRecord
    if (pendingFinds.length === 1) {
      let { store, id, type, snapshot, resolveFind } = pendingFinds[0];
      let payload = await this.findRecord(store, type, id, snapshot);
      
       let serializer = store.serializerFor(type.modelName);
       let normalizedPayload = serializer.normalizeFindRecordResponse(type, payload);
       resolveFind(this._pushRecordIntoStore(store, type, normalizedPayload, snapshot));
       return;
    }
    
    // Else use findMany
    let ids = pendingFinds.mapBy('id');
    let snapshots = pendingFinds.mapBy('snapshot');
    let { store } = pendingFinds[0];
    
    // In reality, this should use this.groupRecordsForFindMany()
    let response = await this.findMany(store, type, ids, snapshots);
    
    let serializer = store.serializerFor(type.modelName);
    let normalizedResponse = serializer.normalizeFindManyResponse(type, payload);

    pendingFinds.forEach(({ id, resolveFind, snapshot }) => {
      let recordPayload = normalizedResponse.data.findBy('id', id);
      resolveFind(this._pushRecordIntoStore(store, type, recordPayload, snapshot));
    });
  }

});
```

`adapter.findResource()` would be called from the store, and would replace the current (private) finder functions.

### findRelationship

The second new method `findRelationship` would be called by the store/internal model when trying to get belongsTo or hasMany relationhips for a given model. It would make the decision if to use `links` or `data`, and call the corresponding methods.

A simplified implementation could look like this. Again, please note that this is not about the exact implementation, but about how the parts of the system play together.

```js
export default RESTAdapter.extend({

  async findRelationship(store, parentModel, relationshipHash, relationshipType) {
    if (this._shouldUseRelationshipLink(store, parentModel, relationshipHash)) {
      let methodName = relationshipType === 'belongsTo' 
        ? 'findBelongsTo' 
        : 'findHasMany';
      let payload = await this[methodName](...);
      
      let serializer = store.serializerFor(...);
      let normalizeMethodName = relationshipType === 'belongsTo' 
        ? 'normalizeFindBelongsTo' 
        : 'normalizeFindHasMany';
      let normalizedPayload = serializer[normalizeMethodName](...);
      
      return normalizedPayload.data.map((recordPayload) => {
        return this._pushRecordIntoStore(...);
      });
    }
    
    // Else just use findResource
    if (relationshipType === 'hasMany') {
      let ids = relationshipHash.data;
      return Promise.all(ids.map((id) => this.findResource(...)));
    }
    
    let id = relationshipHash.data;
    return this.findResource(...);
  }
  
});
```

## How we teach this

The new public methods would need to be added to the API docs. Since they are rather advanced customization options, they probably don't need to be covered in the guides.

## Drawbacks

It would increase the API surface of the adapter. 

Additionally, it would introduce further, similarly named, methods, which could increase confusion - e.g. between `findRecord`, `findResource`, etc.

## Alternatives

We could also make the new methods private (`_findResource`/`_findRelationship`), which would avoid increasing the API surface and "hide" the new terms from developers. However, I do think that there is a very valid use case for attempting to customize this behavior (e.g. when to use links, when to use coalescing, ...), which this RFC would unlock. Currently, this is pretty much hard-coded somewhere in the internals of the store service, which makes it both hard to understand and nearly impossible to customize.

We could also just leave it as it is. However, the current implementation means a very tight entangelement between the store, the adapter, the serializer and the (private) finders. It is both hard to understand (when working on it), as well as hard to customize. It also makes it hard to refactor things.

## Unresolved questions

How exactly should the method signature be for the new methods? For `findResource`, we could just use the same as for `findRecord`. For `findRelationship`, We'll need to include the `store`, `parentModel`, `relationshipType` (belongsTo/hasMany), and detailed information about the relationship itself, including `links` and `data`.
