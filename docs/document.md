# Documents

A document represents the union of a data record with the schema and methods you decide to assign to it. Although Osmos provides a `Document` class, you never initialize it directly—the task of creating and populating a document instance is always delegated to a model.

For the most part, Osmos documents behave like every other JavaScript objects; you can read and write their values, execute their methods, and so forth. They are designed so that they are natural to use under most circumstances.

Underneath, however, a document enforces a few extra rules, the most important of which is that you cannot add or remove properties from it at runtime. This is done because one of Osmos's primary design goals is to create an ODM that puts the preservation of its data before anything else.

Consider the simple scenario in which you write some new code where you mistype the name of a field in your document. Because JavaScript normally allows you to add properties to an object at runtime, that mistake may be very hard to track down—or, worse, it may go unnoticed and cause the loss of data. If you use Osmos, on the other way, the document will notify immediately—in the form of a runtime exception (which will, hopefully, be caught by your unit test coverage).

## Debug vs. Production mode

In debug mode (which is on by default), the Document class enforces read/write constraints by using an ECMA feature called [Direct Proxies](http://wiki.ecmascript.org/doku.php?id=harmony:direct_proxies); this allows Osmos to wrap documents around a “proxy” object that can intercept all attempts to access a property and throw an error when that property is not part of the document.

The downside of this approach is that proxies slow things down and tend to make a mess of Node's memory heap. Therefore, you can turn them off in production by setting the `debug` property of the `Document` class to `false`:

```javascript
if (config.production) // `config` is an object your app defines
  // Turn off debugging mode
  Osmos.Document.debug = false;
}
```

When `debug` is `false`, instead of using Direct Proxies, Osmos gives you direct access to the document objects, which, however, are sealed using `Object.seal`.

## Reading and writing properties

You read and write a document properties the way you normally would, and the same applies to subdocuments and arrays:

- When you read a value from a field, the raw data that was returned by the backing store for a field is run through any transformers configured on your schema, and then returned to you.
- When you write a value to a field, the data is first checked against all the validators configured for the field, then run through all the transformers configured for it, and finally stored into the raw data storage.

The properties that are part of a document are composed of the following:

- Properties defined in the schema
- Dynamic or derived properties defined in the model
- Intrinsic properties of the document itself

## Deleting properties

As of version 1.2.0, Osmos allows you to delete properties in a document as well—provided, of course, that those properties are actually part of the document's schema.

**Note:** this function has been removed as of version 1.3.0. You should, instead, set properties you want to delete to `undefined`.

## Dealing with validation errors

Validation occurs in two phases. As soon as you try to write a value into a field, Osmos notifies you as to whether you attempted to access a property that is not defined in the document. This is done on purpose, and the failure occurs loudly (with an exception) in keeping with Osmos' main directive of data safety.

Additionally, when you attempt to save a document, its contents are validated against the schema. If the validation fails, the data is not written to the underlying raw store; instead, the errors are returned to the `save` callback:

```javascript
doc.save(function(err) {
  if (err) // report err here
});
```

The error object augments the normal Node Error object by providing a `statusCode` property, useful when writing Web services, and an `errors` hash, which can be used to determine which errors occurred on which fields. It is generally safe to output `statusCode` and `errors` to an external, non-trusted source.

## Clearing a document

Starting with version 1.2.0, the Document class implements a `clear` method that completely empties a document, erasing all its contents. It accepts a single Boolean parameter, which, if set to `false` or left empty, preserves the primary key in the document.

## Modifying a document

In addition to addressing individual properties directly, Osmos documents support (as of version 1.0.3) an `update` method that can be used to batch-modify an arbitrary number of properties in one call. The `update` method is designed as a convenient and secure way to allow external actors to modify the contents of a document using a fail-safe approach that automatically blocks sensitive properties from being updated—normally, this means that you can just grab a JSON package from a POST/PUT/PATCH HTTP operation and pass it directly to `update()`, without having to worry about its contents.

To implement its fail-safe mechanism, `update()` only allows whitelisted fields to updated. You whitelist individual fields by adding them to the `updateableProperties` array of the corresponding model. For example:

```javascript
// When declaring the model:

model.updateableProperties = ['id', 'name'];

// Elsewhere in your app, receive a JSON hash from an external caller:

doc.update(jsonData, function(err) {
  // Handle errors here
});
```

Regardless of what information is stored in the `jsonData` blob shown in the previous example, the document object will only allow those listed in `model.updateableProperties` to be updated.

### Update hooks

It's not uncommon to need to perform additional operations as part of an update. For example, if a user wants to update their password, you will most likely want to hash it before it actually gets written to the data store. In order to avoid having to overload the `update()` method, which is hard to do without introducing unwanted side effects or compromising its fail-safe nature, Osmos allows you to hook to the `willUpdate` and `didUpdate` callbacks, where you can access both the original document and the hash that is being used to update it.

## Saving a document

The data associated with a document is not saved to the backing store implicitly; you need to call the document's `save` method:

    document.save(callback(err, document));
    
When `save()` is executed, it first calls up the underlying schema's `validate()` method, and then attempts to write the data to the backing store. If the document is new and a primary key is not set, Osmos calls the `post` method of the underlying driver and makes the new primary key available. Otherwise, it calls `update()` on the driver.

Note that, as of version 1.0.3, `save()` no longer requires a callback; if you don't care about finding out when the save operation is completed (or whether it reports any errors), you can just avoid passing a callback.

As of version 1.2.0, save passes a reference to the document itself as the second argument of its callback.
    
## Deleting a document

The document class implements a `delete` method that can be used to remove the current document from the backing store. This method only works if the document's model has a primary key.

## Preserving the raw data

Osmos is designed to work with largely unstructured data; therefore, it is built with the expectation that there might be variance in the documents returned by the backing store.

However, an Osmos document enforces a strict structure on its underlying data, leaving a conflict in scenarios when the information available in the backing store doesn't match the schema that you have designed.

In these cases, Osmos behaves thusly:

- _All_ the data retrieved from the backing store is preserved, unaltered, and sent back as part of a save operation, even if it is not part of the schema.
- Validation and transformation is only enforced at the application level—in other words, validators and transformers are only called when you read from and write to the store.
- Osmos doesn't implicitly transform the data that is read from the backing store. In other words, declaring a field as a `String` does not ensure that you will receive a string when you read from it.

This ensures that a mismatch between your expectations and the backing store doesn't cause a catastrophic loss of data, although at the cost of additional work in ensuring that the data read from a document is of the proper type. This is preferable, in the minds of the designers, to blindly forcing a data-type conversion that could cause the loss of data.

## Accessing the raw data

To accommodate situations in which it is impossible to predict the exact schema of a document in all cases, Osmos exposes a `__raw__` property that represents the actual raw data as it was received from—and will be sent to when `save()` is called—the backing store. No transformations and validations are performed when accessing this property and its contents—which, needless to say, means you should only use it under extraordinary circumstances.

## Nested subdocuments and arrays

You should be able to nest subdocuments arbitrarily, and even use them inside arrays. 

## Avoiding naming conflicts

It's entirely possible for a document to have fields whose name conflicts with a method or property of the underlying `OsmosDocument` instance. In this case, Osmos _almost always_ gives document data the precedence, which means that the underlying property or method cannot be accessed directly. (The only exception to this rule is the `constructor` property, which is required to identify documents and overrides everything else.)

## Exporting documents

By default, document objects override the `toJSON()` method, forcing it to throw an exception. The reason for this is that Osmos is designed to be used in JSON-based service-oriented systems, where it's easy and convenient to retrieve a document and then return it to the caller in JSON format using JavaScript's own facilities.

The problem with this approach is that it's all too easy to accidentally expose privileged information. For this reason, Osmos forces developers to explicitly write their JSON rendering methods in an attempt to force them to think about exactly what information they want to release in the wild.

## Extending documents

Because Osmos uses proxying to strictly marshal access to documents, they cannot be extended through traditional means, like simply adding a new method to their prototype.

Instead, the Model object provides two hashes, `instanceMethods` and `instanceProperties` that can be used to add methods and virtual properties to every object that is instantiated by a particular model. 

For example:

```javascript
var model = new Model(schema, bucket, db);

model.instanceMethods.outputData = function outputData() {
    console.log(this.data);
};

model.instanceProperties.age = {
    get : function getAge() {
        return this.age;
    },
    
    set : function setAge(value) {
        this.age = 10;
    }
};

model.create(err, doc) {
    doc.outputData(); // will output the contents of the document
    doc.age = 20;
    
    console.log(doc.age);
};
```

## Hooks

Documents do not directly expose any hooks; instead, their hooks are proxied through the respective model (as explained in the Model section).
