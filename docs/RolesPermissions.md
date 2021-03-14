# Roles and Permissions

## Contents

<details>
<summary><strong>Details</strong></summary>

- [Managing Roles and Permissions](#managing-roles-and-permissions)
  - [Create an Application](#create-an-application)
  - [Create a Permission](#create-a-permission)
  - [List Permissions](#list-permissions)
  - [Create a Role](#create-a-role)
  - [Assigning Permissions to each Role](#assigning-permissions-to-each-role)
  - [List Permissions of a Role](#list-permissions-of-a-role)

</details>


## Managing Roles and Permissions

The next step consists in the creation of the proper application, and how to assign roles and permissions to them.
It takes the users and organizations created in the previous sections and ensures that only legitimate users will
have access to resources.

Authorization is the function of specifying access rights/privileges to resources related to information security.
More formally, "to authorize" is to define an access policy. With identity management controlled via the
FIWARE Keyrock Generic Enabler, User access is granted based on permissions assigned to a role.

Every application secured by the Keyrock generic enabler can define a set of permissions - i.e. a set of things
that can be done within the application. For example within the application, the ability to read and modify new
Personal Data . Similarly, the ability to read and modify only your own Personal Data could be secured a proper
defined permission.

These permissions are grouped together in a series of roles - for example read and modify Personal Data could
both be assigned to the Managers role, meaning that Users who are subsequently given that role would gain both
permissions over Personal Data stored in the application. Permissions can overlap and be assigned to multiple
roles - maybe read Personal Data is also assigned to the Users role.

In turn users or organizations will be assigned to one of more roles - each user will gain the sum of all the
permissions for each role they have. For example if Alice is assigned to both Managers and Users roles, she
will gain all permissions for reading and modifying Personal Data.

The concept of a role is unknown to a user - they only know the list of permissions they have been granted,
not how the permissions are split up within the application.

In summary, permissions are all the possible actions that can be done to resources within an application, whereas
roles are groups of actions which can be done by a type of user of that application. The relationship between the
objects can be seen below.

![Roles and Permissions of an Application](https://fiware.github.io/tutorials.Roles-Permissions/img/entities.png)

### Create an Application

Any FIWARE application can be broken down into a collection of microservices. These microservices connect together
to read and alter the state of the real world. Security can be added to these services by restricting actions on
these resources down to users how have appropriate permissions. It is therefore necessary to define an application
to offer a set of permissible actions and to hold a list of permitted users (or groups of users i.e. an Organization).
Therefore, applications are therefore a conceptual bucket holding who can do what on which resource.

To create a new application via the REST API, send a POST request to the `/v1/application` endpoint containing
details of the application such as `name` and `description`, along with OAuth information fields such as the
`url` of the webservice to be protected, and `redirect_uri` (where a user will be challenged for their credentials).
The `grant_types` are chosen from the available list of OAuth2 grant flows. The headers include the `X-Auth-token`
from a previously logged-in user will automatically be granted a provider role over the application.

#### :eight: Request

In the example below, Alice (who holds `X-Auth-token=aaaaaaaa-aaaa-aaaa-aaaa-aaaaaaaaaaaa`) is creating a new
application which accepts three different grant types

```bash
printf '{
  "application": {
    "name": "Personal Data Mgmt. Application",
    "description": "FIWARE Application protected by OAuth2 for managing Personal Data",
    "redirect_uri": "http://localhost:1027/login",
    "url": "http://localhost:1027",
    "grant_type": [
      "authorization_code",
      "implicit",
      "password"
    ]
  }
}'| http  POST http://localhost:3005/v1/applications \
 Content-Type:'application/json' \
 X-Auth-Token:"$TOKEN"
```

#### :eight: Response

The response includes a Client ID and Secret which can be used to secure the application.

```json
{
    "application": {
        "description": "FIWARE Application protected by OAuth2 for managing Personal Data",
        "grant_type": "password,authorization_code,implicit",
        "id": "3fc4e897-a9b5-4b2e-bcce-98849c628972",
        "image": "default",
        "jwt_secret": null,
        "name": "Personal Data Mgmt. Application",
        "redirect_uri": "http://localhost:1027/login",
        "response_type": "code,token",
        "scope": null,
        "secret": "a3e10297-7a67-4f41-a5a8-e065332f2bbc",
        "token_types": "bearer",
        "url": "http://localhost:1027"
    }
}
```

Copy the Application Client ID to be used for all other application requests - in the case above the ID is
`3fc4e897-a9b5-4b2e-bcce-98849c628972` (export APP=3fc4e897-a9b5-4b2e-bcce-98849c628972).

### Create a Permission

An application permission is an allowable action on a resource within that application. Each resource is defined
by a URL (e.g. `/entities`), and the action is any HTTP verb (e.g. GET). The combination will be used to ensure
only permitted users are able to access the `/entities` resource.

It should be emphasized that permissions are always found bound to an application - abstract permissions do not
exist on their own. The standard permission CRUD actions are assigned to the appropriate HTTP verbs (POST, GET,
PATCH and DELETE) under the `/v1/applications/{{application-id}}/permissions` endpoint. As you can see the
`<application-id>` itself is integral to the URL.

Permissions are usually defined once and set-up when the application is created. If the design of your use-case
means that you find you need to alter the permissions regularly, then the definition has probably been defined
incorrectly or in the wrong layer - complex access control rules should be pushed down into the XACML definitions
or moved into the business logic of the application - they should not be dealt with within **Keyrock**.

To create a new permission via the REST API, send a POST request to the `/applications/{{application-id}}/permissions`
endpoint containing the `action`and `resource` along with the `X-Auth-Token` header from a previously logged-in
user (Alice).

#### :nine: Request

```bash
printf '{
  "permission": {
    "name": "Access to a Personal Data entity",
    "action": "GET",
    "resource": "/entities/*",
    "is_regex": true
  }
}' | http  POST "http://localhost:3005/v1/applications/$APP/permissions" \
 Content-Type:'application/json' \
 X-Auth-Token:"$TOKEN"
```

#### :nine: Response

The response returns the details of the newly created permission.

```json
{
    "permission": {
        "action": "GET",
        "id": "6ec726dc-fcad-447b-8222-7b3035de805b",
        "is_internal": false,
        "is_regex": true,
        "name": "Access to a Personal Data entity",
        "oauth_client_id": "3fc4e897-a9b5-4b2e-bcce-98849c628972",
        "resource": "/entities/*"
    }
}
```

We need to repeat this procedure for the rest of resources from which we want to control the access. In our example
we wanted to control the access to OrionLD regarding the creation of entities, creation of several entities. Take a
look in the following table to see the different permissions to be created:

| Perm. | Verb  | Resource                 | Description                                        | Organizations   |
| ----- | ----- | ------------------------ | -------------------------------------------------- | --------------- |
| #1    | GET   | /entities/*              | Get information of an entity (all entities)        | MANAGERS, USERS |
| #2    | GET   | /entities/{{entityID}}   | Get information of an entity (one entity)          | DATA            |
| #3    | POST  | /entityOperations/upsert | Add some entities                                  | MANAGERS        |
| #4    | PATCH | /entities/*/attrs/*      | Update data associated to an entity (all entities) | MANAGERS        |
| #5    | PATCH | /entities/{{ID}}/attrs/* | Update data associated to an entity (one entity)   | DATA            |

We have to mention that the permission #1 include the permission #2, and the permission #2 in generated after
we have the upload the Personal Data associated to a person (e.g. Ole's Personal Data has the entityID
`urn:ngsi-ld:Person:person001`).

> Note: We should manage all the permissions related to the OrionLD API but for this document we will focus on
> the previous resources.
>
> Note: The script `mgmt-users-organizations` will create all the corresponding permissions for this example application.

### List Permissions

Listing the permissions with an application can be done by making a GET request to the
`/v1/applications/{{application-id}}/permissions/` endpoint

#### :one::zero: Request

```bash
http GET "http://localhost:3005/v1/applications/$APP/permissions" \
 X-Auth-Token:"$TOKEN"
```

#### :one::zero: Response

The complete list of permissions includes any custom permission previously created plus all the standard permissions
which are available by default

```bash
{
  "permissions": [
    {
      "action": "PATCH",
      "description": null,
      "id": "d7bd7555-e769-4c71-9143-04bdc327cbe0",
      "name": "Permission to Update the Personal Data information associated to an entity (urn:ngsi-ld:Person:person001)",
      "resource": "/entities/urn:ngsi-ld:Person:person001",
      "xml": null
    },
    {
      "action": "GET",
      "description": null,
      "id": "d2cee587-46bf-4233-a455-8f1abb7f7122",
      "name": "Permission to get Personal Data information of an entity (urn:ngsi-ld:Person:person004)",
      "resource": "/entities/urn:ngsi-ld:Person:person004",
      "xml": null
    },
    {
      "action": "PATCH",
      "description": null,
      "id": "ab95d325-e8fe-43bb-b34e-fe5837b14e28",
      "name": "Permission to update the information associated to an entity (all entities)",
      "resource": "/entities/*",
      "xml": null
    },
    
  etc...
  
  ]
}
```

### Create a Role

A permission is an allowable action on a resource, as noted above. A role consists of a group of permissions, in other
words a series of permitted actions over a group of resources. Roles have a description with a broad scope
so that they can be assigned to a wide range of users or organizations for example a _Reader_ role could be able to
access but not update a series of devices.

There are two predefined roles with **Keyrock** :

- a _Purchaser_ who can
  - Get and assign all public application roles
- a _Provider_ who can:
  - Get and assign public owned roles
  - Get and assign all public application roles
  - Manage authorizations
  - Manage roles
  - Manage the application
  - Get and assign all internal application roles

Using our Personal Data Example, Alice the admin would be assigned the _Provider_ role, she could then create any
additional application-specific roles needed (such as _Manager_, _Users_, _Data_ or _Others_).

Roles are always directly bound to an application - abstract roles do not exist on their own. The standard
CRUD actions are assigned to the appropriate HTTP verbs (POST, GET, PATCH and DELETE) under the
`/v1/applications/{{application-id}}/roles` endpoint.

To create a new role via the REST API, send a POST request to the `/applications/{{application-id}}/roles` endpoint
containing the `name` of the new role, with the `X-Auth-token` header from a previously logged-in user.

#### :eleven: Request

```bash
printf '{
  "role": {
    "name": "Manager"
  }
}'| http  POST "http://localhost:3005/v1/applications/$APP/roles" \
 X-Auth-Token:"$TOKEN"
```

#### :eleven: Response

The details of the created role are returned

```json
{
    "role": {
        "id": "e5aa8b37-701c-4baf-96d4-9021396445dd",
        "is_internal": false,
        "name": "Manager",
        "oauth_client_id": "e295e248-096b-4222-8969-ea3c4e92d409"
    }
}
```

We need to repeat the process for _Users_, _Data_ or _Others_, changing the value `name` in the json payload.

### Assigning Permissions to each Role

Having created a set of application permissions, and a series of application roles, the next step is to assign the
relevant permissions to each role - in other words defining _Who can do What_. To add a permission using the REST
API makes a PUT request as shown, including the `<application-id>`, `<role-id>` and `<permission-id>` in the URL
path and identifying themselves using an `X-Auth-Token` in the header.

The following table summarize the relationship of each *Role* with the different *Permissions*

| *Role*    | *Permissions*                                                                 |
| --------- | ----------------------------------------------------------------------------- |
| Manager   | #1(GET:/entities/*), #3(POST:/entityOperations/upsert), #4(PATCH:/entities/*) |
| Users     | #1(GET:/entities/*)                                                           |
| Data(n)   | #2(GET:/entities/{{entityID}}), #5(PATCH:/entities/{{entityID}})              |
| Others    |                                                                               |

Due to the roles are associated to the application, the Role _Others_ does not have any permission assigned in the
application, therefore the users under the Role Others should be rejected.

#### :twelve: Request

```bash
http PUT "http://localhost:3005/v1/applications/$APP/roles/$MANAGER_ROLE/permissions/$PERMID' \
 X-Auth-Token:"$TOKEN"
```

#### :twelve: Response

The response returns the permissions for the role

```json
{
    "role_permission_assignments": {
        "permission_id": "c21983d5-58f9-4bcc-b2b0-f21819080ad0",
        "role_id": "64535f4d-04b6-4688-a9bb-81b8df7c4e2c"
    }
}
```

> Note: take a look into the applications-roles script to see how we associated the
> different permissions with the corresponding Roles.

### List Permissions of a Role

A full list of all permissions assigned to an application role can be retrieved by making a GET request to the
`/v1/applications/{{application-id}}/roles/{{role-id}}/permissions` endpoint.

#### :one::three: Request

```bash
http GET "http://localhost:3005/v1/applications/$APP/roles/$MANAGERS/permissions" \
  X-Auth-Token:"$TOKEN"
```

#### :one::three: Response

```json
{
    "role_permission_assignments": [
        {
            "action": "GET",
            "description": null,
            "id": "c8f127b2-fabc-420c-93aa-79ef104592d4",
            "is_internal": false,
            "name": "Permission to get Personal Data information of an entity (all entities)",
            "resource": "/entities/*",
            "xml": null
        },
        {
            "action": "POST",
            "description": null,
            "id": "e2f8bd53-0ce5-4179-abaf-66abebb0e582",
            "is_internal": false,
            "name": "Permission to add some entities",
            "resource": "/entityOperations/upsert",
            "xml": null
        },
        {
            "action": "PATCH",
            "description": null,
            "id": "c6e50b77-d34b-40a9-9929-03b1b7ae0886",
            "is_internal": false,
            "name": "Permission to update the information associated to an entity (all entities)",
            "resource": "/entities/*",
            "xml": null
        }
    ]
}
```

In case of the Roles associated to the Person001, the request would be:

#### :one::four: Request

```bash
http GET "http://localhost:3005/v1/applications/$APP/roles/$MANAGERS/permissions" \
  X-Auth-Token:"$TOKEN"
```

#### :one::four: Response

```json
{
    "role_permission_assignments": [
        {
            "action": "GET",
            "description": null,
            "id": "226b3cd7-37f7-4378-99ae-f19bb50469ca",
            "is_internal": false,
            "name": "Permission to get Personal Data information of an entity (urn:ngsi-ld:Person:person001)",
            "resource": "/entities/urn:ngsi-ld:Person:person001",
            "xml": null
        },
        {
            "action": "PATCH",
            "description": null,
            "id": "14738fe7-78b6-4b71-884e-5fda37bafe19",
            "is_internal": false,
            "name": "Permission to Update the Personal Data information associated to an entity (urn:ngsi-ld:Person:person001)",
            "resource": "/entities/urn:ngsi-ld:Person:person001",
            "xml": null
        }
    ]
}
```
