+++
title = "Rust: Baby Steps"
date = 2024-04-29T05:50:58-06:00
draft = false
tags = ["Rust", "development"]
series = "Tilecraft Build Log"
author = "David White"
+++
Last time, I introduced you to Tilecraft, my vision for a highly flexible tile-based map generation API built in Rust. Today's entry will be about my own experience getting started on my first Rust project. 

## Cargo
Before diving into actually writing code, the first step to getting started with Rust is to install the Rust ecosystem's most powerful command line utility: Cargo. Cargo functions as a package manager, build tool, test runner, and all around Swiss army knife for creating and managing a Rust project. Once you have Cargo successfully installed, you can create a new Rust project with the command `cargo new [project_name]`. This creates a new directory with a `Cargo.toml` file and a `src` directory containing a single file: `main.rs`. This is the most basic structure for a Rust project. `Cargo.toml` is used to for project configuration. At first, it contains a section for package details like name and version. It will also contain a list of dependencies, once we add some. The command `cargo build` will compile the source code from the `src` directory and output a binary in the `target` directory. Let's look at some code.

## A First Look at Some Rust Code
If we then open `src/main.rs` in our editor of choice, we're greeted by a very straightforward "Hello, world" program:
```
fn main() {
    println!("Hello, world!");
}
```
Simple, right? There's nothing most developers haven't seen before here. The first line creates a function called "main" which takes no arguments. The next line prints the string `"Hello, world!"`. Notice that Rust uses the keyword `fn` for function declarations and that the `println` function uses a bang as it is a macro. Seeing this as my first introduction to the Rust programming language, I'll admit that I was lulled into thinking that I'd be able to pick up the language in no time flat. Boy was I wrong.

As I mentioned in my previous post, I've been exploring the capabilities of generative AI and have come to use ChatGPT in particular heavily in my development work flow. While I don't believe it's quite savvy enough to build a software project entirely on its own, it's a very useful learning aid as it's (1) highly knowledgeable and (2) able to provide a wider context and answer follow-up questions. I find this mode of learning much more engaging and feel like it meshes very well with my own personal learning style as it makes learning feel more like an active, collaborative process than one of passive absorbtion... but I digress. I embarked on this tangent because my first exposure to the actual Rust language was after asking ChatGPT to write me a simple middleware function to extract a custom header called `X-TENANT-ID` from a request. If you are new to the language,then perhaps you'll be baffled by its output as I was when I first saw it:
``` 
use actix_service::{Service, Transform};
use actix_web::{
    dev::{ServiceRequest, ServiceResponse}, Error, HttpMessage
};
use futures::future::{ok, Ready};
use futures::Future;
use std::pin::Pin;
use std::task::{Context, Poll};
use uuid::Uuid;
use crate::TenantID;

// Middleware struct definition
pub struct FetchTenantID;

// The service factory for the middleware
pub struct FetchTenantIDService<S> {
    service: S,
}

impl<S, B> Transform<S, ServiceRequest> for FetchTenantID
where
    S: Service<ServiceRequest, Response = ServiceResponse<B>, Error = Error> + 'static,
    S::Future: 'static,
    B: 'static,
{
    type Response = ServiceResponse<B>;
    type Error = Error;
    type InitError = ();
    type Transform = FetchTenantIDService<S>;
    type Future = Ready<Result<Self::Transform, Self::InitError>>;

    fn new_transform(&self, service: S) -> Self::Future {
        ok(FetchTenantIDService { service })
    }
}

impl<S, B> Service<ServiceRequest> for FetchTenantIDService<S>
where
    S: Service<ServiceRequest, Response = ServiceResponse<B>, Error = Error> + 'static,
    S::Future: 'static,
    B: 'static,
{
    type Response = ServiceResponse<B>;
    type Error = Error;
    type Future = Pin<Box<dyn Future<Output = Result<Self::Response, Self::Error>>>>;

    fn poll_ready(&self, ctx: &mut Context<'_>) -> Poll<Result<(), Self::Error>> {
        self.service.poll_ready(ctx)
    }

    fn call(&self, req: ServiceRequest) -> Self::Future {
        let tenant_id_header = req.headers().get("X-Tenant-ID");

         match tenant_id_header {
            Some(header_value) => match header_value.to_str() {
                Ok(tenant_id_str) => match Uuid::parse_str(tenant_id_str) {
                    Ok(tenant_id_uuid) => {
                        req.extensions_mut().insert(TenantID(tenant_id_uuid));
                        Box::pin(self.service.call(req))
                    },
                    Err(_) => {
                        let err = actix_web::error::ErrorBadRequest("Invalid UUID format for tenant ID");
                        Box::pin(async move { Err(err) })
                    }
                },
                Err(_) => {
                    let err = actix_web::error::ErrorBadRequest("Invalid tenant ID header format");
                    Box::pin(async move { Err(err) })
                }
            },
            None => {
                let error = actix_web::error::ErrorUnauthorized("Missing tenant ID header");
                Box::pin(async move { Err(error) })
            },
        }
    }
}
```
Coming from hard fought experience learning the language, I now know that this was just about the most dense example imaginable to throw at a beginner as it uses just about every quirky Rust paradigm there is. So let's take a look at some and see if we can get a better grip on what's happening here.

## Quirky Rust Things
### Traits
Traits in Rust are like interfaces in other languages. They define a common set of methods that each implementing struct or enum must also implement. This allows for greater decoupling and more general, modular code as a function can perform an action for any struct which implements the trait. For example: 
```
trait Drawable {
    fn draw(&self);
}

struct Circle {
    radius: f32,
}

struct Square {
    side: f32,
}

impl Drawable for Circle {
    fn draw(&self) {
        println!("Drawing a circle with radius: {}", self.radius);
    }
}

impl Drawable for Square {
    fn draw(&self) {
        println!("Drawing a square with side: {}", self.side);
    }
}

fn render(shape: &impl Drawable) {
    shape.draw();
}
```
Here, the `Drawable` trait defines a function `draw`. In order to use the `render` function, we first need to create a struct that implements `Drawable` as the `render` function calls  the `draw` method on the shape passed in as a parameter. This is enforced by the trait bound `&impl Drawable` applied to the `shape` parameter of `render`.

In the provided middleware example, the `Transform` and `Service` traits are implemented for `FetchTenantID` and `FetchTenantIDService` respectively. These traits facilitate the middleware's ability to plug into the Actix framework.

### Exhaustive Match Statements
Rust requires that match statements be exhaustive; every possible value of the input must be covered. This requirement enhances code safety and integrity, ensuring that no value is accidentally ignored. In the middleware example, all possible outcomes of attempting to retrieve and parse the tenant ID are considered and handled appropriately:
```
match tenant_id_header {
    Some(header_value) => match header_value.to_str() {
        Ok(tenant_id_str) => match Uuid::parse_str(tenant_id_str) {
            Ok(tenant_id_uuid) => { ... },
            Err(_) => { ... }
        },
        Err(_) => { ... }
    },
    None => { ... }
}
```

### Error Handling (Result)
Rust takes a different approach to error handling than the try/catch paradigm that has become prevalent in many other modern languages. Rust implements the throwing and catching of errors through its built-in `Result` enum. A Result can either be Ok and contain a return value, or Err and contain an error. A match statement can then be used to handle in different ways depending on where an Ok or an Err was returned. Here's an example from the middleware code:
```
match Uuid::parse_str(tenant_id_str) {
    Ok(tenant_id_uuid) => {
        req.extensions_mut().insert(TenantID(tenant_id_uuid));
        Box::pin(self.service.call(req))
    },
    Err(_) => {
        let err = actix_web::error::ErrorBadRequest("Invalid UUID format for tenant ID");
        Box::pin(async move { Err(err) })
    }
}
```
The `Uuid::parse_str()` method returns a `Result` enum and we can return a 400 response if the tenant ID from the header is improperly formatted.

### Null Values (Options)
Another place where Rust has opted for an enum type rather than using the kowtowing to convention is with the `Option` type. `Option` takes the place of `null` pointers in other languages for handling cases where there are no values where you might expect them. An `Option` can be `Some` and contain a value, or can simply be `None`. Here's an example of this from the middleware code:
```
let tenant_id_header = req.headers().get("X-Tenant-ID");

match tenant_id_header {
    Some(header_value) => ...,
    None => {
        let error = actix_web::error::ErrorUnauthorized("Missing tenant ID header");
        Box::pin(async move { Err(error) })
    }
}
```

## Conclusion
While I intended to cover some more basic Rust stuff, I realize this post is already getting quite long. Getting into the nitty gritty of learning a programming language can be tedious so thanks for sticking around and let me know if you found this post helpful or maybe you'd even like to see more content about some simple getting started type things for Rust. Next time I'm going to get back into some more project-specific stuff discuss setting up the Diesel ORM for the Tilecraft project and get into some of the data model decisions I made for the project. See ya then!