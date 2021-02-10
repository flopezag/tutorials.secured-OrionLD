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

| Name       | Description                         | UUID                                   |
| ---------- | ----------------------------------- | -------------------------------------- |
| Security   | Security Group for Store Detectives | `security-team-0000-0000-000000000000` |
| Management | Management Group for Store Managers | `managers-team-0000-0000-000000000000` |
| Security   | Security Group for Store Detectives | `security-team-0000-0000-000000000000` |
| Security   | Security Group for Store Detectives | `security-team-0000-0000-000000000000` |

One application, with appropriate roles and permissions has also been created:

| Key           | Value                                  |
| ------------- | -------------------------------------- |
| Client ID     | `tutorial-dckr-site-0000-xpresswebapp` |
| Client Secret | `tutorial-dckr-site-0000-clientsecret` |
| URL           | `http://localhost:3000`                |
| RedirectURL   | `http://localhost:3000/login`          |

### Logging In to Keyrock using the REST API

Enter a username and password to enter the application. The default user has the values `alice-the-admin@test.com`
and `test`.

#### Create Token with Password

The following example logs in using the Admin User, if you want to obtain the corresponding tokens for the other users
after their creation just change the proper name and password data in this request:

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
along with the `X-Auth-Token` header from a previously logged in administrative user (see previous section). Additional
users can be added by making repeated POST requests with the proper information following the previous table. 
For example to create additional accounts for Bob, the Application Manager we should execute the following request

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

---

# Grouping User Accounts under Organizations

For any identity management system of a reasonable size, it is useful to be able to assign roles to groups of users,
rather than setting them up individually. Since user administration is a time consuming business, it is also necessary
to be able to delegate the responsibility of managing these group of users down to other accounts with a lower level of
access.

Consider our supermarket chain for example, there could be a group of users (Managers) who can change the prices of
products within the store, and another group of users (Store Detectives) who can lock and unlock door after closing
time. Rather than give access to each individual account, it would be easier to assign the rights to an organization and
then add users to the groups.

Furthermore, Alice, the **Keyrock** administrator does not need to explicitly add additional user accounts to each
organization herself - she could delegate that right to an owner within each organization. For example Bob the Regional
Manager would be made the owner of the _management_ organization and could add and remove addition manager accounts
(such as `manager1` and `manager2`) to that organization whereas Charlie the Head of Security could be handed an
ownership role in the _security_ organization and add additional store detectives to that organization.

Note that Bob does not have the rights to alter the membership list of the _security_ organization and Charlie does not
have the rights to alter the membership list of the _management_ organization. Furthermore neither Bob nor Charlie would
be able to alter the permissions of the application themselves, merely add and remove existing user accounts to the
organization they control.

Creating an application and setting-up the permissions is not covered here as it is the subject of the next tutorial.

## Organization CRUD Actions

#### GUI

Once signed-in, users are able to create and update organizations for themselves.

![](https://fiware.github.io/tutorials.Identity-Management/img/create-org.png)

#### REST API

Alternatively, the standard CRUD actions are assigned to the appropriate HTTP verbs (POST, GET, PATCH and DELETE) under
the `/v1/organizations` endpoint.

### Create an Organization

To create a new organization, send a POST request to the `/v1/organizations` endpoint containing the `name` and
`description` along with the `X-Auth-token` header from a previously logged in user.

#### 9 Request:

```bash
curl -iX POST \
  'http://localhost:3005/v1/organizations' \
  -H 'Content-Type: application/json' \
  -H 'X-Auth-token: {{X-Auth-token}}' \
  -d '{
  "organization": {
    "name": "Security",
    "description": "This group is for the store detectives"
  }
}'
```

#### Response:

The Organization is created and the user who created it is automatically assigned as a user. The response returns UUID
to identify the new organization.

```json
{
    "organization": {
        "id": "18deea43-e12a-4018-a45a-664c3158780d",
        "image": "default",
        "name": "Security",
        "description": "This group is for the store detectives"
    }
}
```

### Read Organization Details

Making a GET request to a resource under the `/v1/organizations/{{organization-id}}` endpoint will return the
organization listed under that ID. The `X-Auth-token` must be supplied in the headers as only permitted organizations
will be shown.

#### 10 Request:

```bash
curl -X GET \
  'http://localhost:3005/v1/organizations/{{organization-id}}' \
  -H 'Content-Type: application/json' \
  -H 'X-Auth-token: {{X-Auth-token}}'
```

#### Response:

The response returns the details of the organization.

```json
{
    "organization": {
        "id": "18deea43-e12a-4018-a45a-664c3158780d",
        "name": "Security",
        "description": "This group is for the store detectives",
        "website": null,
        "image": "default"
    }
}
```

### List all Organizations

Obtaining a complete list of all organizations is a super-admin permission requiring the `X-Auth-token` - most users
will only be permitted to return users within their own organization. Listing users can be done by making a GET request
to the `/v1/organizations` endpoint

#### 11 Request:

```bash
curl -X GET \
  'http://localhost:3005/v1/organizations' \
  -H 'Content-Type: application/json' \
  -H 'X-Auth-token: {{X-Auth-token}}'
```

#### Response:

The response returns the details of the visible organizations.

```json
{
    "organizations": [
        {
            "role": "owner",
            "Organization": {
                "id": "18deea43-e12a-4018-a45a-664c3158780d",
                "name": "Security",
                "description": "This group is for the store detectives",
                "image": "default",
                "website": null
            }
        },
        {
            "role": "owner",
            "Organization": {
                "id": "a45f9b5a-dd23-4d0f-a0d4-e97e2d7431a3",
                "name": "Management",
                "description": "This group is for the store manangers",
                "image": "default",
                "website": null
            }
        }
    ]
}
```

### Update an Organization

To amend the details of an existing organization, a PATCH request is send to the `/v1/organizations/{{organization-id}}`
endpoint.

#### 12 Request:

```bash
curl -iX PATCH \
  'http://localhost:3005/v1/organizations/{{organization-id}}' \
  -H 'Content-Type: application/json' \
  -H 'X-Auth-token: {{X-Auth-token}}' \
  -d '{
    "organization": {
        "name": "FIWARE Security",
        "description": "The FIWARE Foundation is the legal independent body promoting, augmenting open-source FIWARE technologies",
        "website": "https://fiware.org"
    }
}'
```

#### Response:

The response contains a list of the fields which have been amended.

```json
{
    "values_updated": {
        "name": "FIWARE Security",
        "description": "The FIWARE Foundation is the legal independent body promoting, augmenting open-source FIWARE technologies",
        "website": "https://fiware.org"
    }
}
```

### Delete an Organization

#### 13 Request:

```bash
curl -iX DELETE \
  'http://localhost:3005/v1/organizations/{{organization-id}}' \
  -H 'Content-Type: application/json' \
  -H 'X-Auth-token: {{X-Auth-token}}'
```

## Administrating Users within an Organization

Users within an Organization are assigned to one of types - `owner` or `member`. The members of an organization inherit
all of the roles and permissions assigned to the organization itself. In addition, owners of an organization are able to
add an remove other members and owners.

### Add a User as a Member of an Organization

To add a user to an organization using the GUI, first click on the existing organization, then click on the **Manage**
button:

![](https://fiware.github.io/tutorials.Identity-Management/img/add-user-to-org.png)

To add a user as a member of an organization, an owner must make a PUT request as shown, including the
`<organization-id>` and `<user-id>` in the URL path and identifying themselves using an `X-Auth-Token` in the header.

#### 14 Request:

```bash
curl -iX PUT \
  'http://localhost:3005/v1/organizations/{{organization-id}}/users/{{user-id}}/organization_roles/member' \
  -H 'Content-Type: application/json' \
  -H 'X-Auth-token: {{X-Auth-token}}'
```

#### Response:

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

### Add a User as an Owner of an Organization

An owner can also create new owners by making a PUT request as shown, including the `<organization-id>` and `<user-id>`
in the URL path and identifying themselves using an `X-Auth-Token` in the header.

#### 15 Request:

```bash
curl -iX PUT \
  'http://localhost:3005/v1/organizations/{{organization-id}}/users/{{user-id}}/organization_roles/owner' \
  -H 'Content-Type: application/json' \
  -H 'X-Auth-token: {{X-Auth-token}}'
```

#### Response:

The response lists the user's current role within the organization (i.e. `owner`)

```json
{
    "user_organization_assignments": {
        "role": "owner",
        "user_id": "5e482345-2c48-410e-ae03-203d67a43cea",
        "organization_id": "18deea43-e12a-4018-a45a-664c3158780d"
    }
}
```

### List Users within an Organization

To list the users of an organization using the GUI, just click on the existing organization:

![](https://fiware.github.io/tutorials.Identity-Management/img/org-with-users.png)

Listing users within an organization is an `owner` or super-admin permission requiring the `X-Auth-token` Listing users
can be done by making a GET request to the `/v1/organizations/{{organization-id}}/users` endpoint.

#### 16 Request:

```bash
curl -X GET \
  'http://localhost:3005/v1/organizations/{{organization-id}}/users' \
  -H 'Content-Type: application/json' \
  -H 'X-Auth-token: {{X-Auth-token}}'
```

#### Response:

The response contains the users list.

```json
{
    "organization_users": [
        {
            "user_id": "admin",
            "organization_id": "18deea43-e12a-4018-a45a-664c3158780d",
            "role": "owner"
        },
        {
            "user_id": "5e482345-2c48-410e-ae03-203d67a43cea",
            "organization_id": "18deea43-e12a-4018-a45a-664c3158780d",
            "role": "member"
        }
    ]
}
```

### Read User Roles within an Organization

To find the role of a user within an organization, send a GET request to the
`/v1/organizations/{{organization-id}}/users/{{user-id}}/organization_roles` endpoint.

#### 17 Request:

```bash
curl -X GET \
  'http://localhost:3005/v1/organizations/{{organization-id}}/users/{{user-id}}/organization_roles' \
  -H 'Content-Type: application/json' \
  -H 'X-Auth-token: {{X-Auth-token}}'
```

#### Response:

The response returns the role of the given `<user-id>`

```json
{
    "organization_user": {
        "user_id": "5e482345-2c48-410e-ae03-203d67a43cea",
        "organization_id": "18deea43-e12a-4018-a45a-664c3158780d",
        "role": "member"
    }
}
```

### Remove a User from an Organization

Owners and Super-Admins can remove a user from and organization by making a delete request.

#### 18 Request:

```bash
curl -X DELETE \
  'http://localhost:3005/v1/organizations/{{organization-id}}/users/{{user-id}}/organization_roles/member' \
  -H 'Content-Type: application/json' \
  -H 'X-Auth-token: {{X-Auth-token}}'
```

### Creating Roles and Permissions

To save time, the data creating users and organizations from the
[previous tutorial](https://github.com/FIWARE/tutorials.Roles-Permissions) has been downloaded and is automatically
persisted to the MySQL database on start-up so the assigned UUIDs do not change and the data does not need to be entered
again.

The **Keyrock** MySQL database deals with all aspects of application security including storing users, password etc.;
defining access rights and dealing with OAuth2 authorization protocols. The complete database relationship diagram can
be found [here](https://fiware.github.io/tutorials.Securing-Access/img/keyrock-db.png)

To refresh your memory about how to create users and organizations and applications, you can log in at
`http://localhost:3005/idm` using the account `alice-the-admin@test.com` with a password of `test`.

![](https://fiware.github.io/tutorials.PEP-Proxy/img/keyrock-log-in.png)

and look around.



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
