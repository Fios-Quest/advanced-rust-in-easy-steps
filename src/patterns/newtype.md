NewType
=======

What is a type?
---------------

If I show you this binary value, can you tell me what it is?

`01000001 01010010 01101001 01000101 01010011 01000110 01010100 01010111`

We can rule out a few things.

It's 64bits, which means its definitely not a boolean or a character. It's also too long for an IPv4 address, and too
short for an IPv6 address. More likely it's a number, either a 64bit float `4826389.301167569123208522796630859375` or
a 64-bit integer `4706940307026367575`.

Or, if we run this code:

```rust
# use std::error::Error;
# 
# fn main() -> Result<(), Box<dyn Error>> {
let bytes = vec![
    0b01000001, 
    0b01010010, 
    0b01101001, 
    0b01000101, 
    0b01010011, 
    0b01000110, 
    0b01010100, 
    0b01010111,
];

let what_is_this_number = String::from_utf8(bytes)?;
println!("{what_is_this_number}");
# 
# assert_eq!(what_is_this_number, String::from("ARiESFTW"));
# Ok(())
# }
```

It's an advert for this book!

Types turn data into information. Without knowing that what we're looking at is a number or a sequence of ascii
characters, we'd have a really hard time writing code and, more importantly, what we write would be very error-prone.

Without type information, there's nothing stopping us from accidentally passing a boolean to a function that expects
a complex user structure. We start having to depend constantly on runtime checks to make sure the data our functions
receive is valid before trying to process it.

All modern languages (even ones we may not usually think of being "typed") come with their own built-in types that
usually cover at the very least, floats, strings, lists and objects (or dictionaries). This helps us reason about the
data stored in memory.

> You'll notice I didn't include integers in the bare minimum types, this is because some languages opt to store _all_
> numbers as double precision floating points.

But is it enough to consider `"hello@example.com"` to be a string? It technically is a string, sure, and we could
manipulate it as one, but, in the context of our software, it might be that there is a meaningful difference between
`"hello@example.com"` and `"Hello, example.com"`, in the same way as there is a meaningful difference between 
`4706940307026367575` and `"ARiESFTW"`.

What is a newtype?
------------------

A newtype (or new type), isn't just a type that we create, but it's specifically a type that conveys more meaning around
another type.

For example, if we wanted to represent years, months and days, we could do so with `u64`s:

```rust
# fn main() {
// Todays date as of writing:
let year: u64 = 2025;
let month: u64 = 9;
let day: u64 = 24;
# }
```

However, we can immediately identify some problems with this.

First, there's nothing stopping us from using valid `u64`s to represent invalid data:
```rust
# fn main() {
let year: u64 = 22025;
let month: u64 = 19;
let day: u64 = 2400;
# }
```

Second, there's nothing stopping us passing even a valid day to a function that's supposed to take a month:

```rust
# fn main() {
fn get_english_month_name(month: u64) -> String {
    // ...
#     match month  {
#         1 => "January".to_string(),
#         2 => "February".to_string(),
#         3 => "March".to_string(),
#         4 => "April".to_string(),
#         5 => "May".to_string(),
#         6 => "June".to_string(),
#         7 => "July".to_string(),
#         8 => "August".to_string(),
#         9 => "September".to_string(),
#         10 => "October".to_string(),
#         11 => "November".to_string(),
#         12 => "December".to_string(),
#         _ => "Invalid Month".to_string(),
#     }
}

# let year: u64 = 2025;
# let month: u64 = 9;
# let day: u64 = 24;
# 
println!("{}", get_english_month_name(day));
# assert_eq!(get_english_month_name(day), "Invalid Month".to_string());
# }
```

We need more context about the data. We need to know what "type" of data we're dealing with beyond it just being a
number!

This is what `newtype` is for. First, lets prevent days being passed into functions that take months.

```rust
# fn main() {
# #[derive(Copy, Clone)]
struct Year(u64);
# #[derive(Copy, Clone)]
struct Month(u64);
# #[derive(Copy, Clone)]
struct Day(u64);

let year = Year(2025);
let month = Month(9);
let day = Day(24);

fn get_english_month_name(month: Month) -> String {
    // ...
#     match month.0  {
#         1 => "January".to_string(),
#         2 => "February".to_string(),
#         3 => "March".to_string(),
#         4 => "April".to_string(),
#         5 => "May".to_string(),
#         6 => "June".to_string(),
#         7 => "July".to_string(),
#         8 => "August".to_string(),
#         9 => "September".to_string(),
#         10 => "October".to_string(),
#         11 => "November".to_string(),
#         12 => "December".to_string(),
#         _ => "Invalid Month".to_string(),
#     }
}

// This will work
println!("{}", get_english_month_name(month));
# assert_eq!(get_english_month_name(month), "September".to_string());

// This won't compile, but gives a useful error message!
// println!("{}", get_english_month_name(day));
# }
```
If you try compiling the code with the last line uncommented, you get a wonderful error message:
```text
  --> patterns/new-types.md:165:39
   |
38 | println!("{}", get_english_month_name(day));
   |                ---------------------- ^^^ expected `Month`, found `Day`
   |                |
   |                arguments to this function are incorrect
   |
```

Our second issue is that we can still produce invalid values such as `Month(13)`. We can fix this by restricting the
instantiation of our types and validating the input. The question becomes, what should we do when someone attempts to
use invalid data, I would argue return a Result with a relevant error. Let's focus on `Month`.

```rust
// First we need to make the interior of the struct private which means moving
// it into a seperate module
mod month {
    pub struct Month(u64);

    pub struct InvalidMonthNumber;
    
    impl Month {
        pub fn from_number(month: u64) -> Result<Month, InvalidMonthNumber> {
            if month < 1 || month > 12 {
                Err(InvalidMonthNumber)
            } else {
                Ok(Month(month))
            }
        }
    }
}

use month::Month;

# fn main() {
let maybe_month = Month::from_number(0);
assert!(maybe_month.is_err());

let maybe_month = Month::from_number(9);
assert!(maybe_month.is_ok());
# }
```

I'd say there's still one improvement we can make here, at least for months. Our program is written in English so why
use numbers to represent the months in our code. Furthermore, if we wanted to use the same `get_english_month_name`
function, by matching on the numeric value we still have to do _something_ with a number that's not 1-12.

We can change the code representation of our Month without changing its underlying numeric representation.

```rust
// First we need to make the interior of the struct private which means moving
// it into a seperate module
mod month {
#     #[derive(Debug, PartialEq)]
    #[repr(u64)]
    pub enum Month {
        January = 1,
        February = 2,
        March = 3,
        April = 4,
        May = 5,
        June = 6,
        July = 7,
        August = 8,
        September = 9,
        October = 10,
        November = 11,
        December = 12,
    }

#     #[derive(Debug)]
    pub struct InvalidMonthNumber;
    
    impl Month {
        pub fn from_number(month: u64) -> Result<Month, InvalidMonthNumber> {
            match month  {
                1 => Ok(Month::January),
                2 => Ok(Month::February),
                3 => Ok(Month::March),
                4 => Ok(Month::April),
                5 => Ok(Month::May),
                6 => Ok(Month::June),
                7 => Ok(Month::July),
                8 => Ok(Month::August),
                9 => Ok(Month::September),
                10 => Ok(Month::October),
                11 => Ok(Month::November),
                12 => Ok(Month::December),
                _ => Err(InvalidMonthNumber),
            }
        }

        pub fn get_english_month_name(&self) -> String {
            match self  {
                Month::January => "January".to_string(),
                Month::February => "February".to_string(),
                Month::March => "March".to_string(),
                Month::April => "April".to_string(),
                Month::May => "May".to_string(),
                Month::June => "June".to_string(),
                Month::July => "July".to_string(),
                Month::August => "August".to_string(),
                Month::September => "September".to_string(),
                Month::October => "October".to_string(),
                Month::November => "November".to_string(),
                Month::December => "December".to_string(),
            }
        }
    }
}

use month::Month;

# fn main() {
let month = Month::from_number(9).expect("Somehow not September?!");
assert_eq!(month, Month::September);
println!("{}", month.get_english_month_name());
# assert_eq!(month.get_english_month_name(), "September".to_string());
# }
```

It's worth point out that, in memory, these types are all exactly the same:

```rust
#[repr(u64)]
pub enum MonthEnum {
    // ...snip
#     January = 1,
#     February = 2,
#     March = 3,
#     April = 4,
#     May = 5,
#     June = 6,
#     July = 7,
#     August = 8,
#     September = 9,
#     October = 10,
#     November = 11,
#     December = 12,
}

pub struct MonthStruct(u64);

fn get_memory<'a, T>(input: &'a T) -> &'a [u8] {
    // ...snip
#     // Credit: https://bennett.dev/rust/dump-struct-bytes/
#     unsafe {
#         std::slice::from_raw_parts(
#             input as *const _ as *const u8,
#             std::mem::size_of::<T>()
#         )
#     }
}


let sept_num: u64 = 9;
let sept_struct = MonthStruct(9);
let sept_enum = MonthEnum::September;

let num_bytes = get_memory(&sept_num);
let struct_bytes = get_memory(&sept_struct);
let enum_bytes = get_memory(&sept_enum);

println!("Num bytes: {num_bytes:?}");
println!("Struct bytes: {struct_bytes:?}");
println!("Enum bytes: {enum_bytes:?}");
#
# assert_eq!(num_bytes, struct_bytes);
# assert_eq!(struct_bytes, enum_bytes);
```



