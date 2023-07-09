# Version-1 : The real bridge

## Receiving message from Matrix in Qaul

In the Version-0 we have already a functionality where we were able to listen into the incoming Matrix messages and for this version, We are not just replying back to matrix but instead storing the message from matrix inside of our local peer network.

To do this, We were listening to matrix messages in qaul using `on_room_message` function as seen in Version 0. Now we are extending it with another function `send_qaul` which collects the message content and message sender and sends the information in the qaul where we send it ahead inside of the broadcast feed.

```rust
async fn on_room_message(event: SyncMessageEvent<MessageEventContent>, room: Room) {
    ...
    let msg_text = format!("{} : {}", msg_sender, msg_body);
    send_qaul(msg_text, room.room_id());
}
```

```rust
fn send_qaul(msg_text: String, room_id: &RoomId) {
    // Protobuff for feed messages
    let proto_message = proto::Feed {
        message: Some(proto::feed::Message::Send(proto::SendMessage {
            content: msg_text,
        })),
    };

    // Encode message
    let mut buf = Vec::with_capacity(proto_message.encoded_len());
    proto_message
        .encode(&mut buf)
        .expect("Vec<u8> provides capacity as needed");
    // Send the message in qaul feed
    Rpc::send_message(buf, super::rpc::proto::Modules::Feed.into(), "".to_string());
}
```

This registers all the message in our feed. Now this closes our part of communication from a Matrix Room into the Qaul feed.

## Sending from Qaul to Matrix

This was interesting part since we had to change design pattern multiple times in order to keep things simple and less fancy. The very first attempt was to send the message inside of the matrix room when someone sends the feed message from the CLI. 

In qaul you can send the RPC feed message from CLI via `feed send {msg}` and so what I did is that where the RPC was listening for the qaul feed message I have implemented a function which would send the message in matrix.

```rust
fn matrix_send(message: String) {
    // Get the Room based on RoomID from the client information
    let matrix_client = MATRIX_CLIENT.get();
    let room_id = RoomId::try_from("!nGnOGFPgRafNcUAJJA:matrix.org").unwrap();
    let room = matrix_client.get_room(&room_id).unwrap();
    // Check if the room is already joined or not
    if let Room::Joined(room) = room {
        // Build the message content to send to matrix
        let content = AnyMessageEventContent::RoomMessage(MessageEventContent::text_plain(
            message,
        ));
 
        let rt = Runtime::new().unwrap();
        rt.block_on(async {
            // Sends messages into the matrix room
            room.send(content, None).await.unwrap();
        });
    }
}
```
And we send the message for each incoming feed message which is decoded in RPC as below.
```rust
/// Decodes received protobuf encoded binary RPC message
/// of the feed module.
pub fn rpc(data: Vec<u8>) {
    match proto::Feed::decode(&data[..]) {
        Ok(feed) => {
            match feed.message {
                Some(proto::feed::Message::Received(proto_feedlist)) => {
                    // print all messages in the feed list
                    for message in proto_feedlist.feed_message {
                       ...
                       // Send message to matrix
                            Self::matrix_send(message.content);
                        }      
                    }
                }
                _ => {
                    log::error!("unprocessable RPC feed message");
                },
            }    
        },
        Err(error) => {
            log::error!("{:?}", error);
        },
}
```

The most challanging part for this to happen is with the matrix configuration object state which we need in `matrix_send` because you need to information about the account which would send the message and if it is joined in the room or not ? We had real troubles working with the client to be shared from relay_bot into our feed since If we decide to change the RPC structure to take configuration as object then it may need a major version tweak and would break other binaries which don't take this third argument. We were more concerned about not changing the `libqaul` and avoid it as far as we can.

The best solution to this was suggested very correctly by MathJud and this really helped me solve this chicken and Egg problem. The solution was about using a global storage object stack. So we store the configuration from Matrix Client inside of a storage 

```rust
// Setup a storage object for the Client to make it available globally
pub static MATRIX_CLIENT: state::Storage<Client> = state::Storage::new();

async fn login(
    homeserver_url: &str,
    username: &str,
    password: &str,
) -> Result<(), matrix_sdk::Error> {
   ...
    MATRIX_CLIENT.set(client.clone());
}
```
and now we are able to access this.

## Serious design issue

Remember, Earlier I said we were trying to send message to matrix only when we write a command `feed send {msg}`. But what if the user on our peer network sends this message and is not from the bot account ? This was a serious compatibility issue about which I was unaware. My mentor actually figured this out and then we came to discussion that for this to happen, We may need to constantly listen for all incoming qaul message and send as soon as we receive a message instead of when we send it from CLI. Luckily, Receiving of message also takes place via RPC. And so the code was at right place but only thing was to listen for the message. 

To this front, I have created a ticker which constantly looks every 2 seconds if there is any incoming message from qaul or not ? If there is a message then we can send it to qaul. But it had a new problem. Now, It sends all the messages from qaul because whenever we restart the client, the last_index goes back to 0. 

The solution to this problem was a configuration file.  Next I went to design a configuration file which takes certain things as input and keep track of last message. In next chapter I will go about how I made a configuration file to work with qaul.

Another challenging part was that if you look in `matrix_send` The sending of message to matrix is an asyncronous callback and thus we would need to await it. Again In order to await, We need matrix_rpc function as async. Which would further require the top level RPC function to be async and this would again break the libqaul for other binaries. So, MathJud again helped me with the issue and gave me a link to very good blog post and I read it and integrated the tokio runtime block async in the function. I would also say that we tried async block but since matrix is a tokio async so standard async block was not enough.

Link to async blog : [Async and Sync in Rust](https://greptime.com/blogs/2023-03-09-bridging-async-and-sync-rust)