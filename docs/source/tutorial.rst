.. _tutorial:

Full Tutorial
#############

Here you may find the general aspects used by Colander Swagger to generate
the swagger documentation. Most examples presented on this section refer
to the example on `quickstart`.

Using the Pyramid Hook
======================

In order to enable response documentation, you must add this extension to
your Pyramid config. For that you may use:


.. code-block:: python

    from pyramid.config import Configurator

    def setup():
        config = Configurator()
        config.include('cornice')
        config.include('cornice_swagger')
        config.cornice_enable_openapi_view(
            title='MyAPI',
            description="OpenAPI documentation",
            version='1.0.0'
        )
        config.cornice_enable_openapi_explorer()


If you don't know what this is about or need more information, please check the
`Pyramid documentation <http://docs.pylonsproject.org/projects/pyramid>`_

By default API explorer will be served under `/api-explorer` path in your
application. You can easily configure the paths, required permissions and
Pyramid route factory.

Additional ``kwargs`` passed to this directive will be passed to
``CorniceSwagger.generate`` method.


Extracting path parameters
==========================

Path parameters may be automatically extracted from the service definition,
you may overwrite then with schemas on the view if you wish to add more
view-specific information.

Another example:

.. code-block:: python

    values = Service(name='foo',
                     path='/values/{value}')


.. code-block:: json

    {
        "paths": {
            "/values/{value}": {
                "parameters": [
                    {
                        "name": "value",
                        "in": "path",
                        "required": true,
                        "type": "string"
                    }
                ]
            }
        }
    }


Extracting parameters from cornice schemas
==========================================

When using colander validators such as ``colader_validator`` or
``colander_body_validator``, we can extract the operation parameters
from the request schema. The schemas should comply with
`Cornice 2.0 colander schemas <https://cornice.readthedocs.io/en/latest/schema.html#multiple-request-attributes>`_.


.. code-block:: python

    from cornice.validators import colander_body_validator

    values = Service(name='foo',
                     path='/values/{value}')

    class PutBodySchema(colander.MappingSchema):
        value = colander.SchemaNode(colander.String(),
                                    description='My precious value')


    @values.put(validators=(colander_body_validator, ),
                schema=PutBodySchema())
    def set_value(request):
        """Set the value and returns *True* or *False*."""


.. code-block:: json

    {
        "paths": {
            "/values/{value}": {
                "put": {
                    "parameters": [
                        {
                            "name": "PutBodySchema",
                            "in": "body",
                            "required": true,
                            "schema": {
                                "title": "PutBodySchema",
                                "type": "object",
                                "properties": {
                                    "value": {
                                        "type": "string",
                                        "description": "My precious value",
                                        "title": "Value"
                                    }
                                },
                                "required": [
                                    "value"
                                ]
                            }
                        }
                    ]
                }
            }
        }
    }


When using `colander_validator`, the request should have fields corresponding
the parameters locations as follows:


.. code-block:: python

    from cornice.validators import colander_validator

    class BodySchema(colander.MappingSchema):
        value = colander.SchemaNode(colander.String(),
                                    description='My precious value')


    class QuerySchema(colander.MappingSchema):
        foo = colander.SchemaNode(colander.String(), missing=colander.drop)


    class HeaderSchema(colander.MappingSchema):
        bar = colander.SchemaNode(colander.String(), default='blah')


    class PutRequestSchema(colander.MappingSchema):
        body = BodySchema()
        querystring = QuerySchema()
        header = HeaderSchema()


    @values.put(validators=(colander_validator, ),
                schema=PutRequestSchema())
    def set_value(request):
        """Set the value and returns *True* or *False*."""
        pass


When using custom validators, you can pass a method that transforms your custom schema
into a regular Cornice schema that can be validated with ``colander_validator``, so
Cornice Swagger knows how to convert it to swagger parameters.

.. code-block:: python

    from cornice.validators import colander_body_validator

    MY_IDS = [1, 2, 42]

    def my_custom_validator(request, **kwargs):

        schema = kwargs.get('schema')

        # Peforms random additional validation
        if schema and schema['id'] not in MY_IDS:
            request.errors.add("body", "id", "Invalid id.")

        return colander_body_validator(request, **kwargs)

    @values.put(validators=(my_custom_validator, ),
                schema=PutRequestSchema())
    def set_value(request):
        pass


.. code-block:: python

    from cornice.service import get_services
    from cornice_swagger import CorniceSwagger
    from cornice_swagger.utils import body_schema_converter

    def my_custom_schema_converter(schema, args):
        validators = args.get('validators', [])
        if my_custom_validator in validators:
            return body_schema_converter(schema, args)

    swagger = CorniceSwagger(get_services())
    swagger.schema_transformers.append(my_custom_schema_converter)
    print(swagger.generate())


Extracting produced types from renderers
========================================

The produced content-type field is filled by the cornice renderer you are using.
We currently support `json`, `simplejson` and `xml` renderers. Cornice uses `simplejson`
renderer by default, so if you don't specify a renderer you may expect to find
`application/json` on your operation produce fields.

.. code-block:: python

    values = Service(name='foo',
                     path='/values/{value}')

    @values.put(renderer='xml')
    def set_value(request):
        """Set the value and returns *True* or *False*."""


.. code-block:: json

    {
        "paths": {
            "/values/{value}": {
                "put": {
                    "produces": [
                        "text/xml"
                    ]
                }
            }
        }
    }


Extracting accepted types from content_type field
=================================================

On cornice you can defined the accepted content-types for your view through
the `content_type` field. And we use it to generate the Swagger `consumes` types.

.. code-block:: python

    values = Service(name='foo',
                     path='/values/{value}')

    @values.put(content_type=('application/json', 'text/xml'))
    def set_value(request):
        """Set the value and returns *True* or *False*."""


.. code-block:: json

    {
        "paths": {
            "/values/{value}": {
                "put": {
                    "consumes": [
                        "application/json",
                        "text/xml"
                    ]
                }
            }
        }
    }


Documenting responses
=====================

Unfortunately, on Cornice we don't have a way to provide response schemas, so
this part must be provided separately and handled by Cornice Swagger.

For that you must provide a Response Colander Schema that follows the pattern:

.. code-block:: python

    class ResponseSchema(colander.MappingSchema):
        body = BodySchema()
        headers = HeaderSchema()


    get_response_schemas = {
        '200': ResponseSchema(description='Return my OK response'),
        '404': ResponseSchema(description='Return my not found response')
    }


Notice that the ``ResponseSchema`` class follows the same pattern as the
Cornice requests using ``cornice.validators.colander_validator``
(except for querystrings, since obviously we don't have querystrings on responses).

A response schema mapping, as the ``get_response_schemas`` dict should aggregate
response schemas as the one defined as ``ResponseSchema`` with keys matching the
response status code of for each entry. All schema entries should contain descriptions.
You may also provide a ``default`` response schema to be used if the response doesn't
match any of the status codes provided.

From our minimalist example:


.. code-block:: python

    values = Service(name='foo',
                     path='/values/{value}')

    # Create a body schema for our requests
    class BodySchema(colander.MappingSchema):
        value = colander.SchemaNode(colander.String(),
                                    description='My precious value')


    # Create a response schema for our 200 responses
    class OkResponseSchema(colander.MappingSchema):
        body = BodySchema()


    # Aggregate the response schemas for get requests
    response_schemas = {
        '200': OkResponseSchema(description='Return value')
    }


    @values.put(response_schemas=response_schemas)
    def set_value(request):
        """Set the value and returns *True* or *False*."""


.. code-block:: json

    {
        "paths": {
            "/values/{value}": {
                "put": {
                    "responses": {
                        "200": {
                            "description": "Return value",
                            "schema": {
                                "required": [
                                    "value"
                                ],
                                "type": "object",
                                "properties": {
                                    "value": {
                                        "type": "string",
                                        "description": "My precious value",
                                        "title": "Value"
                                    }
                                },
                                "title": "BodySchema"
                            }
                        }
                    }
                }
            }
        }
    }


Documenting tags
================

Cornice Swagger supports two ways of documenting operation tags. You can either
provide a list of tags on the view decorator or have a ``default_tags``
attribute when calling the generator.


.. code-block:: python

    values = Service(name='foo',
                     path='/values/{value}')

    @values.put(tags=['value'])
    def set_value(request):
        """Set the value and returns *True* or *False*."""


.. code-block:: json

    {
        "tags": [
            {
                "name": "values"
            }
        ],
        "paths": {
            "/values/{value}": {
                "get": {
                    "tags": [
                        "values"
                    ]
                }
            }
        }
    }


When using the ``default_tags`` attribute, you can either use a raw list
of tags or a callable that takes a cornice service and returns a list of tags.


.. code-block:: python

    def default_tag_callable(service):
        return [service.path.split('/')[1]]

    swagger = CorniceSwagger(get_services())
    swagger.default_tags = default_tag_callable
    spec = swagger.generate('IceCreamAPI', '4.2')

.. code-block:: python

    from cornice.service import get_services
    from cornice_swagger import CorniceSwagger

    swagger = CorniceSwagger(get_services())
    swagger.default_tags = ['IceCream']
    spec = swagger.generate('IceCreamAPI', '4.2')


Generating summaries with view docstrings
=========================================

You may use view docstrings to create operation summaries. You may enable
this by passing ``summary_docstrings=True`` when calling the generator.
For example, the following view definition docstring will correspond to
the following swagger summary:

.. code-block:: python

    values = Service(name='foo',
                     path='/values')

    @values.get()
    def get_value(request):
        """Returns the value."""

    swagger = CorniceSwagger(get_services())
    swagger.summary_docstrings = True
    spec = swagger.generate('IceCreamAPI', '4.2')


.. code-block:: json

    {
        "paths": {
            "/values": {
                "get": {
                    "summary": "Returns the value."
                }
            }
        }
    }

Custom Swagger UI <script> bootstrap
====================================

By default standard Swagger UI (https://swagger.io/swagger-ui/)
config is used, but you can customize the generated script tag
by providing your own callable path in config.

The default one is:

``cornice_swagger.swagger_ui_script_generator = cornice_swagger.views:swagger_ui_script_template``

It points to the following callable that accepts a request object:

.. literalinclude:: ../../cornice_swagger/views.py
    :pyobject: swagger_ui_script_template
    :language: python

