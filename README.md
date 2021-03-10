# iOS-dev-notes
Notes I took for an iOS interview
# iOS/Swift Notes
## @escaping
 - When a closure parameter is marked with `@escaping`, it means that closure will outlive the scope of the function you've passed the closure to.
 
- Escaping closures are usually for functions that perform asynchronous task(s) and use the closure as a callback.
 
- `@escaping` means that the caller must take precautions against retain cycles and memory leaks.
 
- Say we have a function that calls `URLSession.shared.datatask()`, called `RequestData(@escaping completionHandler: (response) -> Void)`  this function has to have the closure parameter marked as `@escaping` because when `URLSession.shared.datatask` is called, `RequestData()` will continue executing code **regardless** of whether `URLSession.shared.datatask` has finished. Therefore `RequestData()` may end and remove itself from the call stack without the closure finishing.
 
## concurrent vs serial queues & main dispatch queue
- **Concurrency** means to execute multiple tasks at the same time
- **Serial** means to execute tasks in order.
- Serial queues execute one task at a time in the order they are added to the queue.
- Concurrent queues execute tasks concurrently but tasks are still started in the order of the queue.
- The **main dispatch queue** is a globally available serial queue that can execute code on the main thread either synchronously or asynchronously.
- `.sync` will block the main thread until the task has finished, `.async` means this will happen on a background thread.
 
## async vs sync
- When executing tasks **synchronously**, tasks are run in order of a FIFO queue, the first task will not be started until the second task has finished.
 
- When executing tasks **asynchronously**, the succeeding task will not wait for the preceding task to finish before it is executed.
 
## mvc
- **M** stands for *model*.
- The model is simply a representation of our data, like a `Class` or `Struct`
- **V** stands for *view*.
- The view receives models from the controller and displays it via the app's UI.
- **C** stands for *controller*.
- The model controllers handle interactions with models.
- A common acronym for this is **CRUD**: create, read, update, delete.
- The controller should have code for all CRUD tasks.
 
## persistence
#### FileManager
- Every iOS app has it's own `documents` directory.
- The `FileManager` class can be used to read/write to the filesystem.
- `FileManager` can:
-- Locate files & directories.
-- Create and delete files & directories.
-- Determine whether a file exists.
-- Gather information (attributes) from a file.
-- List contents of directories.
-- and more...
- https://nshipster.com/filemanager
 
#### PropertyListEncoder & PropertyListDecoder
- `PropertyListEncoder` will serialize models into property-list `.plist` files
- `PropertyListDecoder` will deserialize data into models.
 
#### Bundle
- The `Bundle` class can be used to load files inside our xcode project into memory.
- `let fileURL = Bundle.main.URL(forResource: "my-file", withExtension: ".txt")`
- `let fileContents = try! String(contentsOf: fileURL)`
 
#### UserDefaults
- `UserDefaults` is a key-value pair persistence method that is used to save user preferences, such as if the user wants the app in dark or light mode, or if they prefer metric or customary units of measurement.
 
## CoreData
- **CoreData** is an object graph management library. (It is not meant to be a wrapper for SQLite)

- The CoreData **stack** is comprised of the objects that make CoreData work.

- The CoreData stack is comprised of:
-- The Persistent Store
-- The Persistent Store Coordinator
-- The Managed Object Context
-- The Managed Object Model

**Persistent Store:**
- Instance of `NSPersistentStore`.
- Representation of the file on disk, containing our saved data.
- Typically an SQLite file, but can be one of several formats, or custom formats.

**Managed Object Model:**
- Instance of `NSManagedObjectModel`.
- Includes complete information about data schema.
- Typically created as a file in Xcode's graphical editor.
- Can be created programmatically.
- Used to create our entities and define their attributes and relationships.

**Managed Object Context:**
- Instance of `NSManagedObjectContext`.
- Contains Managed Objects.
- Primary interface to the CoreData stack.
- Implements CRUD tasks.
- Cannot have a Managed Object Model outside of a context.

**Persistent Store Coordinator:**
- Instance of `NSPersistentStoreCoordinator`
- Manages Persistent Stores (Representations of files on disk)
- Middle-man between MOC and PS

### Loading Data:
How to load data from the disk:
1. Create an instance of `NSPersistentStoreCoordinator`.
2. Pass a Managed Object Model to the Persistent Store Coordinator.
3. Tell Persistent Store Coordinator to load the data at a file URL location
4. Persistent Store Coordinator will create an instance of `NSPersistentStore` to represent that file.
5. Create an instance of `NSManagedObjectContext` and hook it up to the Persistent Store Coordinator
6. MOC can now do CRUD tasks from the Persistent Store through the coordinator middle-man

### Saving Data:
1. MOC will have changes in it's context, but not saved to the disk yet. (Think of it as a scratchpad)
2. Call .save() on the MOC.
3. Persistent Store Coordinator will receive those changes, and make the changes to the file on disk.

### Discarding Data:
Changes saved to the MOC will be not permanent until .save() is called on it.
- .reset() can be called to discard changes

## CoreData Models
The model tells CoreData the format of the data to be stored.
- To create a model graphically, create a *.xcdatamodeld* file in Xcode using cmd+n
- Tap "*add entity* on the bottom left"
- Change the entity name in the data model inspector
- Define necessary attributes and their type
- Define any relationship connections

## CoreDataStack
The following is code for a `CoreDataStack` class:
```
import Foundation
import CoreData
class CoreDataStack {
  static let shared = CoreDataStack()

  lazy var managedObjectContext: NSManagedObjectContext {
    guard let model = NSManagedObjectModel.mergedModel(from: [Bundle.main]) else { fatalError("CD Model not found") }
    let persistentStoreCoordinator = NSPersistentStoreCoordinator(managedObjectModel: model)
    try! persistentStoreCoordinator.addPersistentStore(ofType: NSSQLiteStoreType, configurationName: nil, at: storeURL, options: nil)
    let context = NSManagedObjectContext(concurrencyType: .mainQueueConcurrencyType)
    context.persistentStoreCoordinator = persistentStoreCoordinator
    return context
  }()

  private var storeURL: URL {
    let documentsDir = try! FileManager.default.url(for: documentsDirectory, in: .userDomainMask, appropriateFor: nil, create: true)
    return documentsDir.appendingPathComponent("db.sqlite")
  }
}
```

## NSPersistentContainer
`NSPersistentContainer` is a class created by Apple to represent a CoreData stack, the programmer does not have to write code for a `CoreDataStack` class this way.
- MOC is exposed through .viewContext when using this method.
The following is code implementing a CoreData stack using `NSPersistentContainer`.
```
import Foundation
import CoreData

let container = NSPersistentContainer(name: "ModelName")
container.loadPersistentStores { (_, error) in
  if let error = error {
    fatalError("Error loading CD stores: \(error)")
  }
}
```

## Hybrid Stack + NSPersistentContainer
It can be convienent to create a `CoreDataStack` class with `NSPersistentContainer` integrated into it, it's effectively the same as using only`NSPersistentContainer`, but with a `shared` property.
```
import Foundation
import CoreData

class CoreDataStack {
	static let shared = CoreDataStack()
	
	lazy var container: NSPersistentContainer {
	  let container = NSPersistentContainer(name: "ModelName")
      container.loadPersistentStores { (_, error) in
      if let error = error {
        fatalError("Error loading CD stores: \(error)")
      }
    }
    return container
  }()
  
  var mainContext: NSManagedObjectContext {
    return container.viewContext
  }
}
```

## NSManagedObject
CoreData objects are always subclasses of `NSManagedObject`.
The subclass is typically created by xcode's codegen, you can set this option in the .xcdatamodeld under:
*entity name* > *attributes inspector* > *codegen* > *class definition*

 If set to manual, subclasses can be generated by xcode for the programmer:
 *Editor* > *Create NSManagedObject Subclass*

This will create a pair of files for each entity:
- *entity*+CoreDataClass
- *entity*+CoreDataProperties

*entity*+CoreDataClass will contain an empty @objc exposed subclass.
*entity*+CoreDataProperties will contain multiple extensions
- One will contain a fetchRequest and properties for the attributes and relationships for the entity.
- Another will contain accessor methods.

### Convenience initializers
Xcode's codegen will not create initializers that have parameters to specify attributes, to get around this a convenience initializer can be used.
1. Create a new file, call it *entity*+convenience.swift
2. Create an extension on *entity*
3. Create a `convenience init(parameters... , context: NSManagedObjectContext)` method
4. If not using `self.init(context: context)`, ` = CoreDataStack.shared.mainContext` can be appended to the `convenience init()`'s `context` argument.

**TIP**: to see an entity's types, locate the *entity*+CoreDataClass.swift (with cmd+click), click on the filename right above the first line of code in the file in xcode, and select the file's parent folder from the list, then the *entity*+CoreDataProperties are available for viewing. 

## NSFetchRequest
`NSFetchRequest` is able to fetch objects from the persistent store.
`NSFetchRequest` is very extensive, it has the ability to specify criteria for objects to be fetched, along with specifying in what sorted order the objects will be returned.

`NSFetchRequest(entity: String)` takes the name of an entity and returns objects, however the preferred method is to call  `.fetchRequest()` on an subclass instance of `NSManagedObject`.
For example, if we had the CoreData object `Building`, `Building.fetchRequest()` would be called using:
```
let fetchRequest: NSFetchRequest<Building> = Building.fetchRequest()
let moc = CoreDataStack.shared.mainContext
do {
  let buildings = try moc.fetch(fetchRequest)
  print(buildings)
} catch {
  print("Error fetching: \(error)")
}
```

## Save Changes in CoreData
Changes in CoreData only exist in the Managed Object Context, they need to be written to the persistent store on disk by saving our changes with `NSManagedObjectContext.save()`:
```
let moc = CoreDataStack.shared.mainContext
do {
  try moc.save()
} catch {
  print("error saving to coredata: \(error)")
}
```

## Delete Changes in CoreData
`NSManagedObjectContext.delete(_ object: NSManagedObject)` tells the MOC to delete the object from the persistent store, this change will take effect once `.save()` is called, writing the MOC changes to disk.
```
let moc = CoreDataStack.shared.mainContext
moc.delete(buildings.first) // delete the first building in our buildings array from coredata
try! moc.save()
```

## Migration
CoreData features the ability to update the format of the persistent data being stored, the process of converting the data's old format into the new format is called *migration*.
CoreData provides three forms of migration:
-   Lightweight migration
-   Migration using a mapping model
-   Fully custom migration

### Lightweight migration
Lightweight migration is entirely automatic. You create a new version of your data model alongside the old one, and Core Data is able to infer the differences between them and how to migrate data saved using the older model version to the newer one. However, lightweight migration only works if the changes to your model are relatively simple.

Changes supported:
-   Addition of new optional attributes
-   Addition of new non-optional attributes with a default value
-   Removal of an attribute
-   A non-optional attribute becoming optional
-   An optional attribute becoming non-optional, and defining a default value
-   Renaming an entity, attribute, or relationship
-   Addition of a new relationship or deletion an existing relationship.


### Migration using a mapping model
If the changes to your model are more complex than lightweight migration allows for, you can use a mapping model. A mapping model is a file that you create to tell Core Data how to convert data from an older model format to a newer one.

### Custom migration
When you do a fully custom migration, you write code that reads objects in your older data store and creates corresponding objects in the new store. CoreData's  API guides this process and encapsulates the migration code.

### Completing Lightweight migration
If using `NSPersistentContainer` for the CoreData stack, nothing has to be done to enable Lightweight migration. Simply make the supported changes to the .xcdatamodeld file.

if **not** using `NSPersistentContainer`, enable lightweight migration by setting the `shouldInferMappingModelAutomatically` and `shouldMigrateStoreAutomatically` properties to true on `NSPersistentStoreDescription` before passing it to `NSPersistentStoreCoordinator.addPersistentStore(with:, completionHandler:)`.

## Fetching
A Fetched Results Controller (instance of `NSFetchedResultsController`) is used to communicate changes in CoreData. Typically to a `UICollectionViewController` or `UITableViewController` in order to keep CoreData and the UI in sync.

`NSFetchResultsController` is given a MOC and FetchRequest, the FRC will monitor the MOC for changes to objects matching the criteria of the fetch request. When a change (CRD) happens to any of the objects, the FRC will inform it's delegate (typically the table/collectionview datasource).

## Sync Persistence
If we wanted to send our CoreData objects to a server, we cannot simply convert them into JSON using `Codeable`.

Representation objects are used to represent  a CoreData object, simply add a new computed property to the CoreData NSManagedObject class through an extension which converts the `Object` into an `ObjectRepresentation`.

## NSPredicate
`NSPredicate` can be used in conjuction with `NSFetchRequest` to narrow down criteria.

They are basically database queries:
```
let adultsPredicate = NSPredicate(format: "age >= %@", 18)
let youngAdultsPredicate = NSPredicate(format: "age >= %@ AND age < %@", 18, 50)
```

Using the predicate:
```
let fetchRequest: NSFetchRequest<Person> = Person.fetchRequest()
let adultsPredicate = NSPredicate(format: "age >= %@", 18)
fetchRequest.predicate = adultsPredicate

let managedObjectContext = container.viewContext
do {
  let adults = try managedObjectContext.fetch(fetchRequest)
  print(adults)
} catch {
  NSLog("Error fetching adults: \(error)")
}
```

 
## dependency injection
- **Dependency injection** is a technique in which an object receives other objects that it depends on. These other objects are called dependencies
- For example, when a preparing for a segue, the destination VC often receives an object before the segue is performed. This is called dependency injection because the destination VC is not tasked with creating the object.
- A benefit to dependency injection is that Unit Testing is made easier in that injecting mock object dependencies into the target.
 
## programmatic views
 
## testing
- **XCTestExpectation** is a function used to test async code.
-- Create **XCTestExpectation** with description.
-- Inside the async code, run `.fulfill()` on the `XCTestExpectation` object.
-- Wait for code to run, then check if expectation was fulfilled:
--  `wait(for: [expectation], timeout: 10.0)
