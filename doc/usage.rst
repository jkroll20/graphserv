GraphServ: Usage
================

The Graph Processor project aims to develop an infrastructure for rapidly analyzing and evaluating Wikipedia's category structure. The `GraphCore <https://github.com/jkroll20/graphcore/>`_ component maintains and processes large directed graphs in memory. `GraphServ <https://github.com/jkroll20/graphserv/>`_ handles access to running GraphCore instances.

This file documents GraphServ usage.

Users can connect to GraphServ via plaintext-TCP or HTTP. GraphServ multiplexes data to and from GraphServ instances and users. Several users can simultaneously execute commands on the same GraphServ instance, or on the same core. 


GraphServ Commands
------------------

GraphServ accepts commands and data in a line-based command language. Its command interface follows the same basic syntax and principles as is used for GraphCore commands (see `GraphCore spec <https://github.com/jkroll20/graphcore/blob/master/spec.rst>`_).

The following is the list of GraphServ commands. Words in square brackets denote access level (see below).

create-graph [admin] ::

	create-graph GRAPHNAME
	create a named graphcore instance.
	graph names may contain only alphabetic characters (a-z A-Z), digits (0-9), hyphens (-) and underscores (_). 
	graph names must start with an alphabetic character, a hyphen or an underscore.

use-graph [read] ::

	use-graph GRAPHNAME
	connect to a named graphcore instance.

authorize [read] ::

	authorize AUTHORITY CREDENTIALS
	authorize with the named authority using the given credentials.
	an authority named 'password' is implemented, which takes credentials of the form user:password.

help [read] ::

	help [COMMAND]
	get help on server and core commands.

drop-graph [admin] ::

	drop-graph GRAPHNAME
	drop a named graphcore instance immediately (terminate the process).

list-graphs [read] ::

	list-graphs
	list currently running graphcore instances.

session-info [read] ::

	session-info
	returns information on your current session.

server-stats [read] ::

	server-stats
	returns information on the server.

protocol-version [read] ::

	protocol-version
	the protocol-version is used to check for compatibility of the server and core binaries.
	this command prints the protocol-version of the server.


Access Control
--------------

Server and core commands are divided into three access levels: *read*, *write* and *admin*. To execute a command, a session's access level must be equal to or higher than the command's access level.

GraphCore commands which modify a graph or its meta variables require *write* access. The *shutdown* command requires *admin* access. 

On connection, GraphServ assigns *read* access to a session. Access levels of a session can be elevated by using the *authorize* command, which tries to authorize with the given authority and credentials. 

Password Authority
++++++++++++++++++

The *password* authority implements access control using a htpassword-file and corresponding unix-style group file. A user authenticates by running the command *authorize password username:password*.

The **htpassword file** contains entries of the form *user:password-hash* and can be created and modified with the `htpasswd <http://httpd.apache.org/docs/2.0/programs/htpasswd.html>`_ tool. GraphServ supports passwords hashed with crypt() (htpasswd -d).

The **group file** contains entries of the form *access_level:password:GID:user_list*. The password and GID fields are ignored, and can be blank. *access_level* must be one of read, write, or admin. *user_list* is a comma-separated list of usernames.

The password authority reads the contents of these files on demand. If one of the files is changed while the server is running, it will be reloaded once a user runs *authorize*. 

If a user name appears in more than one access level, the highest level will be used.

Example htpasswd and group .conf files are included in the repository. All users in the example files use the password 'test'.


TCP Connections
---------------

Users can connect to GraphServ over TCP to execute commands. This example uses `netcat <http://netcat.sourceforge.net/>`_: ::

	$ nc localhost 6666
	help
	OK. available commands:
	# create-graph GRAPHNAME
	# use-graph GRAPHNAME
	# authorize AUTHORITY CREDENTIALS
	# help
	# drop-graph GRAPHNAME
	# list-graphs
	# session-info
	# server-stats




HTTP Connections
----------------

GraphServ contains a rudimentary HTTP Server which implements a subset of `HTTP/1.0 <http://www.w3.org/Protocols/rfc1945/rfc1945>`_. The HTTP Server accepts GET requests. One command can be executed per request. The server will close the connection after responding to the request. 

As a convenience, an HTTP/1.1 version string will also be accepted in GET requests. The version string in the GET request does not change the behaviour of the server or the contents of the response.

In principle, an HTTP client can execute any core or server command. However, because the client is disconnected after executing the first command, an HTTP client can never execute a command which needs an access level above *read*. Also, HTTP clients cannot execute any command which takes a data set. These limitations could be removed in the future by implementing Keep-Alive connections (the default in HTTP/1.1), and/or POST.

The request must follow the form *GET Request-URI Version-String CRLF <header fields> CRLF*. Any header fields following the Request-Line are read and discarded.

The Request-URI can include `percent-encoded <http://en.wikipedia.org/wiki/Percent-encoding>`_ characters. Any '+' characters in the Request-URI will be translated to space (0x20).


Executing Server Commands
+++++++++++++++++++++++++

To execute a server command, simply include the command string in the Request-URI. Example: ::

	$ curl http://localhost:8090/help	# use curl to print help text of GraphServ on localhost listening on the default port.
	GET /help HTTP/1.0			# corresponding Request-Line.

Executing Core Commands
+++++++++++++++++++++++

To send a command to a core, include its name in the Request-URI. Separate core name and command by a forward slash. Example: ::
	
	$ curl http://localhost:8090/core0/list-predecessors+7	# print direct predecessors of node 7 in core0 on localhost.
	GET /core0/list-predecessors+7 HTTP/1.1			# corresponding Request-Line.

HTTP Response and Status Code
+++++++++++++++++++++++++++++

The HTTP server translates Graph Processor command status codes to HTTP Status-Codes in the following way: ::

	Success ('OK.') 				200 OK
	Meta variable query succeeded ('VALUE:')	222 Value
	Failure, graph did not change ('FAILED!') 	400 Bad Request
	Error, graph may have changed ('ERROR!')	500 Internal Server Error
	Success with empty result set ('NONE.')		404 Not Found
	Command not found (special case for HTTP)	501 Not Implemented
	Access Denied ('DENIED!')			401 Not Authorized

Additionally, the untranslated status line is included in the *X-GraphProcessor:* header field of the HTTP response. 

The message-body of the response consists of the status line followed by any result data records or other command output.


| 
| 
| `GraphServ, GraphCore (C) 2011, 2012 Wikimedia Deutschland, written by Johannes Kroll <jkroll at lavabit com>.`
| `Last update to this text: 2012/01/05`


