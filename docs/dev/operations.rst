.. _operations:

.. module:: raider.operations

Operations
==========

*Raider* operations are pieces of code that will be executed when the
HTTP response is received. The most important one is **NextStage**
which controls the authentication flow. But anything can be done with
the operations, and *Raider* allows writing custom ones in hylang to
enable users to add functionality that isn't supported by the main
code.

NextStage
---------

Inside the Authentication object NextStage is used to define the
next step of the authentication process. It can also be used inside
"action" attributes of the other Operations to allow conditional
decision making.

.. code-block:: hylang

   (NextStage "login")

.. autoclass:: NextStage

Print
-----

When this Operation is executed, it will print each of its elements
in a new line.

.. code-block:: hylang

   (Print
     "This will be printed first"
     access_token
     "This will be printed on the third line")

.. autoclass:: Print

Error
-----

Operation that will exit Raider and print the error message.

.. code-block:: hylang

   (Error "Login failed.")

.. autoclass:: Error

Http
----

.. code-block:: hylang

   (Http
      :status 200
      :action
        (NextStage "login")
      :otherwise
        (NextStage "multi_factor"))

.. autoclass:: Http

Grep
----

.. code-block:: hylang

   (Grep
     :regex "TWO_FA_REQUIRED"
     :action
       (NextStage "multi_factor")
     :otherwise
       (Print "Logged in successfully"))

.. autoclass:: Grep       


Writing custom operations
-------------------------

In case the existing operations are not enough, the user can write
their own to add the new functionality. Those new operations should be
written in the project's configuration directory in a ".hy" file. To
do this, a new class has to be defined, which will inherit from
*Raider*'s Operation class:

.. autoclass:: Operation

		