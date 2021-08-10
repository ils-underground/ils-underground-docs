Python -- Using the Sierra REST API
===================================

.. contents::

Getting Started
---------------

There are many different ways in Python to use the Sierra API. One popular way is to use the 
`Requests module <https://docs.python-requests.org/en/master/index.html>`__ (Note: Requests is a 3rd party module which can be installed via pip.). 
Many of the examples below use Requests as the HTTP client.

General Concepts
~~~~~~~~~~~~~~~~

Some of the general concepts for using the REST API revolve around understanding some HTTP concepts. **Mozilla** has an excellent resource on understanding HTTP that can be found here:
`<https://developer.mozilla.org/en-US/docs/Web/HTTP>`__


Working with `HTTP request methods <https://developer.mozilla.org/en-US/docs/Web/HTTP/Methods>`__ 
__________________________________________________________________________________________________

Also called *HTTP verbs*, **request methods** and their intended purposes / descriptions can be found below. The definitions provided here are from the Mozilla.org resource.

GET 
    The HTTP ``GET`` method requests a representation of the specified resource. Requests using GET should only be used to request data 
    
    For example: ``GET`` a list of patrons, bibs etc.

    Example GET request using Python and Requests module:

    .. code-block:: python3
    
       import requests
   
       # the API provides an HTML information summary at the root of the API app ...
       r = requests.get('https://sierra-test.cincinnatilibrary.org:443/iii/sierra-api/')
       print(
           r.status_code, 
           r.text, 
           sep='\n\n'
       )

    the output from the example 
    
    .. code-block:: text
    
       200
   
       <html>
             <body>
               Sierra Rest API<br />
               Version: 6.1.0<br />
               SierraVersion: 5.2<br />
               FullVersion: 5.2_6.1.0<br />
               Revision: 5dfa92fe334be7740a16f70046e473eab5e83745<br />
               RevisionDate:   Thu Jul 23 12:45:53 2020 +0100<br />
               Backwards Compatible: 4.0, 4.1, 4.2, 5.0, 5.1, 5.2, 5.3, 5.4, 5.5, 6.0<br />
             </body>
           </html>

POST
    The HTTP ``POST`` method sends data to the server. The type of the body of the request is indicated by the Content-Type header.
    
    For example: ``POST`` or *create* a bib record given a block of data--representing the properties of the bib record.

    Example POST request using Python and Requests module:
    
    .. code-block:: python3

       # use the `patrons/find` endpoint to "Find a patron by varfield fieldTag ad varField content"
   
       # this refreshes the token, and creates the header we need to send in the request
       # see the "Generating the Access Token" portion of these docs to see this function
       headers = set_access_headers()
       
       # this will set the query string parameters in the URI
       # in this example, we're going to be searching the varfield 'b' barcode for an alt-id value
       params = {
           'varFieldTag': 'b',
           'varFieldContent': 'hsimpson1',
           'fields': ','
       }
       
       # get the response as r
       r = requests.get(base_url + 'patrons/find', 
                        headers=headers, 
                        params=params,
                        verify=True
       )          
       
       # use the python list comprehension to get any field tag 'n' (name) values in the varfields list
       print([varfield['content'] for varfield in r.json()['varFields'] if varfield['fieldTag'] == 'n' ])

    the output from the example

    .. code-block:: text

       # ['Simpson, Homer J']

PUT
    The HTTP ``PUT`` request method creates a new resource or replaces a representation of the target resource with the request payload.

    For example: ``PUT`` or *update* the contents of a bib record given a specified record id number

DELETE
    The HTTP ``DELETE`` request method deletes the specified resource.

    For example: ``DELETE`` a bib record given a specified record id number

HTTP Responses
______________

After sending a request method (like the ones described above), your client should receive a **response** from the server.

The response consists of several pieces of metadata about the response, including a **status code** indicating if the request was successful, or failed.

If the response contains data (like a list of bib record ID numbers for example), that data will be contained in the response **body**

The most common response will be ``200``, indicating a successful request

`200 OK <https://developer.mozilla.org/en-US/docs/Web/HTTP/Status/200>`__
    The request has succeeded. The meaning of the success depends on the HTTP method:

    GET
        The resource has been fetched and is transmitted in the message body.
    
    PUT or POST
        The resource describing the result of the action is transmitted in the message body.



Generating the Access Token
---------------------------

In order to use most of the the Sierra API endpoints, you must provide them with a valid 
**access token** in the `HTTP header 
<https://developer.mozilla.org/en-US/docs/Web/HTTP/Headers>`__ (as *authorization* type *bearer* ) in the request. 

The **access token** is generate by another Sierra REST API ``/token`` endpoint which accepts the `HTTP Authorization <https://developer.mozilla.org/en-US/docs/Web/HTTP/Headers/Authorization>`__ 
request header parameter value which consists of a base64 encoded string formed from concatenated values of your **client key** and **client secret**.

If that all sounds confusing, that's because it is!

Below is a short example using the  `Requests module <https://docs.python-requests.org/en/master/index.html>`__
and Python. 

.. code-block:: python3

    import requests
    from base64 import b64encode
    
    # see the Sierra docs for generating these values for your system
    client_key = 'YOUR_CLIENT_KEY'
    client_secret = 'YOUR_CLIENT_SECRET'
    
    # this is the location of your system's API (change it to match your Sierra system)
    base_url = "https://sierra-test.cincinnatilibrary.org:443/iii/sierra-api/v6/"
    
    auth_string = b64encode(
        (client_key + ':' + client_secret).encode('utf-8')
    ).decode('utf-8')
    
    def set_access_headers():
        """
        use this function to set and refresh the access_headers for future
        authorizing API requests 
        """
    
        headers = {}
        headers['authorization'] = 'basic ' + auth_string
    
        try:
            r = requests.post(base_url + 'token', headers=headers, verify=True)
    
        except requests.ConnectionError as e:
            print('connection error: {}'.format(e))
            return 0
    
        if r.status_code != 200:
            print(r.status_code)
            return 0
    
        access_token = r.json()['access_token']
    
        # set our headers to use the access token
        headers['authorization'] = 'bearer ' + access_token
        
        # Note: depending on the Sierra REST API request endpoint, 
        # you may need to change the types below to fit the request,
        # but these are pretty standard
        headers['content-type'] = 'application/json'
        headers['accept'] = 'application/json'
    
        return headers

Note: this the token in the header as returned from this function is valid for 1 hour (3600 seconds), so you'll need to refresh it if your application needs do do future actions past that amount of time.

Below is an example that uses the **access token** to check the **access token**:

.. code-block:: python3
    
    # this refreshes the token, and creates the header we need to send in the request
    headers = set_access_headers()
    
    # get the response as r
    r = requests.get(base_url + 'info/token', headers=headers, verify=True)          
    
    # get the how many seconds are left before the token expires ...
    print(r.json()['expiresIn'])
    # 3599