# Django DRF Cookbook

A comprehensive collection of recipes, patterns, and solutions for building robust APIs with Django Rest Framework.

## About This Cookbook

This cookbook contains reusable, real-world solutions for Django REST Framework from basic to advanced use cases. Each recipe is designed to solve a specific problem with clear examples and explanations.

## How to Use

Each recipe follows a consistent format:

- **Title:** Concise and descriptive
- **Problem:** What you're solving
- **Solution:** Summary of the approach
- **Code:** The actual DRF code
- **Notes:** Caveats, performance, security tips

## Table of Contents

1. [Setup & Foundation](01-setup-foundation/README.md)
   - [Project Setup and Installation](01-setup-foundation/01-project-setup.md)
   - [Basic Project Structure](01-setup-foundation/02-project-structure.md)
   - [Creating Your First API](01-setup-foundation/03-first-api.md)

2. [Core Concepts](02-core-concepts/README.md)
    - [Request & Response](02-core-concepts/01-request-response.md)
    - [Parsers & Renderers](02-core-concepts/02-parsers-renderers.md)
    - [Exception Handling](02-core-concepts/03-exception-handling.md)
    - [Status Codes](02-core-concepts/04-status-codes.md)
    - [Middleware](02-core-concepts/05-middleware.md)
    - [Model Fields: Types & Usage](02-core-concepts/06-model-fields.md)

3. [Serializers Deep Dive](03-serializers/README.md)
   - [ModelSerializer vs Serializer](03-serializers/01-modelserializer-vs-serializer.md)
   - [Nested Serializers](03-serializers/02-nested-serializers.md)
   - [Writable Nested Serializers](03-serializers/03-writable-nested-serializers.md)
   - [SerializerMethodField](03-serializers/04-serializer-method-field.md)
   - [Custom Validation](03-serializers/05-custom-validation.md)
   - [To Representation and Create/Update Methods](03-serializers/06-to-representation-create-update.md)

4. [Views & ViewSets](04-views-viewsets/README.md)
   - [APIView vs GenericAPIView vs ViewSet](04-views-viewsets/01-apiview-vs-genericapiview-vs-viewset.md)
   - [ModelViewSet CRUD Operations](04-views-viewsets/02-modelviewset-crud.md)
   - [Custom Actions with ViewSets](04-views-viewsets/03-custom-actions.md)
   - [Function-Based Views](04-views-viewsets/04-function-based-views.md)

5. [Routers & URLConf](05-routers-urlconf/README.md)
   - [DefaultRouter and SimpleRouter](05-routers-urlconf/01-default-simple-router.md)
   - [Customizing Routes](05-routers-urlconf/02-customizing-routes.md)
   - [Versioning APIs](05-routers-urlconf/03-versioning-apis.md)
   - [Namespacing APIs](05-routers-urlconf/04-namespacing.md)

6. [Authentication & Permissions](06-authentication-permissions/README.md)
   - [Basic Auth, TokenAuth, and JWT](06-authentication-permissions/01-authentication-methods.md)
   - [Custom Permissions](06-authentication-permissions/02-custom-permissions.md)
   - [Throttling](06-authentication-permissions/03-throttling.md)
   - [Role-Based Access Control](06-authentication-permissions/04-role-based-access.md)
   - [OAuth2 Integration](06-authentication-permissions/05-oauth2.md)

7. [Advanced Querying & Filtering](07-querying-filtering/README.md)
   - [Filtering with django-filter](07-querying-filtering/01-django-filter.md)
   - [Searching with SearchFilter](07-querying-filtering/02-search-filter.md)
   - [Ordering Results](07-querying-filtering/03-ordering-filter.md)
   - [Pagination Techniques](07-querying-filtering/04-pagination.md)
   - [User-Based Filtering](07-querying-filtering/05-user-based-filtering.md)

8. [Relations & Nested Resources](08-relations-nested/README.md)
   - [Many-to-Many and Reverse Relations](08-relations-nested/01-many-to-many.md)
   - [Inline Nested Writes](08-relations-nested/02-inline-nested-writes.md)
   - [Nested Routing with drf-nested-routers](08-relations-nested/03-nested-routers.md)
   - [Optimizing Related Fields](08-relations-nested/04-related-field-optimizations.md)

9. [Testing DRF APIs](09-testing/README.md)
   - [Using APITestCase](09-testing/01-api-test-case.md)
   - [Testing Endpoints with Authentication](09-testing/02-testing-with-auth.md)
   - [Using APIClient](09-testing/03-api-client.md)
   - [Test Data with Factory Boy](09-testing/04-factory-boy.md)

10. [Performance & Optimization](10-performance/README.md)
    - [Query Optimization](10-performance/01-query-optimization.md)
    - [Caching Responses](10-performance/02-caching-responses.md)
    - [Throttling Requests](10-performance/03-throttling.md)
    - [Avoiding N+1 Queries](10-performance/04-avoiding-n-plus-1.md)

11. [Real-World Patterns](11-real-world-patterns/README.md)
    - [User Registration & Login](11-real-world-patterns/01-user-registration-login.md)
    - [Password Reset via Email](11-real-world-patterns/02-password-reset.md)
    - [Social Authentication](11-real-world-patterns/03-social-auth.md)
    - [File Upload](11-real-world-patterns/04-file-upload.md)
    - [Notifications API](11-real-world-patterns/05-notifications.md)
    - [Soft Delete](11-real-world-patterns/06-soft-delete.md)
    - [Audit Trail](11-real-world-patterns/07-audit-trail.md)
    - [Activity Streams](11-real-world-patterns/08-activity-streams.md)

12. [API Documentation](12-api-documentation/README.md)
    - [Using drf-spectacular](12-api-documentation/01-drf-spectacular.md)
    - [Using drf-yasg](12-api-documentation/02-drf-yasg.md)
    - [Generating Swagger & Redoc](12-api-documentation/03-swagger-redoc.md)
    - [Securing Documentation](12-api-documentation/04-securing-docs.md)

13. [Async & Background Tasks](13-async-background/README.md)
    - [Using Celery with DRF](13-async-background/01-celery-integration.md)
    - [Task Status Endpoint](13-async-background/02-task-status.md)
    - [Triggering Jobs from API](13-async-background/03-trigger-jobs.md)

14. [Deployment Considerations](14-deployment/README.md)
    - [Gunicorn + Nginx Setup](14-deployment/01-gunicorn-nginx.md)
    - [Environment Variable Management](14-deployment/02-environment-variables.md)
    - [Database Migrations](14-deployment/03-migrations.md)
    - [Static Files](14-deployment/04-static-files.md)
    - [Media Storage](14-deployment/05-media-storage.md)

15. [Security Practices](15-security/README.md)
    - [Preventing Overposting](15-security/01-preventing-overposting.md)
    - [Disabling Browsable API](15-security/02-disabling-browsable-api.md)
    - [HTTPS-only Cookies](15-security/03-https-cookies.md)
    - [Rate Limiting](15-security/04-rate-limiting.md)
    - [Preventing Information Leakage](15-security/05-information-leakage.md)

16. [Bonus Topics](16-bonus/README.md)
    - [GraphQL with graphene-django](16-bonus/01-graphql.md)
    - [WebSocket APIs with Channels](16-bonus/02-websocket.md)
    - [Multi-tenancy](16-bonus/03-multi-tenancy.md)
    - [Internationalization](16-bonus/04-i18n.md)
    - [CI/CD Pipeline](16-bonus/05-ci-cd.md)

## Project Status

This cookbook is a work in progress. The following sections are currently complete:

- Setup & Foundation (3/3 recipes)
- Serializers Deep Dive (6/6 recipes)
- Views & ViewSets (4/4 recipes)
- Routers & URLConf (2/4 recipes)
- Authentication & Permissions (2/5 recipes)
- Performance & Optimization (1/4 recipes)

See [CONTRIBUTING.md](CONTRIBUTING.md) for information on how to contribute.

## Getting Started

To get started with this cookbook, you need a basic understanding of Django and Django REST Framework. The recipes are designed to be independent, so you can jump to any topic that interests you.

For the code examples to work, you'll need:

- Python 3.8+
- Django 4.0+
- Django REST Framework 3.13+

## Contributing

Contributions are welcome! Please feel free to submit a pull request with your own recipes or improvements to existing ones. Check out [CONTRIBUTING.md](CONTRIBUTING.md) for guidelines.

## License

MIT License
