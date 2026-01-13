# Implementation Plan: Authentication & Messaging System

## Overview
Add authentication and a comprehensive messaging system to a Rails 8.1.2 application supporting:
- **Authentication**: Rails 8's built-in authentication generator
- **1-on-1 Conversations**: Private messages between two users
- **Group Rooms**: Multi-user chat rooms with role-based permissions
- **Real-time Updates**: Action Cable with Turbo Streams for instant message delivery
- **Message Features**: Read/unread status, search, and file attachments

## Architecture Summary

**Database Design**: Polymorphic message architecture where a single `messages` table serves both conversations and rooms via `messageable_type` and `messageable_id`. Separate `message_receipts` table tracks per-user read status.

**Models Structure**:
- User (from auth generator) → has many conversations, rooms, messages
- Conversation (1-on-1) → has many messages (polymorphic), participants
- Room (group) → has many messages (polymorphic), members
- Message (polymorphic) → belongs to messageable (Conversation or Room)
- MessageReceipt → tracks read/unread status per user per message

**Real-time Strategy**: Separate Action Cable channels (ConversationChannel, RoomChannel) with Turbo Streams for server-side rendering of updates.

## Implementation Steps

### Phase 1: Authentication Setup

**Step 1**: Uncomment bcrypt gem
- **File**: `Gemfile` (line 21)
- **Action**: Uncomment `gem "bcrypt", "~> 3.1.7"`
- **Command**: `bundle install`

**Step 2**: Generate Rails 8 authentication
- **Command**: `rails generate authentication`
- **Creates**: User model, Session model, authentication controllers, views, migrations
- **Routes added**: session, passwords, registration resources

**Step 3**: Install Active Storage for file attachments
- **Command**: `rails active_storage:install`
- **Creates**: Migrations for active_storage tables

**Step 4**: Run migrations
- **Command**: `rails db:migrate`

### Phase 2: Messaging Models

**Step 1**: Create Conversation model
- **Command**: `rails generate model Conversation`
- **Migration**: Simple table with timestamps
- **Model associations**:
  - `has_many :conversation_participants`
  - `has_many :participants, through: :conversation_participants`
  - `has_many :messages, as: :messageable`
- **Key method**: `self.between(user1, user2)` - finds existing conversation

**Step 2**: Create ConversationParticipant join model
- **Command**: `rails generate model ConversationParticipant conversation:references user:references`
- **Migration**: Add unique index on `[conversation_id, user_id]`
- **Validation**: User can only participate once per conversation

**Step 3**: Create Room model
- **Command**: `rails generate model Room name:string description:text created_by:references`
- **Migration**: Add indexes on name, created_at
- **Model associations**:
  - `belongs_to :creator` (User)
  - `has_many :room_memberships`
  - `has_many :members, through: :room_memberships`
  - `has_many :messages, as: :messageable`
- **Validations**: Unique name, 3-50 characters

**Step 4**: Create RoomMembership join model
- **Command**: `rails generate model RoomMembership room:references user:references role:string`
- **Migration**: Add unique index on `[room_id, user_id]`, default role: 'member'
- **Roles**: admin, moderator, member (stored in ROLES constant)
- **Add**: `joined_at` datetime field

**Step 5**: Create Message model (polymorphic)
- **Command**: `rails generate model Message messageable:references{polymorphic} user:references body:text`
- **Migration**: Add composite index on `[messageable_type, messageable_id, created_at]`
- **Model associations**:
  - `belongs_to :messageable, polymorphic: true`
  - `belongs_to :user`
  - `has_many :message_receipts`
  - `has_many_attached :attachments` (Active Storage)
- **Validations**:
  - Body presence (unless attachments exist)
  - File type whitelist (images, PDFs, docs)
  - File size limit (10MB)
- **Callbacks**:
  - `after_create_commit :broadcast_message` - real-time updates
  - `after_create_commit :create_receipts_for_participants` - auto-create read receipts

**Step 6**: Create MessageReceipt model
- **Command**: `rails generate model MessageReceipt message:references user:references read_at:datetime`
- **Migration**: Add unique index on `[message_id, user_id]`, index on `[user_id, read_at]`
- **Methods**: `mark_as_read!`, `read?`, `unread?`
- **Scopes**: `unread`, `read`

**Step 7**: Update User model with associations
- **File**: `app/models/user.rb`
- **Add associations**:
  - Conversations: `has_many :conversation_participants`, `has_many :conversations`
  - Rooms: `has_many :room_memberships`, `has_many :rooms`, `has_many :created_rooms`
  - Messages: `has_many :messages`, `has_many :message_receipts`
  - Avatar: `has_one_attached :avatar`
- **Add methods**:
  - `conversation_with(other_user)` - find or create conversation
  - `unread_messages_count` - total unread
  - `unread_count_for_conversation(conv)` - per-conversation unread
  - `unread_count_for_room(room)` - per-room unread

**Step 8**: Run migrations
- **Command**: `rails db:migrate`

### Phase 3: Action Cable (Real-time)

**Step 1**: Generate ConversationChannel
- **Command**: `rails generate channel Conversation`
- **File**: `app/channels/conversation_channel.rb`
- **Logic**:
  - Verify user is participant before subscribing
  - Stream for specific conversation
  - Mark messages as read when channel opened
  - Reject unauthorized subscriptions

**Step 2**: Generate RoomChannel
- **Command**: `rails generate channel Room`
- **File**: `app/channels/room_channel.rb`
- **Logic**:
  - Verify user is room member before subscribing
  - Stream for specific room
  - Mark messages as read when channel opened
  - Reject unauthorized subscriptions

**Step 3**: Update connection authentication
- **File**: `app/channels/application_cable/connection.rb`
- **Logic**: Identify user via encrypted session cookie, reject unauthorized

**Step 4**: Create JavaScript subscriptions
- **Files**:
  - `app/javascript/channels/conversation_channel.js`
  - `app/javascript/channels/room_channel.js`
- **Logic**: Auto-subscribe when conversation/room page loads using data attributes

### Phase 4: Controllers

**Step 1**: ConversationsController
- **Command**: `rails generate controller Conversations index show create`
- **Actions**:
  - `index` - list user's conversations with unread counts
  - `show` - display conversation messages, mark as read
  - `create` - find or create conversation with another user
- **Security**: `before_action :require_authentication`

**Step 2**: RoomsController
- **Command**: `rails generate controller Rooms index show new create edit update destroy`
- **Actions**:
  - `index` - list user's rooms + available rooms to join
  - `show` - display room messages and members, mark as read
  - `new/create` - create room (creator becomes admin)
  - `edit/update` - admin only
  - `destroy` - admin only
- **Security**: Verify membership for show, verify admin for edit/update/destroy

**Step 3**: RoomMembershipsController
- **Command**: `rails generate controller RoomMemberships create destroy`
- **Actions**:
  - `create` - join room as member
  - `destroy` - leave room (prevent last admin from leaving)

**Step 4**: MessagesController
- **Command**: `rails generate controller Messages create destroy`
- **Actions**:
  - `create` - send message to conversation or room, respond with Turbo Stream
  - `destroy` - delete own message only
- **Note**: Uses `set_messageable` to work with both conversations and rooms

**Step 5**: SearchController
- **Command**: `rails generate controller Search index`
- **Actions**:
  - `index` - search messages (SQLite LIKE query), users, and rooms
  - Returns results grouped by type

**Step 6**: DashboardController
- **Command**: `rails generate controller Dashboard index`
- **Actions**:
  - `index` - home page showing recent conversations, rooms, unread count

**Step 7**: UsersController
- **Command**: `rails generate controller Users index show`
- **Actions**:
  - `index` - list all users (for starting conversations)
  - `show` - user profile with "Start Conversation" button

### Phase 5: Routes

**File**: `config/routes.rb`

```ruby
root "dashboard#index"
get "dashboard", to: "dashboard#index"

# Auth routes (from generator)
resource :session
resources :passwords, param: :token
resource :registration

# Conversations
resources :conversations, only: [:index, :show, :create] do
  resources :messages, only: [:create, :destroy]
end

# Rooms
resources :rooms do
  resources :messages, only: [:create, :destroy]
  resources :room_memberships, only: [:create], shallow: true
end
resources :room_memberships, only: [:destroy]

# Search & Users
get "search", to: "search#index"
resources :users, only: [:index, :show]
```

### Phase 6: Views

**Critical Views to Create**:

1. **Layout** (`app/views/layouts/application.html.erb`)
   - Navigation bar with links (Dashboard, Conversations, Rooms, Search)
   - User menu with email and unread badge
   - Flash messages display

2. **Dashboard** (`app/views/dashboard/index.html.erb`)
   - Recent conversations list with unread counts
   - Recent rooms list with member counts
   - Quick action buttons

3. **Conversations**
   - `index.html.erb` - list of conversations with last message preview
   - `show.html.erb` - conversation messages + message form
   - `_conversation.html.erb` - conversation partial with unread badge

4. **Rooms**
   - `index.html.erb` - user's rooms + available rooms to join
   - `show.html.erb` - room messages, members sidebar, message form
   - `new.html.erb` / `edit.html.erb` - room creation/editing forms
   - `_form.html.erb` - room form partial

5. **Messages**
   - `_message.html.erb` - single message with avatar, body, attachments, timestamp
   - `_form.html.erb` - message input with file attachment button
   - `create.turbo_stream.erb` - append new message, reset form
   - `destroy.turbo_stream.erb` - remove message

6. **Search** (`app/views/search/index.html.erb`)
   - Search form
   - Results grouped by: conversation messages, room messages, users, rooms
   - "Start Conversation" buttons for user results

7. **Users**
   - `index.html.erb` - list users with "Start Conversation" button
   - `show.html.erb` - user profile (future)

### Phase 7: Stimulus Controllers (JavaScript)

**Step 1**: Message Form Controller
- **File**: `app/javascript/controllers/message_form_controller.js`
- **Purpose**:
  - Handle Enter key (submit), Shift+Enter (new line)
  - Clear form after successful submission

**Step 2**: Scroll Controller
- **File**: `app/javascript/controllers/scroll_controller.js`
- **Purpose**:
  - Auto-scroll to bottom when messages load
  - Auto-scroll when new messages arrive (MutationObserver)

### Phase 8: Basic Styling

**File**: `app/assets/stylesheets/application.css`

Add CSS for:
- Navigation bar with unread badge
- Flash messages (notice/alert)
- Message list container with scroll
- Message bubbles (own messages vs others)
- Message form with file attachment button
- Grid layouts for rooms and conversations
- Room layout with members sidebar

### Phase 9: Seed Data & Testing

**Step 1**: Create seed data
- **File**: `db/seeds.rb`
- **Create**:
  - 5 test users (user1@example.com through user5@example.com, password: "password123")
  - 1 conversation with 2 messages
  - 2 rooms (General, Random) with members and messages
- **Command**: `rails db:seed`

**Step 2**: Manual testing checklist
- User registration/login/logout
- Start conversation from user list
- Send messages in conversation (real-time)
- Create room and join room
- Send messages in room (real-time)
- Upload file attachments
- Search messages, users, rooms
- Verify unread counts
- Verify read receipts in conversations
- Delete own messages

## Critical Files Reference

**Core Models**:
- `app/models/message.rb` - Polymorphic message with broadcasting logic
- `app/models/user.rb` - User with all associations and helper methods
- `app/models/conversation.rb` - 1-on-1 conversations
- `app/models/room.rb` - Group chat rooms

**Real-time**:
- `app/channels/conversation_channel.rb` - WebSocket for conversations
- `app/channels/room_channel.rb` - WebSocket for rooms
- `app/channels/application_cable/connection.rb` - Authentication

**Controllers**:
- `app/controllers/messages_controller.rb` - Polymorphic message creation
- `app/controllers/conversations_controller.rb` - Conversation management
- `app/controllers/rooms_controller.rb` - Room management

**Configuration**:
- `config/routes.rb` - Nested resource structure
- `Gemfile` - Uncomment bcrypt (line 21)

## Verification Plan

### End-to-End Test Scenario

1. **Setup** (as Admin)
   - Run `rails db:seed`
   - Start server: `rails server`
   - Open two browser windows (Chrome + Firefox or incognito)

2. **Authentication Test**
   - Window 1: Sign up as user1@example.com
   - Window 2: Sign up as user2@example.com
   - Verify both reach dashboard

3. **Conversation Test**
   - Window 1: Navigate to Users, click "Start Conversation" with user2
   - Window 1: Send message "Hello from user1"
   - Window 2: Should see new conversation in dashboard (real-time)
   - Window 2: Open conversation, verify message appears
   - Window 2: Reply "Hello from user2"
   - Window 1: Should see reply instantly (no refresh)
   - Verify unread badge on Window 1 before opening conversation
   - Verify badge clears after opening

4. **File Attachment Test**
   - Window 1: In conversation, attach an image file
   - Send message with attachment
   - Window 2: Verify image displays inline
   - Click image to download

5. **Room Test**
   - Window 1: Create room "Test Room"
   - Window 2: Navigate to Rooms, join "Test Room"
   - Window 1: Send message in room
   - Window 2: Verify message appears instantly
   - Verify members sidebar shows both users

6. **Search Test**
   - Window 1: Navigate to Search
   - Search for "Hello" (should find conversation messages)
   - Search for "user2" (should find user)
   - Search for "Test Room" (should find room)

7. **Read Status Test**
   - Window 1: Send message in conversation
   - Window 1: Should NOT show "Read" receipt initially
   - Window 2: Open conversation
   - Window 1: Should show "Read" receipt on message (real-time)

8. **Unread Count Test**
   - Window 2: Sign out, sign back in as user3@example.com
   - Window 1: Start conversation with user3, send 3 messages
   - Window 2 (as user3): Dashboard should show unread badge with "3"
   - Window 2: Open conversation
   - Verify badge clears from navigation and conversation list

### Performance Check
- Open browser DevTools Network tab
- Verify WebSocket connection established (ws:// or wss://)
- Send message, verify Turbo Stream response
- Check for N+1 queries in Rails logs when loading conversations

### Error Handling Check
- Try sending empty message (should fail validation)
- Try uploading file over 10MB (should fail validation)
- Try uploading invalid file type (.exe) (should fail validation)
- Try accessing conversation you're not part of (should redirect with error)
- Try editing room as non-admin (should redirect with error)

## Notes & Trade-offs

**Polymorphic Messages**: Single messages table serves both conversations and rooms. This simplifies search and reduces code duplication, but makes some queries slightly more complex.

**Separate Receipts Table**: Essential for tracking per-user read status in group rooms. Alternative (boolean on message) would only work for 1-on-1.

**SQLite Search Limitation**: Using LIKE queries for search. When migrating to PostgreSQL in production, upgrade to `pg_search` gem for full-text search with better performance.

**File Upload Security**: Validating file types and sizes. Consider adding virus scanning in production (ClamAV or cloud service).

**Turbo Streams Strategy**: Server-side rendering for real-time updates leverages Rails strengths and minimizes JavaScript. Alternative (React/Vue) would require more client-side code.

## Future Enhancements (Post-MVP)
- Typing indicators
- Online/offline status
- Message editing
- Message reactions (emoji)
- Threaded replies
- Voice messages
- Video calls (WebRTC)
- Push notifications
- Email notifications
- Message pinning
- User @mentions
- Rich text editor (Markdown)
- Link previews
- Admin dashboard for moderation
- User blocking
- Private/invite-only rooms
- Analytics

## Estimated Implementation Time
- Phase 1 (Auth): 1-2 hours
- Phase 2 (Models): 2-3 hours
- Phase 3 (Action Cable): 1-2 hours
- Phase 4 (Controllers): 3-4 hours
- Phase 5 (Routes): 30 minutes
- Phase 6 (Views): 4-6 hours
- Phase 7 (JavaScript): 1 hour
- Phase 8 (CSS): 2-3 hours
- Phase 9 (Testing): 1-2 hours

**Total**: 15-23 hours
