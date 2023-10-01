# Version-2 : Polishing with user menu

Now we were all set functionality wise to launch the very first usable bot version on the qaul community. To improve it further for the users, I have worked a bit on improving the logs on the binary, write comments in code, format the response messages to be sent over the matrix.

One major thing which we did was to improve the matrix menu for the matrix users by allowing them to write the commands to trigger qaul events from matrix. 

```markdown
!qaul : Ping to check if the bot is active or not.
!help : Get the help menu in matrix for possible events.
!users : Get list of all the users on the network.
!invite {qaul_user_id} : To invite a user from the qaul into this matrix room.
!remove {qaul_user_id} : To remove a user from the qaul into this matrix room.
!group-info : Get details for the qaul group with which this matrix room is connected.
```

Here is the code explaining how we took the commands from matrix world and processed things in Qaul world.

```rust
// We don't consider message in matrix from the bot
// since it would be the response being sent from qaul.
if msg_sender != bot_matrix_id {
    let msg_text = format!("{} : {}", msg_sender, msg_body);

    // Send the text to qaul to process the incoming matrix message
    send_qaul(msg_text, room.room_id());

    // on receiving !qaul from matrix, Send message
    if msg_body.contains("!qaul") {
        let content = AnyMessageEventContent::RoomMessage(
            MessageEventContent::text_plain(
                "I am a message sent from qaul network\n",
            ),
        );
        room.send(content, None).await.unwrap();
    }

    // on receiving !help from matrix, Give brief of all possible commands.
    if msg_body.contains("!help") {
        let content = AnyMessageEventContent::RoomMessage(MessageEventContent::text_plain(
            "!qaul : Ping to check if the bot is active or not.\n!users : Get list of all the users on the network.\n!invite {qaul_user_id} : To invite a user from the qaul into this matrix room.\n !remove {qaul_user_id} : To remove a user from the qaul into this matrix room.\n!group-info : Get details for the qaul group with which this matrix room is connected.",
        ));
        room.send(content, None).await.unwrap();
    }

    // on receiving !invite in matrix
    if msg_body.contains("!invite") {
        let matrix_user =
            room.get_member(&msg_sender).await.unwrap().unwrap();
        // Admin Powers
        if matrix_user.power_level() == 100 {
            let mut iter = msg_body.split_whitespace();
            let _command = iter.next().unwrap();
            // TODO : Try to return an error if userID is wrong.
            let qaul_user_id = iter.next().unwrap().to_string();
            let room_id_string = room.room_id().to_string();
            let sender_string = msg_sender.to_string();
            let request_id = format!(
                "invite#{}#{}#{}",
                room_id_string, sender_string, qaul_user_id
            );
            println!("{}", request_id);
            // Create group only if the mapping between a qaul grp and matrix room doesn't exist.
            // If it exist then please check if user already exist or not. If not then invite
            let config = MATRIX_CONFIG.get().write().unwrap().clone();
            let room_id = room.room_id();
            let qaul_group_id: Option<Uuid> = find_key_for_value(
                config.room_map.clone(),
                room_id.clone(),
            );
            if qaul_group_id == None {
                group::Group::create_group(
                    format!("{}", msg_sender.to_owned()).to_owned(),
                    request_id,
                );
                // Acknowledge about sent invitation to qaul user.
                let content = AnyMessageEventContent::RoomMessage(
                MessageEventContent::text_plain("User has been invited. Please wait until user accepts the invitation."),
            );
                room.send(content, None).await.unwrap();
            } else {
                // Get the list of users who are members to the given room.
                group::Group::group_info(
                    chat::Chat::uuid_string_to_bin(
                        qaul_group_id.unwrap().to_string(),
                    )
                    .unwrap(),
                    request_id,
                );
                println!("The Room Mapping already exist for this room");
                // Else Invite the given user in same mapping of the matrix room.
            }
        } else {
            // Not Admin
            let content = AnyMessageEventContent::RoomMessage(
                MessageEventContent::text_plain(
                    "Only Admins can perform this operation.",
                ),
            );
            room.send(content, None).await.unwrap();
        }
    }

    // on receiving !users-list in matrix, Send it to command line
    if msg_body.contains("!users") {
        users::Users::request_user_list(room.room_id().to_string());
    }

    // remove the people from the matrix room
    if msg_body.contains("!remove") {
        let matrix_user =
            room.get_member(&msg_sender).await.unwrap().unwrap();
        // Admin Powers
        if matrix_user.power_level() == 100 {
            let mut iter = msg_body.split_whitespace();
            let _command = iter.next().unwrap();
            // TODO : Try to return an error if userID is wrong.
            let qaul_user_id = iter.next().unwrap().to_string();
            let room_id_string = room.room_id().to_string();
            let sender_string = msg_sender.to_string();
            let request_id = format!(
                "remove#{}#{}#{}",
                room_id_string, sender_string, qaul_user_id
            );
            println!("{}", request_id);

            let config = MATRIX_CONFIG.get().write().unwrap().clone();
            let room_id = room.room_id();
            let qaul_group_id: Option<Uuid> = find_key_for_value(
                config.room_map.clone(),
                room_id.clone(),
            );
            if qaul_group_id == None {
                // No room mapping exist
                let content =
                    AnyMessageEventContent::RoomMessage(MessageEventContent::text_plain(
                        "No qaul group is mapped to this Matrix room. Please invite qaul users to this room.",
                    ));
                room.send(content, None).await.unwrap();
            } else {
                // Check for the group information to see if user is member of the Qaul Room or not
                group::Group::group_info(
                    chat::Chat::uuid_string_to_bin(
                        qaul_group_id.unwrap().to_string(),
                    )
                    .unwrap(),
                    request_id,
                );
            }
        } else {
            // Not Admin
            let content = AnyMessageEventContent::RoomMessage(
                MessageEventContent::text_plain(
                    "Only Admins can perform this operation.",
                ),
            );
            room.send(content, None).await.unwrap();
        }
    }

    // on receiving !group-info in matrix, You get the details of the group information.
    if msg_body.contains("!group-info") {
        let config = MATRIX_CONFIG.get().write().unwrap().clone();
        let room_id = room.room_id();
        let qaul_group_id: Option<Uuid> =
            find_key_for_value(config.room_map.clone(), room_id.clone());
        if qaul_group_id == None {
            // No room mapping exist
            let content =
           AnyMessageEventContent::RoomMessage(MessageEventContent::text_plain(
               "No qaul group is mapped to this Matrix room. Please invite qaul users to this room.",
           ));
            room.send(content, None).await.unwrap();
        } else {
            let request_id = format!("info#{}#_#_", room_id).to_string();
            group::Group::group_info(
                chat::Chat::uuid_string_to_bin(
                    qaul_group_id.unwrap().to_string(),
                )
                .unwrap(),
                request_id,
            );
        }
    }
} else {
    println!("Sent the message in the matrix room by !qaul-bot");
}
```
