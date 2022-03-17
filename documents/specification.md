# Specification

# User side

## Message's features

### Suported types:

1. Text messages from vk to telegram
2. Text messages form telegram to vk
3. Images and descriptions from vk to telegram
4. Images and descriptions from telegram to vk

### Other featutes:

1. Name of the author of the message is displayed on top of the message text when message comes from vk to telegram
2. Informing user if message (created by him or incoming from vk) has unprocessable data

## Control of the bot

### Guiding messages

1. Bot has description for each command
2. Bot has descriptive /help and /about section which tells user how this bot works, what features has and how to use these features
3. Bot tells (if it's not obvious) user what to do next after each user movement
4. When group chat is created:
   1. If there is a pending chat, bot tells user about successful connection to this chat
   2. If there is no pending chat, bot tells user about they have to create a pending chat first and a group can be created only after this step

### General flow

1. Command "/start" -> new user is created and if any information about this user previously existed in the system, this information gets deleted
2. Bot asks user for vk token and doesn't do anything unless token is provided
3. Command "/create" ->
   1. bot sends to the user paged view of buttons with top vk chats (8 chats per page). This view contains buttons "next" (if available), "previous" (if available), "cancel" to stop chat creation process. Any other commands are not available in this period.
   2. When user clicks on the desired chat button bot remembers the desired chat
   3. User creates a group chat and bot automatically connects this chat to a selected vk chat. User doesn't have to create group immediatly after selecting the chat and has access to the full bot functionality.
4. Command "/stop" stops the bot and deletes all informatoin about this user from the database.
5. Command "/token" updates User clicks buttontoken. While waiting for token update the command "/cancel" is available.
6. Command "/help" shows available commands and their description.
7. Command "/about" shows bot description - what this bot can do, my nickname as a bot developer and maybe something more if I will want.

# Program side

## Microservices

### Vk API

Purpose:

1. This service's API:

   1. Start monitoring updates for new user.
   2. Send message to vk (optionally with image) {user_id, chat_id, message and/or image link} -> result (true/false)
   3. Get user's conversations list {user_id} -> list[conversation]
   4. Get user's pending updates {user_id} -> list[update] (service should return updates and forget about them when this method is called)
   5. Get all pending updates {none} -> list[update] for each user (+ forget about updates)

2. Extra jobs:

   1. Run in background threads for fetching vk updates. One thread per user.
   2. Store incoming updates in a RAM memory
   3. Filter incoming updates. Should skip update if:
      1. Type of update is not "new message"
      2. The message was sent by the user himself
   4. Parse updates - should save only {msg_author_vk_id, chat_id, msg_text (if exists), img_link (if exists)}. This data should be converted from array into more readable json.

### Tg API

1. This service's API:

   1. Send message (optionally with image) {user_id, chat_id, message and/or image link} -> result (true/false)
   2. Get user's pending updates {user_id} -> list[update] (service should return updates and forget about them when this method is called)
   3. Get all pending updates {none} -> list[update] for each user (+ forget about updates)
   4. Update currently suggesting chats to join {user_id, new_page_id, list[vk_chat_id, vk_chat_title]} -> reslut(true/false)

2. Extra jobs:
   1. Maintain user's journey in these parts:
      1. When user has to pass the vk_token, wait for token and prohibit aby other actions
      2. When user has to choose vk_chat, maintain current selection state, process clicks on buttons and create corresponding updates (prev_page, next_page (+ calculate the next page's num), chat_selected, cancel)
   2. Always run a thread in a background and receive new updates
   3. Store processed updates in a RAM memory until "processer service" will claim them
   4. Figure out from update wheter it:
      1. Came from a group or a private chat?
      2. Is command (or signal like "New group created") or not?
   5. Ignore any other information from update except of:
      1. Author (message emitter) user_id
      2. Chat_id of update
      3. Message's text and/or message's image
      4. If update is a command:
         1. check whether it is a supported command and if yes, turn the command into command_id but if no, inform user about bad command and prevent update's future propagation
         2. Any command may conatin extra data (such as vk_token when passed) if needed

Purpose:

1. Perform all requests to tg api
2. Convert updates from telegram into our format
3. Filter updates (if will be needed)
4. Save state of user journey in private chat with bot (e.g. waiting for token or for click in the chat selection button)
5. Check if user action is acceptable right now and if now, inform user about it and stop current update from future propagation

### Processer

Purpose:

1. Periodically fetch updates from both vk and tg
2. Process updates:
   1. Register new users
   2. Register user's token
   3. Maintain process of vk chat selection
   4. Register new group chat when created
   5. Propagate messages from vk/tg no tg/vk respectievly based on registered routes
   6. Delete routes if any problems occured (such as vk chat has been deleted)

### GraphQL

Purpose:

1. Standard graphql server
2. Communicates with database
