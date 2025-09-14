# Problem Set 1: Concept and Design: Reading and Writing Concepts

## Table of Contents:

- [Exercise 1](#exercise-1-reading-a-concept)
- [Exercise 2](#exercise-2-extending-a-familiar-concept)
- [Exercise 3](#exercise-3-comparing-concepts)
- [Exercise 4](#exercise-4-defining-familiar-concepts)

## Exercise 1: Reading a Concept

### 1. Invariants

The two invariants of the state are:

1. The count of a request or a purchase should always be greater than or equal to zero.
2. For any item in a registry, the total number of purchases made plus the current remaining unpurchased count in the corresponding request must equal the total number of that item ever requested (including any increases through addItem).

The more important of the two invariants is the second one. It defines the relationship between purchases and requests while also enforcing the first invariant. If purchases were to exceed the requested amount, then the count of a request would be negative; similarly, if a count of purchases was negative, it would imply the registry owner is somehow giving away items rather than receiving them.

The `purchase` action is most affected by the second invariant. It preserves the invariant by requiring that the provided count is less than or equal to the count of the requested item in the precondition.

### 2. Fixing an Action

The `removeItem` action may break the invariant. The precondition only checks that the item to be removed is requested at least once in the registry. If purchases for that item already exist, removing the corresponding request will immediately violate the invariant: the total number of that item ever requested becomes zero because the request is gone; however, the total purchase count for that item remains greater than zero. Thus, the sum of purchases plus remaining count no longer equals the total number of that item ever requested.

To fix this problem, we can strengthen the requires clause so that an item can only be removed if it has no purchases associated with it.

### 3. Inferring Behavior

In the current concept specification, a registry can be opened and closed repeatedly as `open` only requires that a registry exists and is inactive; similarly, `close` only requires that a registry exists and is active. You could allow this functionality in case an owner closes a registry by mistake and needs to reopen it. Alternatively, this functionality allows a registry to act for recurring events. Existing items remain in the registry, so the owner can increment counts or update requests instead of creating a new registry from scratch.

### 4. Registry Deletion

In practice, registry deletion could matter. While closing the registry makes it no longer visible to the public, it remains visible to the owner. Over time, old registries could clutter the owner's view, hindering their experience using the GiftRegistration application. Allowing the owner to delete a registry would give them more control over how their registries are organized; however, it would also remove any historical record of past purchases. The system may want to offer an archiving feature in conjunction for increased flexibility.

### 5. Queries

The two common queries likely to be executed against the concept state are:

1. A registry owner will retrieve items in their registry along with their requested counts and any purchases made.
2. A gift giver will retrieve the items in a public registry that are still available to purchase.

### 6. Hiding Purchases

I'd augment the exisisting concept specificiation as follows. Note '...' indicates that the portion of the concept specification remains unchanged from the original.

- **concept** GiftRegistration [User, Item]
- **purpose** track purchases of requested gifts
- **principle** ...
- **state**
  - a set of Registrys with
    - an owner User
    - an active Flag
    - a set of Requests
    - a hidden Flag
  - a set of Requests with
    - ...
  - a set of Purchases with
    - ...
- **actions**
  - `create(owner: User): (registry: Registry)`
    - effects: create a new registry with this owner, active and hidden set to false and no requests
  - `addItem(registry: Registry, item: Item, count: Number)`
    - ...
  - `removeItem(registry: Registry, item: Item)`
    - ...
  - `open(registry: Registry, hidden: Boolean)`
    - requires: registry exists and it is not active
    - effects: make registry active and sets the registry's hidden flag according to the hidden parameter, hiding purchases from the owner if hidden = True
  - `close(registry: Registry)`
    - ...
  - `purchase(purchaser: User, registry: Registry, item: Item, count: Number)`
    - ...

Queries for the owner now wouldn't display the count of purchases for an item if hidden.

### 6. Generic Types

Using SKU codes instead of representing items can resolve ambiguities. For example, the names or descriptions of two items may be the same/similar, making it unclear what is being purchased and requested. Prices likewise can change over time and may be shared across items, making it ineffective for identifying an item. In contrast, SKU codes are unique identifiers and can enable implementation optimizations such as faster lookups or more compact storage.

## Exercise 2: Extending a Familiar Concept

### 1. Complete the defintion of the concept state.

- **state**
  - a set of Users with
    - a username String
    - a password String

### 2. Write a requires/effects specification for each of the two actions.

- `register(username: String, password: String): (user: User)`
  - requires: no user exists with the given username
  - effects: creates a new user with the provided username and password
- `authenticate(username: String, password: String): (user: User)`
  - requires: a user exists with the given username and the provided password matches that user's password
  - effects: allows the person logging in to act as the user associated with the provided username and password

### 3. Essential invariant must hold

There must not exist multiple users with the same username. This invariant is preserved in the `register` action through its precondition, preventing new users from being created with a provided username if it already belongs to another user.

### 4. Confirmation by email

Note '...' indicates that the portion of the concept specification remains unchanged from the original.

- **concept** PasswordAuthentication [User, Confirmation]
- **purpose** limit access to known users
- **principle** ...
- **state**

  - a set of Users with
    - a username String
    - a password String
    - a confirmed Flag
  - a set of Confirmations with
    - a user User
    - a token String

- **actions**

  - `register(username: String, password: String): (user: User, token: String)`
    - requires: no user exists with the given username
    - effects: creates a new user with the provided username and password, sets the confirmed flag to False, and creates a confirmation linking the new user to the token
  - `confirm(username: String, token: String)`
    - requires: a user with the provided username must exist, and a confirmation must exist linking that user to the provided token
    - effects: sets the confirmed flag of the user with username to True
  - `authenticate(username: String, password: String): (user: User)`
    - requires: a user with the provided username must exist, and that user must have the provided password and be confirmed
    - effects: allows the person logging in to act as the user associated with the provided username and password

## Exercise 3: Comparing Concepts

Note that parameters in the concept spec followed by '?' indicate an optional parameter.

- **concept** PersonalAccessToken [User, DeployKey, Scope, Date]
- **purpose** provide accesses to a service in a controlled, revocable, and optionally scoped way
- **principle** a user creates a personal access token; the user uses their personal access token in scripts, on the command line, and API calls; the token grants access according to its scope, allowing the user to control and limit ongoing access safely

- **state**

  - a set of PersonalAccessTokens with
    - a user User
    - a set of keys DeployKeys
    - a name String \<optional\>
    - an expiration Date \<optional\>
    - a scope Scope \<optional\>
    - a revoked Flag

- **actions**

  - `createPersonalAccessToken(owner: User, name?: String, expiration?: Date, scope?: Scope): (token: PersonalAccessToken)`
    - requires: the owner is an existing, verified User
    - effects: generates a personal access token for the owner with the provided parameters and the revoked flag set to False
  - `deletePersonalAccessToken(owner: User, token: PersonalAccessToken)`
    - requires: the owner and token must exist, and the token must be associated with that owner
    - effects: removes the token from the set of tokens and any deploy keys created by it
  - `revokeToken(token: PersonalAccessToken)`
    - requires: the token exists and is valid (not expired or revoked)
    - effects: sets the revoked flag of the token to be True
  - `useToken(token: PersonalAccessToken)`
    - requires: the token exists and is valid (not expired or revoked)
    - effects: allows holder to act as the associated User for accessing permitted resources

**How it differs**

1. Tokens can be revoked independently of the user's password.
2. Token can have scopes limiting permitted resources or actions.
3. Tokens can be used for automated or repeated access without logging in each session.
4. Passwords are tied to a user, allowing a person to act as a specified user with all permissions.

The GithHub page describing personal access tokens can do a stronger job emphasizing these differences. Currently, it primarily describes the process of creating a perseonal access token and parameter configurations. It does not describe revocability on the provided page or automated access. Examples of both of these should be given. Scopes are mentioned briefly, with more detail existing on a different page.

## Exercise 4: Defining Familiar Concepts

Note that any parameters followed by a '?' in the concept specification are considered optional for the following three specifications.

### Billable Hours Tracking

- **concept** HoursTracker [User, Project, Time, Date]
- **purpose** tracks the hours worked to automate record keeping
- **principle** a user selects a project and enters a description of their work to be done to begin a tracked session; the user performs the works and ends the session when finished
- **state**

  - a set of HoursTracker with
    - a user User
    - a project Project
    - a description String
    - a date Date
    - a start Time
    - an end Time
    - an invalid Flag

- **actions**

  - `createHoursTracker(user: User, project: Project): (tracker: HoursTracker)`
    - requires: user and project exist
    - effects: creates a tracker associated with the provided user and project; sets the invalid flag to True
  - `startHoursTracker(tracker: HoursTracker, description: String, date: Date, start: Time)`
    - requires: tracker exists and has no start time or has not ended
    - effects: sets the tracker's description, date, and start time with the provided information
  - `endHoursTracker(tracker: HoursTracker, end: Time)`
    - requires: tracker exists, has a start time, and no end time; the provided end is after the start time
    - effects: sets the tracker's end time and invalid flag to False

This concept specification is able to handle the case in which someone forgets to end a session through its invalid flag. If a user forgets to end a session, the `endHoursTracker` will not have set the invalid flag to be False, ensuring that it isn't counted because no end time exists. Note that we don't have to check for the date in `endHoursTracker` because it must be after start.

### Conference Room Booking

- **concept** ConferenceRoomBooking [User, Date, Time]
- **purpose** allows user to reserve a conference room for their use
- **principle** a user selects a room, date, start time, and end time; the user goes to the room on that date at that start time and uses the room until the end time
- **state**

  - a set of ConferenceRoomBookings with
    - a date Date
    - a start Time
    - an end Time
    - an owner User
    - a room Number

- **actions**:

  - `createConferenceRoomBooking(owner: User, room: Number, date: Date, start: Number, end: Number): (booking: ConferenceRoomBooking)`
    - requires: a booking does not already exist for the provided room with a time that overlaps with the provided start/end interval on the given date; end is after start; owner must exist
    - effects: creates a booking in the room for the owner from start to end
  - `deleteConferenceRoomBooking(booking: Booking)`
    - requires: booking exists
    - effects: removes the booking from the set of ConferenceRoomBookings
  - `editConferenceRoomBooking(booking: Booking, date: Date, start: Number, end: Number)`
    - requires: booking exists; the new start and end time interval does not conflict with any existing booking for the same room on the provided date; end is after start
    - effects: updates the booking to start at start and end at end on the given date

Note that this implementation of a conference room booking does not allow one booking to extend over multiple days. Additionally, this design allows users to book multiple conference rooms at the same time so long as someone else hasn't booked it previously; this design was chosen because it's possible for one conference room to be insufficient for large party sizes.

### Time-Based One-Time Password (TOTP)

- **concept** OneTimePassword [User, Time]
- **purpose** authenticates users more securely by separating authentication factors and limiting the usefulness of stolen credentials
- **principle** a user logs in using their username and password; a user opens an authentication app on their phone to obtain a time-based one-time password (TOTP); the user inputs the TOTP before it expires to complete authentication and gain access to the system

- **state**

  - a set of Tokens with
    - a user User
    - an expiration Time
  - a set of Users with
    - a username String
    - a password String

- **actions**:

  - `createTOTP(user: User, start: Time, duration?: Number): (token: OneTimePassword)`
    - requires: user exists
    - effects: creates a token for the provided user with an expiration after the specified duration (or a default duration if none is provided)
  - `authenticate(username: String, password: String, token: Token): (user: User)`
    - requires: a user with the provided username exists and password; the provided token exists, has not expired, and is associated with the user
    - effects: allows the person logging in to act as the user associated with the provided username and password

Note that in this design, the duration of the time-based one-time password is configurable but left optional to allow a default value to be set.
