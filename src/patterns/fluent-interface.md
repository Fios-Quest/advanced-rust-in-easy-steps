Fluent Interface
================

We often group data together into structs or objects.

```rust
# #[derive(Debug)]
# struct Username(String);
# #[derive(Debug)]
# struct Email(String);
# #[derive(Debug)]
# struct DateOfBirth(String);
# #[derive(Debug)]
struct User {
    // This field will remain immutable
    username: Username,

    // We want to be able to change these fields
    pub email: Option<Email>,
    pub date_of_birth: Option<DateOfBirth>,
}

impl User {
    fn new(username: Username) -> Self {
        Self {
            username,
            email: None,
            date_of_birth: None,
        }
    }
}

// --- Usage ---

# fn main() -> Result<(), ()> {
# let username = Username("Yuki".to_string());
# let date_of_birth = DateOfBirth("2009-05-01".to_string());
# let email = Email("yuki@example.com".to_string());
let mut yuki = User::new(username);

yuki.email = Some(email);
yuki.date_of_birth = Some(date_of_birth);
# println!("{yuki:#?}");
# Ok(())
# }
```

Depending on the nature of the data we might want to manage access through getters and setters, particularly if we need
to do any validation, or manipulation, etc. In our `User` example above, it might be good to not have to specify the
`Option`.

```rust
# #[derive(Debug)]
# struct Username(String);
# #[derive(Debug)]
# struct Email(String);
# #[derive(Debug)]
# struct DateOfBirth(String);
# #[derive(Debug)]
struct User {
    // Direct access to these properties is forbidden
    username: Username,
    email: Option<Email>,
    date_of_birth: Option<DateOfBirth>,
}

impl User {
    // ... snip ...
#     fn new(username: Username) -> Self {
#         Self {
#             username,
#             email: None,
#             date_of_birth: None,
#         }
#     }

    fn set_email(&mut self, email: Email) {
        self.email = Some(email)
    }

    fn set_date_of_birth(&mut self, date_of_birth: DateOfBirth) {
        self.date_of_birth = Some(date_of_birth);
    }
}

// --- Usage ---

# fn main() -> Result<(), ()> {
# let username = Username("Yuki".to_string());
# let date_of_birth = DateOfBirth("2009-05-01".to_string());
# let email = Email("yuki@example.com".to_string());
let mut yuki = User::new(username);

yuki.set_email(email);
yuki.set_date_of_birth(date_of_birth);
# println!("{yuki:#?}");
# Ok(())
# }
```

However, this is still a little bit clunky as we have to refer to the underlying variable (`yuki`) multiple times.

We can make this more "fluent" simply by returning a reference to that object after each call of a setter.


```rust
# #[derive(Debug)]
# struct Username(String);
# #[derive(Debug)]
# struct Email(String);
# #[derive(Debug)]
# struct DateOfBirth(String);
# #[derive(Debug)]
struct User {
    username: Username,
    email: Option<Email>,
    date_of_birth: Option<DateOfBirth>,
}

impl User {
    // ... snip ...
#     fn new(username: Username) -> Self {
#         Self {
#             username,
#             email: None,
#             date_of_birth: None,
#         }
#     }

    fn set_email(&mut self, email: Email) -> &mut Self {
        self.email = Some(email);
        self
    }

    fn set_date_of_birth(&mut self, date_of_birth: DateOfBirth) -> &mut Self {
        self.date_of_birth = Some(date_of_birth);
        self
    }
}

// --- Usage ---

# fn main() -> Result<(), ()> {
# let username = Username("Yuki".to_string());
# let date_of_birth = DateOfBirth("2009-05-01".to_string());
# let email = Email("yuki@example.com".to_string());
let mut yuki = User::new(username);

yuki.set_email(email)
    .set_date_of_birth(date_of_birth);
# println!("{yuki:#?}");
# Ok(())
# }
```

Rust Quirks
-----------

There are two quirks to this pattern that impact Rust specifically.

Firstly, this pattern works in many languages, and particularly in languages that use things like pass by reference and
copy on write by default, you can sort of stop thinking about it there. Rust has very specific memory management rules
to think about though.

This means we need to consider whether our fluent interface will use references or take and return ownership.

There are pros and cons to each approach.

Using references in our previous example, we need to make sure the owned data exists somewhere before we start modifying
it. That's why we stored `yuki` first, then modified it, effectively two steps.

If we passed ownership back and forth instead, we could create a single chain:

```rust
# #[derive(Debug)]
# struct Username(String);
# #[derive(Debug)]
# struct Email(String);
# #[derive(Debug)]
# struct DateOfBirth(String);
# #[derive(Debug)]
# struct User {
#     username: Username,
#     email: Option<Email>,
#     date_of_birth: Option<DateOfBirth>,
# }
# 
impl User {
    // ... snip ...
#     fn new(username: Username) -> Self {
#         Self {
#             username,
#             email: None,
#             date_of_birth: None,
#         }
#     }

    fn set_email(mut self, email: Email) -> Self {
        self.email = Some(email);
        self
    }

    fn set_date_of_birth(mut self, date_of_birth: DateOfBirth) -> Self {
        self.date_of_birth = Some(date_of_birth);
        self
    }
}

// --- Usage ---

# fn main() -> Result<(), ()> {
# let username = Username("Yuki".to_string());
# let date_of_birth = DateOfBirth("2009-05-01".to_string());
# let email = Email("yuki@example.com".to_string());
let yuki = User::new(username)
    .set_email(email)
    .set_date_of_birth(date_of_birth);
# println!("{yuki:#?}");
# Ok(())
# }
```

In addition to this giving us a slightly nicer flow, you might notice that `yuki` doesn't need to be mutable, we're
encapsulating the mutability inside the methods where we need it.

However, there's a downside to this too. If we _did_ want to modify a single value, we'd have to make sure we take
ownership back from the method:

```rust
# #[derive(Debug)]
# struct Username(String);
# #[derive(Debug)]
# struct Email(String);
# #[derive(Debug)]
# struct DateOfBirth(String);
# #[derive(Debug)]
# struct User {
#     username: Username,
#     email: Option<Email>,
#     date_of_birth: Option<DateOfBirth>,
# }
# 
# impl User {
#     fn new(username: Username) -> Self {
#         Self {
#             username,
#             email: None,
#             date_of_birth: None,
#         }
#     }
# 
#     fn set_email(mut self, email: Email) -> Self {
#         self.email = Some(email);
#         self
#     }
# 
#     fn set_date_of_birth(mut self, date_of_birth: DateOfBirth) -> Self {
#         self.date_of_birth = Some(date_of_birth);
#         self
#     }
# }
# 
# fn main() -> Result<(), ()> {
# let username = Username("Yuki".to_string());
# let date_of_birth = DateOfBirth("2009-05-01".to_string());
# let email = Email("yuki@example.com".to_string());
# let yuki = User::new(username);
let yuki = yuki.set_email(email);
# println!("{yuki:#?}");
# Ok(())
# }
```

Which version you use will be up to you and likely depend on your circumstances. You could even mix and match (though
you will need to give the methods different names).

The other quirk is more of a nice touch. What if our setter could fail? Obviously, it's Rust, we need to return a
`Result`. Luckily, the question mark operator allows us to unwrap an `Ok`, which means we can continue the chain so
long as we're able to bubble the error.

```rust
# #[derive(Debug)]
# struct Username(String);
# #[derive(Debug)]
# struct Email(String);
# #[derive(Debug)]
# struct DateOfBirth(String);
# impl DateOfBirth {
#     fn is_old_enough(&self) -> bool {
#         true
#     }
# }
# #[derive(Debug)]
# struct User {
#     username: Username,
#     email: Option<Email>,
#     date_of_birth: Option<DateOfBirth>,
# }
# struct NotOldEnoughError;
# impl From<NotOldEnoughError> for () {
#     fn from(value: NotOldEnoughError) -> Self {
#         ()
#     }
# }
# 
impl User {
    // ... snip ...
#     fn new(username: Username) -> Self {
#         Self {
#             username,
#             email: None,
#             date_of_birth: None,
#         }
#     }
# 
#     fn set_email(mut self, email: Email) -> Self {
#         self.email = Some(email);
#         self
#     }

    fn set_dob(mut self, dob: DateOfBirth) -> Result<Self, NotOldEnoughError> {
        if !dob.is_old_enough() {
            return Err(NotOldEnoughError);
        }
        self.date_of_birth = Some(dob);
        Ok(self)
    }
}

// --- Usage ---

# fn main() -> Result<(), ()> {
# let username = Username("Yuki".to_string());
# let date_of_birth = DateOfBirth("2009-05-01".to_string());
# let email = Email("yuki@example.com".to_string());
let yuki = User::new(username)
    .set_dob(date_of_birth)? // we can continue the chain despite the result
    .set_email(email);
# println!("{yuki:#?}");
# Ok(())
# }
```

Using this pattern allows us to write much more readable code!
