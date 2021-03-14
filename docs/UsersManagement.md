# Users Management


## Contents

<details>
<summary><strong>Details</strong></summary>

- [Users management](#users-management)
  - [Creating Users](#creating-users)
  - [List all Users](#list-all-users)
  - [Grouping User Accounts under Organizations](#grouping-user-accounts-under-organizations)
  - [Create an Organization](#create-an-organization)
  - [List all Organizations](#list-all-organizations)
  - [Assign users to organizations](#assign-users-to-organizations)
  - [List Users within an Organization](#list-users-within-an-organization)

</details>

## Users management

### Creating Users

In this section, we explain how to create the corresponding users, making use of the corresponding
[Identity Management API](https://keyrock.docs.apiary.io).

> **Note** - an eMail server must be configured to send out invites properly, otherwise the invitation may be deleted as
> spam. For testing purposes, it is easier to update the users table directly: `update user set enabled = 1;`

All the CRUD actions for Users require an `X-Auth-token` header from a previously logged in administrative user to be
able to read or modify other user accounts. The standard CRUD actions are assigned to the appropriate HTTP verbs (POST,
GET, PATCH and DELETE) under the `/v1/users` endpoint.

To create a new user, send a POST request to the `/v1/users` endpoint containing the `username`, `email` and `password`
along with the `X-Auth-Token` header from a previously logged in administrative user (see the previous section). Additional
users can be added by making repeated POST requests with the proper information following the previous table.
For example to create additional accounts for Bob, the Application Manager we should execute the following request

> **Note** You can take a look and execute the create-users script to automatically create all the users accounts.

#### :two: Request

```bash
echo '{
  "user": {
    "username": "Bob",
    "email": "bob-the-appmanager@test.com",
    "password": "test"
  }
}' | http  POST 'http://localhost:3005/v1/users' \
 X-Auth-Token:"$TOKEN"
```

#### :two: Response

The response contains details about the creation of this account:

```json
{
    "user": {
        "admin": false,
        "date_password": "2021-02-10T14:45:47.950Z",
        "eidas_id": null,
        "email": "bob-the-appmanager@test.com",
        "enabled": true,
        "gravatar": false,
        "id": "9943e0bd-1596-4d9d-a438-cdea1b7ec7bb",
        "image": "default",
        "salt": "9b5c3d5bbcf649d2",
        "starters_tour_ended": false,
        "username": "Bob"
    }
}
```

### List all Users

Obtaining a complete list of all users is a super-admin permission requiring the `X-Auth-token` - most users will only
be permitted to return users within their own organization. Listing users can be done by making a GET request to the
`/v1/users` endpoint

#### :three: Request

```bash
http GET 'http://localhost:3005/v1/users' \
 X-Auth-Token:"$TOKEN"
```

#### :three: Response

The response contains basic details of all accounts:

```bash
{
    "users": [
        {
            "date_password": "2021-02-10T15:16:51.000Z",
            "description": null,
            "email": "eve@example.com",
            "enabled": true,
            "gravatar": false,
            "id": "24ab4550-cb7c-45ce-97ea-870051181745",
            "scope": [],
            "username": "Eve",
            "website": null
        },
        {
            "date_password": "2021-02-10T15:16:52.000Z",
            "description": null,
            "email": "torsten-kuehl@xyz.foo",
            "enabled": true,
            "gravatar": false,
            "id": "2548f3a8-aa5c-4dac-90d5-2442d23cd744",
            "scope": [],
            "username": "Torsten",
            "website": null
        },
      
      etc...
      
    ]
}
```

## Grouping User Accounts under Organizations

For any identity management system of a reasonable size, it is useful to be able to assign roles to groups of users,
rather than setting them up individually. Since user administration is a time-consuming business, it is also necessary
to be able to delegate the responsibility of managing these group of users down to other accounts with a lower level of
access.

Consider our Personal Data management example, there could be a group of users (Application Managers) who can introduce
new Personal Data into the system as well as modify existing data introduced previously. Another group of users
(Application Users) who need to access the Personal Data information to produce some report based on them but cannot
modify them. Another group of users (Data Users) correspond to each of the persons that provide the Personal Data,
therefore they can access for reading and modifying their only own data. Finally, Another group of users (Others)
exist that are not related to this application and therefore they cannot access to the Personal Data for neither
reading nor writing. Rather than give access to each individual account, it would be easier to assign the rights to
an organization and then add users to these organizations.

Furthermore, Alice, the **Identity Management** administrator does not need to explicitly add additional user
accounts to each organization herself - she could delegate that right to an owner within each organization.
For example Bob the Project Manager would be made the owner of the _Application Managers_ organization and could
add and remove addition manager accounts to that organization whereas Charlie the Head of _Application Users_
could be handed an ownership role his organization and add additional application users to that organization.

Note that Bob does not have the rights to alter the membership list of the _Users_ organization and Charlie
does not have the rights to alter the membership list of the _Managers_ organization. Furthermore, neither Bob
nor Charlie would be able to alter the permissions of the application themselves, merely add and remove existing
user accounts to the organization they control.

By the execution of this tutorial, alice will be the person in charge of the creation of all organizations for
management purposes. Therefore, Alice will be automatically assigned to all of these groups.

### Create an Organization

The standard CRUD actions are assigned to the appropriate HTTP verbs (POST, GET, PATCH and DELETE) under
the `/v1/organizations` endpoint. To create a new organization, send a POST request to the `/v1/organizations`
endpoint containing the `name` and `description` along with the `X-Auth-token` header from a previously
logged-in user.

> **Note** You can take a look and execute the organization-mgmt script to automatically create all organizations
> and assign the users to each organization.

#### :four: Request

```bash
printf '{
  "organization": {
    "name": "Managers",
    "description": "This group is for the Project Managers of the Personal Data application with full control access"
  }
}'| http  POST http://localhost:3005/v1/organizations \
 Content-Type:'application/json' \
 X-Auth-Token:"$TOKEN"
```

The Organization is created, and the user who created it is automatically assigned as a user. The response returns
UUID to identify the new organization.

#### :four: Response

```json
{
    "organization": {
        "description": "This group is for the Project Managers of the Personal Data application with full control access",
        "id": "e3980d68-4f0e-4f7b-b1d5-d3bbc7125fb1",
        "image": "default",
        "name": "Managers"
    }
}
```

### List all Organizations

Obtaining a complete list of all organizations is a super-admin permission requiring the `X-Auth-token` - most users
will only be permitted to return users within their own organization. Listing users can be done by making a GET request
to the `/v1/organizations` endpoint.

#### :five: Request

```bash
http GET http://localhost:3005/v1/organizations \
 X-Auth-Token:"$TOKEN"
```

#### :five: Response

The response returns the details of the visible organizations.

```json
{
    "organizations": [
        {
            "Organization": {
                "description": "Personal Data owners who can read and modify only their own data",
                "id": "1d157e87-32e3-4812-bde2-c0d1e3967170",
                "image": "default",
                "name": "Data",
                "website": null
            },
            "role": "owner"
        },
        {
            "Organization": {
                "description": "Project Users of the Personal Data application with read control access",
                "id": "531f0f5c-a7c4-4826-96b2-31988caefc11",
                "image": "default",
                "name": "Users",
                "website": null
            },
            "role": "owner"
        },
        {
            "Organization": {
                "description": "Rest of IdM registered users not authorized to access the Personal Data Application",
                "id": "df58f9d2-443a-4375-abf7-00a88677e7b5",
                "image": "default",
                "name": "Others",
                "website": null
            },
            "role": "owner"
        },
        {
            "Organization": {
                "description": "Project Managers of the Personal Data application with full control access",
                "id": "e3980d68-4f0e-4f7b-b1d5-d3bbc7125fb1",
                "image": "default",
                "name": "Managers",
                "website": null
            },
            "role": "owner"
        }
    ]
}
```

### Assign users to organizations

Users within an Organization are assigned to one of types - `owner` or `member`. The members of an organization inherit
all the roles and permissions assigned to the organization itself. In addition, owners of an organization are able to
add and remove other members and owners.

To add a user as a member of an organization, an owner must make a PUT request as shown, including the
`<organization-id>` and `<user-id>` in the URL path and identifying themselves using an `X-Auth-Token` in the header.

#### :six: Request

```bash
http  PUT "http://localhost:3005/v1/organizations/$MANAGERS/users/$BOB/organization_roles/member" \
 Content-Type:'application/json' \
 X-Auth-Token:"$TOKEN"
```

We have to repeat this operation for all the users created previously.

> Note: $MANAGERS corresponds to the organization id of the _Managers_ organization and $BOB correponds to the user id
> of the Bob user. See the mgmt-users-organization script for more details

#### :six: Response

The response lists the user's current role within the organization (i.e. `member`)

```json
{
    "user_organization_assignments": {
        "role": "member",
        "organization_id": "18deea43-e12a-4018-a45a-664c3158780d",
        "user_id": "5e482345-2c48-410e-ae03-203d67a43cea"
    }
}
```

### List Users within an Organization

Listing users within an organization is an `owner` or super-admin permission requiring the `X-Auth-token` Listing
users can be done by making a GET request to the `/v1/organizations/{{organization-id}}/users` endpoint.

#### :seven: Request

```bash
http GET "http://localhost:3005/v1/organizations/$OTHERS/users" \
 X-Auth-Token:"$TOKEN"
```

#### :seven: Response

The response contains the users list.

```json
{
    "organization_users": [
        {
            "organization_id": "f47a6dfb-bf25-4117-be79-723123f11ec4",
            "role": "owner",
            "user_id": "admin"
        },
        {
            "organization_id": "f47a6dfb-bf25-4117-be79-723123f11ec4",
            "role": "member",
            "user_id": "ff7ff1d6-c34e-4784-a511-dfc86ea6c260"
        },
        {
            "organization_id": "f47a6dfb-bf25-4117-be79-723123f11ec4",
            "role": "member",
            "user_id": "4901190d-7233-4a9c-854a-45551b01d912"
        }
    ]
}
```
