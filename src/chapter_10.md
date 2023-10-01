# Version-2 : Sending files and receiving files

Another part of the major bridging features was a choice between puppeting and choosing to work for File exchanges. The file communication was more important since we already gave the users their name in the message as identifier. Puppetting was on the less priorty use case. So we decided to move ahead with the file messages.

From Matrix it was straightforward that you upload a file in room. It gets encoded into MxcURI (Matrix File Specific URL). From this, We got an helper function from SDK to download the file on the device which would Run the bridge (Assume RPi). Now, This would first download the file and send it further to target peer device (Your Phone).

Once it completely sends the file, We delete it from the bridge host (RPi).
Well, There can be debate on why would we not just pass the MxcURI directly to target device and decode it on bridge (which is not secure way to do this). This when I have asked to my mentor, He says that Qaul Application has bunch of version keeping. If we change any single part of code in qaul then it possible means that the new users have to upgrade the application to use it. Since Qaul is application which goes in cases of internet shutdowns and not for regular use, This would not be a great way to introduce it. And the bridge would be part of a local network too. So, It does not harm the security.

```rust
MessageType::File(FileMessageEventContent {
    body: file_name,url: file_url, .. }) => {
    // We don't consider message in matrix from the bot
    // since it would be the response being sent from qaul.
    if msg_sender != bot_matrix_id {
        // generate the File Request Body
        let request = MediaRequest {
            format: MediaFormat::File,
            media_type: MediaType::Uri(file_url.as_ref().unwrap().clone()),
        };

        // get the bytes data decrypted from the matrix into qaul
        let client = MATRIX_CLIENT.get();
        let file_bytes =
            client.get_media_content(&request, true).await.unwrap();

        // Save the file to local storage
        let path_string = Storage::get_path();
        let path = Path::new(path_string.as_str());
        let output_file_path = path.join(file_name);
        let mut file = std::fs::File::create(output_file_path).unwrap();
        let _ = file.write_all(&file_bytes);
        log::info!("File Saved Successfully");

        // Send the file to qaul world
        send_file_to_qaul(
            room.room_id(),
            file_name,
            format!("{} by {}", file_name, msg_sender),
        );
    }
}
```
When we want to send the file from qaul to matrix we have many helper functions written in bridge which can help to analyze the chat contents and files and then keep track of their upload status. Based on it, It will help us to send file to matrix from qaul which is reverse communication of what you have seen above.

You can refer `chat.rs` and `chatfile.rs` for the exact codewise implementation of handling files in chat.

Here is code to review how we worked on the entire logic from bridge to matrix.
```rust
fn send_file_to_matrix(file_path: String, room_id: &RoomId, extension: String, file_name: String) {
    let path = std::env::current_dir().unwrap();
    let mut storage_path = path.as_path().to_str().unwrap().to_string();
    let user = BOT_USER_ACCOUNT_ID.get();
    storage_path.push_str(&format!("/{}", user));
    storage_path.push_str(&format!("/files/{}", file_path));

    let matrix_client = MATRIX_CLIENT.get();
    let room = matrix_client.get_room(&room_id).unwrap();
    if let Room::Joined(room) = room {
        // Build the message content to send to matrix
        let rt = Runtime::new().unwrap();
        rt.block_on(async {
            // Sends messages into the matrix room
            log::info!("{}", storage_path);
            let file_buff = PathBuf::from(storage_path.clone());
            let mut buff = File::open(file_buff).unwrap();
            let mut content_type: &Mime = &STAR_STAR;
            log::info!("{}", extension);
            match extension.as_str() {
                "jpg" | "png" | "jpeg" | "gif" | "bmp" | "svg" => content_type = &mime::IMAGE_STAR,
                "pdf" => content_type = &mime::APPLICATION_PDF,
                _ => {
                    log::info!("Please raise a github ticket since we don't allow this file-type.")
                }
            }
            room.send_attachment(&file_name, content_type, &mut buff, None)
                .await
                .unwrap();
        });
        // Delete the file from bot server.
        log::info!("Deleting file from : {}", storage_path);
        fs::remove_file(storage_path).expect("could not remove file");
    };
```
