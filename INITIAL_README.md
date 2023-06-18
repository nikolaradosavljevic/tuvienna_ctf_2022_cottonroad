CottonRoad
============

Authors:
* Philipp Schoeller, 11907084, Author 1
* Florian Schauer, 12019869, Author 2
* Nikola Radosavljevic, 12222119, Author 3
* Teodor Janez Podobnik, 12206639, Author 4

Categories:<br>
* Server-Side Security: **SQLi**<br>
* Cryptography: **JWT**<br>
* Server-Side Security: **oAuth**<br>
* Server-Side Security: **Path Traversal**
* Server-Side Security: **SSRF**<br>

Repository structure
--------

* `checkers` directory - The checkers code that place and retrieve flags from the service. Additionally it checks whether the service is online and healthy.
* `dist` directory - The root of the repository that will be distributed to each participating team and can be used to patch the service. 
* `exploits` directory - The code to exploit the service vulnerabilities

How to Run
--------
To run the full architecture run:

    make start

Then you can test it, by running unittests inside the ´/webshop´ using:

    pipenv run python api_test.py

Overview
--------
The service is split up into two smaller services:
- The webshop
- The File Upload Server

A webshop where users can view/upload products. When entering the site one must create an account or login (via email + password OR oAuth) to see all available items and to interact with the store. In the store you can view, and query items that exist in the store. The file upload is managed by a separate server which requires a separate account that can be created in the web interface.

All Shop Features:
- View items
- Search for items
- Create an account
- Login to existing account
- Add a personal note in the private section of the profile (with specified character limit)
- View your own profile with personal notes
- Users can create up to 3 shop items
- A shop item has:
    - Name
    - Description

All File Upload Features:
- Create account (username, email, password) to upload files
- Login to account
- Image/TXT file upload; images are only accepted if they are png's and are within a specified maximum size
- Uploaded images will be used to display items (Item Name: filename, Item: the uploaded image)

We plan to use **MySQL**, **Python**, **HTML**, **CSS**, **Javascript** to build the webshop itself, and **Python** to build the apps listening on seperate ports, and **Linux Server** to host the webshop. The backend will be run by either **Flask or Django**, which connects to a **SQLite database**. The App will be containerized and deployed using **Docker** utilities.

### Flag Store 1 - Account Panel (within the personal notes)
The first flag store will be a personal note that one of the users has. The user who has this note is identified using the flag ID.

We plan to use flag ID’s in the following format: **username**

### Flag Store 2 - Flag file uploaded by designated user (stored in the file upload server)
The flag is stored in a file uploaded by a designated user. The goal of the players is to read its content through multiple ways.

We plan to use flag ID’s in the following format: **filename**

Vulnerabilities
---------------

### Flag Store 1, Vuln 1
SQLi to read the personal notes of all users.

The service will have user input sanitization at multiple locations, and one of the many queries will contain a **flaw in the sanitization of the user input**, therefore allowing a SQLi to be possible. Using this SQLi the player gets access to a table containing all the users and their personal notes. One specific user on the site will have the flag in their personal notes which can then be extracted and submitted.

* Difficulty: **easy**
* Discoverability: **easy**
* Patchability: **easy**

### Flag Store 1, Vuln 2
JWT Algorithm Confusion to login as a user by manipulating the JWT correctly.

JWT Algorithm Confusion occurs when an **attacker is able to change the signing algorithm of JWT**. Therefore it can be possible for the attacker to sign its own tokens without knowing the secret key from the server.  

* Difficulty: **medium**
* Discoverability: **medium**
* Patchability: **easy**

Our webserver will be configured to use RS256 as a signing algorithm. However, if the user manipulates the token and changes the algorithm (which is normally configured in the header) to HS256 the webserver will use the public key (as the symmetric key) to verify the token.

### Flag Store 1, Vuln 3
oAuth Login Bypass to login into an arbitrary account (username = flagid).

To implement this vulnerability we will introduce a sub-service, which represents a file-upload service to upload pictures for the webshop. This sub-service will serve as identity provider for the webshop, so it will be possible to login to the webshop using oAuth.
To create an account for the file upload server one needs to enter the following values:

- username (**NOT** unique)
- email (unique)
- password

When logging in using oAuth the webshop will only check that the oAuth-username matches the webshop-database username. Because the username does not need to be unique it will be possible to login into an aribitrary account.

* Difficulty: **medium**
* Discoverability: **hard**
* Patchability: **medium**

### Flag Store 2, Vuln 1
Path Traversal with faulty sanitization

Shop items (images) will be loaded by sending a GET-request to the Web Server endpoint, that retrieves a matching png-file from the File Upload Server (using a GET request):

- GET **<em>example.com/images?name=bike_sold.png</em>**                     &#8594; returns the raw png data of the requested image

- GET **<em>example.com/images?name=../../../etc/passwd</em>**               &#8594; returns some form of error

- GET **<em>example.com/images?name=../../../etc/passwd%00.png</em>**        &#8594; returns some form of error

- GET **<em>example.com/images?name=....//….//….//etc/passwd</em>**          &#8594; returns some form of error

- GET **<em>example.com/images?name=....//....//….//etc/passwd%00.png</em>** &#8594; returns the content of /etc/passwd

**Malicious user is able to create a path to an arbitrary file** and read the flag.

* Difficulty: **medium**
* Discoverability: **easy**
* Patchability: **easy**

### Flag Store 2, Vuln 2
SSRF with an open redirect to send custom GET-requests to the webserver containing arbitrary http-endpoints.

There are two vital parts to this vulnerability:<br>
* The Web Server, serving users
* An internal API, checking and returning availability of an item (the File Upload Server)

The store has a check stock feature, which returns the amount of items available to the user. The Web Server checks the item stock by polling the internal system not reachable by the user. A **malicious user is able to manipulate part of the URL the webserver sends requests to**, by sending custom GET-requests to the webserver containing arbitrary http-endpoints. However, since the domain of the stock checker is hardcoded, the attack has to be paired with an open redirect to be successfully exploited. Since the stock checker has valid credentials, it can be used to access files on the file upload server.

* Difficulty: **medium**
* Discoverability: **medium**
* Patchability: **medium**

Patches
-------

### Flag Store 1, Vuln 1
**Fixing the flawed SQL query** will prevent the player from gaining access to the table and finding the flag. 

### Flag Store 1, Vuln 2
To patch this vulnerability the **JWT header needs to be checked properly**, so that only RS256 is allowed.

### Flag Store 1, Vuln 3
To patch this vulnerability the username and the email address need to be checked. Only if both of them match, the user should be authenticated.

### Flag Store 2, Vuln 1
**Prevent path traversal by making sure '..' (current directory exit) is not included in the path** A proper sanitization of the incoming strings is also necessary to prevent exploitation.

### Flag Store 2, Vuln 2
A possible way to ensure security could be **sanitizing the redirect-to variable of incoming POST-requests**.

Work Packages
-------------

### WP 1, Basic Server- & Frontend Setup & dockerization

### WP 2, Implementing the Vulnerabilities

### WP 3, Implementing the Service-Checkers and Bots

### WP 4, Testing Service and Vulnerabilities

Timeline
--------

WP 1 (done by 02.12.2022)
* Nikola: **Setting up basic server functionality (setting up http listeners)** 
* Florian: **Frontend Web Design of the Shop**
* Philipp: **Create a database with corresponding tables**
* Teodor: **Dockerization**

WP 2 (done by 12.12.2022)
* Philipp: **SQLi**
* Teodor: **Path Traversal**
* Teodor & Nikola: **SSRF with an open redirect**
* Florian: **JWT Algorithm Confusion**
* Florian & Philipp: **oAuth Login Bypass**

WP 3 (done by 27.12.2022)
* All: **Extending Vulnerabilities with a Checker (dispatching the flag + app functionality testing)**

WP 4 (done by 02.01.2023)
* All: **Demonstrating (automated script) exploit and patch of the vulnerability**
