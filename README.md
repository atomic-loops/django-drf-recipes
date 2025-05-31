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

1. [Setup & Foundation](docs/01-setup-foundation/README.md)
   - [Project Setup and Installation](docs/01-setup-foundation/01-project-setup.md)
   - [Basic Project Structure](docs/01-setup-foundation/02-project-structure.md)
   - [Creating Your First API](docs/01-setup-foundation/03-first-api.md)

2. [Serializers Deep Dive](docs/02-serializers/README.md)
   - [ModelSerializer vs Serializer](docs/02-serializers/01-modelserializer-vs-serializer.md)
   - [Nested Serializers](docs/02-serializers/02-nested-serializers.md)
   - [Writable Nested Serializers](docs/02-serializers/03-writable-nested-serializers.md)
   - [SerializerMethodField](docs/02-serializers/04-serializer-method-field.md)
   - [Custom Validation](docs/02-serializers/05-custom-validation.md)
   - [To Representation and Create/Update Methods](docs/02-serializers/06-to-representation-create-update.md)

3. [Views & ViewSets](docs/03-views-viewsets/README.md)
   - [APIView vs GenericAPIView vs ViewSet](docs/03-views-viewsets/01-apiview-vs-genericapiview-vs-viewset.md)
   - [ModelViewSet CRUD Operations](docs/03-views-viewsets/02-modelviewset-crud.md)
   - [Custom Actions with ViewSets](docs/03-views-viewsets/03-custom-actions.md)
   - [Function-Based Views](docs/03-views-viewsets/04-function-based-views.md)

4. [Routers & URLConf](docs/04-routers-urlconf/README.md)
   - [DefaultRouter and SimpleRouter](docs/04-routers-urlconf/01-default-simple-router.md)
   - [Customizing Routes](docs/04-routers-urlconf/02-customizing-routes.md)
   - [Versioning APIs](docs/04-routers-urlconf/03-versioning-apis.md)
   - [Namespacing APIs](docs/04-routers-urlconf/04-namespacing.md)

5. [Authentication & Permissions](docs/05-authentication-permissions/README.md)
   - [Basic Auth, TokenAuth, and JWT](docs/05-authentication-permissions/01-authentication-methods.md)
   - [Custom Permissions](docs/05-authentication-permissions/02-custom-permissions.md)
   - [Throttling](docs/05-authentication-permissions/03-throttling.md)
   - [Role-Based Access Control](docs/05-authentication-permissions/04-role-based-access.md)
   - [OAuth2 Integration](docs/05-authentication-permissions/05-oauth2.md)

6. [Advanced Querying & Filtering](docs/06-querying-filtering/README.md)
   - [Filtering with django-filter](docs/06-querying-filtering/01-django-filter.md)
   - [Searching with SearchFilter](docs/06-querying-filtering/02-search-filter.md)
   - [Ordering Results](docs/06-querying-filtering/03-ordering-filter.md)
   - [Pagination Techniques](docs/06-querying-filtering/04-pagination.md)
   - [User-Based Filtering](docs/06-querying-filtering/05-user-based-filtering.md)

7. [Relations & Nested Resources](docs/07-relations-nested/README.md)
   - [Many-to-Many and Reverse Relations](docs/07-relations-nested/01-many-to-many.md)
   - [Inline Nested Writes](docs/07-relations-nested/02-inline-nested-writes.md)
   - [Nested Routing with drf-nested-routers](docs/07-relations-nested/03-nested-routers.md)
   - [Optimizing Related Fields](docs/07-relations-nested/04-related-field-optimizations.md)

8. [Testing DRF APIs](docs/08-testing/README.md)
   - [Using APITestCase](docs/08-testing/01-api-test-case.md)
   - [Testing Endpoints with Authentication](docs/08-testing/02-testing-with-auth.md)
   - [Using APIClient](docs/08-testing/03-api-client.md)
   - [Test Data with Factory Boy](docs/08-testing/04-factory-boy.md)

9. [Performance & Optimization](docs/09-performance/README.md)
   - [Query Optimization](docs/09-performance/01-query-optimization.md)
   - [Caching Responses](docs/09-performance/02-caching-responses.md)
   - [Throttling Requests](docs/09-performance/03-throttling.md)
   - [Avoiding N+1 Queries](docs/09-performance/04-avoiding-n-plus-1.md)

10. [Real-World Patterns](docs/10-real-world-patterns/README.md)
    - [User Registration & Login](docs/10-real-world-patterns/01-user-registration-login.md)
    - [Password Reset via Email](docs/10-real-world-patterns/02-password-reset.md)
    - [Social Authentication](docs/10-real-world-patterns/03-social-auth.md)
    - [File Upload](docs/10-real-world-patterns/04-file-upload.md)
    - [Notifications API](docs/10-real-world-patterns/05-notifications.md)
    - [Soft Delete](docs/10-real-world-patterns/06-soft-delete.md)
    - [Audit Trail](docs/10-real-world-patterns/07-audit-trail.md)
    - [Activity Streams](docs/10-real-world-patterns/08-activity-streams.md)

11. [API Documentation](docs/11-api-documentation/README.md)
    - [Using drf-spectacular](docs/11-api-documentation/01-drf-spectacular.md)
    - [Using drf-yasg](docs/11-api-documentation/02-drf-yasg.md)
    - [Generating Swagger & Redoc](docs/11-api-documentation/03-swagger-redoc.md)
    - [Securing Documentation](docs/11-api-documentation/04-securing-docs.md)

12. [Async & Background Tasks](docs/12-async-background/README.md)
    - [Using Celery with DRF](docs/12-async-background/01-celery-integration.md)
    - [Task Status Endpoint](docs/12-async-background/02-task-status.md)
    - [Triggering Jobs from API](docs/12-async-background/03-trigger-jobs.md)

13. [Deployment Considerations](docs/13-deployment/README.md)
    - [Gunicorn + Nginx Setup](docs/13-deployment/01-gunicorn-nginx.md)
    - [Environment Variable Management](docs/13-deployment/02-environment-variables.md)
    - [Database Migrations](docs/13-deployment/03-migrations.md)
    - [Static Files](docs/13-deployment/04-static-files.md)
    - [Media Storage](docs/13-deployment/05-media-storage.md)

14. [Security Practices](docs/14-security/README.md)
    - [Preventing Overposting](docs/14-security/01-preventing-overposting.md)
    - [Disabling Browsable API](docs/14-security/02-disabling-browsable-api.md)
    - [HTTPS-only Cookies](docs/14-security/03-https-cookies.md)
    - [Rate Limiting](docs/14-security/04-rate-limiting.md)
    - [Preventing Information Leakage](docs/14-security/05-information-leakage.md)

15. [Bonus Topics](docs/15-bonus/README.md)
    - [GraphQL with graphene-django](docs/15-bonus/01-graphql.md)
    - [WebSocket APIs with Channels](docs/15-bonus/02-websocket.md)
    - [Multi-tenancy](docs/15-bonus/03-multi-tenancy.md)
    - [Internationalization](docs/15-bonus/04-i18n.md)
    - [CI/CD Pipeline](docs/15-bonus/05-ci-cd.md)

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
