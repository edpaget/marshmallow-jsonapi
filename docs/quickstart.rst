**********
Quickstart
**********

.. note:: The following guide assumes some familiarity with the marshmallow API. To learn more about marshmallow, see its official documentation at `https://marshmallow.readthedocs.org <https://marshmallow.readthedocs.org>`_.

Declaring schemas
=================

Declare your schemas as you would with marshmallow.

A `Schema` **MUST** define:

- An ``id`` field
- The ``type_`` class Meta option

It is **RECOMMENDED** to set strict mode to `True`.

.. code-block:: python

    from marshmallow_jsonapi import Schema, fields

    class ArticleSchema(Schema):
        id = fields.Str(dump_only=True)
        title = fields.Str()

        class Meta:
            type_ = 'articles'
            strict = True


Serialization
=============

Objects will be serialized to `JSON API documents <http://jsonapi.org/format/#document-structure>`_ with primary data.

.. code-block:: python

    ArticleSchema().dump(article).data
    # {
    #     'data': {
    #         'id': '1',
    #         'type': 'articles',
    #         'attributes': {'title': 'Django is Omakase'}
    #     }
    # }

Relationships
=============

The `Relationship <marshmallow_json.fields.Relationship>` field is used to serialize `relationship objects <http://jsonapi.org/format/#document-resource-object-relationships>`_.

To serialize links, pass a URL format string and a dictionary of keyword arguments. String arguments enclosed in `< >` will be interpreted as attributes to pull from the object being serialized.

.. code-block:: python
    :emphasize-lines: 5-10

    class ArticleSchema(Schema):
        id = fields.Str(dump_only=True)
        title = fields.Str()

        author = fields.Relationship(
            self_url='/articles/{article_id}/relationships/author',
            self_url_kwargs={'article_id': '<id>'},
            related_url='/authors/{author_id}',
            related_url_kwargs={'author_id': '<author.id>'}
        )

        class Meta:
            type_ = 'articles'
            strict = True

    ArticleSchema().dump(article).data
    # {
    #     'data': {
    #         'id': '1',
    #         'type': 'articles'
    #         'attributes': {'title': 'Django is Omakase'},
    #         'relationships': {
    #             'author': {
    #                 'links': {
    #                     'self': '/articles/1/relationships/author'
    #                     'related': '/authors/9',
    #                 }
    #             }
    #         }
    #     }
    # }

Resource linkages
-----------------

You can serialize `resource linkages <http://jsonapi.org/format/#document-resource-object-linkage>`_ by passing ``include_data=True`` .

.. code-block:: python
    :emphasize-lines: 8-10

    class ArticleSchema(Schema):
        id = fields.Str(dump_only=True)
        title = fields.Str()

        comments = fields.Relationship(
            related_url='/posts/{post_id}/comments',
            related_url_kwargs={'post_id': '<id>'},
            # Include resource linkage
            many=True, include_data=True,
            type_='comments'
        )
        class Meta:
            type_ = 'articles'
            strict = True

    ArticleSchema().dump(article).data
    # {
    #     "data": {
    #         'id': '1',
    #         'type': 'articles'
    #         'attributes': {'title': 'Django is Omakase'},
    #         "relationships": {
    #             "comments": {
    #                 "links": {
    #                     "related": "/posts/1/comments/"
    #                 }
    #                 "data": [
    #                     {"id": 5, "type": "comments"},
    #                     {"id": 12, "type": "comments"}
    #                 ],
    #             }
    #         },
    #     }
    # }

Errors
======

``Schema.load`` and ``Schema.validate`` will return JSON API-formatted `Error objects <http://jsonapi.org/format/#error-objects>`_.

.. code-block:: python

    from pprint import pprint

    from marshmallow_jsonapi import Schema, fields
    from marshmallow import validate, ValidationError


    class AuthorSchema(Schema):
        id = fields.Str(dump_only=True)
        first_name = fields.Str(required=True)
        last_name = fields.Str(required=True)
        password = fields.Str(load_only=True, validate=validate.Length(6))
        twitter = fields.Str()

        class Meta:
            type_ = 'people'
            strict = True

    schema = AuthorSchema()
    input_data = {
        'data': {
            'type': 'people',
            'attributes': {
                'first_name': 'Dan',
                'password': 'short'
            }
        }
    }

    try:
        schema.validate(input_data)
    except ValidationError as err:
        pprint(err.messages)
    # {'errors': [{'detail': 'Shorter than minimum length 6.',
    #              'source': {'pointer': '/data/attributes/password'}},
    #             {'detail': 'Missing data for required field.',
    #              'source': {'pointer': '/data/attributes/last_name'}}]}

Validating ``type``
-------------------

If an invalid "type" is passed in the input data, an `IncorrectTypeError <marshmallow_jsonapi.exceptions.IncorrectTypeError>` is raised.


.. code-block:: python

    from marshmallow_jsonapi.exceptions import IncorrectTypeError

    input_data = {
        'data': {
            'type': 'invalid-type',
            'attributes': {
                'first_name': 'Dan',
                'last_name': 'Gebhardt',
                'password': 'verysecure'
            }
        }
    }
    try:
        schema.validate(input_data)
    except IncorrectTypeError as err:
        pprint(err.messages)
    # {'errors': [{'detail': 'Invalid type. Expected "people".',
    #              'pointer': '/data/type'}]}

Inflection
==========

You can optionally specify a function to transform attribute names. For example, you may decide to follow JSON API's `recommendation <http://jsonapi.org/recommendations/#naming>`_ to use "dasherized" names.

.. code-block:: python

    from marshmallow_jsonapi import Schema, fields

    def dasherize(text):
        return text.replace('_', '-')

    class AuthorSchema(Schema):
        id = fields.Str(dump_only=True)
        first_name = fields.Str(required=True)
        last_name = fields.Str(required=True)

        class Meta:
            type_ = 'people'
            inflect = dasherize

    result = AuthorSchema().dump(author)
    result.data
    # {
    #     'data': {
    #         'id': '9',
    #         'type': 'people',
    #         'attributes': {
    #             'first-name': 'Dan',
    #             'last-name': 'Gebhardt'
    #         }
    #     }
    # }

Flask integration
=================

Marshmallow-jsonapi includes optional utilities to integrate with Flask.

For example, the ``Relationship`` field in the ``marshmallow_jsonapi.flask`` module allows you to pass view names instead of path templates.


.. code-block:: python

    from marshmallow_jsonapi import Schema, fields
    from marshmallow_jsonapi.flask import Relationship

    class ArticleSchema(Schema):
        id = fields.Str(dump_only=True)
        title = fields.Str()

        author = fields.Relationship(
            self_view='article_author',
            self_url_kwargs={'article_id': '<id>'},
            related_view='author_detail',
            related_view_kwargs={'author_id': '<author.id>'}
        )

        comments = Relationship(
            related_view='article_comments',
            related_view_kwargs={'article_id': '<id>'},
            many=True, include_data=True,
            type_='comments'
        )

        class Meta:
            type_ = 'posts'

See `here <https://github.com/marshmallow-code/marshmallow-jsonapi/blob/master/examples/flask_example.py>`_ for a full example.
