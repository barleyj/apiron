##############
Advanced usage
##############

`httpbin.org <https://httpbin.org>`_ is a great testing tool
for various situations you may run into when interacting with RESTful services.
Let's define a :class:`Service <apiron.service.base.Service>` that points at `httpbin.org <https://httpbin.org>`_
and hook up a few interesting endpoints.

*********************
Service and endpoints
*********************

.. code-block:: python

    # test_service.py
    from apiron.service.base import Service
    from apiron.endpoint import Endpoint, JsonEndpoint, StreamingEndpoint

    class HttpBinService(Service):
        domain = 'https://httpbin.org'

        getter = JsonEndpoint(path='/get')
        poster = JsonEndpoint(path='/post', default_method='POST')
        status = Endpoint(path='/status/{status_code}/')
        anything = JsonEndpoint(path='/anything/{anything}')
        slow = JsonEndpoint(path='/delay/5')
        streamer = StreamingEndpoint(path='/stream/{num_lines}')


**********************
Using all the features
**********************

.. code-block:: python

    import requests

    from apiron.client import ServiceCaller, Timeout

    from test_service import HttpBinService

    httpbin = HttpBinService()

    # A normal old GET call
    ServiceCaller.call(httpbin, httpbin.getter, params={'foo': 'bar'})

    # A normal old POST call
    ServiceCaller.call(httpbin, httpbin.poster, data={'foo': 'bar'})

    # A GET call with parameters formatted into the path
    ServiceCaller.call(httpbin, httpbin.anything, path_kwargs={'anything': 42})

    # A GET call with a 500 response, raises RetryError since we successfully tried but got a bad response
    try:
        ServiceCaller.call(httpbin, httpbin.status, path_kwargs={'status_code': 500})
    except requests.exceptions.RetryError:
        pass

    # A GET call to a slow endpoint, raises ConnectionError since our connection failed
    try:
        ServiceCaller.call(httpbin, httpbin.slow)
    except requests.exceptions.ConnectionError:
        pass

    # A GET call to a slow endpoint with a longer timeout
    ServiceCaller.call(
        httpbin,
        httpbin.slow,
        timeout_spec=Timeout(connection_timeout=1, read_timeout=6)
    )

    # A streaming response
    response = ServiceCaller.call(httpbin, httpbin.streamer, path_kwargs={'num_lines': 20})
    for chunk in response:
        print(chunk)


*****************
Service discovery
*****************

You may want to interact with a service whose name is known but whose hosts are resolved via another application.
Here is an example where the resolver application always resolves to ``https://www.google.com`` for the host.

.. code-block:: python

    from apiron.client import ServiceCaller
    from apiron.service.discoverable import DiscoverableService

    class AlwaysGoogleResolver:
        @staticmethod
        def resolve(service_name):
            return ['https://www.google.com']

    class GoogleService(DiscoverableService):
        service_name = 'google-service'
        host_resolver_class = AlwaysGoogleResolver

        search = Endpoint(path='/search')

    google = GoogleService()
    ServiceCaller.call(google, google.search, params={'q': 'rare puppers'})

An application may wish to use a load balancer application
or a more complex service discovery mechanism (like Netflix's `Eureka <https://github.com/Netflix/eureka>`_)
to resolve the hostnames of a given service.


********************
Workflow consistency
********************

It's common to have an existing :class:`requests.Session` object you'd like to use to make additional requests.
This is enabled in ``apiron`` with the ``session`` argument to :func:`apiron.client.ServiceCaller.call`.
The passed in session object will be used to send the request.
This is useful for workflows where cookies or other information need to persist across multiple calls.

It's often more useful in logs to know which module initiated the code doing the logging.
``apiron`` allows for an existing logger object to be passed
into :func:`apiron.client.ServiceCaller.call` as well
so that logs will indicate the caller module rather than :mod:`apiron.client`.
