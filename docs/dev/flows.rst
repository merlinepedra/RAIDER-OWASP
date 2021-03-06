.. _flows:

Flows
=====

:term:`Flows <Flow>` are the main concept in **Raider**, used to
define the HTTP information exchange. Each :term:`request` you want to
send needs its own Flow object. Inside the ``request`` attribute of
the object, needs to be a :class:`Request <raider.request.Request>`
object containing the definition of the request. This definition can
contain :class:`Plugins <raider.plugins.Plugin>` whose value will be
used when sending the HTTP request.

There are two types of Flow, the regular one using the :class:`Flow
<raider.flow.Flow>` class, and the authentication Flows using the
:class:`AuthFlow <raider.flow.AuthFlow>` class. Only difference is
that AuthFlow ones are treated as changing the authentication state
while the regular ones don't. Use AuthFlow to define the process
necessary to reach from unauthenticated state to authenticated
one. Use regular Flows for any other requests you want to test using
Raider.

.. automodule:: raider.flow
   :members:


Examples
--------

Create the variable ``initialization`` with the AuthFlow. It'll send a
GET request to ``https://example.com/admin/``. If the HTTP response
code is 200 go to next stage ``login``.

.. code-block:: hylang

    (setv initialization
          (AuthFlow
            :request (Request
                       :method "GET"
                       :url "https://example.com/admin/")
            :operations [(Http
                           :status 200
                           :action (NextStage "login"))]))
    

Define AuthFlow ``login``. It will send a POST request to
``https://example.com/admin/login`` with the username and the password
in the body. Extract the cookie ``PHPSESSID`` and store it in the
``session_id`` plugin. If server responds with HTTP 200 OK, print
``login successfully``, otherwise quit with the error message ``login
error``.

.. code-block:: hylang

    (setv username (Variable "username"))
    (setv password (Variable "password"))
    (setv session_id (Cookie "PHPSESSID"))

    (setv login
          (AuthFlow
            :request (Request
                       :method "POST"
                       :url "https://www.example.com/admin/login"
                       :data
                       {"password" password
                        "username" username})
            :outputs [session_id]
            :operations [(Http
                           :status 200
                           :action (Print "login successfully")
                           :otherwise (Error "login error"))]))
    		


Define another ``login`` Flow. Here what's different is the
``csrf_name`` and ``csrf_value`` plugins. In this application both the
name and the value of the token needs to be extracted, since they change
all the time. They were defined as :class:`Html <raider.plugins.Html>`
objects. Later they're being used in the body of the :class:`Request
<raider.request.Request>`.

If the HTTP response code is 200 means the :term:`MFA <Multi-factor
authentication (MFA)>` was enabled and the ``multi_factor`` :term:`stage`
needs to run next. Otherwise, try to log in again. Here the password
is asked from the user by a :class:`Prompt <raider.plugins.Prompt>`.

Also define the regular Flow named ``get_nickname`` to extract the
username of the logged in user. This request doesn't affect the
authentication state which is why Flow is used instead of AuthFlow.

.. code-block:: hylang

    ;; Gets `username` from active user's object defined in `users`.
    (setv username (Variable "username"))

    ;; Gets the password by manual input.
    (setv password (Prompt "password"))

    ;; Gets `PHPSESSID` from the cookie.
    (setv session_id (Cookie "PHPSESSID"))

    ;; Gets the OTP code by manual input.
    (setv mfa_code (Prompt "OTP code"))

    ;; Extract nickname from the HTML code. It looks for a tag like this:
    ;; <input id="nickname" value="admin">
    ;; and returns `admin`.
    (setv nickname
          (Html
            :name "nickname"
            :tag "input"
            :attributes
            {:id "nickname"}
            :extract "value"))

    ;; Extracts the name of the CSRF token from HTML code. It looks
    ;; for a tag similar to this:
    ;; <input name="0123456789" value="0123456789012345678901234567890123456789012345678901234567890123" type="hidden">
    ;; and returns 0123456789.
    (setv csrf_name
          (Html
            :name "csrf_name"
            :tag "input"
            :attributes
            {:name "^[0-9A-Fa-f]{10}$"
             :value "^[0-9A-Fa-f]{64}$"
             :type "hidden"}
            :extract "name"))

    ;; Extracts the value of the CSRF token from HTML code. It looks
    ;; for a tag similar to this:
    ;; <input name="0123456789" value="0123456789012345678901234567890123456789012345678901234567890123" type="hidden">
    ;; and returns 0123456789012345678901234567890123456789012345678901234567890123.    
    (setv csrf_value
          (Html
            :name "csrf_value"
            :tag "input"
            :attributes
            {:name "^[0-9A-Fa-f]{10}$"
             :value "^[0-9A-Fa-f]{64}$"
             :type "hidden"}
            :extract "value"))

    ;; Defines the `login` AuthFlow. Sends a POST request to
    ;; https://example.com/login.php. Use the username, password
    ;; and both the CSRF name and values in the POST body.
    ;; Extract the new CSRF values, and moves to the next stage
    ;; if HTTP response is 200.
    (setv login
          (AuthFlow
            :request (Request
                       :method "POST"
                       :url "https://example.com/login.php"
                       :cookies [session_id]
                       :data
                       {"password" password
                        "username" username
                        csrf_name csrf_value})
            :outputs [csrf_name csrf_value]
            :operations [(Http
                           :status 200
                           :action (NextStage "multi_factor")
                           :otherwise (NextStage "login"))]))

    ;; Defines the `multi_factor` AuthFlow. Sends a POST request to
    ;; https://example.com/login.php. Use the username, password,
    ;; CSRF values, and the MFA code in the POST body.
    (setv multi_factor
          (AuthFlow
            :request (Request
                       :method "POST"
                       :url "https://example.com/login.php"
                       :cookies [session_id]
                       :data
                       {"password" password
                        "username" username
			"otp" mfa_code
                        csrf_name csrf_value})
            :outputs [csrf_name csrf_value]))

    ;; Extracts the nickname and print it. Send a GET request to
    ;; https://example.com/settings.php and extract the nickname
    ;; from the HTML response.
    (setv get_nickname
          (Flow
            :request (Request
                       :method "GET"
                       :url "https://example.com/settings.php"
                       :cookies [session_id])
            :outputs [nickname]
	    :operations [(Print nickname)]))
		
