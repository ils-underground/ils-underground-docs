Python & the Sierra REST API
============================

Generating the Access Token
---------------------------

In order to use most of the the Sierra API endpoints, you must provide them with a valid 
**access token** in the `HTTP header 
<https://developer.mozilla.org/en-US/docs/Web/HTTP/Headers>`__ (as *authorization* type *bearer* ) in the request. 

The **access token** is generate by another Sierra REST API ``/token`` endpoint which accepts the `HTTP Authorization <https://developer.mozilla.org/en-US/docs/Web/HTTP/Headers/Authorization>`__ 
request header parameter value which consists of a base64 encoded string formed from concatenated values of your **client key** and **client secret**.

If that all sounds confusing, that's because it is!

Below is a short example using the `Requests module <https://docs.python-requests.org/en/master/index.html>`__ 
and Python. (Note: Requests is a 3rd party module which can be installed via pip.)

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