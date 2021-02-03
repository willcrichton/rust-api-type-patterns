# Access control

A common pattern in system design is that X can only happen if Y is true.
* A user can only access their account settings if logged in.
* A piece of data can only be accessed if its mutex is locked.
* A file can only be read if a user has read permissions.

These properties are all instances of **access control**, where the ability to execute an action is predicated on access privileges.

In terms of our representational principle, the two conceptual elements are the **action** (eg opening a webpage, reading data) and the **access** (logged in, locked a mutex). We will look at how to design APIs to ensure consistency between the action and the access.
