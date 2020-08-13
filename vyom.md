# Vyom

The surmised version of how to write a Reddit Bot in  [[rust]]
# Part I 

Recently while browsing [reddit](https://old.reddit.com) I came up on a [post](https://www.reddit.com/r/rust/comments/i1satq/webference_rusty_days_2020_all_recorded_talks/g01rwq8/?context=3) in the [/r/rust](https://old.reddit.com) subreddit which was a link to a YouTube playlist for the Rusty-Days conference, however there was no way I could find the contents of the playlist without going to YouTube on my phone. This was a nuance so I went to YouTube and curated the list.

![Manually listing the entries](https://dev-to-uploads.s3.amazonaws.com/i/fxof8cdxfizue63qkz7d.png)

This was going to be tiresome if I'd have to do it every time I see a post that links to a YouTube playlist. 
So here we are writing a bot do this task for everyone. This bot will run on a server somewhere (hopefully forever) and curate playlist info for all the people who avail its service.

# Creating Credentials For Our Bot
In order to write our bot we first need to get some credentials from reddit so that we can access [reddit apis](https://old.reddit.com/dev/api) programmatically.

First we need an application id and secret so that reddit can know our application. We can get this information by going to [preferences/app](https://www.reddit.com/prefs/apps) and clicking `are you a developer? create an app...` button cause **we definitely are.**

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
We can now hard code the credentials in our source code and use like this.

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

This is a very easy and clear way to handle credentials but it is flawed.

- If we need to change the credentials then we would have to change the code, rebuild the app and restart the app.

- If we decide to share the code with someone or push it github, it will expose our credentials, which can be used to hijack our account and do bad things.

So lets see if we can fix the first problem, by moving the credentials out of the source code. But where do we put it then ? If you're thinking about environment variables then you're absolutely right. Environment variables are a good place to store such values and they are fairly easy to change.

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

Most of the time we don't really want to export a lot of environment variables manually. It is exhausting. We could fix this problem by writing a shell script that has all our `export` statements... or we can use [dotenv](https://crates.io/crates/dotenv). Dotenv is a crate that provides us a way to put environment variables in a `.env` file and read them. Dotenv is smart to enough to only read from the file if the Environment Variable is **not set** on the system.

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
Test=DeezTests
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
The bot will listen to a mention like `/u/VyomBot` and would check if the post is a link to a YouTube playlist or  at a later stage if the parent comment of the mention is a YouTube playlist.

# Setting up Reddit
We can follow these steps to setup reddit for testing/developing this bot

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
> Psst.., I'll let you in on something cool. In rust `type` is a reserved keyword. In most programming languages you can use a keyword only as keyword, e.g. you *cannot* have a variable called `for`. In rust we can use `type` as an attribute of a struct and access it by specifying it as a raw string using the `r#` like `message.data.r#type`

Now that we have written the code lets run it and see what happens..

```shell
(base) DeltaManiac @ ~/git/rust/vyom
└─ $ cargo run
    Finished dev [unoptimized + debuginfo] target(s) in 4.45s
     Running `target/debug/vyom`
```
Nice! It logged that we replied to the mention. Lets run it again, this time it should not reply to an already replied message as we have read it.
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
Damn! 
IT REPLIED AGAIN!!!😞
And this is what the subreddit looks like now.

![replies,Replies,REPLIES](https://dev-to-uploads.s3.amazonaws.com/i/fdjrhs96s7yujerdqbu8.png)

Time to find that pesky bug and get rid of it for good. 

Lets go to reddit and see what the inbox looks like.

![Unread Message](https://dev-to-uploads.s3.amazonaws.com/i/hjb7uxcod9jfirud71qy.png)

Well, its just as we suspected, when we reply to a mention with the `comment` function it does not change the status of the message. Sifting through the [documentation](https://docs.rs/roux/1.0.0/roux/?search=read) of `roux` we can find a method that marks a message as `read`. 

The place we are at right now reminds of a the poem [The Road Not Taken](https://www.poetryfoundation.org/poems/44272/the-road-not-taken) by Robert Frost. It talks about how the author finds two roads diverging in the wood and he ponders which one to travel upon. I ask you to take a few minutes and read the poem, its beautiful.

I'll be waiting!

Oh BTW the code can be found on the `part-I` branch [here](https://github.com/DeltaManiac/VyomBot)
 
# Part II
If you had read the poem mentioned in the previous part, you can be pretty sure what we are going to do right now. You Betcha! We are going to go down the Rabbit Hole. 

Just as in the poem it would have been easy for us to change the library to something that already has a `mark as read` method like many do and continue on, but like Frost we will take the road not taken and that might make all the difference. 😉

## Down the Rabbit Hole

We actually got stumped on the last part because there was no method to mark a message as read in `roux`. This makes one wonder if there isn't such an api for reddit or that `roux` just didn't implement it.

Lets head to [reddit api docs](https://www.reddit.com/dev/api) and try our luck.

Yep, reddit does have a [`read_message`](https://www.reddit.com/dev/api#POST_api_read_message) api for us to use exactly for this purpose. The api accepts a list of [fullnames](https://www.reddit.com/dev/api#fullnames) with an HTTP POST method.

What is the `fullname` for our message ? Its nothing but the `name` parameter of the struct.

Now to fix `roux`, so that we can mark the message as read.

Lets clone the [roux source code](https://github.com/halcyonnouveau/roux.rs) into another directory.

Since the `comment` method we used is an api which POSTS the comment data to reddit. Perhaps we can ~~reuse~~, who are we kidding ? We can definitely *copy-paste* and modify the code to send some data to the `read_message` api.

> While searching for a way to mark a message as read, we came up across another api [`message/unread`](https://www.reddit.com/dev/api#GET_message_unread) which returns only the unread messages from our inbox, so we don't have to filter out on the `new` flag of the response anymore. Yay!
```rust
# src/me/mod.rs
...
/// Get user's submitted posts.
    pub async fn inbox(&self) -> Result<BasicListing<InboxItem>, RouxError> {
        Ok(self
            .get("message/inbox")
            .await?
            .json::<BasicListing<InboxItem>>()
            .await?)
    }

/** This is our addition **/
///  Get users unread messages
    pub async fn unread(&self) -> Result<BasicListing<InboxItem>, RouxError> {
        Ok(self
            .get("message/unread")
            .await?
            .json::<BasicListing<InboxItem>>()
            .await?)
    }

/** This is our addition **/
/// Mark message as read
    pub async fn mark_read(&self, ids: &str) -> Result<Response, RouxError> {
        let form = [("id", ids)];
        self.post("api/read_message", &form).await
    }

/** This is our addition **/
/// Mark messages as unread
    pub async fn mark_read(&self, ids: &str) -> Result<Response, RouxError> {
        let form = [("id", ids)];
        self.post("api/unread_message", &form).await
    }

    pub async fn comment(&self, text: &str, parent: &str) -> Result<Response, RouxError> {
        let form = [("text", text), ("parent", parent)];
        self.post("api/comment", &form).await
    }
...
```
> I've submitted a [PR](https://github.com/halcyonnouveau/roux.rs/pull/13) to roux with these changes.
{% github https://github.com/halcyonnouveau/roux.rs/pull/13 %}

So all is good and well with the change, but how do we use this changed version with our code ?

`Cargo.toml` is the answer. We can tell `Cargo.toml` to use the code from a directory or from a url for a specified crate. Since we have a the modified source code in our system, we can point to that to get it working.
```toml
# Cargo.toml

[package]
name = "vyom"
version = "0.1.0"
authors = ["Harikrishnan Menon <harikrishnan.menon@sap.com>"]
edition = "2018"

[dependencies]
roux={path="../roux.rs"} #This points to our local modified copy
# roux={git = "https://github.com/DeltaManiac/roux.rs"} #This points to the modified version on github
dotenv_codegen="0.15.0"
tokio = {version="0.2.22", features=["macros"]}
env_logger ="0.7.1"
log = "0.4.11"
```
When we build our project now, we can see that it picks up the roux source code from the new path specified by us.
```shell
(base) DeltaManiac @ ~/git/rust/vyom
└─ $ cargo build
   Compiling roux v1.0.1-alpha.0 (/Users/DeltaManiac/git/rust/roux.rs)
   Compiling vyom v0.1.0 (/Users/DeltaManiac/git/rust/vyom)
   Finished dev [unoptimized + debuginfo] target(s) in 6.70s
```
> If we don't want go through the hassle of doing this, we can point cargo to my fork which has the necessary changes. This is how the code would be in the repository.

## Actually Squashing the Bug

Now that we are back from our exceedingly educating trip down the rabbit hole, lets see how we can finally mark a message as read after we reply to it.

```rust
# main.rs
...
...
// Fetch only the unread messages form the inbox of the logged in user
Ok(client) => match client.unread().await {
    Ok(listing) => {
        for message in listing.data.children.iter() {
            // We have removed the `new` check
            if message.data.r#type == "username_mention" {
                match client
                    .comment(
                        "Thank you for standing by while we squished a bug. You shouldn't be seeing this message again!",
                        &message.data.name.as_str(),
                    )
                    .await
                {
                    Ok(_) => {
                        info!("Replied to {}", message.data.name);
                        match client.mark_read(message.data.name.as_str()).await {
                            Ok(_) => info!("Marked {} as read", message.data.name),
                            Err(_) => {
                                error!("Failed to mark {} as read", message.data.name)
                            }
                        }
                    }
                    Err(_) => error!("Failed to reply to mention {}", message.data.name),
                };
            }
        }
    }
...
...
```
We have changed the reply text so that we can identify from reddit that it is actually the new reply that is being sent, and we call the `mark_read` method form the modified crate to mark the message as read.

Lets run the code and see if it works. Fingers Crossed.
```shell
(base) DeltaManiac @ ~/git/rust/vyom
└─ $ cargo run   
   Compiling vyom v0.1.0
    Finished dev [unoptimized + debuginfo] target(s) in 4.41s
     Running `target/debug/vyom`
[2020-08-09T16:38:11Z INFO  vyom] Replied to t1_g0vfbra
[2020-08-09T16:38:11Z INFO  vyom] Marked t1_g0vfbra as read
```
Cool, but does it actually mark the message as read? Lets run the program again a couple more times and figure it out.
```shell
(base) DeltaManiac @ ~/git/rust/vyom
└─ $ cargo run
    Finished dev [unoptimized + debuginfo] target(s) in 0.10s
     Running `target/debug/vyom`
(base) DeltaManiac @ ~/git/rust/vyom
└─ $ cargo run
    Finished dev [unoptimized + debuginfo] target(s) in 0.10s
     Running `target/debug/vyom`
(base) DeltaManiac @ ~/git/rust/vyom
└─ $
```

Since there doesn't seem to to be any logs being printed we can confirm that we are not replying again to the message. But it programming and you never know if you're right until you completely verify from reddit side too. Let go take a look at the subreddit.

![Alt Text](https://dev-to-uploads.s3.amazonaws.com/i/0zxm1grbg2p41xjmmmjh.png)

Yep, it has only one comment.

## Extracting the Intel

Now that we have figured out how to respond to comments, lets get to the actual crux of the problem. 

When a VyomBot gets mentioned where should he look for the Youtube link? There can be many answers to this question like
1. The immediate parent comment of the mention
2. The title of the post
3. It could be part of the message sent to VyomBot

These all seem relevant, but to keep it simple lets start with 2, i.e if the parent is a YouTube playlist link then we fetch the information and post it as a comment.

In order to get the link of the playlist we are not going to use the roux library but instead handwrite it ourselves. Why you ask ? CAUSE ITS GONNA BE FUN!!

> Note to reader the next part of the code is kind of hacky code and do not follow idiomatic rust. Here be Dragons 🐉🐉🐉

### Constructing the Reddit Post link

We can use the `context` field of the response which looks like 
```json
{
...
...
    name: "t1_g0vfbra",
    created: 1596987973.0,
    created_utc: 1596959173.0,
    context: "/r/VyomBot/comments/i6fk15/test_playlist/g0vfbra/?context=3",
}
```
 to construct the url of the post.

The first 5 parts `r`, `VyomBot`, `comments`, `i6fk15`, `test_playlist` can be used to for the url to the post. 
Let's do this right now.

```rust
# main.rs
...
 if message.data.r#type == "username_mention" {
let post_url = format!(
    "https://www.reddit.com/{}/.json",
    message
        .data
        .context // /r/VyomBot/comments/i6fk15/test_playlist/g0vfbra/?context=3
        .trim() // remove any trailing and leading spaces
        .split('/') // [ "", "r", "VyomBot", "comments", "i6fk15", "test_playlist", "g0vfbra", "?context=3" ]
        .skip(1) // [ "r", "VyomBot", "comments", "i6fk15", "test_playlist", "g0vfbra", "?context=3" ]
        .collect::<Vec<&str>>()[0..=4] // Take the first 5 [ "r", "VyomBot", "comments", "i6fk15", "test_playlist" ]
        .join("/") // /r/VyomBot/comments/i6fk15/test_playlist/
    );
...
```

We now have constructed the url `https://www.reddit.com/{}/r/VyomBot/comments/i6fk15/test_playlist/.json`. The `.json` at the end tells reddit to return the JSON formatted  and not the HTML page of the post id specified

### Extracting the Playlist ID

In order to query the url that we crafted above we would be using the [`reqwest`](https://crates.io/crates/reqwest) crate and the [`url`](https://crates.io/crates/url) crate. We fire a GET request to reddit and extract the `url` parameter from the response body which would have our link.

We then would convert the response body to a dynamic json using the [`serde_json`](https://crates.io/crates/serde_json) crate and then extract the link from the `url` property of the response.

Then the [`url`](https://crates.io/crates/url) crate to parse and extract the playlist id from the YouTube link. For our link 
`https://www.youtube.com/playlist?list=PLf3u8NhoEikhTC5radGrmmqdkOK-xMDoZ` the playlist id is `PLf3u8NhoEikhTC5radGrmmqdkOK-xMDoZ`.

```toml
# Cargo.toml

[package]
name = "vyom"
version = "0.1.0"
authors = ["Harikrishnan Menon <harikrishnan.menon@sap.com>"]
edition = "2018"

[dependencies]
roux={path="../roux.rs"} #This points to our local modified copy
# roux={git = "https://github.com/DeltaManiac/roux.rs"} #This points to the modified version on github
dotenv_codegen="0.15.0"
tokio = {version="0.2.22", features=["macros"]}
env_logger ="0.7.1"
log = "0.4.11"
reqwest = {version="0.10.7",features=["json"]} // New
serde_json = "1.0.57" // New
url = "2.1.1"  // New

```

```rust
# main.rs
...
...
// Make an http request to the post url
let playlist_id = match reqwest::get(&post_url).await {
    // If the response is received convert it in to dynamic json
    Ok(response) => match response.json::<serde_json::Value>().await {
        Ok(json) => {
            // Get json[0]["data"]["children][0]["url}
            // NB: DO NOT USE THIS CODE IN PRODUCTION
            let url = match json
                .get(0)
                .unwrap()
                .get("data")
                .unwrap()
                .get("children")
                .unwrap()
                .get(0)
                .unwrap()
                .get("data")
                .unwrap()
                .get("url")
            {
                // Parse the youtube url from the string 
                // "https://www.youtube.com/playlist?list=PLf3u8NhoEikhTC5radGrmmqdkOK-xMDoZ"
                // after trimming `"`
                Some(url) => match url::Url::parse(
                    &url.to_string().trim_matches('\"'),
                ) {
                    Ok(url) => {
                        match (
                            // From the query parameters
                            // find the parameter with key "list"
                            url.query_pairs().find(|q| {
                                q.0 == "list"
                            }),
                            // Also check if the host is youtube
                            (url.host_str() == Some("youtube.com")
                                || url.host_str()
                                    == Some("www.youtube.com")),
                        ) {
                            (Some((_, val)), true) => {
                                // Return the url
                                Some(val.into_owned())
                            }
                            (_, _) => {
                                error!(
                                    "Couldn't find `list` param in url {} for message : {}",
                                    &url.to_string(),
                                    &message.data.name
                                );
                                None
                            }
                        }
                    }
                    // Error Handling
...

dbg!(playlist_id);
...
...
```

> A better way to handle the response is to create a struct that mimics the reponsed and just let the `.json()` method of reqwest do the heavy lifting of converting it into rust types. This will help avoid all the calls to `unwrap`. 



> The nested matches statements should be replaced by `.and_then()` for a more cleaner and readable code.

```shell
(base) DeltaManiac @ ~/git/rust/vyom
└─ $ cargo run   
Compiling vyom v0.1.0
Finished dev [unoptimized + debuginfo] target(s) in 5.09s     Running `target/debug/vyom`
[src/main.rs:200] "playlist_id" = "PLf3u8NhoEikhTC5radGrmmqdkOK-xMDoZ"
[src/main.rs:200] "playlist_id" = "PLf3u8NhoEikhTC5radGrmmqdkOK-xMDoZ"
```

Hooray! We've come a long way since we started, written some atrocious code, contributed to a library and even rolled out our own code instead of using library code to talk to talk to reddit.

I know its getting boring now and we're gonna wrap it up in the next part.

Oh BTW the code can be found on the `part-II` branch [here](https://github.com/DeltaManiac/VyomBot)

# Part III

We've reached the final chapter. The spoils are just ahead of us, lets go grab em.

# Obtaining Credentials

We need to generate a different set of credentials to talk with YouTube. Lets go do that step by step..

1. Logon to [Google Developer Console](https://console.developer.google.com)

2. Click `Enable Apis and Service`
![Enable API](https://dev-to-uploads.s3.amazonaws.com/i/hdqpmrbd1hu5hsoulzdj.png)

3. Search for `YouTube Data API v3`
![YouTube Data API v3](https://dev-to-uploads.s3.amazonaws.com/i/wl3mybevggxlqbmgvesf.png)

4. Click the `Enable` button to enable the API for our account
![Enable YouTube API](https://dev-to-uploads.s3.amazonaws.com/i/69ipwnu5ntnmz4jctdc8.png)

5. Click the `Create Credentials` button to start creating credentials for us to use.
![Create Credentials](https://dev-to-uploads.s3.amazonaws.com/i/iiu41fyb8h89h4xizu8j.png)

6. We need to first describe what kind of credentials have to be generated. Don't worry, just follow the screenshot.
![Generate Credentials](https://dev-to-uploads.s3.amazonaws.com/i/bgix1krigatizp0qbt6b.png)

7. We're Done! 
![Done](https://dev-to-uploads.s3.amazonaws.com/i/kjb0e9q1e27mi3fbzqsl.png)

# Handling JSON better

In the last part we used a dynamic JSON to to retrieve the playlist url from Reddit. This time to interact with the YouTube API we wont do that, instead we will one up ourselves and de-serialize the JSON into structs that we define in rust.
> We use [`serde`](https://crates.io/crates/serde) to do handle the heavy lifting of JSON de-serialization

The response of the YouTube API is a bit more well defined than the Reddit API.
```json
{
  "kind": "youtube#playlistItemListResponse",
  "etag": "Fij-lGuELswW5Y6HXEJsEVAZ6Xg",
  "nextPageToken": "CAUQAA",
  "items": [
    {
      "kind": "youtube#playlistItem",
      "etag": "KC_3PIeEyspbfuA_AplI4dv2ITA",
      "id": "UExmM3U4TmhvRWlraFRDNXJhZEdybW1xZGtPSy14TURvWi45ODRDNTg0QjA4NkFBNkQy",
      "snippet": {
        "publishedAt": "2020-08-05T19:31:06Z",
        "channelId": "UC9X86dyEwpbCnpC18qjt33Q",
        "title": "Rusty Days 2020 - Hackathon Submissions",
        "description": "Rules ► https://rusty-days.org/hackathon/\n\nTeams ►\narrugginiti https://github.com/Rust-Wroclaw/rd-hack-arrugginiti\nBox-Team https://github.com/Rust-Wroclaw/rd-hack-Box-Team\nBrighter3D https://github.com/Rust-Wroclaw/rd-hack-Brighter3D\nhexyoungs https://github.com/Rust-Wroclaw/rd-hack-hexyoungs\nLastMinute https://github.com/Rust-Wroclaw/rd-hack-LastMinute\nplanters https://github.com/Rust-Wroclaw/rd-hack-planters\n\nFollow ►\nFacebook: https://rusty-days.org/facebook\nTwitch: https://rusty-days.org/twitch\nTwitter: https://rusty-days.org/twitter",
        "thumbnails": {
          "default": {
            "url": "https://i.ytimg.com/vi/QaCvUKrxNLI/default.jpg",
            "width": 120,
            "height": 90
          },
          "medium": {
            "url": "https://i.ytimg.com/vi/QaCvUKrxNLI/mqdefault.jpg",
            "width": 320,
            "height": 180
          },
          "high": {
            "url": "https://i.ytimg.com/vi/QaCvUKrxNLI/hqdefault.jpg",
            "width": 480,
            "height": 360
          },
          "standard": {
            "url": "https://i.ytimg.com/vi/QaCvUKrxNLI/sddefault.jpg",
            "width": 640,
            "height": 480
          }
        },
        "channelTitle": "Rust Wrocław",
        "playlistId": "PLf3u8NhoEikhTC5radGrmmqdkOK-xMDoZ",
        "position": 0,
        "resourceId": {
          "kind": "youtube#video",
          "videoId": "QaCvUKrxNLI"
        }
      }
    },
    {
      "kind": "youtube#playlistItem",
      "etag": "EgGMmoAJ81l2BJFspcg1idaKy-8",
      "id": "UExmM3U4TmhvRWlraFRDNXJhZEdybW1xZGtPSy14TURvWi5EMEEwRUY5M0RDRTU3NDJC",
      "snippet": {
        "publishedAt": "2020-08-01T12:17:09Z",
        "channelId": "UC9X86dyEwpbCnpC18qjt33Q",
        "title": "Rusty Days 2020 - Tim McNamara: How 10 open source projects manage unsafe code",
        "description": "Agenda ► https://rusty-days.org/agenda\nSlides ►https://rusty-days.org/assets/slides/08-how-10-open-source-projects-manage-unsafe-code.pdf\nPlaylist with all talks ► https://www.youtube.com/playlist?list=PLf3u8NhoEikhTC5radGrmmqdkOK-xMDoZ\n\nFollow ►\nFacebook: https://rusty-days.org/facebook\nTwitch: https://rusty-days.org/twitch\nTwitter: https://rusty-days.org/twitter\n\nThis video ►\nIs it safe to use unsafe? Learn why some projects need unsafe code and how projects manage its risks.\n\nThis talk will briefly discuss what the unsafe keyword enables and what its risks are. The bulk of time will be spent discussing how projects manage those risks. It finishes by providing recommendations based on that analysis.\n\nProjects surveyed include:\n* Servo (Mozilla)\n* Fuchsia OS (Google)\n* fast_rsync (Dropbox)\n* winrt-rs (Microsoft)\n* Firecracker (AWS)\n* Linkerd2",
        "thumbnails": {
          "default": {
            "url": "https://i.ytimg.com/vi/9M0NQI5Cp2c/default.jpg",
            "width": 120,
            "height": 90
          },
          "medium": {
            "url": "https://i.ytimg.com/vi/9M0NQI5Cp2c/mqdefault.jpg",
            "width": 320,
            "height": 180
          },
          "high": {
            "url": "https://i.ytimg.com/vi/9M0NQI5Cp2c/hqdefault.jpg",
            "width": 480,
            "height": 360
          },
          "standard": {
            "url": "https://i.ytimg.com/vi/9M0NQI5Cp2c/sddefault.jpg",
            "width": 640,
            "height": 480
          },
          "maxres": {
            "url": "https://i.ytimg.com/vi/9M0NQI5Cp2c/maxresdefault.jpg",
            "width": 1280,
            "height": 720
          }
        },
        "channelTitle": "Rust Wrocław",
        "playlistId": "PLf3u8NhoEikhTC5radGrmmqdkOK-xMDoZ",
        "position": 1,
        "resourceId": {
          "kind": "youtube#video",
          "videoId": "9M0NQI5Cp2c"
        }
      }
    },
    {
      "kind": "youtube#playlistItem",
      "etag": "3gm-0cEUcjfm1v1vgh_3EjS6mJg",
      "id": "UExmM3U4TmhvRWlraFRDNXJhZEdybW1xZGtPSy14TURvWi40NzZCMERDMjVEN0RFRThB",
      "snippet": {
        "publishedAt": "2020-08-01T12:16:07Z",
        "channelId": "UC9X86dyEwpbCnpC18qjt33Q",
        "title": "Rusty Days 2020 - Luca Palmieri: Are we observable yet?",
        "description": "Agenda ► https://rusty-days.org/agenda\nSlides ►https://rusty-days.org/assets/slides/07-are-we-observable-yet.pdf\nPlaylist with all talks ► https://www.youtube.com/playlist?list=PLf3u8NhoEikhTC5radGrmmqdkOK-xMDoZ\n\nFollow ►\nFacebook: https://rusty-days.org/facebook\nTwitch: https://rusty-days.org/twitch\nTwitter: https://rusty-days.org/twitter\n\nThis video ►\nIs Rust ready for mainstream usage in backend development?\n\nThere is a lot of buzz around web frameworks while many other (critical!) Day 2 concerns do not get nearly as much attention.\n\nWe will discuss observability: do the tools currently available in the Rust ecosystem cover most of your telemetry needs?\n\nI will walk you through our journey here at TrueLayer when we built our first production backend system in Rust, Donate Direct.\n\nWe will be touching on the state of Rust tooling for logging, metrics and distributed tracing.",
        "thumbnails": {
          "default": {
            "url": "https://i.ytimg.com/vi/HtKnLiFwHJM/default.jpg",
            "width": 120,
            "height": 90
          },
          "medium": {
            "url": "https://i.ytimg.com/vi/HtKnLiFwHJM/mqdefault.jpg",
            "width": 320,
            "height": 180
          },
          "high": {
            "url": "https://i.ytimg.com/vi/HtKnLiFwHJM/hqdefault.jpg",
            "width": 480,
            "height": 360
          },
          "standard": {
            "url": "https://i.ytimg.com/vi/HtKnLiFwHJM/sddefault.jpg",
            "width": 640,
            "height": 480
          },
          "maxres": {
            "url": "https://i.ytimg.com/vi/HtKnLiFwHJM/maxresdefault.jpg",
            "width": 1280,
            "height": 720
          }
        },
        "channelTitle": "Rust Wrocław",
        "playlistId": "PLf3u8NhoEikhTC5radGrmmqdkOK-xMDoZ",
        "position": 2,
        "resourceId": {
          "kind": "youtube#video",
          "videoId": "HtKnLiFwHJM"
        }
      }
    },
    {
      "kind": "youtube#playlistItem",
      "etag": "2C9sX2xuTowOxjn0m95AH53JiA4",
      "id": "UExmM3U4TmhvRWlraFRDNXJhZEdybW1xZGtPSy14TURvWi5GNjNDRDREMDQxOThCMDQ2",
      "snippet": {
        "publishedAt": "2020-07-31T11:28:34Z",
        "channelId": "UC9X86dyEwpbCnpC18qjt33Q",
        "title": "Rusty Days 2020 - Jan-Erik Rediger: Leveraging Rust to build cross-platform mobile libraries",
        "description": "Agenda ► https://rusty-days.org/agenda\nSlides ►https://rusty-days.org/assets/slides/06-cross-platform-mobile-libraries.pdf\nPlaylist with all talks ► https://www.youtube.com/playlist?list=PLf3u8NhoEikhTC5radGrmmqdkOK-xMDoZ\n\nFollow ►\nFacebook: https://rusty-days.org/facebook\nTwitch: https://rusty-days.org/twitch\nTwitter: https://rusty-days.org/twitter\n\n\nThis video ►\nAt Mozilla, Firefox is not the only product we ship. Many others — including a variety of smartphone applications, and certainly not just web browsers — are built by various teams across the organization. These applications are composed of a multitude of libraries which, when possible, are reused across platforms.\n\nIn the past year we used Rust to rebuild one of these libraries: the library powering the telemetry in our mobile applications is now integrated into Android and iOS applications and will soon be powering our Desktop platforms as well.\n\nThis talk will showcase how this small team managed to create a cross-platform Rust library, and ship it to a bunch of platforms all at once.",
        "thumbnails": {
          "default": {
            "url": "https://i.ytimg.com/vi/j5rczOF7pzg/default.jpg",
            "width": 120,
            "height": 90
          },
          "medium": {
            "url": "https://i.ytimg.com/vi/j5rczOF7pzg/mqdefault.jpg",
            "width": 320,
            "height": 180
          },
          "high": {
            "url": "https://i.ytimg.com/vi/j5rczOF7pzg/hqdefault.jpg",
            "width": 480,
            "height": 360
          },
          "standard": {
            "url": "https://i.ytimg.com/vi/j5rczOF7pzg/sddefault.jpg",
            "width": 640,
            "height": 480
          },
          "maxres": {
            "url": "https://i.ytimg.com/vi/j5rczOF7pzg/maxresdefault.jpg",
            "width": 1280,
            "height": 720
          }
        },
        "channelTitle": "Rust Wrocław",
        "playlistId": "PLf3u8NhoEikhTC5radGrmmqdkOK-xMDoZ",
        "position": 3,
        "resourceId": {
          "kind": "youtube#video",
          "videoId": "j5rczOF7pzg"
        }
      }
    },
    {
      "kind": "youtube#playlistItem",
      "etag": "pB_4gb7ai1HOgVLz8Jx9SJB1P_g",
      "id": "UExmM3U4TmhvRWlraFRDNXJhZEdybW1xZGtPSy14TURvWi45NDk1REZENzhEMzU5MDQz",
      "snippet": {
        "publishedAt": "2020-07-31T09:06:09Z",
        "channelId": "UC9X86dyEwpbCnpC18qjt33Q",
        "title": "Rusty Days 2020 -  Nell Shamrell - Harrington: The Rust Borrow Checker - A Deep Dive",
        "description": "Agenda ► https://rusty-days.org/agenda\nSlides ►https://rusty-days.org/assets/slides/05-the-rust-borrow-checker.pdf\nPlaylist with all talks ► https://www.youtube.com/playlist?list=PLf3u8NhoEikhTC5radGrmmqdkOK-xMDoZ\n\nFollow ►\nFacebook: https://rusty-days.org/facebook\nTwitch: https://rusty-days.org/twitch\nTwitter: https://rusty-days.org/twitter\n\nThis video ►\n\nThe Rust compiler's borrow checker is critical for ensuring safe Rust code. Even more critical, however, is how the borrow checker provides useful, automated guidance on how to write safe code when the check fails. \n\nEarly in your Rust journey, it may feel like you are fighting the borrow checker. Come to this talk to learn how you can transition from fighting the borrow checker to using its guidance to write safer and more powerful code at any experience level. Walk away not only understanding the what and the how of the borrow checker - but why it works the way it does - and why it is so critical to both the technical functionality and philosophy of Rust.",
        "thumbnails": {
          "default": {
            "url": "https://i.ytimg.com/vi/knhpe5IUnlE/default.jpg",
            "width": 120,
            "height": 90
          },
          "medium": {
            "url": "https://i.ytimg.com/vi/knhpe5IUnlE/mqdefault.jpg",
            "width": 320,
            "height": 180
          },
          "high": {
            "url": "https://i.ytimg.com/vi/knhpe5IUnlE/hqdefault.jpg",
            "width": 480,
            "height": 360
          },
          "standard": {
            "url": "https://i.ytimg.com/vi/knhpe5IUnlE/sddefault.jpg",
            "width": 640,
            "height": 480
          },
          "maxres": {
            "url": "https://i.ytimg.com/vi/knhpe5IUnlE/maxresdefault.jpg",
            "width": 1280,
            "height": 720
          }
        },
        "channelTitle": "Rust Wrocław",
        "playlistId": "PLf3u8NhoEikhTC5radGrmmqdkOK-xMDoZ",
        "position": 4,
        "resourceId": {
          "kind": "youtube#video",
          "videoId": "knhpe5IUnlE"
        }
      }
    }
  ],
  "pageInfo": {
    "totalResults": 9,
    "resultsPerPage": 5
  }
}
```
This JSON response can be constructed from simple structs that we can define.
The `#[derive(Deserialize)]` helps `serde` understand that it can use this struct to deserialize json into by matching the fields of the struct to those of that in the JSON body.
> `serde` is an amazing library and a bit too vast to explain in this post.

```rust
# main.rs

#[derive(Debug, Deserialize)]
struct Snippet {
    title: String,
    position: i32,
}
#[derive(Debug, Deserialize)]
struct Item {
    kind: String,
    snippet: Snippet,
}
#[derive(Debug, Deserialize)]
struct YoutubeResponse {
    items: Vec<Item>,
}
```

Now that we have defined our struct lets go ahead and call the YouTube API.
```rust
# main.rs

let mut reply: String =
    "Sorry couldn't find the YouTube Link! :(".to_string();

if playlist_id.is_some() {
    // Generate api url
    let url = format!("https://www.googleapis.com/youtube/v3/playlistItems?part=snippet&playlistId={}&key={}&maxResults={}",playlist_id.unwrap(), YT_KEY, YT_MAX_RESULT);
                        // Fire the API
    let playlist_items = match reqwest::get(&url).await {
                        // Try to convert the response to our struct
        Ok(response) => match response.json::<YoutubeResponse>().await {
                               // Return the array of Item            Ok(yt_response) => Some(yt_response.items),
            Err(e) => {
                error!(
                    "Couldn't parse playlist response for comment {} reason : {}",
                        &message.data.name, e
                            );
                None
            }
        },
        Err(e) => {
            error!(
                "Couldn't fetch YouTube data for comment {} reason : {}",
                &message.data.name, e
            );
            None
        }
    };
    //Loop over each item and then create the message.
    if playlist_items.is_some() {
        let items = playlist_items.unwrap();
        if items.len() > 0 {
            reply = "Playlist Items: \n".to_string();
            for item in items {
                reply.push_str(
                    format!("\n {} \n", item.snippet.title).as_str(),
                )
            }
        }
    }
}
```

We first define `reply` with the string that we want to respond with if we fail to identify the playlist id.
If we have a playlistID we then call YouTube API with the key we generated earlier. We then extract the items of the playlist generated our reply text. The code that we have written in the previous part already handles replying to the message.
Lets try it out!
```shell
(base) DeltaManiac @ ~/git/rust/vyom
└─ $ cargo run
   Compiling vyom v0.1.0
    Finished dev [unoptimized + debuginfo] target(s) in 5.79s
     Running `target/debug/vyom`
[2020-08-11T18:46:46Z INFO  vyom] Replied to t1_g14nya9
[2020-08-11T18:46:46Z INFO  vyom] Marked t1_g14nya9 as read
```
And on Reddit it looks just as beautiful!
![Final](https://dev-to-uploads.s3.amazonaws.com/i/poxb0f9lmqtufptsvo4j.png)

![](https://media.giphy.com/media/3oKIPf3C7HqqYBVcCk/giphy.gif)

# Fin
Thanks for joining along while we built our first bot :heart:.
If this journey has taught you something, feel free to give a shout out!

[//begin]: # "Autogenerated link references for markdown compatibility"
[rust]: rust "Rust"
[//end]: # "Autogenerated link references"
