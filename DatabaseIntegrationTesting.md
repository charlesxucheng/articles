
Testing with Database
==================

Unit Testing and Integration Testing
-----------
With regard to a typical web application layered architecture:
> Controller->Service->Repository->Database

When doing unit testing each layer, we mock the layer it depends on, except for the lowest layer (Database). 

We do not need to perform unit testing with database layer if there is no logic inside (no stored procedures, triggers, logic in views, etc). 

For repository layer, unit testing should be done for logic that is non-trivial. However, if a repository method is performing some simple database operations, usually it does not require unit testing.

However, we still need to perform integration testing between the repository layer and database layer. Otherwise there is no assurance that the repository code will work correctly with the database. 

Such integration testing can be run separately from the running of unit testing cases so that unit testing execution will not be slowed down by database accesses. 

Integration testing can happen less frequently than unit testing. Typically integration testing is done:

 - after the completion of a logical unit of work (a story, a log, etc)
 - during integration testing phase of an iteration

We can then use a database that is of the same type as production database for integration testing. Compared to using an in-memory database, it has the following benefits:

- captures database specific issues early and provides better assurance.
- eliminates the need to write database scripts that are compatible to different types of products (e.g. Oracle vs Hsqldb), which introduces additional complexity and unforeseeable limitations, even if using tools such as Liquibase.
- makes it possible to use database vendor specific features that are not available in the in-memory database.

The integration testing should follow good test practices, such as:

- Independent/orthogonal
- Repeatable
- Do not assume data is always there. Manage your test data via scripts and version control system.
- If unit testing is sufficient, do not use integration testing.

Separating Unit Tests and Integration Tests using Gradle
-------------------------------------------------------------------

To be added later


Test Database Setup
------------------------
Since we need a controlled test environment, we need a separate database/schema from our development database.

We can define a task to migrate database changes to the test database (using Flyway or other similar database migration tools). The integration testing task can depend on this task.

Injecting Test Database
----------------------------

Using Spring profiles

Schema Validation
----------------------
hibernate.hbm2ddl.auto = validate


 



http://www.toptal.com/qa/how-to-write-testable-code-and-why-it-matters

https://lostechies.com/jimmybogard/2013/06/18/strategies-for-isolating-the-database-in-tests/
http://stackoverflow.com/questions/22209181/gradle-how-to-exclude-some-tests#














> Written with [StackEdit](https://stackedit.io/).