# Concept Specification

**concept** Authentication

**purpose** manage user accounts and authentication sessions

**principle** a person creates an account, logs in once, and receives a session token that represents them for all subsequent actions; sessions can be terminated when no longer needed

**state**

- set of Users with
  - a username String
  - a password String
- a set of Sessions with
  - a sessionToken String
  - a user User

**actions**

- `register(username: String, password: String): (user: User)`
  - requires: username is unique
  - effects: creates a new user
- `login(username: String, password: String): (sessionToken: String)`
  - requires: a user exists with the provided username and matching password
  - effects:
    - generates a unique session token and creates a new session associated with the user
    - adds the session to the set of Sessions and returns the sessionToken
- `logout(sessionToken: String)`
  - requires: session with sessionToken exists
  - effects: removes the session with the given sessionToken from the set of Sessions
- `deleteUser(sessionToken: String)`
  - requires: session with sessionToken exists
  - effects:
    - removes the session's associated user from the set of Users
    - removes sessions with the associated user from the set of Sessions

---

**concept** File

**purpose** represent uploaded content

**principle** a user uploads content as a file, either as raw binary data or text; the file belongs to the user who created it and can later be accessed, renamed, or deleted by that same user

**state**

- a set of Files with
  - an owner User
  - a name String
  - a content Binary | String

**actions**

- `makeFile(sessionToken: String, name: String, content: Binary | String): (f: File)`
  - requires: sessionToken is valid
  - effects: creates a file owned by the session user with the provided name and content
- `deleteFile(sessionToken: String, f: File)`
  - requires:
    - f exists
    - sessionToken is valid
    - file owner is the same as the session user
  - effects: removes f from the set of files
- `renameFile(sessionToken: String, f: File, newName: String)`
  - requires:
    - f exists
    - sessionToken is valid
    - file owner is the same as the session user
  - effects: renames f with newName

---

**concept** Chat \[User\]

**purpose** engage in a conversation

**principle** a user starts a new chat, exchanges messages with an LLM, and can save, rename, or delete the conversation

**state**

- a set of Chats with
  - an owner User
  - a name String
  - a messages List
- a messages List with
  - a role String
  - a content String
  - a timestamp Date

**action**

- `makeChat(sessionToken: String, name: String): (chat: Chat)`
  - requires:
    - sessionToken is valid
  - effects: returns a new chat with the provided name and makes the owner the session user
- `sendMessage(sessionToken: String, chat: Chat, content: String)`
  - requires:
    - sessionToken is valid
    - chat exists
    - chat owner is the session user
  - effects:
    - appends a message with role "user" to the chat's message list
- `receiveResponse(sessionToken: String, chat: Chat)`
  - requires:
    - sessionToken is valid
    - chat exists
    - chat owner is the session user
  - effects:
    - appends a new message with role = "llm" to the chat's message list
- `deleteChat(sessionToken: String, chat: Chat)`
  - requires:
    - sessionToken is valid
    - chat exists
    - chat owner is the session user
  - effects:
    - deletes the chat conversation
- `renameChat(sessionToken: String, chat: Chat, newName: String)`
  - requires:
    - sessionToken is valid
    - chat exists
    - chat owner is the session user
  - effects: changes name of chat to newName

---

**concept** Folder \[User, File, Chat\]

**purpose** organize items in a hierarchy

**principle** after you create a folder and insert items into it, you can move the folder into another folder and all the elements will still belong to it

**state**

- a set of Folders with
  - a name String
  - an owner User
  - a parent Folder | null (the root directory)
  - a children set of Folders
  - an elements set of Items
- a set of Items with
  - a set of Files
  - a set of Chats

**actions**

- `makeFolder(sessionToken: String, name: String, parentFolder: Folder | null): (f: Folder)`
  - requires:
    - sessionToken is valid
    - parent folder exists
    - session user owns the parent folder
    - a folder with the provided name must not exist in the provided parent folder
  - effects: creates an empty folder with the provided name in the parent folder owned by the session user
- `deleteFolder(sessionToken: String, f: Folder): (content: List<Items>)`
  - requires:
    - sessionToken is valid
    - f exist
    - session user owns f
  - effects: deletes f and all subfolders, outputting its content
- `moveFolder(sessionToken: String, f: Folder, newLocation: Folder)`
  - requires:
    - sessionToken is valid
    - f and newLocation exist
    - f has the same owner as newLocation
    - session user matches that owner
    - newLocation does not have a folder with the same name as f
    - newLocation is not in f's descendants
  - effects: moves f to newLocation and sets f's parent to newLocation
- `addItem(sessionToken: String, f: Folder, newItem: File | Chat)`
  - requires:
    - sessionToken is valid
    - f and newItem exist
    - f's owner is session user
    - newItem must not exist in another folder
  - effects: adds a reference to newItem in f
- `moveItem(sessionToken: String, f: Folder, newLocation: Folder, item: File | Chat)`
  - requires:
    - sessionToken is valid
    - f and newItem exist
    - f and newLocation's owner is session user
    - newLocation doesn't contain an item with the same name as item's
  - effects: adds a reference to item in newLocation and removes it from f
- `removeItem(sessionToken, f: Folder, item: File | Chat)`
  - requires:
    - sessionToken is valid
    - f and item exist
    - f's owner is session user
    - item is in f
  - effects: removes item from f
- `renameItem(sessionToken, f: Folder, item: File | Chat, newName: String)`
  - requires:
    - sessionToken is valid
    - session user owns f
    - item exists in f
    - no other item in f has name newName
  - effects: renames item to the new name

# Essential Synchronizations

**File Creation**

- **when** `Request.makeFile(sessionToken, name, content)`
- **then** `File.makeFile(sesionToken, name, content): (f)`

- **when**
  - `Request.makeFile(sessionToken, name, content, fileLocation)`
  - `File.makeFile(sessionToken, name, content): (f)`
- **then** `Folder.addItem(sessionToken, fileLocation, f)`

---

**File Deletion**

- **when** `Request.deleteFile(sessionToken, f, item)`
- **then** `Folder.removeItem(sessionToken, f, item)`

- **when**

  - `Request.deleteFile(sessionToken, f, item)`
  - `Folder.removeItem(sessionToken, f, item)`

- **then** `File.deleteFile(sessionToken, f)`

---

**Chat Creation**

- **when** `Request.makeChat(sessionToken, name)`
- **then** `Chat.makeChat(sessionToken, name): (chat)`

- **when**
  - `Request.makeChat(sessionToken, name, f)`
  - `Chat.makeChat(sessiontoken, name): (chat)`
- **then** `Folder.addItem(sessionToken, f, chat)`

---

**Chat Deletion**

- **when** `Request.deleteChat(sessionToken, f, chat)`
- **then** `Folder.removeItem(sessionToken, f, item)`

- **when**
  - `Request.deleteChat(sessionToken, f, chat)`
  - `Folder.removeItem(sessionToken, f, item:)`
- **then** `Chat.deleteChat(sessionToken, chat)`

---

**Rename File**

- **when** `Request.renameFile(sessionToken, f, item, newName)`
- **then** `Folder.renameItem(sessionToken, f, item, newName)`

---

**Rename Chat**

- **when** `Request.renameChat(sessionToken, f, chat, newName)`
- **then** `Folder.renameItem(sessionToken, f, chat, newName)`

---

**Delete Folder**

- **when** `Request.deleteFolder(sessionToken, f)`
- **then** `Folder.deleteFolder(sessionToken, folfder): (content)`

- **when**
  - `Request.deleteFolder(sessionToken, folder)`
  - `Folder.deleteFolder(sessionToken, folder): (content)`
- **where** content elm is of type chat
- **then** `Chat.deleteChat(sessionToken, chat: content_elm)`

- **when**
  - `Request.deleteFolder(sessionToken, folder)`
  - `Folder.deleteFolder(sessionToken, folder): (content)`
- **where** content elm is of type file
- **then** `File.deleteFile(sessionToken, f: content_elm)`

# A Brief Note

The Authentication concept is central to the app, managing user accounts and sessions, and controlling access to all other resources. Only a valid session token allows a user to create, modify, or delete files, chats, or folders, ensuring that each user interacts only with their own data. The File concept allows users to upload and manage documents—such as research papers, notes, or other content—providing a persistent, personal repository. The Chat concept enables interactive conversations with the AskMedi chatbot, allowing users to ask questions, receive explanations, and preserve these conversations for later reference. Finally, the Folder concept organizes files and chats into hierarchies, so users can group related items together, maintain a clear structure, and easily access their materials. Generic parameters, such as the user type in Folder or Chat, are instantiated with the users defined in the Authentication concept, ensuring consistent ownership and access control across the app.
