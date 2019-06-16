# Police Man
This is a standalone IAM (Identity and Access Management) microservice, built to easily integrate into your existing backend microservice architecture/solution so you don't have to think about building your own IAM solution, which is tedious and difficult.  
This allows you to be your own standalone identity provider just like using google/facebook/github as identity providers for social logins, except that now you are no longer dependant on other platforms and infrastructure for your user's identity.  
This Backend microservice is API based, designed only for providing Identity and Access management of user accounts over its API.  
This project is open source and available free of charge to use, but hopefully will be offering a freemium service after first stable major release is out!  

There are 2 parts to this story, Identity provisioning and Authorization/Access-management. But before we talk more about it, right below is the definition of the different stakeholders that will be interacting with this service.


## Service stakeholder definitions
- Developer / Dev:
    - A "developer" is defined as the person who is creating the application
    - A developer is also referenced as the person who can build upon the API for their app
- User / Client:
    - A "user" is defined as a user of the developer's application
    - The app that users use, is defined as the "Client"
    - Clients can be Single page applications in browsers or Mobile apps, and they are able to communicate with the service via APIs
- Account Admin:
    - An "Account Admin" is defined as someone who has access to modify settings in the IAM but is not neccesarily a developer, although a developer is also an account admin.
- Organization
    - "Organization" is defined as either a company that collectively publishes application(s), or a group of people who have control over a set of applications.
    - Admin accounts can be grouped together by their organizations.

--------------------------------------------------

## Identity management
The sole purpose of Identity Managment is to:
- Store and manage your users' credentials tied to their User UUIDs.
- Act as the single source of truth for the user's credential and as the only valid identity token provider.
- Handle all user logins/authentications and token provisioning.
    - Return a signed token pair in exchange for valid user credentials.
    - Refresh token pair given a valid refresh token.
- Account management
    - Delete user accounts when requested
    - Update user accounts. Like update of passwords and/or user roles


## For Access management
The sole purpose of Access Management is to:
- Act as the single source of truth for the user's roles.
- Handles all of the user's Access rights.
    - CRUD operations on the User's access rights.
- Handle user object's state property
    - Maintain the state of either active/deactivated/deleted
    - Allow API caller to do CRUD operations on the state
- Defines every single role
    - What the role is allowed to do
    - Resource Access rights of each role

The configurations for access management can be viewed and modified through a web-portal (Admin Portal coming soon) for admin accounts.

--------------------------------------------------

## Database and Data schema
In this section we will talk about the database and storage choices for a service like this, and also discuss some F.A.Qs on this topic. Main focus of this is to explain the preferred choice of data storage of user creds and more.

- There are 2 datastores owned by this service. 1 for Identity management and 1 for access rights management.  
    - Note that I said Datastore and not Database. Because it means that there are 2 types of data that are stored but they can be stored on the same database. You can think of it as meaning 2 tables in a relational database.
- NoSQL is suggested as the choice of database, thanks to the abundance of NoSQL based DBaaS that offer great scalibility and ease of use for developers.
- However if your business model requires you to use SQL/Relational-Databases for whatever reasons, it's fine too. In fact it is really simple to replace the NoSQL DB connector with a SQL DB connector. You can refer to the current DB connector and simply rewrite it in SQL with any of your preferred SQL DB connector. Watch this repo, as I will be writing out an example SQL connector for reference in the future.

### Access management, role and action definitions Datastore/Document
- Access management owns a datastore, to store the defitintions of every single type of user-role/user-group.  
- Data is modified by API calls made from the web-portal

### User Datastore/Document
- Identity management owns the user datastore, to store the user Objects.  
- The database for this service only deals with the Users'
    - unique Identifier or a "userID" (username or their "email")
        - Email, however is not suggested because if DB is ever hacked, all the emails are exposed and can be used for things like spam which can damage the reputation of your brand. Even a simple hash of the emails is more preferable.
    - corresponding credential hash and salt
        - Assuming if BCrypt is used, the salt is together with the password, thus there will be no need for a salt attribute
    - Access rights (a.k.a roles in your application)

#### User datastore (Key Value pair document based storage form)
Below is an example on how the User store will look like:
```js
{
    ${user_UID} : {
        userID:
        hash:
   }
}
```
Legend:
- user_UID
    - This must be an `unique identifier` for every single user that MUST be unique in the whole datastore. Generally speaking each organization gets 1 set of datastore, but they can choose to create more.
    - "user_UID" is used as the key for user objects
    - It is possible to share users across different applications (This can be configured)

Every single application is a role. Example, users can use both "my_app" and "his_app", you just give user's a role depending on which app they use and the permission within that app. So if both my_app and his_app are used, then the user will have 2 groups of roles.

To keep the database clean and the schema relatively flat. There should not be too much nesting. Instead of nesting application specific roles as an array that is binded to the application as a role and storing the full name of the role every single time, we should store an enumerated item representing all the roles.
So like create an enum of possible roles, and then store the enumerate value in the user DB so that the user DB is really flat and simple. And this also have added security benefits too, if the user DB is hacked, you cant't tell which user has what role, because all of the enumerated values are stored in the roles database instead. And since these are 2 seperate databases, there is an added layer of security.

- userID
- hash
- roles  (an array of enumerated values representing the roles this user have)
- permissions  (an array of enumerated values representing the permissions this user have)


### Database/Datastore notes
- To keep the persisted state clean and lean. Nothing else is included, not even other user details.
    - Because this service is purely for identity matching and token provisioning, application specific user details should be handled seperately by a "user service" that does application specific user details management in your application's microservice architecture. Thus the police man service WILL NOT store any application specific user state for you.
    - It is important to note that this Auth service is, and has to be independant from the rest of your services.
- Client services or client code calling API of this service must make sure that they have a unique and valid user ID.  
    - If it is a new user, then another service like a user service should deal with account creation first, this service only deals with auth and credentials. You need to have a UID to use the identity management.
    - If your app accepts anonymous users you may encounter a problem on how to uniquely identify this user.
        - In this case it is most likely that you have an app, and that app allows users to use and store data without needing to create an account. To deal with this situation, you would still need to be able to uniquely identify this user in order to provide Access management.
        - To do so, you should either generate 2 true random strings to use as their UID and password, or take a System value like IMEI that is unique to the mobile app user. You should then create an account for that user using that unique identifier and store that UID on the user's device too. So everytime you want to get a token for this user, send the UID & password to the service for a token. Thus achieving a Identity for the anonymous user.
        - Once the user decides to create an account, you can use the previously used Credentials to update the service with new credentials that are created by the user or your user service.
- What about new users?? How do they create an account?
    - Since we assume that all communication with policeman about user accounts have a Unique ID already, the datastore only deals with userID and the corresponding credentials. As mentioned in the previous point, creation of users needs to be dealt by your user service. You only send the user details over to police-man to create an identity for this user that you can trust.

--------------------------------------------------

## Design notes and considerations
### Why email isn't used as the username
Because of privacy reasons, every email can have a username attached to it and that username is changeable, but not the email.  
User can only log in with their username. This prevents people from logging in with email addresses if other sites are hacked.  
Email can be used as the "username" or userID because this app is designed for use with the end user.
So assuming the db is hacked, you cant really hide the email...  

### Info required for sign up:
- username (unique)
- email (unique)
- password (min. strength required)

### Signup flow
After signup:
- A first login token link will be sent to the user's Email for user to visit
- Upon using the login token, user's account will be activated
- User will be redirected to the login page with a banner showing activation successful.

User account state:
- Active (Actively used)
- dormant (Sometime since last login / use)
- De-activated (Deleted data, but email still black listed for now, but username now free for others to use)

--------------------------------------------------

## License and Contributing
This project is developed under the "BSD 3-Clause License"  
Feel free to use this project as you see fit and do contribute your changes too!  
If you have any questions feel free to contact me via [email](mailto:jaimeloeuf@gmail.com) as I know that although this is a lengthy README, it might still lack some details and can be confusing too.