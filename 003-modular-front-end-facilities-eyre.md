+++
title = "Modular Front End Facilities For Eyre"
authors = "~ponmep-litsem <mikolajp@protonmail.com>"
created = 2017-12-19
status = deprecated
+++

# Overview

Basic front-end facilities of Urbit such as the welcome page, login and related facilities, redirections, and client-side session management are currently hard-coded in Eyre.

We would like to propose decoupling of front-end and client side session management from Eyre, in order to make the development of Urbit interface possible without hacking the code of Eyre vane itself. 

# Specification

Currently, Eyre hides frond-end specific code inside its many arms. For example '++ login-page' contains both the html layout and the scripts for user authentication. One way to make Eyre independent of any particular front end implementation is to move the relevant pages to separate files under the "portal" (working name for Urbit front-end facilities) clay node. Thus the the login page would be under ```/web/portal/login``` Similarly, logout screen would be located under: ```/web/portal/logout``` The welcome screen would just be ```/web/portal```. And so on.

In this way, an alternative portal implementation could be easily implemented by either augmenting the files in `/web/portal` in the case of static page implementation. This has additional advantage of making it possible to have a Portal SAP, which would just reside under `/web/portal.js`.

The authentication related JS code should be moved into Urb.js, which already contains much of client-side authentication. In this way, the Eyre only becomes responsible for handling the authentication API, setting cookies. 

# Rationale

Currently Urbit's front-end implementation is scattered throughout different places. The session management front-end and code is located in the bowels of Eyre and also in urb.js Error page, and default welcome screen are on the other hand served from /web. In order to facilitate greater UI modularity, it would make sense to decouple Eyre from any presentation-related code, and have it only handle the default entry point. This results in greater flexibility, as UI is now located in one specific place and amenable to modifications/alternative implementations. 

# Integration Plan

Once the correct solution to the problem emerges, the integration would proceed as follows:

1. Moving all front-end related code into specified directory in web, into separate files. 
2. Moving authentication related code urb.js. Authentication with eyre should be handled exclusively via API endpoints.
3. Verification of new architecture by building a demo portal SAP with the same functionalities as the current static page implementation. 