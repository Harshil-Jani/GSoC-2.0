# Version-2 : Send DM from qaul to matrix

The very first part of this work was to figure out how we can create a new matrix room based on willingness of a Qaul user. We get RPC event for invited groups in Qaul. The steps are simple
- Qaul user creates a new group with the matrix ID of matrix user.
- Invites the Qaul Bridge user into the group. 
- Bridge user accepts the invitation automatically and joins the group.
- Bridge user triggers the room creation on matrix and invites the matrix user.

This is very simple auto-joining and invitation part in Qaul which can be explained as below.

```rust
_invited_ticker = invited_fut => {
    group::Group::group_invited();
    None
}
```
This ticker keeps running after few miliseconds and we keep listening to new group invitation on the bridge user.

Once the invitation is received we invite matrix user into the group by creating a new room. But we always check that the user is an admin of the qaul group who is trying to create a new room. We actually also have a similar check on Matrix side too where we check the person inviting is admin or not. If admin then only we proceed further.

```rust
let all_groups = group_list_response.groups.clone();
let mut config = MATRIX_CONFIG.get().write().unwrap();
for group in all_groups {
    // If Mapping exist let it be. Else create new room.
    let group_id =
        uuid::Uuid::from_bytes(group.group_id.try_into().unwrap());
    // qaul_groups.insert(group_id, group.group_name.clone());
    let mut qaul_room_admin = format!("@qaul://{}", "[username]");
    for member in group.members {
        if member.role == 255 {
            let user_id = PeerId::from_bytes(&member.user_id).unwrap();
            qaul_room_admin.push_str(&user_id.to_string());
        }

    // If group mapping does not exist
    // If group_name contains matrix user name then only do this.
    if let Ok(user) = UserId::try_from(group.group_name.clone()) {
    if !config.room_map.contains_key(&group_id) {
        let matrix_client = MATRIX_CLIENT.get();
        let rt = Runtime::new().unwrap();
        rt.block_on(async {
            println!("{:#?}", group_id);
            // Check if user exist on matrix
            // Create a group on matrix with qaul user name.
            let mut request = CreateRoomRequest::new();
            let room_name =
                RoomNameBox::try_from(qaul_room_admin).unwrap();
            request.name = Some(&room_name);
            let room_id = matrix_client
                .create_room(request)
                .await
                .expect("Room creation failed")
                .room_id;

            // Check if the room is joined
            if let Some(joined_room) = matrix_client.get_joined_room(&room_id) {
                joined_room.invite_user_by_id(&user).await.unwrap();
            } else {
                println!("Wait till the bot joins the room");
            }

            // Save things to Config file
            let room_info = MatrixRoom {
                matrix_room_id: room_id,
                qaul_group_name: group.group_name,
                last_index: 0,
            };
            config.room_map.insert(group_id, room_info);
            MatrixConfiguration::save(config.clone());
                });
            }
        }
    }
}
```

The second part of this Direct communication messages was to allow the messages sent from the qaul private DM into the matrix world.

Luckily, In qaul we already had a mechanism to detect the message in a qaul group over the network. Each message would contain its own meta-data like the group-id, sender-id and the message content. We can map with the group-id to check in which group it is contained in and from our matrix configuration we can figure the matrix room which contains the mapping and send the message over the network.

The qaul way of detecting the message is that for every few milliseconds we constantly poll over the message which we receive on our network
```rust
if let Ok(config) = MATRIX_CONFIG.get().read() {
    group::Group::group_list();
    let qaul_groups: Vec<Uuid> = config.room_map.keys().cloned().collect();

    // Check unread messages from Libqaul groups
    for group in qaul_groups {
        let matrix_room = config.room_map.get(&group).unwrap();
            let last_index_grp = matrix_room.last_index;
        let group_id = group.as_bytes().to_vec();
        chat::Chat::request_chat_conversation(group_id,last_index_grp);
    }
} else {
    println!("Waiting for the configuration to Sync")
}
None
```

Here the function method `request_chat_conversation()` is responsible for giving us the list of unread messages in a given group for all the groups in network.

From the RPC event of all the unread messages which we receive, We extract the details for formatting our matrix message and prepare it to send it on the matrix bridge. 
Details like `Qaul User Name`, `Mapped Room ID` and `Message Content` are used to be sent over the network using `matrix_send()` method which I will elaborate below after the following snippet.

```rust
Some(proto::chat::Message::ConversationList(proto_conversation)) => {
    // Conversation table
    let group_id =
        uuid::Uuid::from_bytes(proto_conversation.group_id.try_into().unwrap());
    let mut config = MATRIX_CONFIG.get().write().unwrap();
    if !config.room_map.contains_key(&group_id) {
        println!("No Mapping found");
    } else {
        let matrix_room = config.room_map.get_mut(&group_id).unwrap();
        let last_index_grp = matrix_room.last_index;
        let room_id = matrix_room.clone().matrix_room_id;
        for message in proto_conversation.message_list {
            if message.index > last_index_grp {
                if let Ok(ss) = Self::analyze_content(&message, &room_id) {
                    print! {"{} | ", message.index};
                    // message.sender_id is same as user.id
                    match proto::MessageStatus::from_i32(message.status)
                        .unwrap()
                    {
                        proto::MessageStatus::Sending => print!(".. | "),
                        proto::MessageStatus::Sent => print!("✓. | "),
                        proto::MessageStatus::Confirmed => print!("✓✓ | "),
                        proto::MessageStatus::ConfirmedByAll => print!("✓✓✓| "),
                        proto::MessageStatus::Receiving => print!("🚚 | "),
                        proto::MessageStatus::Received => print!("📨 | "),
                    }

                    print!("{} | ", message.sent_at);
                    let users = QAUL_USERS.get().read().unwrap();
                    println!("{:#?}", users);
                    let sender_id =
                        bs58::encode(message.sender_id).into_string();
                    println!("{}", sender_id);
                    let user_name =
                        Self::find_user_for_given_id(users.clone(), sender_id)
                            .unwrap();
                    println!(
                        " [{}] {}",
                        bs58::encode(message.message_id).into_string(),
                        message.received_at
                    );

                    for s in ss {
                        // This part is mapped with the matrix room.
                        // Allow inviting the users or removing them.
                        Self::matrix_send(&s, &room_id, user_name.clone());
                        println!("\t{}", s);
                    }
                    println!("");
                    matrix_room.update_last_index(message.index);
                }
            }
        }
        MatrixConfiguration::save(config.clone());
    }
}
```

This is how we analyse an unread message in qaul group and then send to matrix bridge using `matrix_send()` method.

```rust
fn matrix_send(message: &String, room_id: &RoomId, user: String) {
    // Get the Room based on RoomID from the client information
    let matrix_client = MATRIX_CLIENT.get();
    let room = matrix_client.get_room(&room_id).unwrap();
    // Check if the room is already joined or not
    if let Room::Joined(room) = room {
        // Build the message content to send to matrix
        let final_msg = format!("{} : {}", user, message);
        let content = AnyMessageEventContent::RoomMessage(MessageEventContent::text_plain(final_msg));

        let rt = Runtime::new().unwrap();
        rt.block_on(async {
            // Sends messages into the matrix room
            room.send(content, None).await.unwrap();
        });
    }
}
```

With this chunk of code, We have successfully achieved the integration of private group messages from both ends Qaul->Matrix and Matrix->Qaul.
