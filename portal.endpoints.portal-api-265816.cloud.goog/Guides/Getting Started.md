# Overview

The Transport API is a [gRPC](https://grpc.io/) interface with a [Protocol Buffers v3](https://developers.google.com/protocol-buffers)
service specification. It also exposes a REST interface through [transcoding](https://cloud.google.com/endpoints/docs/grpc/transcoding).

To get started please read about [authorization](./Authorization).

# Transport API Design Considerations

The Transport API departs from some of the common conventions of gRPC + proto3 APIs.

## Resource Fields with Special Values and Field Masks

The Transport API handles resource fields that may have a special value differently than normal. This concept is often referred to as 
[**"nullability."**](https://en.wikipedia.org/wiki/Nullable_type)

For example, a typical API might specify such a field in the following manner:

```protobuf    
import "google/protobuf/wrappers.proto"

message User
{
    string                      user_id  = 1;
    repeated string             comments = 2;
    google.protobuf.StringValue nickname = 3;  // can be unknown
}
```

Then the presence or absence of the `nickname` submessage can be used to indicate, respectively, whether a User resource has a normal or
special value for its `nickname`.

Wrapping the `nickname` string into a StringValue is necessary because proto3 does not support field absence for scalar types. Instead,
scalar types are effectively always present with some normal value. Wrapping the value into one of the well known wrapper type messages 
allows for field absence to be used to indicate a special value and to be unambiguously determined.

Unfortunately, this approach conflicts with other commonly supported concepts such as 
operations that operate on or return only partial resources. For example, an update that only seeks to update some subset of a resource's 
fields. Because field presence is already being used to indicate whether a field has a normal or special value, such operations must 
explicitly re-specify which fields should **actually** be operated on and which should be entirely ignored.

```protobuf
import "google/protobuf/field_mask.proto"

message UpdateUserRequest
{
    User      user       = 1;
    FieldMask field_mask = 2;  // explicitly specifies the field names that this request is updating
}
```

This approach has several drawbacks:

* It encodes field names, rather than just field numbers, into serializations. This means that field names in the protobuf 
and auto-generated code can never change in the future without breaking backwards compatability with older versions.

* Because field names are relatively verbose compared to field numbers, this heads back in the direction of JSON by encoding 
redundant, inefficient, verbose, human readable field specifications.

* FieldMasks come with restrictions such as only one repeated field is allowed to be specified and it must be in the last 
position of the FieldMask. For update operations, a repeated field will append onto the existing list.

* Field mask negation is not directly supported, except that the absence of a FieldMask is interpreted to mean a field 
mask with all fields specified.

* A JSON encoding of a FieldMask (e.g. - for REST interface) must convert the specified field names to / from camel case.

For all of these reasons, the Transport API took a different approach to support the concepts of nullability and partial resource 
operations.

First, in the protobuf representation of actual resources (as opposed to request-response helper messages) we made every field truly 
optional. Therefore, a field being absent in a resource's message has no intrinsic meaning in and of itself. If a field is absent in 
the protobuf representation of a resource, then that says nothing about whether the underlying resource actually has a normal or special 
value for that field. The absence only means that *this* protobuf message says nothing about that field of the underlying resource.

Second, for fields that can have a special value, we allow them to be explicitly set to a special NULL value.

Taken together, this means that a typical field in a protobuf message representing an underlying resource can be in one of three states: 
unspecified, specified as a normal value, or specified as NULL. We accomplish this by abusing the oneof feature of proto3:

```protobuf
import "google/protobuf/struct.proto"

message User
{
    oneof user_id_     { string user_id  = 1; }
    repeated             string comments = 2; bool                      comments_set  = 3;
    oneof nickname_    { string nickname = 4; google.protobuf.NullValue nickname_null = 5; }  // can be NULL
}
```

Wrapping scalars in oneofs allows their presence / absence to be unambiguously determined. For nullable fields, we add another field 
inside the oneof, which allows it to be explicitly specified as NULL.

Field presence / absence can then be explicitly tested in code through the "which one of" method on the oneof. Alternatively, field 
presence / absence can be tested using the "has" method on the field **and** its null counterpart (if it exists). Additionally, iterations 
over the specified fields will work as expected: specified fields (or their null counterpart) will be in the iteration, while unspecified 
fields will not be.

Repeated fields cannot exist inside oneofs. For repeated fields, we instead use an explicit boolean to disambiguate if an empty list 
indicates whether the field is being specified or not. If the list is not empty, then it is specified, *regardless of the value of the 
accompanying boolean*. In field iterations, if the list is not empty, then it will be in the iteration. If the "\_set" boolean is 
true, then the boolean will be in the iteration.  Otherwise, the field will not be in the iteration at all.

The above naming conventions are consistently applied throughout the Transport API. Scalar fields are wrapped in a oneof that has the same 
name as the scalar field but with an underscore character ("\_") appended. If the scalar is nullable, then another field is added to the 
oneof with the same name as the scalar but with "\_null" appended. For repeated fields, a boolean is added with the same name as the field 
but with "\_set" appended.

This approach allows us to deal with field masks quite differently. Because every field is now truly optional and by default 
unspecified, partial updates can now simply specify the fields they want to update and leave the rest unspecified.

```protobuf
message UpdateUserRequest
{
    User user = 1;  // only specifies the resource name and the other fields to update with their new values
    ...             // other common fields temporarily elided to reduce confusion; keep reading below for details
}
```

Get and list operations that only want partial results can specify a field mask using the resource itself with any value specified
for the fields they want returned.

```protobuf
message GetUserRequest
{
    string user_id         = 1;
    User   field_mask      = 2;  // specifies the fields to return or not to return, depending on field_mask_pstv
    bool   field_mask_pstv = 3;
}
```

The `field_mask_pstv` boolean indicates whether `field_mask` is a positive or negative field mask. A positive field mask specifies the 
fields that should be returned, while a negative field mask indicates the fields that should not be returned. By default, the `field_mask` 
itself will not be specified (interpreted the same as a default / empty field_mask) and `field_mask_pstv` will be false, therfore, all the 
fields of the resource will be returned. Together, `field_mask` and `field_mask_pstv`, can be easily manipulated to filter the results for 
only the fields in which the caller is interested.

Returning to update operations discussed above, we allow a field mask to be specified for updates as well. However, be aware that this 
field mask is **only filtering the returned result** and **NOT** specifying which fields to update! So, the meaning of a field mask here 
is **different** than what a field mask is typically used for in update operations in other APIs! The same filtering functionality is 
present for create operations too.

```protobuf
message CreateUserRequest
{
    User user            = 1;  // unspecified fields will get their default value (e.g. - NULL) or the operation will fail if no default
    User field_mask      = 2;  // specifies the fields to return or not to return, depending on field_mask_pstv
    bool field_mask_pstv = 3;
}

message UpdateUserRequest
{
    User user            = 1;  // only specifies the resource name and the other fields to update with their new values
    User field_mask      = 2;  // specifies the fields to return or not to return, depending on field_mask_pstv
    bool field_mask_pstv = 3;
}
```

## List Operations

In the transport API, a typical list request looks something like:

```protobuf
message ListUsersRequest
{
    string filter          = 1;
    User   field_mask      = 2;
    bool   field_mask_pstv = 3;
    string order_by        = 4;
    uint32 offset          = 5;
    uint32 page_size       = 6;
    string page_token      = 7;    
}

message ListUsersResponse
{
    repeated User users           = 1;
    string        next_page_token = 2;
    uint32        total_size      = 3;
}
```

The `filter` field is a string that specifies a restricted [SQL WHERE](https://en.wikipedia.org/wiki/Where_(SQL)) clause to return a 
reduced result set. The "WHERE" is implied and shouldn't be specified in the string. All the basic logical and mathematical operators and 
predicates on singular resource fields are supported (e.g. - =, <>, ==, !=, >, >=, <, <=, +, -, \*, /, %, IN, BETWEEN, LIKE, MATCH, 
REGEXP, IS NULL, IS NOT NULL, etc.) in the syntax.  Function calls (e.g. - COUNT(\*)), sub-selects, comments, early termination, and other 
SQL operations are disallowed. Field names should not be quoted.

For enum fields, values should be represented by the enum name but wrapped in a single quoted string (e.g. - `enum_field = 
'ENUM_VALUE_NAME'`).

As explained in the previous section, `field_mask` and `field_mask_pstv` filter the fields of the returned result set (i.e. - the contents 
of each User in the returned `users` field).
