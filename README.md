SwiftData
=========

Working with SQLite is a pain in Swift - that's where SwiftData comes in.

SwiftData is a simple and effective wrapper around the SQLite3 C API written completely in Swift.


##Features

- Execute SQL statements directly
- Bind objects conveniently to a string of SQL
- Queries return an easy to use array
- Support for transactions and savepoints
- Inline error handling
- Completely thread-safe by default
- Convenience functions for common tasks, including table and index creation/deletion/querying
- Database connection operations (opening/closing) are automatically handled


##Installation

Currently, it's as easy as adding the file 'SwiftData.swift' as a git submodule, and dragging it into your project.
Ensure that you've added 'libsqlite3.dylib' as a linked framework and that you've added `#import "sqlite3.h"` to your Briding-Header.h file.


##System Requirements

Xcode Version:

- Xcode 6

Can be used in applications with operating systems:

- iOS 7.0+
- Mac OS X 10.9+


##Usage

This section runs through some sample usage of SwiftData.

The full API documentation can be found [here](http://ryanfowler.github.io/SwiftData)


###Table Creation

By default, SwiftData creates and uses a database called 'SwiftData.sqlite' in the 'Documents' folder of the application.

To create a table in the database, you may use the convenience function:

```swift
if let err = SD.createTable("Cities", withColumnNamesAndTypes: ["Name": .StringVal, "Population": .IntVal, "IsWarm": .BoolVal, "FoundedIn": .DateVal]) {
    //there was an error during this function, handle it here
} else {
    //no error, the table was created successfully
}
```

Similar convenience functions are provided for:

- deleting a table:
```swift
let err = SD.deleteTable("TableName")
```
- finding all existing tables in the database:
```swift
let (tables, err) = SD.existingTables()
```

Alternatively, a table could be created using a SQL statement directly, as shown in the 'Execute A Change' section below.


=================
###Execute A Change

The `SD.executeChange()` function can be used to execute any non-query SQL statement (e.g. INSERT, UPDATE, DELETE, CREATE, etc.).

To create a table using this function, you could use the following:

```swift
if let err = SD.executeChange("CREATE TABLE Cities (ID INTEGER PRIMARY KEY AUTOINCREMENT, Name TEXT, Population INTEGER, IsWarm BOOLEAN, FoundedIn DATE)") {
    //there was an error during this function, handle it here
} else {
    //no error, the table was created successfully
}
```

The table created by this function call is the equivalent of the convenience function used in the earlier section.

Now that we've created our table, "Cities", we can insert a row into it like so:

```swift
if let err = SD.executeChange("INSERT INTO Cities (Name, Population, IsWarm, FoundedIn) VALUES ('Toronto', 2615060, 0, '1793-08-27')") {
    //there was an error during the insert, handle it here
} else {
    //no error, the row was inserted successfully
}
```

#####Binding Values

Alternatively, we could insert a row with object binding:

```swift
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

Be aware that although this uses similar syntax to prepared statements, it actually uses the public function:
```swift
let escValue = SD.escapeValue(object)
```
to escape objects internally, which you may also use yourself.
This means that the objects will attempt to bind to *ALL* '?'s in the string of SQL, including those in strings and comments.

The objects are escaped and will bind to a SQL string in the following manner:

- A String object is escaped and surrounded by single quotes (e.g. 'sample string')
- An Int object is left untouched (e.g. 10)
- A Double object is left untouched (e.g. 10.8)
- A Bool object is converted to 0 for false, or 1 for true (e.g. 1)
- An NSDate object is converted to a string with format 'yyyy-MM-dd HH:mm:ss' and surrounded by single quotes (e.g. '2014-08-26 10:30:28')
- An NSData object is prefaced with an 'X' and converted to a hexadecimal string surrounded by single quotes (e.g. X'1956a76c')
- A UIImage object is saved to disk, and the ID for retrieval is saved as a string surrounded by single quotes (e.g. 'a98af5ca-7700-4abc-97fb-60737a7b6583')

All other object types will bind to the SQL string as 'NULL', and a warning message will be printed to the console.

#####Binding Identifiers

If an identifier (e.g. table or column name) is provided by the user and needs to be escaped, you can use the characters 'i?' to bind the objects like so:

```swift
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
```swift
let escIdentifier = SD.escapeIdentifier(identifier)
```
Objects provided to bind as identifiers must be of type String.


====================
###Execute A Query

Now that our table has some data, we can query it:

```swift
let (resultSet, err) = SD.executeQuery("SELECT * FROM Cities")
if err != nil {
    //there was an error during the query, handle it here
} else {
    for row in resultSet {
        if let name = row["Name"]?.asString() {
            println("The City name is: \(name)")
        }
        if let population = row["Population"]?.asInt() {
            println("The population is: \(population)")
        }
        if let isWarm = row["IsWarm"]?.asBool() {
            if isWarm {
                println("The city is warm")
            } else {
                println("The city is cold")
            }
        }
        if let foundedIn = row["FoundedIn"]?.asDate() {
            println("The city was founded in: \(foundedIn)")
        }
    }
}
```

A query function returns a tuple of:

- the result set as an Array of SDRow objects
- the error code as an Optional Int

An SDRow contains a number of corresponding SDColumn objects.
The values for each column can be obtained by using the column name in subscript format, much like a Dictionary.
In order to obtain the column value in the correct data type, you may use the convenience functions:

- asString()
- asInt()
- asDouble()
- asBool()
- asDate()
- asData()
- asAnyObject()
- asUIImage()

If one of the above functions is not used, the value will be an SDColumn object.

For example, if you want the string value for the column "Name":

```swift
if let name = row["Name"]?.asString() {
    //the value for column "Name" exists as a String
} else
    //the value is nil, cannot be cast as a String, or the column requested does not exist
}
```

You may also execute a query using object binding, similar to the row insert example in an earlier section:

```swift
let (resultSet, err) = SD.executeQuery("SELECT * FROM Cities WHERE Name = ?", withArgs: ["Toronto"])
if err != nil {
    //there was an error during the query, handle it here
} else {
    for row in resultSet {
        if let name = row["Name"]?.asString() {
            println("The City name is: \(name)") //should be "Toronto"
        }
        if let population = row["Population"]?.asInt() {
            println("The population is: \(population)")
        }
        if let isWarm = row["IsWarm"]?.asBool() {
            if isWarm {
                println("The city is warm")
            } else {
                println("The city is cold")
            }
        }
        if let foundedIn = row["FoundedIn"]?.asDate() {
            println("The city was founded in: \(foundedIn)")
        }
    }
}
```

The same binding rules apply as described in the 'Execute a Change' section.


=================
###Error Handling

You have probably noticed that almost all SwiftData functions return an 'error' value.

This error value is an Optional Int corresponding to the appropriate error message, which can be obtained by calling the function:

```swift
let errMsg = SD.errorMessageForCode(err)
```

It is recommended to compare the error value with nil to see if there was an error during the operation, or if the operation was executed successfully.

By default, error and warning messages are printed to the console when they are encountered.


=================
###Creating An Index

To create an index, you may use the provided convenience function:

```swift
if let err = SD.createIndex("NameIndex", onColumns: ["Name"], inTable: "Cities", isUnique: true) {
    //there was an error creating the index, handle it here
} else {
    //the index was created successfully
}
```

Similar convenience functions are provided for:
- removing an index:
```swift
let err = removeIndex("IndexName")
```
- finding all existing indexes:
```swift
let (indexes, err) = existingIndexes()
```
- finding all indexes for a specified table:
```swift
let (indexes, err) = existingIndexesForTable("TableName")
```


=================
###Custom Connection

You may create a custom connection to the database and execute a number of functions within a provided closure.
An example of this can be seen below:

```swift
let task: ()->Void = {
    if let err = SD.executeChange("INSERT INTO Cities VALUES ('Vancouver', 603502, 1, '1886-04-06')") {
        println("Error inserting city")
    }
    if let err = SD.createIndex(name: "NameIndex", onColumns: ["Name"], inTable: "Cities", isUnique: true) {
        println("Index creation failed")
    }
}

if let err = SD.executeWithConnection(.ReadWrite, task) {
    //there was an error opening or closing the custom connection
} else {
    //no error, the closure was executed
}
```

The available custom connection flags are:

- .ReadOnly (SQLITE_OPEN_READONLY)
- .ReadWrite (SQLITE_OPEN_READWRITE)
- .ReadWriteCreate (SQLITE_OPEN_READWRITE | SQLITE_OPEN_CREATE)

All operations that occur within the provided closure are executed on the single custom connection.

For more information, see the SQLite documentation for [opening a new database connection](http://www.sqlite.org/c3ref/open.html).


=================
###Transactions and Savepoints

If we wanted to execute the above closure `task: ()->Void` inside an exclusive transaction, it could be done like so:

```swift
if let err = transaction(task) {
    //there was an error starting, closing, committing, or rolling back the transaction as per the error code
} else {
    //the transaction was executed without errors
}
```

Similarly, a savepoint could be executed like so:

```swift
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
###Using UIImages

Convenience functions are provided for working with UIImages.

To easily save a UIImage to disk and insert the corresponding ID into the database:

```swift
let image = UIImage(named:"SampleImage")
if let imageID = SD.saveUIImage(image) {
    if let err = SD.executeChange("INSERT INTO SampleImageTable (Name, Image) VALUES (?, ?)", withArgs: ["SampleImageName", imageID]) {
        //there was an error inserting the new row, handle it here
    }
} else {
    //there was an error saving the image to disk
}
```

Alternatively, object binding can also be used:

```swift
let image = UIImage(named:"SampleImage")
if let err = SD.executeChange("INSERT INTO SampleImageTable (Name, Image) VALUES (?, ?)", withArgs: ["SampleImageName", image]) {
    //there was an error inserting the new row, handle it here
} else {
    //the image was saved to disk, and the ID was inserted into the database as a String
}
```

In the examples above, a UIImage is saved to disk and the returned ID is inserted into the database as a String.
In order to easily obtain the UIImage from the database, the function '.asUIImage()' called on an SDColumn object may be used:

```swift
let (resultSet, err) = SD.executeQuery("SELECT * FROM SampleImageTable")
if err != nil {
    //there was an error with the query, handle it here
} else {
    for row in resultSet {
        if let image = row["Image"]?.asUIImage() {
            //'image' contains the UIImage with the ID stored in this column
        } else {
            //the ID is invalid, or the image could not be initialized from the data at the specified path
        }
    }
}
```

The '.asUIImage()' function obtains the ID as a String and returns the UIImage associated with this ID (or will return nil if the ID was invalid or a UIImage could not be initialized).

If you would like to delete the photo, you may call the function:

```swift
if SD.deleteUIImageWithID(imageID) {
    //image successfully deleted
} else {
    //there was an error deleting the image with the specified ID
}
```

This function should be called to delete the image with the specified ID from disk *before* the row containing the image ID is removed.
Removing the row containing the image ID from the database does not delete the image stored on disk.


=================
###Thread Safety

All SwiftData operations are placed on a custom serial queue and executed in a FIFO order.

This means that you can access the SQLite database from multiple threads without the need to worry about causing errors.


##API Documentation

Full API Documentation can be found [here](http://ryanfowler.github.io/SwiftData)


##Contact

Please open an issue for any comments, concerns, ideas, or potential pull requests - all welcome :)


##License

SwiftData is released under the MIT License.

See the LICENSE file for full details.
