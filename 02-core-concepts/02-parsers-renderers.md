# Parsers & Renderers in DRF

DRF uses parsers to handle incoming request data and renderers to control the format of outgoing responses.

## Parsers
- Parsers convert the raw request body into Python data.
- Common parsers: `JSONParser`, `FormParser`, `MultiPartParser`.

**Example:**
```python
from rest_framework.decorators import api_view, parser_classes
from rest_framework.parsers import MultiPartParser, FormParser

@api_view(['POST'])
@parser_classes([MultiPartParser, FormParser])
def upload_file(request):
    file = request.data.get('file')
    return Response({'filename': file.name})
```

## Renderers
- Renderers control how response data is formatted (e.g., JSON, HTML, XML).
- Common renderers: `JSONRenderer`, `BrowsableAPIRenderer`, `TemplateHTMLRenderer`.

**Example:**
```python
from rest_framework.decorators import api_view, renderer_classes
from rest_framework.renderers import JSONRenderer, BrowsableAPIRenderer

@api_view(['GET'])
@renderer_classes([JSONRenderer, BrowsableAPIRenderer])
def example(request):
    return Response({'message': 'Hello, world!'})
```

**References in this repo:**
- See `03-views-viewsets/04-function-based-views.md` and `14-security/02-disabling-browsable-api.md`. 