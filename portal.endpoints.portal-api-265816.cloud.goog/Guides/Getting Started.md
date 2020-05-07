# Overview

The Transport API is a [gRPC](https://grpc.io/) interface with a [Protocol Buffers v3](https://developers.google.com/protocol-buffers)
service specification. It also exposes a REST interface through [transcoding](https://cloud.google.com/endpoints/docs/grpc/transcoding).

To get started please read about [authorization](./Authorization).

# Transport API Design Considerations

The Transport API departs from some of the common conventions of gRPC + proto3 APIs.

## Unknown Values and Field Masks

The Transport API handles fields whose value may be unknown differently than usual. 

### Problems with the Typical Design

A typical API might specify a field whose value can be unknown using field absence like this:

```protobuf    
import "google/protobuf/wrappers.proto"

message User
{
    string                      user_id  = 1;
    repeated string             comments = 2;
    google.protobuf.StringValue nickname = 3;  // nickname may be absent indicating that it is unknown
}
```

Wrapping `nickname` into a StringValue is necessary because proto3 does not truly support field presence / absence for scalar types. In 
proto3, a regular scalar is considered absent if it has the default value regardless of whether it was explicitly set or not. 
Unfortunately, oftentimes the default value of a scalar (e.g. - 0) is a perfectly valid value for the field and can't also be used to 
indicate that it is unknown. Wrapping the value into one of the well known wrapper type messages allows for field absence to be 
unambiguously determined, regardless of value, which can be used to indicate that the field's value is unknown.

Unfortunately again, this approach conflicts with other common concepts such as operating on or returning partial resources. For example, 
an update that only seeks to update some subset of a resource's fields. Because field absence is already being used to indicate unknown 
values, such operations must explicitly re-specify which fields should **actually** be considered and which should be entirely ignored.

```protobuf
import "google/protobuf/field_mask.proto"

message UpdateUserRequest
{
    User      user       = 1;
    FieldMask field_mask = 2;  // explicitly specifies the field names that this request is actually updating
}
```

This allows a user's `nickname` to be set to unknown by specifying the field name in the field mask but having it be absent in `user`.

This approach has several drawbacks:

* It encodes field names, rather than just field numbers, into serializations. This means that field names in the protobuf 
and auto-generated code can never change in the future without breaking backwards compatability with older versions.

* Because field names are relatively verbose compared to field numbers, this heads back in the direction of JSON by encoding 
redundant, inefficient, verbose, human readable field specifications. It would be far preferable if we could determine which fields
to update simply by examining `user` instead, but overloading the meaning of field absence prevents this.

* FieldMasks come with restrictions such as only one repeated field is allowed to be specified and it must be in the last 
position of the FieldMask. For update operations, a repeated field will append onto the existing list.

* Field mask negation is not directly supported, except that the absence of a FieldMask is interpreted to mean a field 
mask with all fields specified.

* A JSON encoding of a FieldMask (e.g. - REST interface) must convert the specified field names to / from camel case.

### Unknown Values and Field Masks in the Transport API

For all of the above reasons and more, the Transport API took a different approach to support the concepts of unknown values and partial 
resource operations.

First, in the protobuf representation of actual resources (as opposed to request-response helper protobuf messages) we made every field 
truly optional and each field's presence or absence can be unambiguously determined regardless of value. Because every field is truly 
optional, a field being absent in a protobuf representation of a resource has no intrinsic meaning in and of itself. If a field is absent 
in the protobuf representation of a resource, then that says nothing about whether the underlying resource actually has a normal 
or unknown value for that field. Instead, a field's absence only means that *this* protobuf message says *nothing* about that field of the 
underlying resource at all.

Second, for fields that can have an unknown value, we allow them to be explicitly set to a special NULL value. This concept is known as
["nullability"](https://en.wikipedia.org/wiki/Nullable_type).

Taken together, this means that a typical field in a protobuf message representing an underlying resource in the Transport API can be in 
one of three states: unspecified, specified as a normal value, or specified as NULL. We accomplish this through the oneof feature of 
proto3:

```protobuf
import "google/protobuf/struct.proto"

message User
{
    oneof user_id_  { string user_id  = 1; }
    repeated          string comments = 2; bool                      comments_set  = 3;
    oneof nickname_ { string nickname = 4; google.protobuf.NullValue nickname_null = 5; }  // can be NULL meaning it is unknown
}
```

Wrapping scalars in oneofs allows them to be truly optional. Their field presence or absence can be unambiguously determined regardless of 
value using the "which one of" method on the oneof. For scalars that can have an unknown value, we add another field inside its 
oneof, which allows it to be explicitly specified as NULL. Iterations over a protobuf's present fields will work as expected: specified 
fields (or their null counterparts) will be in the iteration, while unspecified fields will not be.

Repeated fields cannot exist inside oneofs. For repeated fields, we instead add an explicit boolean to disambiguate if an empty list 
indicates whether the field is being specified (true) or not (false). If the list is not empty, then it is specified, *regardless of the 
value of the accompanying boolean*. In field iterations, if the list is not empty, then it will be in the iteration. If the "\_set" 
boolean is true, then the boolean will be in the iteration.  Otherwise, the field will not be in the iteration at all.

The above naming conventions are consistently applied throughout the Transport API. Scalar fields are wrapped in a oneof that has the same 
name as the scalar field but with an underscore character ("\_") appended. If the scalar can be unknown, then another field is added to 
the oneof with the same name as the scalar but with "\_null" appended. For repeated fields, a boolean is added with the same name as the 
field but with "\_set" appended.

This approach allows the Transport API to deal with field masks quite differently. Because every field is now truly optional and by 
default unspecified, partial updates can now simply specify the fields they want to update and leave the rest unspecified.

```protobuf
message UpdateUserRequest
{
    User user = 1;  // only specifies the resource name and the other fields to set with their new values
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
fields of the resource will be returned. Together, `field_mask` and `field_mask_pstv` can be easily manipulated to filter the results for 
only the fields in which the caller is interested.

Returning to update operations discussed above, we allow a field mask to be specified for updates as well. However, be aware that this 
field mask is **only filtering the *returned result*** and **NOT** specifying which fields to update! So, the meaning of a field mask here 
is **different** than what a field mask is typically used for in update operations in other APIs! The same filtering functionality is 
provided for create operations too.

Another difference about update operations is that if a repeated field is specified in an update, then it completely sets the values
of the repeated field rather than appending onto them as is more typical.

```protobuf
message CreateUserRequest
{
    User user            = 1;  // unspecified fields will get their default value (e.g. - NULL) or the operation will fail if no default
    User field_mask      = 2;  // specifies the fields to return or not to return, depending on field_mask_pstv
    bool field_mask_pstv = 3;
}

message UpdateUserRequest
{
    User user            = 1;  // only specifies the resource name and the other fields to set with their new values
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
reduced result set. The "WHERE" is implied and must not be specified in the string. All the basic logical and mathematical operators and 
predicates on singular (i.e. - non-repeated) resource fields are supported (e.g. - =, <>, ==, !=, >, >=, <, <=, +, -, \*, /, %, IN, 
BETWEEN, LIKE, MATCH, REGEXP, IS NULL, IS NOT NULL, etc.) in the syntax.  Function calls (e.g. - COUNT(\*)), sub-selects, comments, early 
termination, and other SQL operations are not allowed. Field names must not be quoted. Operations on repeated fields are currently not 
supported and using them will cause an error.

For enum fields, values should be represented by the enum name but wrapped in a single quoted string. E.g.: `"enum_field = 
'ENUM_VALUE'"`.

As explained previously, `field_mask` and `field_mask_pstv` filter the fields of the returned result set (e.g. - the contents 
of each User in the returned `users` field).

The `order_by` field is a string that specifies a restricted [SQL ORDER BY](https://en.wikipedia.org/wiki/Order_by) clause to order a 
result set. The "ORDER BY" is implied and must not be specified in the string. Only singular (i.e. - non-repeated) fields may be specified
with an optional "ASC" or "DESC" operator appended.

The `offset`, `page_size`, and `page_token` fields control pagination of the returned result set. `offset` controls at what position 
offset from the beginning of the overall result set the returned result sets will begin. `page_size` controls how large a returned result 
set can be at most. `page_token` allows an overall result set to be iterated over through repeated calls using the same other parameters. 
On the initial call, `page_token` must be an empty string.  Then each repeated call must use the `next_page_token` returned from the 
previous call.  When `next_page_token` returns as an empty string, then the overall result set has been exhausted. If the overall result 
set is no longer available, then the call will return an error.
