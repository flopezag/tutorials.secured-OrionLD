# Architecture

## Contents

<details>
<summary><strong>Details</strong></summary>

- [Architecture](#architecture)
  - [Start Up](#start-up)
  - [Dramatis Personae](#dramatis-personae)
  - [Logging In to Keyrock using the REST API - Getting admin token](#logging-in-to-keyrock-using-the-rest-api---getting-admin-token)

</details>

## Architecture

This application protects access to the existing Stock Management and Sensors-based application by adding PEP Proxy
instances around the services created in previous tutorials and uses data pre-populated into the **MySQL** database
used by **Keyrock**. It will make use of four FIWARE components - the
[Orion Context Broker](https://fiware-orion.readthedocs.io/en/latest/), the
[Keyrock](https://fiware-idm.readthedocs.io/en/latest/) Generic enabler and adds one or two instances
[Wilma](https://fiware-pep-proxy.rtfd.io/) PEP Proxy dependent upon which interfaces are to be secured.

The Orion-LD Context Broker rely on open source [MongoDB](https://www.mongodb.com/) technology to keep persistence
of the information they hold. **Keyrock** uses its own [MySQL](https://www.mysql.com/) database. The architecture
consists of the following elements:

- The FIWARE [Orion Context Broker](https://fiware-orion.readthedocs.io/en/latest/) which will receive requests using
  [NGSI-LD](https://forge.etsi.org/swagger/ui/?url=https://forge.etsi.org/gitlab/NGSI-LD/NGSI-LD/raw/master/spec/updated/full_api.json).
- FIWARE [Keyrock](https://fiware-idm.readthedocs.io/en/latest/) offer a complement Identity Management System
  including:
  - An OAuth2 authentication system for Applications and Users.
  - A site graphical frontend for Identity Management Administration.
  - A REST API for Identity Management via HTTP requests.
- FIWARE [Wilma](https://fiware-pep-proxy.rtfd.io/) is a PEP Proxy securing access to the **Orion** microservice.
- The underlying [MongoDB](https://www.mongodb.com/) database:
  - Used by the **Orion-LD Context Broker** to hold context data information such as data entities, subscriptions and
    registrations.
  - Used by the **IoT Agent** to hold device information such as device URLs and Keys.
- A [MySQL](https://www.mysql.com/) database:
  - Used to persist user identities, applications, roles and permissions.

Since all interactions between the elements are initiated by HTTP requests, the entities can be containerized and run
from exposed ports.

The specific architecture of each section of the tutorial is discussed below.

### Start Up

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

### Dramatis Personae

The following people at `test.com` legitimately have accounts within the Application

- Alice, she will be the Administrator of the **Identity Management** Application. The account is created in the
  initialization process of the Identity Management.
- Bob, administrator of the application, he has access to read and write the Personal Data store in the application.
- Charlie, he is an application's user. He needs to read the Personal Data of the users but cannot modify them.

The following people at `example.com` have signed up for accounts, but have no reason to be granted access
to the data

- Eve - Eve the Eavesdropper.
- Mallory - Mallory the malicious attacker.

The following people at `xyz.foo` have signed up for accounts and can access to their Personal Data for reading
and writing only:

- Ole.
- Torsten.
- Frank.
- Lothar.

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
| Managers   | Project Managers of the Personal Data application with full control access.          |
| Users      | Project Users of the Personal Data application with read control access.             |
| Data       | Personal Data owners who can read and modify only their own data.                    |
| Others     | Rest of IdM registered users not authorized to access the Personal Data Application. |

One application, with appropriate roles and permissions has also been created:

| Key           | Value                                  |
| ------------- | -------------------------------------- |
| Client ID     | `tutorial-dckr-site-0000-xpresswebapp` |
| Client Secret | `tutorial-dckr-site-0000-clientsecret` |
| URL           | `http://localhost:3000`                |
| RedirectURL   | `http://localhost:3000/login`          |

### Logging In to Keyrock using the REST API - Getting admin token

Enter a username and password to enter the application. The default user has the values `alice-the-admin@test.com`
and `test`. The following example logs in using the Admin User, if you want to obtain the corresponding tokens for
the other users after their creation just change the proper name and password data in this request:

#### :one: Request

```console
http POST http://localhost:3005/v1/auth/tokens \
  name=alice-the-admin@test.com \
  password=test
```

#### :one: Response

The response header returns an `X-Subject-token` which identifies who has logged on the application. This token is
required in all subsequent requests to gain access

```bash
HTTP/1.1 201 Created
Cache-Control: no-cache, private, no-store, must-revalidate, max-stale=0, post-check=0, pre-check=0
Connection: keep-alive
Content-Length: 138
Content-Security-Policy: default-src 'self' img-src 'self' data:;script-src 'self' 'unsafe-inline'; ...
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
