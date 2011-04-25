.. include:: <s5defs.txt>

Pyramid Views
=============

:Authors: Chris McDonough, Agendaless Consulting
:Date: 4/29/2011 (Pylons Minicon, SF)

..  footer:: Chris McDonough, Agendaless Consulting

View Callables
--------------

A view callable is:

- Any Python object with a ``__call__`` method that accepts a Request object
  and returns a Response object.

- *or* a class that accepts a Request object as an ``__init__`` argument and
  which possesses a different method which accepts no arguments; that method
  returns a Response object.

A View Callable as a Function
-----------------------------

Here's a simple view callable (``aview``) implemented as a function:

.. sourcecode:: python

   from pyramid.response import Response

   def aview(request):
       return Response('OK')

A View Callable as an Instance
------------------------------

Here's a simple view callable (``aview``) implemented as an instance:

.. sourcecode:: python

   from pyramid.response import Response

   class AViewFactory(object):
       def __call__(self, request):
           return Response('OK')

   aview = AViewFactory()

A View Callable as a Class
--------------------------

Here's a simple view callable (``aview``) implemented as a class:

.. sourcecode:: python

   from pyramid.response import Response

   class aview(object):
       def __init__(self, request):
           self.request = request

       def __call__(self):
           return Response('OK')

View Class (Multiple Methods)
-----------------------------

Any method of a view class can return a response:

.. sourcecode:: python

   class aview(object):
       def __init__(self, request):
           self.request = request
       def ok(self):
           return Response('OK')
       def notok(self):
           return Response('Not OK')

View Class (Multiple Methods)
-----------------------------

- Individual response-returning methods of a class used as a view callable need
  to be registered separately.

- In the previous slide, the ``ok`` and ``notok`` methods will need to be
  registered separately to be eligible for view execution.

View Declaration!=Registration
------------------------------

- Merely creating a view callable or a view class does not make it eligible
  to be considered for view execution.

- You need to *register* a view callable to make it eligible for view
  execution.

- View registration can be done imperatively or by invoking a ``scan``.

View Registration (Imperative)
------------------------------

.. sourcecode:: python

   from pyramid.config import Configurator
   from pyramid.response import Response

   def aview(request):
       return Response('OK')

   if __name__ == '__main__':
      config = Configurator()
      config.add_route('home', '/')
      config.add_view(aview, route_name='home')

View Registration (Imperative)
------------------------------

- View callables are registered imperatively using the ``add_view`` method of
  a Configurator object named ``config``.

- The previous script registers the ``__main__.aview`` view callable
  as a view.

- This view will be found when the ``PATH_INFO`` portion of the request URL is
  ``/``.

- The ``route_name`` argument passed to ``add_view`` refers to the name
  of the single route added via ``add_route`` named ``home``.

View Class (Multiple Methods)
-----------------------------

.. sourcecode:: python

   class aview(object):
       def __init__(self, request):
           self.request = request
       def ok(self): return Response('OK')
       def notok(self): return Response('Not OK')
   if __name__ == '__main__':
       # ... 
       config.add_route('ok', '/ok')
       config.add_route('notok', '/notok')
       config.add_view(aview,route_name='ok',attr='ok')
       config.add_view(aview,route_name='notok',
                       attr='notok')

View Class (Multiple Methods)
-----------------------------

- When the view is a class and the ``attr`` argument is *not* passed to
  ``add_view``, the ``__call__`` method of the class is assumed to be
  something that returns a response.

- When ``attr`` *is*, passed, however, it names the method of the
  class you like to behave as the response generating callable.

Registering A View Declaratively
--------------------------------

.. sourcecode:: python

   from pyramid.view import view_config
   from pyramid.response import Response

   @view_config(route_name='home')
   def aview(request):
       return Response('OK')

   if __name__ == '__main__':
       # ...
       config.add_route('home', '/')
       config.scan('__main__')

Registering a View Declaratively
--------------------------------

- In the previous example, we changed the example to use ``view_config``
  decorator.  Instead of calling ``add_view`` for each view callable
  imperatively, a view callable can be marked up with a ``view_config``
  decorator.

- ``view_config`` - marked view callables and classes are registered
  as the result of a ``scan``.

- The scanning process calls ``add_view`` when it finds a marked-up view
  callable.  ``@view_config`` + ``scan`` == ``add_view``.

Registering a View Declaratively
---------------------------------

- A scan will consider the declarations made in the package it scans and any
  subpackages recursively.

- To do so, it imports each module it finds in the package as necessary, and
  will do the same recursively for subpackages.

Class Registered Declaratively
------------------------------

.. sourcecode:: python

   @view_config(route_name='home')
   class aview(object):
       def __init__(self, request):
           self.request = request
       def __call__(self):
           return Response('OK')

   if __name__ == '__main__':
       # ...
       config.add_route('home', '/')
       config.scan('__main__')

Method Registered Declaratively
-------------------------------

.. sourcecode:: python

   class aview(object):
       def __init__(self, request):
           self.request = request
       @view_config(route_name='ok')
       def ok(self): return Response('OK')
       @view_config(route_name='notok')
       def notok(self): return Response('Not OK')
   if __name__ == '__main__':
       # ...
       config.add_route('ok', '/ok')
       config.add_route('notok', '/not_ok')
       config.scan('__main__')

Impact of view_config Decorator
-------------------------------

- ``view_config`` *only* marks view callables as registerable during a scan.
  It is a passthrough that just marks the callable so a later scan
  can find it.

- A derivation of the view callable is registered during a scan.

- When you import the decorated callable for testing purposes or other
  direct interaction, it won't behave any differently than it would if the
  decorator was not attached.

View Configuration
------------------

- View configuration is the set of arguments passed to ``add_view`` or
  ``view_config``. 

- There are many arguments.  The more arguments used, the more specific the
  scenario must be in order for the view to be executed.

- We've seen one named ``route_name``.  This argument associates the view
  with a *route* defined by ``add_route``.

View Lookup and Execution
-------------------------

Pyramid responds to a request by:

- Looking up a view.

- Executing the view that is found via view lookup.

- The Pyramid "router" orchestrates view lookup and execution.

- The router is invoked when a WSGI request enters the application.

- Developers usually don't interact with the router, unless they're creating
  a functional test.

Router Responsibilities
-----------------------

The router (among other things) is responsible for these steps:

- Create a request.

- Look for a matching route.

- Traverse to find a context.

- Using the context and the request, find a view callable.

- Invoke the view callable.

Router Timeline
---------------

- The router always has a global root factory and a request factory.

- A new Request object is created using the request factory.

Router Timeline (2)
-------------------

- We look for a matching route by iterating over the existing routes in
  insertion order.

- Some routes have predicates.  The first route in insertion
  order that matches all predicates "wins".

- If we find a matching route, we find a request interface associated with
  the matched route.

- If the matched route has a root factory, we use that instead of the global
  root factory.

Router Timeline (3)
-------------------

- A root object is created using the root factory.

- We obtain a context and a view name via traversal using the root object,
  even if a route matched.  This is how "hybrid" urldispatch/traversal is
  accomplished.

- We figure out what interfaces are provided by the context.

- We look up a view based on the request interface and the context interface.

Router Timeline (4)
-------------------

- We call the found view using the context and the request; it will return a
  Response.

- The view might be a "multiview", in which case predicates are checked to
  find "the best" concrete view that is part of the multiview.

- A multiview is a group of views that share the same
  context-type/request-type/view-name triad.  Each concrete view that is part
  of a multiview differs only by the predicates it possesses.
