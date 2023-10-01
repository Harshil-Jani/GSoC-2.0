# Version-2 : Invite and remove from group

We have thought of invitation and removal of the qaul users from the matrix chat. Now since the matrix room was open for participation by multiple members we were more inclined to set restrictions on who can `!invite` or `!remove` the members from the matrix chat. 

Architecturally, We implemented that the users who have Admin Powers in Matrix can only perform the invitation and removal commands. This would restrict the unmonitered chaos which would happen if we allow everyone to invite or remove.

In order to invite a member you can write the command in matrix room as 
`!invite {peer-id of qaul user}`

The code from matrix side looks as below :
```rust
// on receiving !invite in matrix
if msg_body.contains("!invite") {
    let matrix_user =
        room.get_member(&msg_sender).await.unwrap().unwrap();
    // Check for Admin Powers
    if matrix_user.power_level() == 100 {
        let mut iter = msg_body.split_whitespace();
        let _command = iter.next().unwrap();
        let qaul_user_id = iter.next().unwrap().to_string();
        let room_id_string = room.room_id().to_string();
        let sender_string = msg_sender.to_string();
        let request_id = format!(
            "invite#{}#{}#{}",
            room_id_string, sender_string, qaul_user_id
        );
        log::info!("{}", request_id);
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
            log::info!("The Room Mapping already exist for this room");
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
```

As of now, In qaul we do not have request numbering mechanism, so we don't know that which request was received first and others following. There may be chances that the sequence would be much different in case of traffic on matrix servers and qaul peer network. So we changed our libqaul code a bit to support the id based recognition since we were using RPC. In case of all the invite request the RPC request id is given as `#invite#matrix_room_id#msg_sender_id#qaul_user_peer_id`

This would then be decoded on the RPC Response.

```rust
// Receiving GroupInfoResponse which we are polling every 10 ms.
Some(proto::group::Message::GroupInfoResponse(group_info_response)) => {
    let group_id = uuid::Uuid::from_bytes(
        group_info_response.group_id.try_into().unwrap(),
    );

    if request_id != "" {
        // reqeust_id = qaul_user_id#room_id
        let mut iter = request_id.split('#');
        let cmd = iter.next().unwrap();
        log::info!("cmd : {}", cmd);
        let room_id = iter.next().unwrap();
        log::info!("room : {}", room_id);
        let _sender = iter.next().unwrap();
        log::info!("sender : {}", _sender);
        let qaul_user_id = iter.next().unwrap();
        log::info!("qaul user : {}", qaul_user_id);

        if cmd == "invite" {
            let grp_members = group_info_response.members.clone();
            let user_id =
                chat::Chat::id_string_to_bin(qaul_user_id.to_owned()).unwrap();
            let mut all_members = Vec::new();
            for member in grp_members {
                all_members.push(member.user_id);
            }
            if all_members.contains(&user_id) {
                matrix_rpc(
                    "User already exist in the qaul group".to_owned(),
                    RoomId::try_from(room_id).unwrap(),
                );
            } else {
                // Invite user into this group.
                let users = QAUL_USERS.get().read().unwrap();
                let user_name = chat::Chat::find_user_for_given_id(
                    users.clone(),
                    qaul_user_id.to_owned(),
                )
                .unwrap();
                matrix_rpc(
                    format!("User {} has been invited. Please wait until user accepts the invitation.", 
                    user_name
                ).to_owned(), RoomId::try_from(room_id).unwrap());
                matrix_rpc("User has been invited. Please wait until user accepts the invitation.".to_owned(), RoomId::try_from(room_id).unwrap());
                Self::invite(
                    chat::Chat::uuid_string_to_bin(group_id.to_string())
                        .unwrap(),
                    user_id,
                );
            }
        }
    }
}
```

Similarly we do have operations for removal. Do checkout our codebase to understand the implementations at their present state.