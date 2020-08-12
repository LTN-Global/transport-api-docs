# Transport API Authorization

## API Key

To access any portion of the Transport API, all requests must 
[specify a valid API key](https://cloud.google.com/endpoints/docs/grpc/restricting-api-access-with-api-keys#calling_an_api_using_an_api_key).

Please contact an administrator in the Transport division to obtain an API key.

## API Authentication

To access most any portion of the Transport API, a request must also specify a valid JWT bearer token. 

To obtain a token, a user must first authenticate themselves to the API using [CreateToken](../../methods/Auth/CreateToken) 
(or via [REST](../../routes/v1/oauth/token/post)).  Please note that even this request must specify the API key.

The caller must specify a valid Transport login consisting of a username and password. Currently, only users in the NOC 
account are authorized to use this API.

Please contact an administrator in the Transport division to obtain a Transport login in the NOC account.

Upon successful authentication, the API will return a JWT bearer token string that is valid for the next hour.

If you try to make a request after your token has expired, then you will get the following error letting you know you have to make a request for another token.

    {
        "code": 16,
        "message": "JWT validation failed: TIME_CONSTRAINT_FAILURE",
        "details": [
            {
                "@type": "type.googleapis.com/google.rpc.DebugInfo",
                "stackEntries": [],
                "detail": "auth"
            }
        ]
    }

## API Access Control

Access to specific API resources and procedures is controlled by your user login specified in your JWT bearer token. Currently, only users in the NOC account are 
authorized to use this API.

## Curl Examples

### Generate a JWT:

    api.txt contains: <API key>
    login.txt contains: { "username": "<portal username>", "password": "<portal password>" }

    > curl -X POST "https://${hostname}:${hostport}/v1/oauth/token?key=$(cat api.txt)" --data-binary @login.txt

Returns:
    
    {"token":"<JWT body>"}
    
    Write jwt.txt to contain: Authorization: Bearer <JWT body>

### Accessing an Endpoint:

    api.txt contains: <API key>
    jwt.txt contains: Authorization: Bearer <JWT body>

    > curl "https://${hostname}:${hostport}/v1/users/jschultz@ltnglobal.com?key=$(cat api.txt)" -H @jwt.txt
