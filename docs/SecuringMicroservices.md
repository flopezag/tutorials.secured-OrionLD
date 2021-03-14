# Securing Microservices

## Contents

<details>
<summary><strong>Details</strong></summary>

- [Securing Microservices with an Identity Management and a PEP Proxy](#securing-microservices-with-an-identity-management-and-a-pep-proxy)
  - [Introduction to the solution](#introduction-to-the-solution)
  - [Standard Concepts of Identity Management](#standard-concepts-of-identity-management)
  - [Video: Introduction to Keyrock](#arrow_forward-video-introduction-to-keyrock)
  - [Video: Introduction to Wilma PEP Proxy](#arrow_forward-video-introduction-to-wilma-pep-proxy)
  - [Prerequisites](#prerequisites)
    - [Docker and Docker Compose](#docker-and-docker-compose-)
    - [Cygwin (for Windows)](#cygwin-for-windows-)
    - [Postman](#postman-)
    - [http](#http-)
    - [jq](#jq-)
</details>

## Securing Microservices with an Identity Management and a PEP Proxy

### Introduction to the solution

> "Oh, it's quite simple. If you are a friend, you speak the password, and the doors will open."
>
> — Gandalf (The Fellowship of the Ring by J.R.R Tolkien)

The [previous tutorial](https://github.com/FIWARE/tutorials.Securing-Access) demonstrated that is possible to Permit
or Deny access to resources based on an authenticated user identifying themselves within an application. It is a
matter of the code following a different line of execution if the `access_token` was not found (Level 1 -
_Authentication Access_), or confirming that a given `access_token` had appropriate rights (Level 2 - _Basic
Authorization_). The same method of securing access can be applied by placing a Policy Enforcement Point (PEP) in front
of other services within a FIWARE-based Smart Solution.

A **PEP Proxy** lies in front of a secured resource and is an endpoint found at "well-known" public location. It serves
as a gatekeeper for resource access. Users or other actors must supply information to the **PEP Proxy** to allow
their request to succeed and pass through the **PEP proxy**. The **PEP proxy** then passes the request on to the real
location of the secured resource itself - the actual location of the secured resource is unknown to the outside
user - it could be held in a private network behind the **PEP proxy** or found on a different machine altogether.

FIWARE [Wilma](https://fiware-pep-proxy.rtfd.io/) is a simple implementation of a **PEP proxy** designed to work with
the FIWARE [Keyrock](https://fiware-idm.readthedocs.io/en/latest/) Generic Enabler. Whenever a user tries to gain access
to the resource behind the **PEP proxy**, the PEP will describe the user's attributes to the Policy Decision Point
(PDP), request a security decision, and enforce the decision. (Permit or Deny). There is minimal disruption of access
for authorized users - the response received is the same as if they had accessed the secured service directly.
Unauthorized users are answered with a **401 - Unauthorized** response.

### Standard Concepts of Identity Management

The following common objects are found with the **Keyrock** Identity Management database:

- **User**, any signed-up user able to identify themselves with an eMail and password. Users can be assigned rights
  individually or as a group.
- **Application**, any securable FIWARE application consisting of a series of microservices.
- **Organization**, a group of users who can be assigned a series of rights. Altering the rights of the organization
  effects the access of all users of that organization.
- **OrganizationRole**, users can either be members or admins of an organization - Admins are able to add and remove
  users from their organization, members merely gain the roles and permissions of an organization. This allows each
  organization to be responsible for their members and removes the need for a super-admin to administer all rights.
- **Role**, a role is a descriptive bucket for a set of permissions. A role can be assigned to either a single user
  or an organization. A signed-in user gains all the permissions from their own roles plus the roles associated to
  their organization.
- **Permission**, an ability to do something on a resource within the system.

Additionally, two further non-human application objects can be secured within a FIWARE application:

- **IoTAgent**, a proxy between IoT Sensors and the Context Broker.
- **PEPProxy**, a middleware for use between generic enablers challenging the rights of a user.

The relationship between the objects can be seen below - the entities marked in red are used directly within this
tutorial:

![Definition of entities](https://fiware.github.io/tutorials.PEP-Proxy/img/entities.png)

### :arrow_forward: Video: Introduction to Keyrock

[![Introduction to Keyrock - Identity Management](https://fiware.github.io/tutorials.Step-by-Step/img/video-logo.png)](https://www.youtube.com/watch?v=dHyVTan6bUY "Introduction")

Click on the image above to watch an introductory video

### :arrow_forward: Video: Introduction to Wilma PEP Proxy

[![Introduction to Wilma - PEP Proxy](https://fiware.github.io/tutorials.Step-by-Step/img/video-logo.png)](https://www.youtube.com/watch?v=8tGbUI18udM "Introduction")

Click on the image above to see an introductory video

### Prerequisites

#### Docker and Docker Compose <img src="https://www.docker.com/favicon.ico" align="left"  height="30" width="30">

To keep things simple both components will be run using [Docker](https://www.docker.com). **Docker** is a container
technology which allows to different components isolated into their respective environments.

- To install Docker on Windows follow the instructions [here](https://docs.docker.com/docker-for-windows/)
- To install Docker on Mac follow the instructions [here](https://docs.docker.com/docker-for-mac/)
- To install Docker on Linux follow the instructions [here](https://docs.docker.com/install/)

**Docker Compose** is a tool for defining and running multi-container Docker applications. A
[YAML file](https://raw.githubusercontent.com/Fiware/tutorials.Identity-Management/master/docker-compose.yml) is used
configure the required services for the application. This means all container services can be brought up in a single
command. Docker Compose is installed by default as part of Docker for Windows and Docker for Mac, however Linux users
will need to follow the instructions found [here](https://docs.docker.com/compose/install/)

You can check your current **Docker** and **Docker Compose** versions using the following commands:

```bash
docker-compose -v
docker version
```

Please ensure that you are using Docker version 18.03 or higher and Docker Compose 1.21 or higher and upgrade if
necessary.

#### Cygwin (for Windows) <img src="https://www.cygwin.com/favicon.ico" align="left"  height="30" width="30">

We will start up our services using a simple bash script. Windows users should download
[cygwin](http://www.cygwin.com/) to provide a command-line functionality similar to a Linux
distribution on Windows.

#### Postman <img src="https://www.postman.com/favicon-32x32.png" align="left"  height="30" width="30">

Postman is a collaboration platform for API development. Postman's features simplify each step of building an API and
streamline collaboration, therefore you can create better APIs—faster. To install Postman, follow the instructions
[here](https://www.postman.com/downloads).

#### http <img src="https://httpie.io/static/img/favicon-32x32.png" align="left" height="30" width="30">

This is a command line HTTP client, similar to curl or wget, with JSON support, syntax highlighting, persistent
sessions, and wget-like downloads with an expressive and intuitive syntax. `http` can be installed on each
operating system. Follow the instructions described [here](https://httpie.io/docs#installation).

#### jq <img src="https://stedolan.github.io/jq/jq.png" align="left" width="35" height="35">

This is a program to slice, filter and map the content of JSON data. This is a useful tool to extract certain
information automatically from the HTTP responses. `jq` is written in C with no dependencies and can be use
on nearly any platform. Prebuilt binaries are available for Linux, OS X and Windows. For more details how to install
the tool you can go [here](https://stedolan.github.io/jq/download).