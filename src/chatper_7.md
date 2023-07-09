# Version-2 : Group/Room Creation

There are two aspects which we want to listen in qaul and act accordingly on matrix.

1. Listening for creation of new peer groups.
2. Listening for messages in peer network connected to bot account.

## Group Creation

The use-case for this part was that we would create a new group with the name of matrix user and the bot should have powers to open up the new matrix room with the name of qaul users.

I have created a group ticker in order to do this just like feed ticker and extended our configuration for a mapping between

```
Qaul group ID
    - Matrix-Room-Id
    - Last index for a given group
    - Qaul group name/Matrix User name
```

Now, We listen for each new group. If any new group is created and detected by group ticker, We open a matrix room with the name of qaul group admin and then invite the matrix user with whome the qaul user wanted to chat.

```rust
    let mut group_ticker = Ticker::new(Duration::from_secs(2));
    loop {
        let evt = {
            let group_fut = group_ticker.next().fuse();
            pin_mut!(group_fut);
            select! {
                _group_ticker = group_fut => {
                    // Checks every two seconds if any new group is created or not
                    group::Group::group_list();
                    None
                }
            }
        };
    }
```

```rust
let mut config = MATRIX_CONFIG.get().write().unwrap();
for group in all_groups {
    let group_id =
        uuid::Uuid::from_bytes(group.group_id.try_into().unwrap());
    let mut qaul_room_admin = format!("@qaul://{}","");
    for member in group.members {
        // Member who is admin
        if member.role == 255 {
            let user_id = PeerId::from_bytes(&member.user_id).unwrap();
            qaul_room_admin.push_str(&user_id.to_string());
        }
    }

    // If the group is not stored in the configuration then create matrix room.
    if !config.room_map.contains_key(&group_id) {
        let matrix_client = MATRIX_CLIENT.get();
        let rt = Runtime::new().unwrap();
        rt.block_on(async {
            // Creating new room
            let mut request = CreateRoomRequest::new();
            let room_name =
                RoomNameBox::try_from(qaul_room_admin).unwrap();
            request.name = Some(&room_name);
            let room_id = matrix_client
                .create_room(request)
                .await
                .expect("Room creation failed");

            // Invite matrix user
            matrix_client.get_joined_room(&room_id).unwrap().invite_user_by_id(&UserId::try_from(group.group_name.clone()).unwrap()).await.unwrap();

            // Send the (matrix roomID, qaul group name, last_index) into the below HashMap
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
```

We have also extended our matrix incomming message function `on_room_message` for listening to the incoming message from a given matrix room and then we check which room has sent the message from the Map and in qaul we store message in respective group only. So no one can intrude to know that the message was apart from the group admin. If the message is received from a room which is not inside our map then we send it to our feed since without mapping no matrix room should fundamentally exist and it is meant as a broadcast message.

```rust
fn send_qaul(msg_text: String, room_id: &RoomId) {
    let mut config = MATRIX_CONFIG.get().write().unwrap();
    let qaul_id = find_key_for_value(config.room_map.clone(), room_id.clone());
    if qaul_id.is_some() {
        // send message to qaul group
        // create group send message
        // logic here
    } else {
        // send to feed from matrix
        // logic here
    }
    MatrixConfiguration::save(config.clone()); // save config back since in logic we might already update last index for group/feed.
}
```

With this we have completed another functionality where we are able to 
1. Create private DMs between a qaul user and matrix user.
2. Matrix user sending message to qaul user.
