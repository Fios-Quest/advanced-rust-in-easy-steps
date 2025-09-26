Builder
=======

The builder pattern allows us to construct complex types in steps.

For example, imagine we have a 'User' struct that holds a lot of information on a user. Some of that information is
required and some of it is not.

```rust
struct User {
  primary_email: Email,
  username: Username,
  secondary_emails: Vec<EmailAddress>,
  phone_number: Option<PhoneNumber>,
}

```

There are arguably two versions of this pattern, a simpler version that will work in just about any language that I'm
going to refer to as "Builder Lite", and a more complex version that only works in languages with robust type systems 
that I'm going to refer to as the "Typed Builder". Each has their own pros and cons, and we'll go over those too.

Builder Lite
------------


Typed Builder
-------------

Pro's and Con's
---------------

