---
layout: post
title:  "Better E2E API Mocks"
date:   2021-10-26 10:44:25 -0500
categories: frontend software testing mocks testing e2e api
---

The first time I wrote an end-to-end test suite for a frontend application. I approached the problem naively and based each test case on user flows that I could understand and replicate by navigating through the app that was currently in production.

Replicating the flows in an automated browser was simple enough as I could copy the network responses from my prod session and save them locally as individual JSON files to serve to the api requests coming from the automated browser through an http intercepter. The shortcoming of this solution is obvious, for every variation of the underlying response object, a new json file has to be added to the codebase and any tests which need to use this response need to import it. At worst, this method of storing reponses results in a new file being saved for every variation of every response used by your application. Worse still is that if any object in these JSON files has their definition changed outside the scope of the frontend application, all of the files which reference that object also have to be updated.

The solution to this issue is object factories, functions that produce an object with object properties based on input parameters. Instead of saving massive amounts of JSON responses locally, you can create a single function that arbitrarily populates an object's values. Better yet, since a single function is in-charge of creating all possible variants of the underlying object, if a change is made to the definition of the object, that change only needs to be made in the factory function.

This method solves the problem of scalability and maintainability. It puts the developer in a position where they can *start* to mock APIs knowing that it won't lead to a ridiculous amount of static json objects. With factories in place, the focus of a developer can shift to *what* objects should one make a factory *for*? One might be tempted to keep this scheme simple and create one object factory for each distinct API route so that these factories represent an http response object. While this solution seems reasonable at the outset and may meet the needs of some developers, it is dependant on the values returned by these responses being consistent and independant of the other routes being mocked.

For example, let's suppose a library has and API that provides 2 routes that users can send a `GET` request to:

* `/books`

  * Returns an array of books with author information:

    ```json
    [{
      "id": 1000,
    	"title": "To Kill a Mockingbird",
      "author": {
        "first_name": "Harper",
        "last_name": "Lee"
      }
    }
    ```

* `/authors?author_id={id}`

  * Returns an authors personal information

    ```json
    {
      "id": 1001,
      "first_name": "Harper",
      "last_name": "Lee"
    }
    ```

If a change were introduced into the object representing `author`, adding a `date_of_birth` field for example, then an app with factories for the responses for `/book` and `/author` would need to see both factories updated to have them maintain accuracy with the source data.

This example makes the next step of abstraction obvious, introduce more object factories for the objects that are returned by these API responses. But wait! Just as it was important to have a strategy to limit the number of local JSON files, the number of factories and a structured way to define them should also be considered.

Specificially in an end-to-end frontend test suite, I have found it helpful to separate factories into two distinct groups: Response Factories and Object Factories. Response factories represent objects that will be consumed directly by the frontend application code and Object factories represent some values that are stored in a database and composed into responses by the backend. With this in mind, a defined structure starts to emerge:

```
/Factories
	/Objects
		- BookFactory.js
		- AuthorFactory.js
	/Responses
		- BookResponseFactory.js
		- AuthorResponseFactory.js
			
```

Now the responses are separate from the underlying objects which they are composed of and changes to either the `book` object or `author` object can again be made in a single place. The amount of factories now scales with the amount of objects returned from the backend (or defined in the database), which is almost certainly smaller than the amount of possible variations if we *only* created factories for responses.

Still this can be improved, while it is good to have the *ability* to mock all possible variations of the known objects and responses, it is rarely the case where one actually has the *need* to mock all possible object variations. A neat aspect of writing these object factories is that it provides specific targets to analyze for equivalence partitioning.

The subset of valid object values is likely well understood or easy to arrive at and some values can be expected or even guaranteed. These values are of particular interest because they help significantly limit the variations an object created by this factory might have and the knowledge of what values can be expected or guaranteed provides a better understanding for what the object itself is meant to represent, particularly for people who aren't familiar with the project. e.g.: As a developer, I will know that book objects are expected to have certain non-null values and have a relation to its author.

Storing common factory arguments gives the developer a good idea of what the objects used in production will be, or atleast what kinds of objects will be expected. I'll refer to the collection of common factory arguments as a "preset". If a factory is considered a blueprint, a preset can be considered the building materials.

Book Factory:

```javascript
export default const BookFactory => ({
	id: null,
	title: ''
	author: {}
} = {}){
	return {
    id,
    title,
    author
  };
}
```

Book Preset:

```
export default const BookPreset = {
	id: 1000,
	title: 'Preset Book Title',
	author: AuthorFactory(AuthorPreset)
}
```

The tightly coupled nature of presets and factories prompts us to make a small change in the directory structure of our factories:

```
/Factories
	/Objects
		/Book
			- BookFactory.js
			- BookPresets.js
		/Author
			- AuthorFactory.js
			- AuthorPresets.js
	/Responses
		/book
			/GET
				- BookResponseFactory.js
				- BookResponsePresets.js
		/author
			/GET
				- AuthorResponseFactory.js
				- AuthorResponsePresets.js
```

\* Added in a `/GET` route to make a distinction between different kinds of HTTP responses.

This is close to a solution that I have implemented in the past to tack the issue of complex API mocking for end-to-end tests. The final nuance that I have added to this system of mocks is to limit the wordiness of all the imports and to expose a common collection of mocked response factories as a single "bundle" that defines a common user configuration and exports it all from a single file. I've found it helpful to bundle factories in a single file because, depending on the complexity of a e2e user session, if all relevant factories and its presets need to be imported in a single test spec, the resulting list of imports can be quite cumbersome.

ex: `Factories.js` that exports all objects this far:

```
import BookFactory from '...'
import BookPresets from '...'
import AuthorFactory from '...'
import AuthorPresets from '...'

export default const Factories = {
	Book: BookFactory(BookPreset),
	Author: AuthorFactory(AuthorPreset)
}
```

or if there are multiple presets defined for a factory, a default export structure can be defined that exposes all the presets:

```
import BookFactory from '...'
import * as BookPresets from '...'
import AuthorFactory from '...'
import * as AuthorPresets from '...'

export default const Factories = {
	Book: {
		factory: BookFactory,
		presets: BookPresets
	},
	Author: {
		factory: AuthorFactory,
		presets: AuthorPresets
	}
}
```

This approach will probably better suit some projects than others. This system was created to mock API's coming from a django backend and the number of object factories roughly matched the number of ways database objects were serialized to JSON. A similar relationship existed for the number of response serializers. This method can be expanded to handle cases of common serialization patterns such as paginated responses or error responses but I have not yet had the time to get into the weeds with this system. What I do know is that this pattern is significantly more scalable than any e2e api mocks that I've worked with in the past and still has plenty of room to be improved.