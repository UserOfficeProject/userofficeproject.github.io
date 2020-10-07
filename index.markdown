---
# Feel free to add content and custom Front Matter to this file.
# To modify the layout, see https://jekyllrb.com/docs/themes/#overriding-theme-defaults

layout: default

---


# Step-by-step guide for creating new functionality

![Image edit metadata](img/uo_overview.png)

Use this document to understand how different parts of UserOffice code base is connected together. 

For this example we will implement an imaginary feature, we will
imagine our product owner want's to give user officer (i.e. an
administrative role) possibility to **lock out a user**.

## Backend

### Database changes

Start by adding a new column "is_locked" to the database by creating a new file under /db\_patches/00XX\_AlterUserAddIsLocked.sql (Note the naming convention) . This patch will be automatically executed on container startup and therefore the database will be modified. 

```  
ALTER TABLE users ADD COLUMN is_locked BOOL DEFAULT FALSE; 
```
   
### Code changes

Now that we have DB column, let's write function in our datasource layer that will update the database. 

#### Create new database interface function 

Add a new interface function under /src/datasources/AdminMutations.ts . The Interface function will accept one parameter user\_id, and return a value object of type BasicUserDetails containing the user who was locked

```
lockUser(user_id: number): Promise<BasicUserDetails>;
```

#### Implement the interface in PostGreSQL

Add the function to handle the database request in the file by implementing the interface in /src/datasources/postgres/AdminDataSource.ts

```
lockUser(user_id: number): Promise<BasicUserDetails> {
// Knex is a powerful library that in many ways simplifies writing SQL in JS. You can check out http://knexjs.org/ if you have questions regarding syntax or how it works. They host great documentation pages
    return database // database is imported in imports on top and it references ready to use instance of Knex
      .update({
        is_locked: true
      })
      .from("users")
      .where("user_id", user_id)
      .returning("*")
      .then((updatedRows: Array<UserRecord>) => {
        if (updatedRows.length !== 1) {
          throw new Error(`Could not lock user:${user_id}`); // throwing error according to design https://confluence.esss.lu.se/display/SWAP/UO+Diagram
        }
        return createBasicUserObject(updatedRows[0]); // createBasicUserObject is a utility function to convert DB records into value objects used in UserOffice, if there is a new table you would need to also add a new utility function that performs the conversion
      });
  }
```

#### Implement the interface in mockups

Also implement the interface in mockup db in /src/datasources/mockups/AdminDataSource.ts. Mockup DB layer is used in unit tests

```
async lockUser(
   user_id: number
 ): Promise<BasicUserDetails> {
   return new BasicUserDetails(user_id, 'Carl', 'Young', 'ESS', 'Pharmacist')
 }
 ```

#### Add function to logic layer for handling requests
Now that we have DB layer ready let's implement mutation. Mutations contains DB error handling,authorization and other important business logic.\ Navigate to /src/mutations/AdminMutations.ts and add new function

```
async lockUser(
   agent: User | null, // this is the reference to currently logged in user, this will be passed in from resolver (in next step)
   userId: number
 ): Promise<BasicUserDetails | Rejection> { // Rejection is part of UserOffice value objects. This value object contains the reason
   if (!(await this.userAuth.isUserOfficer(agent))) { // as per PO request we will only allow userOfficer to call this method
     return rejection("NOT_AUTHORIZED");
   }
   return this.dataSource
     .lockUser(userId) // call to our newly implemented function on DB layer
     .then(user => {
       return user;
     })
     .catch(error => {
       logger.logException("Could not lock user", error, {
         agent,
         userId
       }); // this part is crucial for debugging. Make error message descriptive that can be aggregated on and add specifics in context object.
       return rejection("INTERNAL_ERROR");
     });
 }
```

### Create a resolver that exposes mutation functionality to GraphQL
Create new file in /src/resolvers/mutations/LockUserMutation.ts . Note that Resolvers rely heavily on decorators ([https://www.typescriptlang.org/docs/handbook/decorators.html](https://www.typescriptlang.org/docs/handbook/decorators.html))

```

@Resolver() // classify class as resolver by using @Resolver decorator
export class LockUserMutation { // the name of class does not matter, but by convention is <functionName>Mutation
  @Mutation(() => BasicUserDetailsResponseWrap) // Classify method as Mutation by using decoration @Mutation, and signifying response type. Response in this case is BasicUserDetailsResponseWrap which is commonly shared. If you introduce new ValueObject you want to also add new responseWrap. Please check /src/resolvers/wrappers/CommonWrappers.ts for more details.
 
// All ResponseWrapers contains response field and error field. Error field is string always called "error", response field name and type varies.
// i.e. for BasicUserDetails, BasicUserDetailsResponseWrap contains user: BasicUserDetails and error: string
// if mutation ends in failure, response field will be null and error will be populated with error
// if mutation succeeds response field will be the result but error will be null
 
  lockUser( // the name of the mutation that will show up in GraphQL API
    @Arg("userId", () => Int) userId: number // mutation argument. For mutations with more arguments you might choose to create separate Arguments class to group them
    @Ctx() context: ResolverContext // Mark argument with @Ctx decorator and it will contain context populated by TypeGraphql framework. Context contains references to mutation as well as currently logged in user details object
  ) {
    return wrapResponse( // wrapResponse will evaluate your mutation call and format response
      context.mutations.admin.lockUser(context.user, userId), //tip: typescript will check the compatibility of mutation response and wrapper compile time
      BasicUserDetailsResponseWrap // ResponseWrapper to use
    );
  }
}

```

And that is it. Test if things are working by navigating
to <http://localhost:4000/graphql>, there you should see a new endpoint
mutations. Tip: check backend output terminal for errors if any\
![](View%20Source_files/image2020-2-14_16-54-51.png "Scientific Web Applications > How to implement new functionality in User Office > image2020-2-14_16-54-51.png")

-----------------------------

## Frontend

Now that we managed to get backend working, calling it from frontend is
surprisingly easy.

### Write graphql mutation:
Create new file lockUserMutation and write the mutation:
```
mutation lockUser($userId: Int!) { // name your query
  lockUser(userId: $userId) { // call graphql endpoint
    user {
      firstname
    }
    error
  }
}
```
### Regenerate SDK
```
npm run generate:local
```
This will regenerate sdk (src/generated/sdk.ts) file on client side containing new signatures of GraphQL layer, now including our new lockUser method. This will also typecheck your mutation and make sure it is existing and valid.

### Call the method from code
Create a hook and invoke a method

```
import { useDataApi } from "../hooks/useDataApi";
 
// ...
 
const api = useDataApi();
 
//...
 
api().lockUser({userId:1})
```