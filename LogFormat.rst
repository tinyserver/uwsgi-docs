Formatting uWSGI requests logs (from 1.3-dev)
====================================

Starting from 1.3-dev uWSGI exports a --logformat option allowing you to build custom request loglines

The syntax is simple:

.. code-block:: ini

   [uwsgi]
   logformat = i am a logline reporting "%(method) %(uri) %(proto)" returning with status %(status) 

All of the %() marked variables are substitued using specific rules.

Currently 3 kind of logvars are defined:

offsetof
********

they are taken blindly from the wsgi_request structure of the current request

.. code-block:: sh

   %(uri) -> REQUEST_URI
   %(method) -> REQUEST_METHOD
   %(user) -> REMOTE_USER
   %(addr) -> REMOTE_ADDR
   %(host) -> HTTP_HOST
   %(proto) -> SERVER_PROTOCOL

and from 1.4.5

.. code-block:: sh

   %(uagent) -> HTTP_USER_AGENT
   %(referer) -> HTTP_REFERER



functions
*********

they are simple functions called for generating the logvar value

.. code-block:: sh

   %(status) -> HTTP response status code
   %(micros) -> response time in microseconds
   %(msecs) -> respone time in milliseconds
   %(time) -> timestamp of the start of the request
   %(ctime) -> ctime of the start of the request
   %(epoch) -> the current time in unix format

and from 1.4.5

.. code-block:: sh

   %(size) -> response body size + response headers size
   %(ltime) -> human-formatted (apache style) request time
   %(hsize) -> response headers size
   %(rsize) -> response body size
   %(cl) -> request content body size

and from 1.4.6

.. code-block:: sh

   %(pid) -> pid of the worker handling the request
   %(wid) -> id of the worker handling the request
   %(switches) -> number of async switches
   %(vars) -> number of CGI vars in the request
   %(headers) -> number of generated response headers
   %(core) -> the core running the request
   %(vsz) -> address space/virtual memory usage (in bytes)
   %(rss) -> RSS memory usage (in bytes)
   %(vszM) -> address space/virtual memory usage (in megabytes)
   %(rssM) -> RSS memory usage (in megabytes)
   %(pktsize) -> size of the internal request uwsgi packet
   %(modifier1) -> modifier1 of the request
   %(modifier2) -> modifier2 of the request

user-defined logvars
********************

you can define logvars directly from your code (they are freed after each request)

.. code-block:: python

   import uwsgi
   def application(env, start_response):
       uwsgi.set_logvar('foo', 'bar')
       # returns 'bar'
       print uwsgi.get_logvar('foo')
       uwsgi.set_logvar('worker_id', str(uwsgi.worker_id()))
       ...

if you set a logformat like that


.. code-block:: sh

   uwsgi --logformat "worker id = %(worker_id) for request \"%(method) %(uri) %(proto)\" test = %(foo)"

you will be able to access code-defined logvars

Apache style combined request logging
*************************************

If you want to generate apache-compatible logs, just apply what you have learnt

.. code-block:: ini

   [uwsgi]
   ...
   log-format = %(addr) - %(user) [%(ltime)] "%(method) %(uri) %(proto)" %(status) %(size) "%(referer)" "%(uagent)"
   ...


Hacking logformat
*****************

If you want to add more c-based variables, open logging.c and go to the end of the file.

Adding vars is really easy

.. code-block:: c

                if (!uwsgi_strncmp(ptr, len, "uri", 3)) {
                        logchunk->pos = offsetof(struct wsgi_request, uri);
                        logchunk->pos_len = offsetof(struct wsgi_request, uri_len);
                }
                else if (!uwsgi_strncmp(ptr, len, "method", 6)) {
                        logchunk->pos = offsetof(struct wsgi_request, method);
                        logchunk->pos_len = offsetof(struct wsgi_request, method_len);
                }
                else if (!uwsgi_strncmp(ptr, len, "user", 4)) {
                        logchunk->pos = offsetof(struct wsgi_request, remote_user);
                        logchunk->pos_len = offsetof(struct wsgi_request, remote_user_len);
                }
                else if (!uwsgi_strncmp(ptr, len, "addr", 4)) {
                        logchunk->pos = offsetof(struct wsgi_request, remote_addr);
                        logchunk->pos_len = offsetof(struct wsgi_request, remote_addr_len);
                }
                else if (!uwsgi_strncmp(ptr, len, "host", 4)) {
                        logchunk->pos = offsetof(struct wsgi_request, host);
                        logchunk->pos_len = offsetof(struct wsgi_request, host_len);
                }
                else if (!uwsgi_strncmp(ptr, len, "proto", 5)) {
                        logchunk->pos = offsetof(struct wsgi_request, protocol);
                        logchunk->pos_len = offsetof(struct wsgi_request, protocol_len);
                }
                else if (!uwsgi_strncmp(ptr, len, "status", 6)) {
                        logchunk->type = 3;
                        logchunk->func = uwsgi_lf_status;
                        logchunk->free = 1;
                }


for function-based vars the prototype is

.. code-block:: c

   ssize_t uwsgi_lf_foobar(struct wsgi_request *wsgi_req, char **buf);

where buf is the destination buffer for the logvar value (this will be automatically freed if you set logchunk->free as in the "status" related c-code previously reported)

.. code-block:: c

   ssize_t uwsgi_lf_status(struct wsgi_request *wsgi_req, char **buf) {
           *buf = uwsgi_num2str(wsgi_req->status);
           return strlen(*buf);
   }
