# GraphQL with graphene-django

## Problem
How do you add a GraphQL API to your Django project alongside or instead of REST endpoints?

## Solution
Use the `graphene-django` package to define GraphQL schemas and expose a `/graphql/` endpoint.

## Code

### Install graphene-django
```bash
pip install graphene-django
```

### settings.py (add to INSTALLED_APPS)
```python
INSTALLED_APPS += [
    'graphene_django',
]
GRAPHENE = {
    'SCHEMA': 'myapp.schema.schema',
}
```

### schema.py (in your app)
```python
import graphene
from graphene_django.types import DjangoObjectType
from .models import MyModel

class MyModelType(DjangoObjectType):
    class Meta:
        model = MyModel

class Query(graphene.ObjectType):
    all_my_models = graphene.List(MyModelType)

    def resolve_all_my_models(self, info):
        return MyModel.objects.all()

schema = graphene.Schema(query=Query)
```

### urls.py
```python
from django.urls import path
from graphene_django.views import GraphQLView

urlpatterns = [
    path('graphql/', GraphQLView.as_view(graphiql=True)),
]
```

## Notes
- The `/graphql/` endpoint provides an interactive GraphiQL interface for testing queries.
- You can use both REST and GraphQL in the same Django project.
- For authentication, use DRF or Django authentication as usual and pass tokens in headers. 