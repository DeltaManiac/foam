# Vyom
This bot is in [[rust]]

Recently while browsing [reddit](https://old.reddit.com) I came up on a [post](https://www.reddit.com/r/rust/comments/i1satq/webference_rusty_days_2020_all_recorded_talks/g01rwq8/?context=3) in the [/r/rust](https://old.reddit.com) subreddit which was a link to a youtube playlist for the Rust-Wroclaw conference/meetup, however there was no way I could find the contents of the playlist without going to Youtube. This was a nuance so I went to Youtube and curated the list.

![Manually listing the entries](https://dev-to-uploads.s3.amazonaws.com/i/fxof8cdxfizue63qkz7d.png)

This was going to be tiresome if I'd have to do it every time I see a link that is a YouTube playlist. So here we are writing a bot do this task for everyone. This bot will run on a server somewhere (hopefully forever) and curate playlist info for all the people who avail its service.

# Creating Credentials For Our Bot
In order to write our bot we first need to get some credentials from reddit so that we can access the [reddit apis](https://old.reddit.com/dev/api) programmatically.

First you need an application id and secret so reddit knows your application. We can get this information by going to [preferences/app](https://www.reddit.com/prefs/apps) and clicking `are you a developer? create an app...` button

Reddit lets us choose the type of the app we want to build. The three types of app are :

- Web app: Runs as part of a web service on a server you control. Can keep a secret.

- Installed app: Runs on devices you don't control, such as the user's mobile phone. Cannot keep a secret, and therefore, does not receive one.

- Script app: Runs on hardware you control, such as your own laptop or server. Can keep a secret. Only has access to your account.

More info about about the apps can be found [here](https://github.com/reddit-archive/reddit/wiki/oauth2-app-types).

We choose the `script` type, enter a name and description for our bot, and use  the dummy url `http://www.example.com/unused/redirect/uri` for the redirect url.

![Create App](https://dev-to-uploads.s3.amazonaws.com/i/weyxca7xkhcnfrwo36kj.png)

We have now created the credentials with Client Id : `TjC0s2uTaTHYCg` and Client Secret : `mrkAaWitnXLf_DiRagIRS_33cD8`.

![Credentials](https://dev-to-uploads.s3.amazonaws.com/i/xj0q1m7li5abg060ari6.png)

# Using and Storing the credentials
We can now use the credentials in our source code and use it in our code.

```rust
# main.rs

static  CLIENT_ID:&str="TjC0s2uTaTHYCg";
static  CLIENT_SECRET:&str="mrkAaWitnXLf_DiRagIRS_33cD8";

fn main(){
    println!("Client ID: {}",CLIENT_ID);
    println!("Client Secret: {}",CLIENT_SECRET);
}
```
```shell
DeltaManiac @ ~/git/rust/vyom
└─ $ cargo run
    Finished dev [unoptimized + debuginfo] target(s) in 0.08s
     Running `target/debug/vyom`
Client ID: SmQ7CzGkKA62yA
Client Secret: UItY35BYBEN_rFVnGVzud9Pig6g
```

This is a very easy and clear way to handle credentials but its flawed in two ways.

- If we need to change the credentials then we would have to change the code, rebuild the app and redeploy the app.

- If we decide to share the code with someone or push it github, it will expose our credentials, which can be used to hijack our account and do bad things.

So lets see if we can fix the first problem, by moving the credentials out of the source code. But where do we put it then ? If you're thinking about environment variables then youre absolutely right. Environment variables are a good place to store such values and they are fairly easy to change.

```rust
# main.rs

fn main(){
    match std::env::var("CLIENT_ID1") {
        Ok(client_id) => println!("Client ID: {}", client_id),
        Err(e) => panic!("Couldn't read CLIENT_ID ({})", e),
    };
    match std::env::var("CLIENT_SECRET1") {
        Ok(client_secret) => println!("Client Secret: {}", client_secret),
        Err(e) => panic!("Couldn't read CLIENT_SECRET ({})", e),
    };
}
```
Since our bot wont work without a `client_id` and a `client_secret` we call [panic!](https://doc.rust-lang.org/stable/std/macro.panic.html) so that the application exits with an error.

```shell
(base) DeltaManiac @ ~/git/rust/vyom
└─ $ cargo run
    Finished dev [unoptimized + debuginfo] target(s) in 0.41s
     Running `target/debug/vyom`
thread 'main' panicked at \'Couldn\'t read CLIENT_ID (environment variable not found), 
src/main.rs:9:19
note: run with `RUST_BACKTRACE=1` environment variable to display a backtrace

# Set the environment variables
(base) DeltaManiac @ ~/git/rust/vyom
└─ $ export CLIENT_SECRET=UItY35BYBEN_rFVnGVzud9Pig6g
(base) DeltaManiac @ ~/git/rust/vyom
└─ $ export CLIENT_ID=SmQ7CzGkKA62yA

(base) DeltaManiac @ ~/git/rust/vyom
└─ $ cargo run
    Finished dev [unoptimized + debuginfo] target(s) in 0.47s
     Running `target/debug/vyom`
Client ID: SmQ7CzGkKA62yA
Client Secret: UItY35BYBEN_rFVnGVzud9Pig6g
```

Most of the time we don't really want to export a lot of environment variables manually. Its exhausting. We could fix this problem by writing a shell script that has all our `export` statement... or we can use [dotenv](https://crates.io/crates/dotenv). Dotenv is a way to put environment variables in a `.env` file and read them. Dotenv is smart to enough to only read from the file if the Environment Variable is not set on the system.

We first add the `dotenv` dependency to our `Cargo.toml` file.
```toml
# Cargo.toml
[package]
name = "vyom"
version = "0.1.0"
authors = ["DeltaManiac <maxpaynered@gmail.com>"]
edition = "2018"

[dependencies]
dotenv_codegen="0.15.0" # dotenv dependency
```

We then setup the environment variables in the `.env` file.
```shell
# .env
CLIENT_ID=test_123
CLIENT_SECRET=test_321
Test=DisTest
```

We finally modify our code to use the `dotenv` crate. 
```rust
# main.rs

#[macro_use]
extern crate dotenv_codegen;

fn main(){
    println!("Env Not on Sys: {}",dotenv!("Test"));
    println!("Client ID: {}",dotenv!("CLIENT_ID"));
    println!("Client Secret: {}",dotenv!("CLIENT_SECRET"));
}
```
```shell
(base) DeltaManiac @ ~/git/rust/vyom
└─ $ cargo run
    Finished dev [unoptimized + debuginfo] target(s) in 0.08s
     Running `target/debug/vyom`
Env Not on Sys: ss #Value from the .env file
Client ID: SmQ7CzGkKA62yA #Value from the system
Client Secret: UItY35BYBEN_rFVnGVzud9Pig6g #Value from the system
```
# How will the bot work ? 
The bot will listen to a mention like `/u/VyomBot` and would check if the post is a link to a youtube playlist or the parent comment of the mention is a YouTube playlist.

# Setting up Reddit
I've followed the following steps to setup reddit for testing/developing this bot

1. Created a new user called [VyomBot](https://old.reddit.com/user/VyomBot) so that the bot can be mentioned via `/u/VyomBot`

2. Registered a new app of `script` type for `/u/VyomBot`

3. Create a new [subreddit](https://old.reddit.com/ur/VyomBot) `/r/VyomBot` as a test play ground.

![/r/VyomBot](https://dev-to-uploads.s3.amazonaws.com/i/vgu4s3kk5s6bxvdrhdrh.png)

4. Create a new [post](https://www.reddit.com/r/VyomBot/comments/i6fk15/test_playlist/?) with the link to the playlist.

5. Mention `/u/VyomBot` in the comments.
![Alt Text](https://dev-to-uploads.s3.amazonaws.com/i/vriyg18m5dgc5k8rklc4.png)

# Talking to Reddit
## Getting Messages from Inbox
Lets start off by querying reddit to see if we have a new mention and printing the message. We will use the [roux](https://crates.io/crates/roux) crate for interacting with the reddit apis. 
Direct quote from the description of the crate
> A simple, asynchronous Reddit API wrapper implemented in Rust.
This means that we have to use a framework like [tokio](https://crates.io/crates/tokio) to provide the async runtime for our bot. 
Lets go about doing that.

Add the dependencies to our Cargo.toml file.
```toml
# Cargo.toml
[package]
name = "vyom"
version = "0.1.0"
authors = ["DeltaManiac <maxpaynered@gmail.com>"]
edition = "2018"

[dependencies]
dotenv_codegen="0.15.0" # dotenv dependency
roux="1.0.0" # roux dependency
tokio = {version="0.2.22",features=["macros"]} # tokio dependency and only enable the macro feature
```
Update our code to use the library and call the reddit apis.
```rust
# main.rs

#[macro_use]
extern crate dotenv_codegen;
#[macro_use]
extern crate log; // Used for logging
use roux::Reddit;

#[tokio::main]
async fn main() {
    match Reddit::new(
        dotenv!("VYOM_USERAGENT"),
        dotenv!("VYOM_CLIENT_ID"),
        dotenv!("VYOM_CLIENT_SECRET"),
    )
    .username(dotenv!("VYOM_USERNAME"))
    .password(dotenv!("VYOM_PASSWORD"))
    .login()
    .await
    {   // Try to make a new client with the credentials
        Ok(client) => match client.inbox().await {
            // Fetch the inbox of the logged in user
            Ok(listing) => {
                println!("Message Count {}", listing.data.children.len());
                dbg!(listing.data.children.get(0).unwrap());
            }
            Err(_) => {
                error!("Failed to fetch messages");
            }
        },
        Err(e) => panic!(e),
    }
}

```

When we run the program we get the number of messages we have and the `dbg!` macro shows what the passed in variable which in this case is a `InboxItem` struct, looks like. 

```shell
(base) DeltaManiac @ ~/git/rust/vyom
└─ $ cargo run
    Finished dev [unoptimized + debuginfo] target(s) in 3.71s
     Running `target/debug/vyom`
Message Count 5
[src/main.rs:24] &listing.data.children.get(0).unwrap().data = InboxItem {
    id: "g0vfbra",
    subject: "username mention",
    was_comment: true,
    author: Some(
        "DeltaManiac",
    ),
    parent_id: Some(
        "t3_i6fk15",
    ),
    subreddit_name_prefixed: Some(
        "r/VyomBot",
    ),
    new: true, 
    type: "username_mention",
    body: "/u/VyomBot",
    dest: "VyomBot",
    body_html: "&lt;!-- SC_OFF --&gt;&lt;div class=\"md\"&gt;&lt;p&gt;&lt;a href=\"/u/VyomBot\"&gt;/u/VyomBot&lt;/a&gt;&lt;/p&gt;\n&lt;/div&gt;&lt;!-- SC_ON --&gt;",
    name: "t1_g0vfbra",
    created: 1596987973.0,
    created_utc: 1596959173.0,
    context: "/r/VyomBot/comments/i6fk15/test_playlist/g0vfbra/?context=3",
}
```
We can use the `new` property to identify if this is a message that we had previously read.
The type property can be used to determine if the item is a comment or a username mention.
We can use this to iterate over the messages retrieved and  and determine the messages that we have to reply to. 

# Replying to the message
Roux provides us a convenient method aptly name `comment` to reply to the message. Let's go ahead and use this to reply to the message.
```rust
# main.rs

async fn main() {
...
...
// Fetch the inbox of the logged in user
    Ok(listing) => {
        for message in listing.data.children.iter() {
            is message unread and of type "username_mention"
            if message.data.new && message.data.r#type == "username_mention" {
                match client
                    .comment(
                        "You have been Noted by Vyom. Please Stand By!",
                        &message.data.name.as_str(),
                    )
                    .await
                {
                    Ok(_) => info!("Replied to {}", message.data.name),
                    Err(_) => error!("Failed to reply to mention"),
                };
            }
        }
    }
...
...
```
> Psst.., I'll let you in on something cool. In rust `type` is a reserved keyword. In most programming languages you can use a keyword only as keyword, e.g. you *cannot* have a variable called `for`. In rust we can use `type` as an attribute of a struct and access it by specifing it as a raw string using the `r#` like `message.data.r#type`

Now that we have written the code lets run it and see what happens..

```shell
(base) DeltaManiac @ ~/git/rust/vyom
└─ $ cargo run
    Finished dev [unoptimized + debuginfo] target(s) in 4.45s
     Running `target/debug/vyom`
```
Nice! It logged that we replied to the mention. Lets run it again, this time it should not reply to an allready replied message as we have read it.
```shell
[2020-08-09T12:59:25Z INFO  vyom] Replied to t1_g0vfbra
(base) DeltaManiac @ ~/git/rust/vyom
└─ $ cargo run
    Finished dev [unoptimized + debuginfo] target(s) in 0.10s
     Running `target/debug/vyom`
[2020-08-09T12:59:29Z INFO  vyom] Replied to t1_g0vfbra
(base) DeltaManiac @ ~/git/rust/vyom
└─ $ cargo run
    Finished dev [unoptimized + debuginfo] target(s) in 0.09s
     Running `target/debug/vyom`
[2020-08-09T12:59:32Z INFO  vyom] Replied to t1_g0vfbra
```
Damn! IT REPLIED AGAIN!!! And this is what the subbreddit looks like.

![replies,Replies,REPLIES](https://dev-to-uploads.s3.amazonaws.com/i/fdjrhs96s7yujerdqbu8.png)

I guess this a good place to stop. 
In the next part we will find the reason for this pesky little bug and squash it.

[//begin]: # "Autogenerated link references for markdown compatibility"
[rust]: rust "Rust"
[//end]: # "Autogenerated link references" 
