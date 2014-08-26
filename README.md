SwiftData
=========

C api's are a pain in Swift - that's where SwiftData comes in.

SwiftData is a simple and effective wrapper around the SQLite3 C api written completely in Swift.

##Features

- Execute direct SQL statements
- Bind objects conveniently to a string of SQL
- Queries return an easy to handle array of data
- Support for transactions and savepoints
- Inline error handling
- Completely thread-safe by default
- Convenience functions for common tasks, including table and index creation/deletion/querying
- Database connection operations (opening/closing) are automatically handled

##Installation

Currently, it's as easy as dragging the file 'SwiftData.swift' into your project.
Ensure that you've added 'libsqlite3.dylib' as a linked framework and that you've added `#import "sqlite3.h"` to your Briding-Header.h file.

##Requirements

- Xcode 6
- iOS 7.0+

##Usage

This section runs through some sample usage of SwiftData.

The full API documentation can be found [here](http://ryanfowler.github.io/SwiftData)

####Table Creation

A Table can be created using the convenience function:

```
if let err = SD.createTable("Cities", withColumnNamesAndTypes: ["Name": .string, "Population": .int, "IsWarm": .bool, "FoundedIn": .date]) {
    //there was an error during this function, handle it here
} else {
    //no error, the table was created successfully
}
```

=================
###Execute Change

Alternatively, a Table could be created using a direct SQL statement:

```
if let err = SD.executeChange("CREATE TABLE Cities (ID INTEGER PRIMARY KEY AUTOINCREMENT, Name TEXT, Population INTEGER, IsWarm BOOLEAN, FoundedIn DATE)") {
    //there was an error during this function, handle it here
} else {
    //no error, the table was created successfully
}
```

The `SD.executeChange(...)` function can be used to execute any non-query SQL statement.

Note that by default, error and warning messages are printed to the console. If an error code is returned from a function call, the related error message can be obtained by calling the function:

`let errMsg = SD.errorMessageFromCode(err)`

Now that we've created our Table, "Cities", we can insert a row into it:

```
if let err = SD.executeChange("INSERT INTO Cities (Name, Population, IsWarm, FoundedIn) VALUES ('Toronto', 2615060, 0, '1793-08-27')") {
    //there was an error during the insert, handle it here
} else {
    //no error, the row was inserted successfully
}
```

#####Binding Values

Or we can insert a row with object binding:

```
//from user input
let name: String = //user input
let population: Int = //user input
let isWarm: Bool = //user input
let foundedIn: NSDate = //user input

if let err = SD.executeChange("INSERT INTO Cities (Name, Population, IsWarm, FoundedIn) VALUES (?, ?, ?, ?)", withArgs: [name, population, isWarm, foundedIn]) {
    //there was an error during the insert, handle it here
} else {
    //no error, the row was inserted successfully
}
```

The provided objects will be escaped and will bind to the '?' characters (in order) in the string of SQL.
You may escape objects yourself using the function:

`let escValue = SD.escapeValue(object)`

The above function will escape the provided object and include single quotes around string values. 'escValue' is ready to be directly inserted into a SQL statement.

Object types supported to bind as values are:

- String
- Int
- Double
- Bool
- NSDate
- NSData
- nil

All other object types will bind to the SQL string as 'NULL'.

#####Binding Identifiers

If an identifier (e.g. table or column name) is provided by the user and needs to be escaped, you can use the characters 'i?' to bind the objects like so:

```
//from user input
let columnName1 = //user input
let columnName2 = //user input

if let err = SD.executeChange("INSERT INTO Cities (Name, Population, i?, i?) VALUES (?, ?, ?, ?)", withArgs: [columnName1, columnName2, name, population, isWarm, foundedIn]) {
    //there was an error during the insert, handle it here
} else {
    //no error, the row was inserted successfully
}
```

The objects 'columnName1' and 'columnName2' will bind to the characters 'i?' in the string of SQL as identifiers. Double quotes will be placed around each identifier.
You may escape an identifier string yourself using the function:

`let escIdentifier = SD.escapeIdentifier(identifier)`

Objects provided to bind as identifiers must be of type String.

====================
###Execute Query

Now that our table has some data, we can query it:

```
let (resultSet, err) = SD.executeQuery("SELECT * FROM Cities")
if err != nil {
    //there was an error during the query, handle it here
} else {
    for row in resultSet {
        if let name = row["Name"].asString() {
            println("The City name is: \(name)")
        }
        if let population = row["Population"].asInt() {
            println("The population is: \(population)")
        }
        if let isWarm = row["IsWarm"].asBool() {
            if isWarm {
                println("The city is warm")
            } else {
                println("The city is cold")
            }
        }
        if let foundedIn = row["FoundedIn"].asDate() {
            println("The city was founded in: \(foundedIn)")
        }
    }
}
```

A query function returns a tuple of:

- the result set as an Array of SDRow objects
- the error code as an Optional Int

The error code should always be checked against nil and handled if it exists.
If there is no error, the result set can be used.

An SDRow contains a number of SDColumn objects.
The values for each column can be obtained by using the column name in subscript format, much like a Dictionary.
In order to obtain the column value as the correct type, you may use the convenience functions:

- asString()
- asInt()
- asDouble()
- asBool()
- asDate()
- asData()

So for example, if you want the string value for the column "Name":

```
if let name = row["Name"].asString() {
    //the value for column "Name" exists as a String
} else

    //the value is nil, cannot be cast as a String, or the column requested does not exist
}
```

You may also execute a query using object binding, similar to the insert example above:

```
let (resultSet, err) = SD.executeQuery("SELECT * FROM Cities WHERE Name = ?", withArgs: ["Toronto"])
if err != nil {
    //there was an error during the query, handle it here
} else {
    for row in resultSet {
        if let name = row["Name"].asString() {
            println("The City name is: \(name)") //should be "Toronto"
        }
        if let population = row["Population"].asInt() {
            println("The population is: \(population)")
        }
        if let isWarm = row["IsWarm"].asBool() {
            if isWarm {
                println("The city is warm")
            } else {
                println("The city is cold")
            }
        }
        if let foundedIn = row["FoundedIn"].asDate() {
            println("The city was founded in: \(foundedIn)")
        }
    }
}
```

The same binding rules apply as described in the earlier Insert example.

=================
###Custom Connection

You may create a custom connection and execute a number of functions within a provided closure.
This can be done like so:

```
let task: ()->Void = {
    if let err = SD.executeChange("INSERT INTO Cities VALUES ('Vancouver', 603502, 1, '1886-04-06')") {
        println("Error inserting city")
    }
    if let err = SD.createIndex(name: "NameIndex", onColumns: ["Name"], inTable: "Cities", isUnique: true) {
        println("Index creation failed")
    }
}

if let err = SD.executeWithConnection(.readWrite, task) {
    //there was an error opening or closing the custom connection
} else {
    //no error, the closure was executed
}
```

The custom connection flags are:

- .readOnly (SQLITE_OPEN_READONLY)
- .readWrite (SQLITE_OPEN_READWRITE)
- .readWriteCreate (SQLITE_OPEN_READWRITE | SQLITE_OPEN_CREATE)

All operations that occur within the provided closure are executed on the single custom connection.


=================
###Transactions and Savepoints

If we wanted to execute the above closure `task: ()->Void` inside an exclusive transaction, it could be done like so:

```
if let err = transaction(task) {
    //there was an error starting, closing, committing, or rolling back the transaction as per the error code
} else {
    //the transaction was executed without errors
}
```

Similarly, a savepoint could be executed like so:

```
if let err = savepoint(task) {
    //there was an error starting, closing, releasing, or rolling back the savepoint as per the error code
} else {
    //the savepoint was executed without errors
}
```

It should be noted that transactions *cannot* be embedded into another transaction or savepoint.
Unlike transactions, savepoints may be embedded into other savepoints or transactions.

For more information, see the SQLite documentation for [transactions](http://sqlite.org/lang_transaction.html) and [savepoints](http://www.sqlite.org/lang_savepoint.html).


=================
###Thread Safety

All SwiftData operations are placed on a custom serial queue and executed in a FIFO order.

This means that you can access the SQLite database from multiple threads without the need to worry about causing errors.


##API Documentation

Full API Documentation can be found [here](http://ryanfowler.github.io/SwiftData)


##Contact

Email: ryan.fowler19 [at] gmail.com

Comments, concerns, ideas, and pull requests welcome :)


##License

SwiftData is released under the MIT License.

See the LICENSE file for full details.
