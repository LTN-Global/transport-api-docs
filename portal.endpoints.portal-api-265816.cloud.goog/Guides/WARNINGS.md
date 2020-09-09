# WARNINGS

## Documentation

This documentation has auto-converted all field names to use CamelCase for both gRPC and REST whereas the actual API uses snake_case. 
This is a known limitation of Google's web docs tool.  Therefore, unfortunately, you must convert all CamelCase specifications back to 
snake_case in your head.

The REST documentation also lists incorrect URLs.  In particular, the listed hostname is not the actual API's hostname.  Please contact 
someone on the transport division's development team to get the correct hostname and port.


## View Resources

Most of the API resources map directly to a database table; however, there are a some *view* resources which require in depth python logic rather than a sql query to retrieve its dataset. Table and view resources should be treated the same as far as users are concerned, but there are a few caveats.

### List Filters

As specified in the List Operations section of [Getting Started](./Getting%20Started), a typical table filter supports all the basic logical and mathematical operators and predicates on non-repeated resource fields  (e.g. - =, <>, ==, !=, >, >=, <, <=, +, -, *, /, %, IN, BETWEEN, LIKE, MATCH, REGEXP, IS NULL, IS NOT NULL, etc.).

For now, list filters on *view* resources are the exceptions, and only support the equals to and AND operators (e.g. =, AND).