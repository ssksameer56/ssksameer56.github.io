# Migrating From Node JS to Golang - Experience

# Overview
Recently, we moved a major API consumed by our mobile app from NodeJS to Golang. Since, the project was for migrating an existing API and not creating one, the challenges were far greater than you normally encounter. Lots of features that Golang offers simply become a roadblock in trying to recreate the same functionality and sometimes the functionality simply doesnt exist. When we combine this with a need to make the API faster, it becomes even tougher to find the solution to all these problems. I'll try to cover what problems we encountered, how we tried to solve them and any other learnings we got from this.

## The Problem At Hand
Our API was written in NodeJS with a PostgresDB and Redis cache. A fairly normal setup for any modern API. Since the API caters to UX in the app, it was important to have as fast response time as possible. We also did not have the liberty to design our own response or request bodies as those were defined in the front end. Any change would require us to spend time on front end, which we simply didnt have.

This was exacerbated by the fact that the Node API leveraged a lot of JS and JSON specific functionality which doesnt simply translate to Go easily. 

## Using JSON in Postgres
For effeciency, some data was computed as JSON already in the database for our API. While it is tempting to map the database as structs using ORM or by hand, it often just adds excess computation. Every unmarshal/marshall operation will be costly and its important to compute it once and be done with it. Operating on marshalled JSON is costly in Go, so use it sparingly. We could break down the query into simple SQL components and read into GO structs(we use PGX) or keep the JSON part as it and unmarshal into a struct. We found the second approach better - you wont have to test for changes in your query and still work on data in Go structs. 

In essence, if the data is to be read from database and not operated on, you can forgo some principles and create a JSON in Postgres itself. Unmarshal if some things are to be read or operated and marshal again.

## Structs, Objects and Types
Go is a strongly typed language while JS isnt. As such, when developing APIs in Node, we often took advantage of that fact. Fields and variables changed types easily. The structure of any object changed quite a lot as requirements came and features were added. If we tried to map the structure into Go structs as would be expected, we ended up with almost 6-7 structs for one simple object in JS all because of the changed structure, recursive calls, renaming fields, and changing types of variables. While this is the way you should write in Go, it becomes unmanagable very fast and you end up with a lot of transformators/adaptors to convert the structs as the logic flows.

What helped is a mega struct which encompasses all the fields used in the logic or the function. all operations can be simply be done by assigning values to the fields. To get the desired structure, empty or assign the required fields. With omitempty, when you finally marshal, the response should be exactly what you want. This also removes the overload of transformators or convertors. *This is especially useful for logic that modifies fields due to recursion.* 

In case an object is being modified or its structure changing, its easier to create a Go struct with all the possible fields and mutate the individual fields as needed.


### Un-breaking the Front End
Go either keeps a field in marshalling or removes if `omitempty` is present. It is important therefore that the client consuming does not only check for null but undefined. This would not introduce breaking changes.


### Making It Better
The easiest way to speed up the JSON part of the API is to use libraries such as EasyJSON rather than the default encoding/json. This gives the performance boost.

Second would be to avoid maps of interfaces. Its very easy to use maps of interfaces when the structure of the response is not known, but if you have to interact with data it becomes a terrible choice. Between typecasting, nil checks and manipulating you spend a lot of useless time as well as increases a risk of unhandled panics and errors. 


### Quality of Life Changes

#### string Tags
A lot of times, values which are numbers get often passed as strings to an API. While JS can handle these cases, Go cant. The unmarshalling function panics. You can handle it by adding the `string` tag to the Go tag property and use `json.Number`. Json.Number gives a string which can be formatted to string/number as required.  

#### VSCode Tools

The Marketplace has tools that help you autocomplete generating tags for json,db properties which are useful when generating your API structs.