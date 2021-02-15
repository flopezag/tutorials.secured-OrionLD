[![FIWARE Banner](https://fiware.github.io/tutorials.Working-with-Linked-Data/img/fiware.png)](https://www.fiware.org/developers)
[![NGSI LD](https://img.shields.io/badge/NGSI-LD-d6604d.svg)](https://www.etsi.org/deliver/etsi_gs/CIM/001_099/009/01.03.01_60/gs_cim009v010301p.pdf)

[![FIWARE Security](https://nexus.lab.fiware.org/repository/raw/public/badges/chapters/security.svg)](https://github.com/FIWARE/catalogue/blob/master/security/README.md)
[![License: MIT](https://img.shields.io/github/license/fiware/tutorials.PEP-Proxy.svg)](https://opensource.org/licenses/MIT)
[![Support badge](https://img.shields.io/badge/tag-fiware-orange.svg?logo=stackoverflow)](https://stackoverflow.com/questions/tagged/fiware)
[![JSON LD](https://img.shields.io/badge/JSON--LD-1.1-f06f38.svg)](https://w3c.github.io/json-ld-syntax/) <br/>
[![Documentation](https://img.shields.io/readthedocs/fiware-tutorials.svg)](https://fiware-tutorials.rtfd.io)


This tutorial uses the FIWARE [Wilma](https://fiware-pep-proxy.rtfd.io/) PEP Proxy combined with **Keyrock** to secure
access to Orion-LD endpoints exposed by FIWARE generic enablers. Users (or other actors) must log-in and use a token 
to gain access to services. The application code created in the
[previous tutorial](https://github.com/FIWARE/tutorials.Securing-Access) is expanded to authenticate users throughout a
distributed system. The design of FIWARE Wilma - a PEP Proxy is discussed, and the parts of the Keyrock GUI and REST API
relevant to authenticating other services are described in detail.

[http](https://httpie.io) commands are used throughout to access the **Keyrock** and **Wilma** REST APIs -
[Postman documentation](https://fiware.github.io/tutorials.PEP-Proxy/) for these calls is also available.

[![Run in Postman](https://run.pstmn.io/button.svg)](https://app.getpostman.com/run-collection/6b143a6b3ad8bcba69cf)

## Contents

<details>
<summary><strong>Details</strong></summary>

-   [Securing Microservices with a PEP Proxy](#securing-microservices-with-a-pep-proxy)
    -   [Standard Concepts of Identity Management](#standard-concepts-of-identity-management)
    -   [:arrow_forward: Video : Introduction to Wilma PEP Proxy](#arrow_forward-video--introduction-to-wilma-pep-proxy)
-   [Prerequisites](#prerequisites)
    -   [Docker](#docker)
    -   [Cygwin](#cygwin)
-   [Architecture](#architecture)
-   [Start Up](#start-up)
    -   [Dramatis Personae](#dramatis-personae)
        -   [Introduction](...)
        -   [Administrating Users](...)
        -   [Creating Roles and Permissions](...)
    -   [Logging In to Keyrock using the REST API](#logging-in-to-keyrock-using-the-rest-api)
        -   [Create Token with Password](#create-token-with-password)
        -   [Get Token Info](#get-token-info)
-   [Securing the Orion Context Broker](#securing-the-orion-context-broker)
    -   [Securing Orion - PEP Proxy Configuration](#securing-orion---pep-proxy-configuration)
    -   [Securing Orion - Start up](#securing-orion---start-up)
        -   [:arrow_forward: Video : Securing A REST API](#arrow_forward-video--securing-a-rest-api)
    -   [User Logs In to the Application using the REST API](#user-logs-in-to-the-application-using-the-rest-api)
        -   [PEP Proxy - No Access to Orion-LD without an Access Token](#pep-proxy---no-access-to-orion-without-an-access-token)
        -   [Keyrock - User Obtains an Access Token](#keyrock---user-obtains-an-access-token)
        -   [PEP Proxy - Accessing Orion-LD with an Authorization](pep-proxy---accessing-orion-awith-an-authorization)
-   [Integration with eIDAS](...)

</details>

# Securing Microservices with a PEP Proxy

> "Oh, it's quite simple. If you are a friend, you speak the password, and the doors will open."
>
> — Gandalf (The Fellowship of the Ring by J.R.R Tolkien)

The [previous tutorial](https://github.com/FIWARE/tutorials.Securing-Access) demonstrated that it is possible to Permit
or Deny access to resources based on an authenticated user identifying themselves within an application. It was simply a
matter of the code following a different line of execution if the `access_token` was not found (Level 1 -
_Authentication Access_), or confirming that a given `access_token` had appropriate rights (Level 2 - _Basic
Authorization_). The same method of securing access can be applied by placing a Policy Enforcement Point (PEP) in front
of other services within a FIWARE-based Smart Solution.

A **PEP Proxy** lies in front of a secured resource and is an endpoint found at "well-known" public location. It serves
as a gatekeeper for resource access. Users or other actors must supply sufficient information to the **PEP Proxy** to
allow their request to succeed and pass through the **PEP proxy**. The **PEP proxy** then passes the request on to the
real location of the secured resource itself - the actual location of the secured resource is unknown to the outside
user - it could be held in a private network behind the **PEP proxy** or found on a different machine altogether.

FIWARE [Wilma](https://fiware-pep-proxy.rtfd.io/) is a simple implementation of a **PEP proxy** designed to work with
the FIWARE [Keyrock](https://fiware-idm.readthedocs.io/en/latest/) Generic Enabler. Whenever a user tries to gain access
to the resource behind the **PEP proxy**, the PEP will describe the user's attributes to the Policy Decision Point
(PDP), request a security decision, and enforce the decision. (Permit or Deny). There is minimal disruption of access
for authorized users - the response received is the same as if they had accessed the secured service directly.
Unauthorized users are simply returned a **401 - Unauthorized** response.

## Standard Concepts of Identity Management

The following common objects are found with the **Keyrock** Identity Management database:

-   **User** - Any signed up user able to identify themselves with an eMail and password. Users can be assigned rights
    individually or as a group
-   **Application** - Any securable FIWARE application consisting of a series of microservices
-   **Organization** - A group of users who can be assigned a series of rights. Altering the rights of the organization
    effects the access of all users of that organization
-   **OrganizationRole** - Users can either be members or admins of an organization - Admins are able to add and remove
    users from their organization, members merely gain the roles and permissions of an organization. This allows each
    organization to be responsible for their members and removes the need for a super-admin to administer all rights
-   **Role** - A role is a descriptive bucket for a set of permissions. A role can be assigned to either a single user
    or an organization. A signed-in user gains all the permissions from all of their own roles plus all of the roles
    associated to their organization
-   **Permission** - An ability to do something on a resource within the system

Additionally two further non-human application objects can be secured within a FIWARE application:

-   **IoTAgent** - a proxy between IoT Sensors and the Context Broker
-   **PEPProxy** - a middleware for use between generic enablers challenging the rights of a user.

The relationship between the objects can be seen below - the entities marked in red are used directly within this
tutorial:

![](https://fiware.github.io/tutorials.PEP-Proxy/img/entities.png)

## :arrow_forward: Video : Introduction to Wilma PEP Proxy

[![](https://fiware.github.io/tutorials.Step-by-Step/img/video-logo.png)](https://www.youtube.com/watch?v=8tGbUI18udM "Introduction")

Click on the image above to see an introductory video

# Prerequisites

## Docker

To keep things simple both components will be run using [Docker](https://www.docker.com). **Docker** is a container
technology which allows to different components isolated into their respective environments.

-   To install Docker on Windows follow the instructions [here](https://docs.docker.com/docker-for-windows/)
-   To install Docker on Mac follow the instructions [here](https://docs.docker.com/docker-for-mac/)
-   To install Docker on Linux follow the instructions [here](https://docs.docker.com/install/)

**Docker Compose** is a tool for defining and running multi-container Docker applications. A
[YAML file](https://raw.githubusercontent.com/Fiware/tutorials.Identity-Management/master/docker-compose.yml) is used
configure the required services for the application. This means all container services can be brought up in a single
command. Docker Compose is installed by default as part of Docker for Windows and Docker for Mac, however Linux users
will need to follow the instructions found [here](https://docs.docker.com/compose/install/)

## Cygwin

We will start up our services using a simple bash script. Windows users should download [cygwin](http://www.cygwin.com/)
to provide a command-line functionality similar to a Linux distribution on Windows.

## Postman

## http

## jq


# Architecture

This application protects access to the existing Stock Management and Sensors-based application by adding PEP Proxy
instances around the services created in previous tutorials and uses data pre-populated into the **MySQL** database used
by **Keyrock**. It will make use of four FIWARE components - the
[Orion Context Broker](https://fiware-orion.readthedocs.io/en/latest/), the
[Keyrock](https://fiware-idm.readthedocs.io/en/latest/) Generic enabler and adds one or two instances
[Wilma](https://fiware-pep-proxy.rtfd.io/) PEP Proxy dependent upon which interfaces are to be secured.

The Orion-LD Context Broker rely on open source [MongoDB](https://www.mongodb.com/) technology to keep persistence 
of the information they hold. **Keyrock** uses its own [MySQL](https://www.mysql.com/) database.

Therefore the overall architecture will consist of the following elements:

-   The FIWARE [Orion Context Broker](https://fiware-orion.readthedocs.io/en/latest/) which will receive requests using
    [NGSI-LD](https://forge.etsi.org/swagger/ui/?url=https://forge.etsi.org/gitlab/NGSI-LD/NGSI-LD/raw/master/spec/updated/full_api.json)
-   FIWARE [Keyrock](https://fiware-idm.readthedocs.io/en/latest/) offer a complement Identity Management System
    including:
    -   An OAuth2 authentication system for Applications and Users
    -   A site graphical frontend for Identity Management Administration
    -   An equivalent REST API for Identity Management via HTTP requests
-   FIWARE [Wilma](https://fiware-pep-proxy.rtfd.io/) is a PEP Proxy securing access to the **Orion** microservice.
-   The underlying [MongoDB](https://www.mongodb.com/) database :
    -   Used by the **Orion-LD Context Broker** to hold context data information such as data entities, subscriptions and
        registrations
    -   Used by the **IoT Agent** to hold device information such as device URLs and Keys
-   A [MySQL](https://www.mysql.com/) database :
    -   Used to persist user identities, applications, roles and permissions

Since all interactions between the elements are initiated by HTTP requests, the entities can be containerized and run
from exposed ports.

The specific architecture of each section of the tutorial is discussed below.

# Start Up

To start the installation, do the following:

```console
git clone https://github.com/FIWARE/tutorials.PEP-Proxy.git
cd tutorials.PEP-Proxy
git checkout NGSI-v2

./services create
```

> **Note** The initial creation of Docker images can take up to three minutes

Thereafter, all services can be initialized from the command-line by running the
[services](https://github.com/flopezag/tutorials.secured-OrionLD/blob/main/services) 
Bash script provided within the repository:

```console
./services <command>
```

Where `<command>` will be help, start, stop or create.

> :information_source: **Note:** If you want to clean up and start over again you can do so with the following command:
>
> ```console
> ./services stop
> ```

## Dramatis Personae

### Introduction

The following people at `test.com` legitimately have accounts within the Application

-   Alice, she will be the Administrator of the **Identity Management** Application. The account is created in the 
    initialization process of the Identity Management.
-   Bob, administrator of the application, he has access to read and write the Personal Data store in the application.
-   Charlie, he is an application's user. He needs to read the Personal Data of the users but cannot modify them.

The following people at `example.com` have signed up for accounts, but have no reason to be granted access
to the data

-   Eve - Eve the Eavesdropper
-   Mallory - Mallory the malicious attacker

The following people at `xyz.foo` have signed up for accounts and can access to their Personal Data for reading 
and writing only:
-   Ole
-   Torsten
-   Frank
-   Lothar

<details>
  <summary>
   For more details <b>(Click to expand)</b>
  </summary>

   | Name       | eMail                          | Password |
   | ---------- | ------------------------------ | -------- |
   | Alice      | `alice-the-admin@test.com`     | `test`   |
   | Bob        | `bob-the-appmanager@test.com`  | `test`   |
   | Charlie    | `charlie-the-appuser@test.com` | `test`   |

   | Name    | eMail                 | Password |
   | ------- | --------------------- | -------- |
   | Eve     | `eve@example.com`     | `test`   |
   | Mallory | `mallory@example.com` | `test`   |

   | Name    | eMail                     | Password |
   | ------- | ------------------------- | -------- |
   | Ole     | `ole-lahm@xyz.foo`        | `test`   |
   | Torsten | `torsten-kuehl@xyz.foo`   | `test`   |
   | Frank   | `frank-king@xyz.foo`      | `test`   |
   | Lothar  | `lothar-lammich@xyz.foo`  | `test`   |

</details>

Three organizations have also been set up by Alice:

| Name       | Description                          |
| ---------- | ------------------------------------ |
| Managers   | This group is for the Project Managers of the Personal Data application with full control access.          |
| Users      | This group is for the Project Users of the Personal Data application with read control access.             |
| Data       | This group is for the Personal Data owners who can read and modify only their own data.                    |
| Others     | This group is for the rest of IdM registered users not authorized to access the Personal Data Application. |

One application, with appropriate roles and permissions has also been created:

| Key           | Value                                  |
| ------------- | -------------------------------------- |
| Client ID     | `tutorial-dckr-site-0000-xpresswebapp` |
| Client Secret | `tutorial-dckr-site-0000-clientsecret` |
| URL           | `http://localhost:3000`                |
| RedirectURL   | `http://localhost:3000/login`          |

### Logging In to Keyrock using the REST API. Getting admin token

Enter a username and password to enter the application. The default user has the values `alice-the-admin@test.com`
and `test`. The following example logs in using the Admin User, if you want to obtain the corresponding tokens for 
the other users after their creation just change the proper name and password data in this request:

##### :one: Request:

```console
http POST http://localhost:3005/v1/auth/tokens \
  name=alice-the-admin@test.com \
  password=test
```

##### Response:

The response header returns an `X-Subject-token` which identifies who has logged on the application. This token is
required in all subsequent requests to gain access

```yaml
HTTP/1.1 201 Created
Cache-Control: no-cache, private, no-store, must-revalidate, max-stale=0, post-check=0, pre-check=0
Connection: keep-alive
Content-Length: 138
Content-Security-Policy: default-src 'self' img-src 'self' data:;script-src 'self' 'unsafe-inline';style-src 'self' https: 'unsafe-inline'
Content-Type: application/json; charset=utf-8
Date: Wed, 10 Feb 2021 08:31:27 GMT
ETag: W/"8a-SCtuhPlCxvhqNChN4qFnlyzMINs"
Expect-CT: max-age=0
Referrer-Policy: no-referrer
Set-Cookie: session=eyJyZWRpciI6Ii8ifQ==; path=/; expires=Wed, 10 Feb 2021 09:31:27 GMT; httponly
Set-Cookie: session.sig=80Qc1EoFnglVR7H5hG9_Rad6txc; path=/; expires=Wed, 10 Feb 2021 09:31:27 GMT; httponly
Strict-Transport-Security: max-age=15552000; includeSubDomains
X-Content-Type-Options: nosniff
X-DNS-Prefetch-Control: off
X-Download-Options: noopen
X-Frame-Options: SAMEORIGIN
X-Permitted-Cross-Domain-Policies: none
X-Subject-Token: f66fe9ee-1910-4d3c-9710-79795ca37ac3
X-XSS-Protection: 0
```

```json
{
    "idm_authorization_config": {
        "authzforce": false,
        "level": "basic"
    },
    "token": {
        "expires_at": "2021-02-10T09:31:27.065Z",
        "methods": [
            "password"
        ]
    }
}
```

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

#### 5 Request:

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

#### Response:

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

#### 6 Request:

```bash
http GET 'http://localhost:3005/v1/users' \
 X-Auth-Token:"$TOKEN"
```

#### Response:

The response contains basic details of all accounts:

```json
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


### Grouping User Accounts under Organizations

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
reading or writing. Rather than give access to each individual account, it would be easier to assign the rights to 
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

#### Create an Organization

The standard CRUD actions are assigned to the appropriate HTTP verbs (POST, GET, PATCH and DELETE) under
the `/v1/organizations` endpoint. To create a new organization, send a POST request to the `/v1/organizations` 
endpoint containing the `name` and `description` along with the `X-Auth-token` header from a previously 
logged-in user.

> **Note** You can take a look and execute the organization-mgmt script to automatically create all organizations
> and assign the users to each organization.

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

#### List all Organizations

Obtaining a complete list of all organizations is a super-admin permission requiring the `X-Auth-token` - most users
will only be permitted to return users within their own organization. Listing users can be done by making a GET request
to the `/v1/organizations` endpoint.

```bash
http GET http://localhost:3005/v1/organizations \
 X-Auth-Token:"$TOKEN"
```

The response returns the details of the visible organizations.

```json
{
    "organizations": [
        {
            "Organization": {
                "description": "This group is for the Personal Data owners who can read and modify only their own data",
                "id": "1d157e87-32e3-4812-bde2-c0d1e3967170",
                "image": "default",
                "name": "Data",
                "website": null
            },
            "role": "owner"
        },
        {
            "Organization": {
                "description": "This group is for the Project Users of the Personal Data application with read control access",
                "id": "531f0f5c-a7c4-4826-96b2-31988caefc11",
                "image": "default",
                "name": "Users",
                "website": null
            },
            "role": "owner"
        },
        {
            "Organization": {
                "description": "This group is for the rest of IdM registered users not authorized to access the Personal Data Application",
                "id": "df58f9d2-443a-4375-abf7-00a88677e7b5",
                "image": "default",
                "name": "Others",
                "website": null
            },
            "role": "owner"
        },
        {
            "Organization": {
                "description": "This group is for the Project Managers of the Personal Data application with full control access",
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


#### Assign users to organizations

Users within an Organization are assigned to one of types - `owner` or `member`. The members of an organization inherit
all the roles and permissions assigned to the organization itself. In addition, owners of an organization are able to
add and remove other members and owners.

To add a user as a member of an organization, an owner must make a PUT request as shown, including the
`<organization-id>` and `<user-id>` in the URL path and identifying themselves using an `X-Auth-Token` in the header.

##### 14 Request:

```bash
http  PUT "http://localhost:3005/v1/organizations/$MANAGERS/users/$BOB/organization_roles/member" \
 Content-Type:'application/json' \
 X-Auth-Token:"$TOKEN"
```

We have to repeat this operation for all the users created previously.

> Note: $MANAGERS corresponds to the organization id of the _Managers_ organization and $BOB correponds to the user id
> of the Bob user. See the mgmt-users-organization script for more details

##### Response:

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

#### List Users within an Organization

Listing users within an organization is an `owner` or super-admin permission requiring the `X-Auth-token` Listing 
users can be done by making a GET request to the `/v1/organizations/{{organization-id}}/users` endpoint.

#### 16 Request:

```bash
http GET "http://localhost:3005/v1/organizations/$OTHERS/users" \
 X-Auth-Token:"$TOKEN"
```

#### Response:

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

![](https://fiware.github.io/tutorials.Roles-Permissions/img/entities.png)

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
from a previously logged in user will automatically be granted a provider role over the application.

#### Request:

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

#### Response:

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
endpoint containing the `action`and `resource` along with the `X-Auth-Token` header from a previously logged in 
user (Alice).

#### 8 Request:

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

#### Response:

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

| Permission | Verb  | Resource                     | Description                                                   | Organizations   |
| ---------- | ----- | ---------------------------- | ------------------------------------------------------------- | --------------- |
| #1         | GET   | /entities/*                  | Get detailed information of an entity (all entities)          | MANAGERS, USERS |
| #2         | GET   | /entities/{{entityID}}       | Get detailed information of an entity (one entity)            | DATA            |
| #3         | POST  | /entityOperations/upsert     | Add some entities                                             | MANAGERS        |
| #4         | PATCH | /entities/*/attrs            | Update the information associated to an entity (all entities) | MANAGERS        |
| #5         | PATCH | /entities/{{entityID}}/attrs | Update the information associated to an entity (one entity)   | DATA            |

We have to mention that the permission #1 include the permission #2, and the permission #2 in generated after
we have the upload the Personal Data associated to a person (e.g. Ole's Personal Data has the entityID 
`urn:ngsi-ld:Person:person001`).

> Note: We should manage all the permissions related to the OrionLD API but for this document we will center only
> on the previous resources.
> 
> Note: The script `mgmt-users-organizations` will create all the corresponding permissions for this example application.

### List Permissions

Listing the permissions with an application can be done by making a GET request to the
`/v1/applications/{{application-id}}/permissions/` endpoint

#### 10 Request:

```bash
http GET "http://localhost:3005/v1/applications/$APP/permissions" \
 X-Auth-Token:"$TOKEN"
```

#### Response:

The complete list of permissions includes any custom permissions created previously plus all the standard permissions
which are avaiable by default

```json
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
    
  ...
  
  ]
}
```

### Create a Role

A permission is an allowable action on a resource, as noted above. A role consists of a group of permissions, in other
words a series of permitted actions over a group of resources. Roles are usually given a description with a broad scope
so that they can be assigned to a wide range of users or organizations for example a _Reader_ role could be able to
access but not update a series of devices.

There are two predefined roles with **Keyrock** :

-   a _Purchaser_ who can
    -   Get and assign all public application roles
-   a _Provider_ who can:
    -   Get and assign only public owned roles
    -   Get and assign all public application roles
    -   Manage authorizations
    -   Manage roles
    -   Manage the application
    -   Get and assign all internal application roles

Using our Personal Data Example, Alice the admin would be assigned the _Provider_ role, she could then create any
additional application-specific roles needed (such as _Manager_, _Users_, _Data_ or _Others_).

Roles are always directly bound to an application - abstract roles do not exist on their own. The standard
CRUD actions are assigned to the appropriate HTTP verbs (POST, GET, PATCH and DELETE) under the
`/v1/applications/{{application-id}}/roles` endpoint.

To create a new role via the REST API, send a POST request to the `/applications/{{application-id}}/roles` endpoint
containing the `name` of the new role, with the `X-Auth-token` header from a previously logged in user.

#### 13 Request:

```bash
printf '{
  "role": {
    "name": "Manager"
  }
}'| http  POST "http://localhost:3005/v1/applications/$APP/roles" \
 X-Auth-Token:"$TOKEN"
```

#### Response:

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

| *Role*    | *Permissions*                                                           |
| --------- | ----------------------------------------------------------------------- |
| Manager   | #1(/entities/*), #3(/entityOperations/upsert), #4(/entities/*/attrs)    |
| Users     | #1(/entities/*)                                                         |
| Data      | #2(/entities/{{entityID}}), #5(/entities/{{entityID}}/attrs)            |
| Others    |                                                                         |

Due to the roles are associated to the application, the Role _Others_ does not have any permission assigned in the
application, therefore the users under the Role Others should be rejected.

#### 18 Request:

```bash
http PUT "http://localhost:3005/v1/applications/$APP/roles/$MANAGER_ROLE/permissions/$PERMID' \
 X-Auth-Token:"$TOKEN"
```

#### Response:

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
`/v1/applications/{{application-id}}/roles/{{role-id}}/permissions` endpoint

#### 19 Request:

```bash
http GET "http://localhost:3005/v1/applications/$APP/roles/$MANAGERS/permissions" \
  X-Auth-Token:"$TOKEN"
```

#### Response:

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

```bash
http GET "http://localhost:3005/v1/applications/$APP/roles/$MANAGERS/permissions" \
  X-Auth-Token:"$TOKEN"
```

and the response:
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

### Create a PEP Proxy

By default, the docker-compose is created with default credentials to be used by the PEP Proxy, but it is not a good
example to use in production environment, and it is recommended to create a new PEP Proxy account. To create a new PEP 
Proxy account within an application, send a POST request to the `/v1/applications/{{application-id}}/pep_proxies` 
endpoint along with the `X-Auth-Token` header from a previously logged in administrative user.

Provided there is no previously existing PEP Proxy account associated with the application, a new account will be 
created with a unique id and password and the values will be returned in the response. The first two data are obteined
in the creation of the PEP Proxy, the Application Id is obtained after the creation if you request the PEP Proxy details
associated to the application.

The following table summarize the Data that are needed, where you can find them and which are the configuration 
parameters in PEP Proxy associated to this value.

| Data               | Request response          | Configuration parameter |
| ------------------ | ------------------------- | ----------------------- |
| Pep Proxy Username | pep.proxy.id              | PEP_PROXY_USERNAME      |
| PEP Proxy Password | pep_proxy.password        | PEP_PASSWORD            |
| Application Id     | pep_proxy.oauth_client_id | PEP_PROXY_APP_ID        |

Finally, there will be only one credential associated to an application for a PEP Proxy, therefore a subsequent request
will produce a 409 Conflict with the message `Pep Proxy already registered`.

#### Request:

```bash
http POST "http://localhost:3005/v1/applications/$APP/pep_proxies" \
Content-Type:application/json \
X-Auth-Token:"$TOKEN"
```

#### Response:

```json
{
    "pep_proxy": {
        "id": "pep_proxy_5551b5d3-8293-41cb-b569-5836097224ab",
        "password": "pep_proxy_d7a44050-7c61-4f1e-ae9d-49bb626c41c7"
    }
}
```

### Read PEP Proxy details

Making a GET request to the `/v1/applications/{{application-id}}/pep_proxies` endpoint will return the details of the 
associated PEP Proxy Account. The `X-Auth-Token` must be supplied in the headers. It is important to see that if you
want to obtain the `oauth_client_id`, you need to request this information with the API.

#### Request:

```bash
http GET "http://localhost:3005/v1/applications/$APP/pep_proxies" \
Content-Type:application/json \
X-Auth-Token:"$TOKEN"
```

#### Response:

```json
{
    "pep_proxy": {
        "id": "pep_proxy_5551b5d3-8293-41cb-b569-5836097224ab",
        "oauth_client_id": "6f0d6fa9-888e-4371-b9e8-3863e503d242"
    }
}
```

> Note: To update the PEP Proxy credentials just change the configuration parameters *PEP_PROXY_APP_ID*, *PEP_PASSWORD*,
> and *PEP_PROXY_USERNAME* in the docker-compose file and launch again the docker-compose. It automatically updates the 
> PEP Proxy container with the new data. For your convenience, the script application-management execute all the process.

### Authorizing Application Access

In the end, a user logs into an application , identifies himself and then is granted a list of permissions that the user
is able to do. However it should be emphasized that it is the application, not the user that holds and offers the
permissions, and the user is merely associated with a aggregated list of permissions via the role(s) they have been
granted.

The application can grant roles to either Users or Organizations - the latter should always be preferred, as it allows
the owners of the organization to add new users - delegating the responsibility for user maintenance to a wider group.

For example, imagine the supermarket gains another store detective. Alice has already created role called Security and
assigned it to the Security team. Charlie is the owner of the Security team organization, and is able to add the new
`detective3` user to his team. `detective3` can then inherit all the rights of his team without further input from
Alice.

Granting roles to individual Users should be restricted to special cases - some roles may be very specialized an only
contain one member so there is no need to create an organization. This reduced the administrative burden when setting up
the application, but any further changes (such as removing access rights when someone leaves) will need to be done by
Alice herself - no delegation is possible.

## Authorizing Organizations

A role cannot be granted to an organization unless the role has already been defined within the application itself. The
organization must also have be created as was demonstrated in the previous tutorial.

### Grant a Role to an Organization

To grant an organization access to an application, click on the appliation to get to the details page and scroll to the
bottom of the page, click the **Authorize** button and select the relevant organization.

![](https://fiware.github.io/tutorials.Roles-Permissions/img/add-role-to-org.png)

A Role can be granted to either `members` or `owners` of an Organization. Using the REST API, the role can be granted
making a PUT request as shown, including the `<application-id>`, `<role-id>` and `<organzation-id>` in the URL path and
identifying themselves using an `X-Auth-Token` in the header.

#### 21 Request:

This example adds the role to all members of the organization

```bash
curl -iX PUT \
  'http://localhost:3005/v1/applications/{{application-id}}/organizations/{{organization-id}}/roles/{{role-id}}/organization_roles/member' \
  -H 'Content-Type: application/json' \
  -H 'X-Auth-token: {{X-Auth-token}}'
```

#### Response:

The response lists the role assignment as shown:

```json
{
    "role_organization_assignments": {
        "role_id": "64535f4d-04b6-4688-a9bb-81b8df7c4e2c",
        "organization_id": "security-0000-0000-0000-000000000000",
        "oauth_client_id": "3782c5e3-88f9-481a-9b3c-2f2d6f604482",
        "role_organization": "member"
    }
}
```

### List Granted Organization Roles

A full list of roles granted to an organization can be retrieved by making a GET request to the
`/v1/applications/{{application-id}}/organizations/{{organization-id}}/roles` endpoint

#### 22 Request:

```bash
curl -X GET \
  'http://localhost:3005/v1/applications/{{application-id}}/organizations/{{organization-id}}/roles' \
  -H 'Content-Type: application/json' \
  -H 'X-Auth-token: {{X-Auth-token}}'
```

#### Response:

The response shows all roles assigned to the organization

```json
{
    "role_organization_assignments": [
        {
            "organization_id": "security-0000-0000-0000-000000000000",
            "role_id": "64535f4d-04b6-4688-a9bb-81b8df7c4e2c"
        }
    ]
}
```

## Authorizing Individual User Accounts

A defined role cannot be granted to a user unless the role has already been associated to an application

### Grant a Role to a User

Granting User access via the GUI can be done in the same manner as for organizations.

A Role can be granted to either `members` or `owners` of an Organization. Using the REST API, the role can be granted
making a PUT request as shown, including the `<application-id>`, `<role-id>` and `<user-id>` in the URL path and
identifying themselves using an `X-Auth-Token` in the header.

#### 24 Request:

```bash
curl -iX PUT \
  'http://localhost:3005/v1/applications/{{application-id}}/users/{{user-id}}/roles/{{role-id}}' \
  -H 'Content-Type: application/json' \
  -H 'X-Auth-token: {{X-Auth-token}}'
```

#### Response:

```json
{
    "role_user_assignments": {
        "role_id": "64535f4d-04b6-4688-a9bb-81b8df7c4e2c",
        "user_id": "bbbbbbbb-good-0000-0000-000000000000",
        "oauth_client_id": "3782c5e3-88f9-481a-9b3c-2f2d6f604482"
    }
}
```

### List Granted User Roles

To list the roles granted to an Individual user, make a GET request to the
`v1/applications/{{application-id}}/users/{{user-id}}/roles` endpoint

#### 25 Request:

```bash
curl -X GET \
  'http://localhost:3005/v1/applications/{{application-id}}/users/{{user-id}}/roles' \
  -H 'Content-Type: application/json' \
  -H 'X-Auth-token: {{X-Auth-token}}'
```

#### Response:

The response returns all roles assigned to the user

```json
{
    "role_user_assignments": [
        {
            "user_id": "bbbbbbbb-good-0000-0000-000000000000",
            "role_id": "64535f4d-04b6-4688-a9bb-81b8df7c4e2c"
        }
    ]
}
```

## List Application Grantees

By creating a series of roles and granting them to Users and Organizations, we have made an association between them.
The REST API offers two convienience methods exist to list all the grantees of an application

### List Authorized Organizations

To list all organizations which are authorized to use an application, make a GET request to the
`/v1/applications/{{application-id}}/organizations` endpoint.

#### 27 Request:

```bash
curl -X GET \
  'http://localhost:3005/v1/applications/{{application-id}}/organizations/{{organizations-id}}/roles' \
  -H 'Content-Type: application/json' \
  -H 'X-Auth-token: {{X-Auth-token}}'
```

#### Response:

The response returns all organizations which can access the application and the roles they have been assigned.
Individual members are not listed.

```json
{
    "role_organization_assignments": [
        {
            "organization_id": "security-0000-0000-0000-000000000000",
            "role_organization": "member",
            "role_id": "64535f4d-04b6-4688-a9bb-81b8df7c4e2c"
        }
    ]
}
```

### List Authorized Users

To list all individual users who are authorized to use an application, make a GET request to the
`/v1/applications/{{application-id}}/users` endpoint.

#### 28 Request:

```bash
curl -X GET \
  'http://localhost:3005/v1/applications/{{application-id}}/users/{{user-id}}/roles' \
  -H 'Content-Type: application/json' \
  -H 'X-Auth-token: {{X-Auth-token}}'
```

#### Response:

The response returns all individual users who can access the application and the roles they have been assigned. Note
that users of an organization granted access are not listed.

```json
{
    "role_user_assignments": [
        {
            "user_id": "aaaaaaaa-good-0000-0000-000000000000",
            "role_id": "provider"
        },
        {
            "user_id": "bbbbbbbb-good-0000-0000-000000000000",
            "role_id": "64535f4d-04b6-4688-a9bb-81b8df7c4e2c"
        }
    ]
}
```

# Securing the OrionLD Context Broker

![](https://fiware.github.io/tutorials.PEP-Proxy/img/pep-proxy-orion.png)

## Securing Orion-LD - PEP Proxy Configuration

The `orion-proxy` container is an instance of FIWARE **Wilma** listening on port `1027`, it is configured to forward
traffic to `orion` on port `1026`, which is the default port that the Orion-LD Context Broker is listening to for NGSI
Requests.

```yaml
orion-proxy:
    image: fiware/pep-proxy
    container_name: fiware-orion-proxy
    hostname: orion-pepproxy
    networks:
        default:
            ipv4_address: 172.18.1.10
    depends_on:
        - keyrock
    ports:
        - "1027:1027"
    expose:
        - "1027"
    environment:
        - PEP_PROXY_APP_HOST=orionld
        - PEP_PROXY_APP_PORT=1026
        - PEP_PROXY_PORT=1027
        - PEP_PROXY_IDM_HOST=keyrock
        - PEP_PROXY_HTTPS_ENABLED=false
        - PEP_PROXY_AUTH_ENABLED=false
        - PEP_PROXY_IDM_SSL_ENABLED=false
        - PEP_PROXY_IDM_PORT=3005
        - PEP_PROXY_APP_ID=tutorial-dckr-site-0000-xpresswebapp
        - PEP_PROXY_USERNAME=pep_proxy_00000000-0000-0000-0000-000000000000
        - PEP_PASSWORD=test
        - PEP_PROXY_PDP=idm
        - PEP_PROXY_MAGIC_KEY=1234
```

The `PEP_PROXY_APP_ID` and `PEP_PROXY_USERNAME` would usually be obtained by adding new entries to the application in
**Keyrock**, however, in this tutorial, they have been predefined by populating the **MySQL** database with data on
start-up.

The `orion-pepproxy` container is listening on a single port:

-   The PEP Proxy Port - `1027` is exposed purely for tutorial access - so that cUrl or Postman can requests directly to
    the **Wilma** instance without being part of the same network.

| Key                       | Value                                            | Description                                            |
| ------------------------- | ------------------------------------------------ | ------------------------------------------------------ |
| PEP_PROXY_APP_HOST        | `orionld`                                        | The hostname of the service behind the PEP Proxy       |
| PEP_PROXY_APP_PORT        | `1026`                                           | The port of the service behind the PEP Proxy           |
| PEP_PROXY_PORT            | `1027`                                           | The port that the PEP Proxy is listening on            |
| PEP_PROXY_IDM_HOST        | `keyrock`                                        | The hostname for the Identity Manager                  |
| PEP_PROXY_HTTPS_ENABLED   | `false`                                          | Whether the PEP Proxy itself is running under HTTPS    |
| PEP_PROXY_AUTH_ENABLED    | `false`                                          | Whether the PEP Proxy is checking for Authorization    |
| PEP_PROXY_IDM_SSL_ENABLED | `false`                                          | Whether the Identity Manager is running under HTTPS    |
| PEP_PROXY_IDM_PORT        | `3005`                                           | The Port for the Identity Manager instance             |
| PEP_PROXY_APP_ID          | `tutorial-dckr-site-0000-xpresswebapp`           |                                                        |
| PEP_PROXY_USERNAME        | `pep_proxy_00000000-0000-0000-0000-000000000000` | The Username for the PEP Proxy                         |
| PEP_PASSWORD              | `test`                                           | The Password for the PEP Proxy                         |
| PEP_PROXY_PDP             | `idm`                                            | The Type of service offering the Policy Decision Point |
| PEP_PROXY_MAGIC_KEY       | `1234`                                           |                                                        |

For this example, the PEP Proxy is checking for Level 1 - _Authentication Access_ not Level 2 - _Basic Authorization_ or
Level 3 - _Advanced Authorization_.

## Securing Orion-LD 

### :arrow_forward: Video : Securing A REST API

[![](https://fiware.github.io/tutorials.Step-by-Step/img/video-logo.png)](https://www.youtube.com/watch?v=coxFQEY0_So "Securing a REST API")

Click on the image above to see a video about securing a REST API using the Wilma PEP Proxy

## User Logs In to the Application using the REST API

### PEP Proxy - No Access to Orion-LD without an Access Token

Secured Access can be ensured by requiring all requests to the secured service are made indirectly via a PEP Proxy (in
this case the PEP Proxy is found in front of the Context Broker). Requests must include an `X-Auth-Token`, failure to
present a valid token results in a denial of access.

#### :one::two: Request:

If a request to the PEP Proxy is made without any access token as shown:

```console
http GET http://localhost:1027/ngsi-ld/v1/entities/urn:ngsi-ld:Person:001?options=keyValues \           
Link:'<https://fiware.github.io/tutorials.Step-by-Step/tutorials-context.jsonld>; rel="http://www.w3.org/ns/json-ld#context"; type="application/ld+json"'  
```

#### Response:

The response is a **401 Unauthorized** error code, with the following explanation:

```
Auth-token not found in request header
```

### Keyrock - User obtains an Access Token

#### :one::three: Request:

To log in to the application using the user-credentials flow send a POST request to **Keyrock** using the `oauth2/token`
endpoint with the `grant_type=password`. Additionally, the authorization filed is constructed as follow:

For example to log-in as Alice the Admin:
* The Client ID and Client Secret created in the IDM for your application are combined with a single colon (:). 
  This means that the Client ID itself cannot contain a colon.
* The resulting string is encoded using a variant of Base64. For your convenience you can use the following 
  command line instruction:
  
  ```console
  #! echo -n "<Client ID>:<Client Secret>" | base64
  ```
* The authorization method and a space (e.g. "Basic ") is then prepended to the encoded string.

For example to log-in as Alice the Admin:

```console
http --form POST 'http://localhost:3005/oauth2/token' \
 username=alice-the-admin@test.com \  
 password=test \   
 grant_type=password \  
 Authorization:'Basic dHV0b3JpYWwtZGNrci1zaXRlLTAwMDAteHByZXNzd2ViYXBwOnR1dG9yaWFsLWRja3Itc2l0ZS0wMDAwLWNsaWVudHNlY3JldA==' \
 Content-Type:'application/x-www-form-urlencoded'
```

#### Response:

The response returns an access code to identify the user:

```json
{
    "access_token": "25c5f5a1a53984d7322b36a6227f36201399a471",
    "expires_in": 3599,
    "refresh_token": "6527f0b041eb890ea361fdd8db91ce2781f60618",
    "scope": [
        "bearer"
    ],
    "token_type": "bearer"
}
```

This can also be done by entering the Tutorial Application on http:/localhost and logging in using any of the OAuth2
grants on the page. A successful log-in will return an access token. For the next step, we export a TOKEN variable 
to keep the information of the oAuth token.

```console
export TOKEN={{access_token}}
```

### PEP Proxy - Accessing Orion-LD with an Authorization

The standard `Authorization: Bearer` header can also be used to identity the user, the request from an authorized user
is permitted and the service behind the PEP Proxy (in this case the Orion-LD Context Broker) will return the data as
expected.

#### :one::five: Request:

```console
http GET http://localhost:1027/ngsi-ld/v1/entities/urn:ngsi-ld:Person:person001?options=keyValues \
 Link:'<https://schema.lab.fiware.org/ld/context>; rel="http://www.w3.org/ns/json-ld#context"; type="application/ld+json"' \
 Authorization:"Bearer $TOKEN"
```

#### Response:

```json
{
    "@context": "https://schema.lab.fiware.org/ld/context",
    "address": {
        "addressLocality": "Berlin",
        "addressRegion": "Berlin",
        "postalCode": "14199",
        "streetAddress": "Detmolder Str. 10"
    },
    "email": "ole-lahm@xyz.foo",
    "id": "urn:ngsi-ld:Person:person001",
    "name": "Ole Lahm",
    "telephone": "0049 1522 99999999",
    "type": "Person"
}
```


## License

[MIT](LICENSE) © 2018-2020 FIWARE Foundation e.V.
