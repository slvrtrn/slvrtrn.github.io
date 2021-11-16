---
layout: post
title:  "Application configuration using environment variables in Rust"
date:   2021-11-16 22:17:18 +0100
categories: rust typescript
---
Environment variables are one of the ways to deal with your application configuration. Generally speaking, it is even preferable in case you deploy your application into a Kubernetes cluster. In a NodeJS application, this is typically done via [dotenv](https://www.npmjs.com/package/dotenv) which reads variables from either environment itself or `.env` file.

As I was using NodeJS mostly with TypeScript, I was familiar with libraries such as [dotenv-safe](https://www.npmjs.com/package/dotenv-safe) which is used for additional verification with `.env.example` file and [dotenv-parse-variables](https://www.npmjs.com/package/dotenv-parse-variables) that allows parsing Booleans, Numbers, and Arrays instead of having everything as Strings.

Combining all three together, it is possible to read the configuration file with proper types without too much boilerplate, as well as crash the application if any of the variables are missing. For example:

```ts
import dotenvParseVariables from 'dotenv-parse-variables'
import dotenvSafe from 'dotenv-safe'

export type Env = {
  NODE_ENV: 'production' | 'development'
  MAGIC_STRING: string
  MAGIC_NUMBER: number
}

export const ENV = {
  ...dotenvParseVariables(dotenvSafe.config().parsed),
  IS_DEVELOPMENT: process.env.NODE_ENV !== 'production',
  IS_PRODUCTION: process.env.NODE_ENV === 'production',
} as Env
```

Recently, I decided to try Rust as a language for a typical backend application containing a few Kafka consumers inside, as well as a web server to provide liveness probes and other low-key utility endpoints. The necessity to deal with the configuration files popped up immediately since it is required to configure Kafka broker host, group id, topic names, and a lot of other things. Digging through GitHub projects, eventually, I stumbled upon [envy](https://github.com/softprops/envy), which provided almost exact functionality as I was searching for. However, out of the box, it does not support `.env` files, so the following code:

```rs
use serde::Deserialize;

#[derive(Deserialize, Debug)]
pub struct Config {
    pub magic_string: String,
    pub magic_number: u16,
    pub rust_env: String,
}

impl Config {
    pub fn is_production(&self) -> bool {
        self.rust_env == "production"
    }
    pub fn is_development(&self) -> bool {
        !self.is_production()
    }
}

fn main() -> std::io::Result<()> {
    envy::from_env::<Config>()?;
    Ok(())
}
```

will panic despite having `.env` file with the following contents next to it:

```
RUST_ENV=development
MAGIC_STRING=foobar
MAGIC_NUMBER=42
```

Not a big deal! This issue can be worked around quite easily. In case of failure to read an environment variable from the environment itself, we can implement an error handling branch that will read `.env` file contents, load key/value pairs into the environment, and call `envy` again! 

Let's try to do that, adding the following function and changing the main call:

```rs
fn load_config() -> std::io::Result<Config> {
    let env = envy::from_env::<Config>();
    match env {
        // if we could load the config using the existing env variables - use that
        Ok(config) => Ok(config),
        // otherwise, try to load the .env file
        Err(_) => {
            // simulate https://www.npmjs.com/package/dotenv behavior
            let mut file = File::open(".env")?;
            let mut content = String::new();
            file.read_to_string(&mut content)?;
            for line in content.lines() {
                let pair = line.split('=').collect::<Vec<&str>>();
                let (key, value) = match &pair[..] {
                    &[key, value] => (key, value),
                    _ => panic!("Expected env variable pairs, got {}", content),
                };
                std::env::set_var(key, value);
            }
            match envy::from_env::<Config>() {
                Ok(config) => Ok(config),
                Err(e) => panic!("Failed to read the config from env: {}", e),
            }
        }
    }
}

fn main() -> std::io::Result<()> {
    let config = load_config()?;
    Ok(println!("Magic number is {}", config.magic_number))
}
```

After that, if we run the application, there is no more panic, and we can see the proper output:

```
Magic number is 42
```

Other than that, there is one more improvement we can do to our config - make it a static const available from everywhere using [lazy_static](https://docs.rs/lazy_static/1.4.0/lazy_static/) crate. With that crate, we will allow more complex initialization to a static const (in our case, using a function call). 

Let's just move all the code except the main function into some `globals.rs`, adding 

```rs
lazy_static::lazy_static! {
    pub static ref CONFIG: Config = load_config().unwrap();
}
```

there as well. In `main.rs`, we can have it as 

```rs
use globals::CONFIG;

mod globals;

fn main() -> std::io::Result<()> {
    lazy_static::initialize(&CONFIG);
    Ok(println!("Magic number is {}", CONFIG.magic_number))
}
```

Now, the configuration will be initialized in a non-lazy way with either environment variables or `.env` file, if present, and will be available as a static const to any other module in the application. Goal completed!

For the full source code listing, please refer to [my pet project repo](https://github.com/slvrtrn/rust-service/blob/9e293c2fa15098e7ebee88621ab5fcad43e92f7b/src/globals.rs).