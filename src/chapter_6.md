# Version-1 : Configuring bridge

## Structure of Config File

```yaml
relay_bot:
  homeserver: https://matrix.org
  bot_id: qaul-bot
  bot_password: not-gonna-tell

feed:
  last_index: 25
```

Here the most important part is with the last_index. It helps in two ways. One is while we run our ticker which looks for new messages then on restarting it is not 0. So, We actually only send newer messages. Another benefit is that we can avoid the duplication of the matrix message. Since the incoming message from matrix is stored inside the qaul feed we increase the index so that message is not sent back into our feed.

Again use follow the same approach to store the configuration inside of a Storage object and then use it globally across the binary.

```rust
pub static MATRIX_CONFIG: state::Storage<RwLock<MatrixConfiguration>> = state::Storage::new();

pub async fn connect() -> Result<(), matrix_sdk::Error> {
    println!("Connecting to Matrix Bot");

    // Configuration for starting of the bot
    let path_string = Storage::get_path();
    let path = Path::new(path_string.as_str());
    let config_path = path.join("matrix.yaml");
    let config: MatrixConfiguration = match Config::builder()
        .add_source(File::with_name(&config_path.to_str().unwrap()))
        .build()
    {
        Err(_) => {
            log::error!("no configuration file found, creating one.");
            MatrixConfiguration::default()
        }
        Ok(c) => c.try_deserialize::<MatrixConfiguration>().unwrap(),
    };

    MATRIX_CONFIG.set(RwLock::new(config.clone()));
    login(
        &config.relay_bot.homeserver,
        &config.relay_bot.bot_id,
        &config.relay_bot.bot_password,
    )
    .await?;
    Ok(())
}
```

and we can now use this matrix configuration to take care of last_index and feed messages

#**`request_feed_list` Implementation**
```rust
let mut config = MATRIX_CONFIG.get().write().unwrap();
let last_index_matrix = config.feed.last_index;
// print all messages in the feed list.
for message in proto_feedlist.feed_message {
    print!{"[{}] ", message.index}; // This is what we are getting from ticker logic as can be seen below
    println!("Time Sent - {}", message.time_sent);
    println!("Timestamp Sent - {}", message.timestamp_sent);
    println!("Time Received - {}", message.time_received);
    println!("Timestamp Received - {}", message.timestamp_received);
    println!("Message ID {}", message.message_id_base58);
    println!("From {}", message.sender_id_base58);
    println!("\t{}", message.content);
    println!("");
    // Only send if the message index is greater than last known index.
    if message.index> last_index_matrix {
        Self::matrix_send(message.content);
        config.feed.last_index = message.index;
        // save the configuration file locally.
        MatrixConfiguration::save(config.clone());
    }
}
// set the configuration back to object.
MATRIX_CONFIG.set(config.clone().into());
```

## Ticker logic

```rust
let mut feed_ticker = Ticker::new(Duration::from_secs(2));
 loop {
        let evt = {
            let feed_fut = feed_ticker.next().fuse();
            pin_mut!(feed_fut);
            select! {
                _feed_ticker = feed_fut => {
                    let config = MATRIX_CONFIG.get().read().unwrap();
                    let last_index = &config.feed.last_index;
                    // Check unread messages from Libqaul
                    feed::Feed::request_feed_list(*last_index);
                    None
                }
            }
        };
 }
```
The function `request_feed_list` allows us to know send messages to matrix and print in qaul. This is the logic which you can see in the above snippet.

By this time, We were able to close our communication completely and reliabaly from 
1. Matrix Room to Qaul Feed
2. Qaul Feed to Matrix Room
