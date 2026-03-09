# Celipa Mobile App Architecture - Complete Analysis

## Table of Contents
1. [App Overview](#app-overview)
2. [Technology Stack](#technology-stack)
3. [App Structure](#app-structure)
4. [Navigation Architecture](#navigation-architecture)
5. [State Management](#state-management)
6. [Screen Flow & User Journeys](#screen-flow--user-journeys)
7. [Receipt Flow - Deep Dive](#receipt-flow---deep-dive)
8. [API Integration](#api-integration)
9. [Real-time Communication](#real-time-communication)
10. [Data Flow Diagrams](#data-flow-diagrams)
11. [Microservices Integration](#microservices-integration)

---

## App Overview

**Celipa** is a React Native mobile application built with Expo that enables users to split restaurant bills and receipts among groups. The app provides:

- Receipt scanning via OCR
- Real-time collaborative receipt splitting
- Payment tracking and confirmation
- User profile management
- Receipt history

---

## Technology Stack

### Core Framework
- **React Native** (0.72.6)
- **Expo SDK** (~49.0.15)
- **React** (17.0.2)
- **React Navigation** (v6) - Stack, Tab, Drawer navigators

### State Management
- **Redux** (4.1.2)
- **Redux Persist** (6.0.0) - AsyncStorage persistence
- **Redux Thunk** (2.4.0)

### Real-time Communication
- **Socket.io Client** (4.4.1)

### UI Libraries
- **React Native Paper** (4.10.0)
- **React Native Elements** (3.4.2)
- **Expo Vector Icons**

### Key Features
- **Expo Camera** - Receipt scanning
- **Expo Image Picker** - Photo selection
- **Expo Notifications** - Push notifications
- **Expo Location** - Location tracking
- **Expo Contacts** - Contact integration
- **React Native QR Code SVG** - QR code generation

---

## App Structure

```
celipa_app/
├── App.js                    # Root component, Redux Provider
├── index.js                  # Entry point
├── Navigation/               # Navigation configuration
│   ├── AuthChanger.js        # Auth state router
│   ├── RootStackScreen.js    # Auth screens stack
│   ├── HomeStackNavigation.js # Main app stack
│   └── MainTabNavigationV2.js # Bottom tab navigator
├── Screens/                  # Screen components (25 screens)
│   ├── Home.js              # Dashboard/home screen
│   ├── ReceiptPicture.js    # Camera/receipt scanning
│   ├── UserReceipt.js       # Receipt creation/editing
│   ├── VirtualReceipt.js    # Main receipt splitting screen
│   ├── JoinReceipt.js       # Join receipt by code
│   ├── PaymentUpdates.js    # Payment tracking
│   └── ...
├── Components/              # Reusable components
│   ├── modals/              # Modal components (29 modals)
│   ├── cards/               # Card components
│   └── ...
├── Apis/                    # API layer
│   ├── Services/
│   │   └── api.js           # API service functions
│   └── Utils/
│       ├── Request.js       # HTTP request wrapper
│       └── strings.js       # Constants
├── redux/                    # State management
│   ├── actions/             # Redux actions
│   ├── reducers/            # Redux reducers
│   └── middleware/          # Redux middleware
├── Auth/                     # Authentication screens
├── calculations/            # Business logic
│   └── receiptTotals.js     # Receipt calculations
└── assets/                  # Images, fonts, etc.
```

---

## Navigation Architecture

### Navigation Hierarchy

```
App.js (Root)
│
├── AuthChanger (Conditional Router)
│   │
│   ├── RootStackScreen (Unauthenticated)
│   │   ├── SplashScreen
│   │   ├── CellPhoneNumber
│   │   ├── VerifyCode
│   │   └── SignUpUpdate
│   │
│   └── HomeStackScreen (Authenticated)
│       │
│       ├── MainTabNavigatorV2 (Bottom Tabs)
│       │   ├── Receipts Tab → PreviousReceiptScreen
│       │   ├── Home Tab → HomeScreen
│       │   └── Profile Tab → ProfileScreen
│       │
│       └── Stack Screens (Modal/Full Screen)
│           ├── ReceiptPictureScreen
│           ├── UserReceiptScreen
│           ├── VirtualReceiptScreen
│           ├── JoinReceiptScreen
│           ├── PaymentUpdates
│           ├── EditProfileScreen
│           └── ... (20+ screens)
```

### Navigation Flow Diagram

```
┌─────────────────────────────────────────────────────────┐
│                    App Launch                            │
└────────────────────┬────────────────────────────────────┘
                     │
                     ▼
            ┌─────────────────┐
            │   AuthChanger   │
            │  (Check Token)  │
            └────────┬─────────┘
                     │
        ┌────────────┴────────────┐
        │                         │
        ▼                         ▼
┌───────────────┐        ┌───────────────┐
│ No Token      │        │ Has Token     │
│               │        │               │
│ RootStack     │        │ HomeStack     │
│ (Auth Flow)   │        │ (Main App)    │
└───────┬───────┘        └───────┬───────┘
        │                         │
        │                         │
        ▼                         ▼
┌───────────────┐        ┌───────────────┐
│ Splash        │        │ MainTabs      │
│ → Phone       │        │   ├─ Home     │
│ → Verify      │        │   ├─ Receipts │
│ → SignUp      │        │   └─ Profile  │
└───────────────┘        └───────┬───────┘
                                 │
                    ┌────────────┼────────────┐
                    │            │            │
                    ▼            ▼            ▼
            ┌───────────┐ ┌──────────┐ ┌──────────┐
            │Scan Receipt│ │Join Code │ │View Past │
            │            │ │          │ │Receipts  │
            └─────┬──────┘ └────┬─────┘ └────┬────┘
                  │             │            │
                  ▼             ▼            ▼
            ┌──────────────────────────────┐
            │    VirtualReceipt Screen      │
            │  (Main Receipt Interaction)   │
            └──────────────────────────────┘
```

---

## State Management

### Redux Store Structure

```javascript
{
  currentUser: {
    user: {
      _id: String,
      firstName: String,
      lastName: String,
      email: String,
      phoneNumber: Number,
      profilePicture: String,
      preferredPayment: Array,
      receipts: [ObjectId],
      userColor: String,
      UserLocation: { lat, lng },
      detailedLocation: Object,
      fcmToken: String,
      expoPushToken: String
    },
    userToken: String
  },
  login: {
    // Login state
  },
  loader: {
    // Loading state
  },
  resetPasswordId: {
    // Password reset state
  },
  loadingBar: {
    // Loading bar state
  }
}
```

### Redux Actions

- `setUser(user, token)` - Set authenticated user
- `setNoUser()` - Clear user state
- `logout()` - Logout user
- `retrieveToken(token)` - Retrieve stored token

### Redux Persist

- **Storage**: AsyncStorage
- **Key**: "root"
- **Persisted**: `currentUser` state

---

## Screen Flow & User Journeys

### 1. Authentication Flow

```
SplashScreen
    │
    ▼
CellPhoneNumberScreen
    │ (Enter phone number)
    ▼
VerifyCodeScreen
    │ (Enter verification code)
    ▼
SignUpUpdateScreen
    │ (Complete profile)
    ▼
HomeScreen (Authenticated)
```

### 2. Receipt Creation Flow

```
HomeScreen
    │ (Tap "New receipt" button)
    ▼
ReceiptPictureScreen
    │ (Take photo or select from gallery)
    │ (Upload to Veryfi OCR)
    ▼
UserReceiptScreen
    │ (Review/Edit extracted items)
    │ (Edit restaurant name, tax, items)
    │ (Confirm receipt)
    ▼
VirtualReceiptScreen
    │ (Assign items to participants)
    │ (Add participants)
    │ (Set tip)
    │ (Confirm payment)
    ▼
PaymentUpdatesScreen (if creator)
    OR
ConfirmationScreen (if participant)
```

### 3. Join Receipt Flow

```
HomeScreen
    │ (Tap "Join receipt")
    ▼
JoinReceiptScreen
    │ (Enter 5-digit code OR scan QR)
    │ (API: GET /receipt/join/:userId/:code)
    ▼
VirtualReceiptScreen
    │ (View receipt, assign items)
    │ (Confirm payment)
    ▼
ConfirmationScreen
```

### 4. Receipt History Flow

```
MainTabs → Receipts Tab
    │
    ▼
PreviousReceiptScreen
    │ (List of all user receipts)
    │ (Search receipts)
    │ (Tap receipt)
    ▼
SelectedReceiptScreen
    │ (View receipt details)
    │ (Tap "View receipt")
    ▼
VirtualReceiptScreen
```

### 5. Profile Management Flow

```
MainTabs → Profile Tab
    │
    ▼
ProfileScreen
    │ (View profile)
    │ (Tap "Edit Profile")
    ▼
EditProfileScreen
    │ (Update profile, upload picture)
    │ (API: PATCH /user/profile)
    │
    │ (Tap "Payment Methods")
    ▼
PrefferedPaymentScreen
    │ (Update payment preferences)
    │ (API: PUT /user/payment/preferred)
```

---

## Receipt Flow - Deep Dive

### VirtualReceipt Screen - Complete Analysis

The **VirtualReceipt** screen is the core of the receipt splitting functionality. Here's a complete breakdown:

#### Screen Purpose
- Display receipt items
- Allow participants to assign items to themselves
- Show participant avatars
- Calculate individual totals
- Handle tip and tax
- Confirm payments
- Real-time updates via Socket.io

#### Key State Variables

```javascript
// Receipt Data
const [receipt, setReceipt] = useState(null);
const [receiptId, setReceiptId] = useState(null);
const [menuItems, setMenuItems] = useState([]);
const [participants, setParticipants] = useState([]);

// Calculations
const [receiptSubtotal, setReceiptSubtotal] = useState(0);
const [receiptTax, setReceiptTax] = useState(0);
const [receiptTip, setReceiptTip] = useState(0);
const [total, setTotal] = useState(0);

// UI State
const [receiptEditMode, setReceiptEditMode] = useState(false);
const [loading, setLoading] = useState(true);
const [refreshing, setRefreshing] = useState(false);

// Modals
const [assignItemsModal, setAssignItemsModal] = useState(false);
const [tipModal, setAddTipModal] = useState(false);
const [acceptPaymentModal, setAcceptPaymentModal] = useState(false);
const [inviteParticipant, setShowInviteParticipant] = useState(false);
const [singleReceipt, setSingleReceipt] = useState(false);

// Participant Selection (for receipt creator)
const [currentSelectorID, setCurrentSelectorID] = useState(null);
const [createdParticipants, setCreatedParticipants] = useState([]);
```

#### Component Structure

```
VirtualReceipt
├── Header
│   ├── Back Button
│   └── Title: "Tap to select the items you ordered"
│
├── Participant Section
│   ├── Participant Avatars (ScrollView horizontal)
│   │   ├── Current User (highlighted)
│   │   ├── Receipt Creator (bordered)
│   │   └── Other Participants
│   ├── Add Participant Button
│   └── Participant Selector Dropdown (if creator)
│
├── Receipt Details
│   ├── Receipt Name
│   └── Edit Receipt Button (if creator)
│
├── Items List (ScrollView)
│   ├── ViewReceiptItemsCard (view mode)
│   │   └── OR
│   └── EditReceiptCard (edit mode)
│       ├── Item Name
│       ├── Item Quantity
│       ├── Item Price
│       ├── Assigned Participants (avatars)
│       └── Item Actions Menu
│
├── Receipt Totals
│   ├── Subtotal
│   ├── Tax (editable if creator)
│   ├── Tip (editable if creator)
│   └── Total
│
└── Action Button
    └── "Confirm selection" Button
```

#### Key Functions

**1. Initial Data Loading**
```javascript
const queryReceiptData = async () => {
  // 1. Initialize Socket.io connection
  const socket = io(SocketEndpoint, { transports: ["websocket"] });
  
  // 2. Fetch receipt data
  const receipt = await getReceipt(receiptId);
  const menuItems = await getMenuItems(receiptId);
  const participants = await getParticipants(receiptId);
  const allReceipts = await getAllMenuItemsFromAllParticipants(receiptId);
  
  // 3. Set up Socket.io listener
  socket.on(receiptId, (info) => {
    // Update state when receipt changes
    setMenuItems([...info.receiptMenu]);
    setParticipants([...info.receiptParticipants]);
    setReceipt(info.receipt);
    // ... update other state
  });
  
  // 4. Set local state
  setMenuItems([...menuItems.receipt]);
  setParticipants([...participants.receipt]);
  setReceipt(receipt.receipt);
  // ... calculate totals
};
```

**2. Assign Item to Participant**
```javascript
const assignItem = async (userId, item) => {
  // If item quantity > 2, show quantity modal
  if (item.itemQuantity > 2) {
    setItemQuantityModalOpen(item);
  } else {
    // Directly assign item
    await addParticipantToMenu(userId, item._id, receiptId, 1);
    // Socket.io will broadcast update automatically
  }
};
```

**3. Edit Receipt (Creator Only)**
```javascript
const submitEditedItems = async () => {
  const editedItems = menuItemsEdit.filter(item => item.edited);
  const itemsAdded = menuItemsEdit.filter(item => item.addedItem);
  
  // Update via API
  await updateReceiptItemsAndTipDetails(receiptId, {
    editedItems,
    itemsAdded,
    tipEdited: editedTip,
    taxEdited: receiptTax,
    receiptTotal: calculatedTotal
  });
  
  // Socket.io broadcasts update to all participants
};
```

**4. Confirm Payment**
```javascript
const confirmPayment = async (userTotal) => {
  // 1. Update payment status
  await updateConfirmPaymentStatus(userId, receiptId);
  
  // 2. Navigate based on role
  if (isCreator) {
    navigation.navigate(PaymentUpdates);
  } else {
    navigation.navigate(ConfirmationScreen);
  }
};
```

#### Socket.io Integration

**Connection Setup**
```javascript
const socket = io(SocketEndpoint, { transports: ["websocket"] });

// Listen for receipt updates
socket.on(receiptId, (info) => {
  // Receipt was updated (item assigned, tip changed, etc.)
  setMenuItems([...info.receiptMenu]);
  setParticipants([...info.receiptParticipants]);
  setReceipt(info.receipt);
  calculateReceiptTotal(info.receiptMenu);
});

// Listen for payment updates
socket.on(`${receiptId}pu`, (info) => {
  // Payment status changed
  setAllUsersReceipts([...info.allitemsParticipants]);
});
```

**Events Received**
- `receiptId` - Full receipt update (items, participants, totals)
- `${receiptId}pu` - Payment updates only

**Events Triggered** (via API calls that trigger server-side Socket.io)
- Item assignment → Server broadcasts to all participants
- Tip update → Server broadcasts to all participants
- Participant added → Server broadcasts to all participants
- Payment confirmed → Server broadcasts to all participants

#### Modals Used

1. **InviteParticipantsModal** - Add participants via phone/contacts
2. **AssignItemsToParticipantsModal** - Assign item to specific participant
3. **SingleReceiptModal** - View individual participant's items/total
4. **AddTipModal** - Add/edit tip percentage
5. **AcceptPaymentModal** - Confirm payment and select payment method
6. **PaymentMethodsModal** - Choose payment method (Venmo, CashApp, etc.)
7. **EditItemModal** - Edit item details
8. **RemoveItemModal** - Delete item from receipt
9. **ItemQuantityModal** - Select quantity for items with quantity > 2
10. **AddLocalContactsModal** - Add participants from phone contacts
11. **ExitReceiptModal** - Confirm exit from receipt

---

## API Integration

### API Service Layer (`Apis/Services/api.js`)

All API calls go through a centralized service layer:

```javascript
// Base URL
const SERVER_URL = "https://server.celipa.co";

// Request wrapper (Apis/Utils/Request.js)
// - Adds JWT token from AsyncStorage
// - Handles JSON/FormData
// - Error handling
```

### API Endpoints Used

#### Authentication
- `POST /user/login` - Login
- `POST /user/signup` - Register
- `POST /user/confirm` - Verify code
- `POST /user/phoneNumber/confirm` - Verify phone
- `POST /user/email` - Reset password

#### User Management
- `GET /user/:id` - Get user
- `PATCH /user/profile` - Update profile
- `PUT /user/payment/preferred` - Update payment preferences

#### Receipt Management
- `POST /receipt/create` - Create receipt
- `GET /current/user/:id` - Get receipt
- `GET /receipt/join/:userId/:receiptCode` - Join receipt
- `GET /user/receipt/:id` - Get user's receipts
- `GET /user/menuitems/:receiptId` - Get menu items
- `GET /user/participants/:receiptId` - Get participants

#### Receipt Items
- `PUT /menu/addparticipant/:userId/:menuId/:receiptId/:quantity` - Assign item
- `PUT /receipt/update/item/:itemId/:receiptId` - Update item
- `DELETE /receipt/item/:itemId/:receiptId` - Delete item
- `POST /receipt/update/items/tip/:receiptId` - Update items & tip

#### Participants
- `PUT /receipt/add` - Add participant
- `POST /receipt/participants/localparticipants/:receiptId` - Add from contacts
- `PUT /receipt/status/:userId/:receiptId` - Update send status
- `PUT /receipt/confirmStatus/:userId/:receiptId` - Confirm payment

#### OCR
- `GET /global/credentials` - Get Veryfi credentials
- (Direct Veryfi API call) - Upload image for OCR

#### Analytics
- `GET /user/receipts/totals/:userId` - Get total spent

### Request Flow

```
Component
    │
    ▼
api.js (Service Function)
    │
    ▼
Request.js (Wrapper)
    │
    ├── Get token from AsyncStorage
    ├── Add Authorization header
    ├── Format body (JSON/FormData)
    │
    ▼
fetch(SERVER_URL + endpoint)
    │
    ▼
Server Response
    │
    ▼
Component State Update
```

---

## Real-time Communication

### Socket.io Architecture

**Connection Pattern**
```javascript
// Each screen that needs real-time updates creates its own connection
const socket = io(SocketEndpoint, { transports: ["websocket"] });

// Join receipt room (implicit via event name)
socket.on(receiptId, (data) => {
  // Handle receipt update
});
```

**Screens Using Socket.io**
1. **VirtualReceipt** - Main receipt updates
2. **PaymentUpdates** - Payment status updates

**Event Flow**

```
User Action (e.g., assign item)
    │
    ▼
API Call (PUT /menu/addparticipant/...)
    │
    ▼
Server Processes Request
    │
    ├── Updates Database
    ├── Fetches Updated Receipt Data
    └── Emits Socket.io Event
        │
        ▼
Socket.io Broadcasts to Receipt Room
    │
    ▼
All Connected Clients Receive Update
    │
    ▼
React State Updates
    │
    ▼
UI Re-renders
```

**Socket Events**

**Client → Server** (via API calls, not direct Socket events)
- All updates happen via REST API
- Server then broadcasts via Socket.io

**Server → Client**
- `receiptId` - Full receipt update
  ```javascript
  {
    receiptParticipants: [...],
    receiptMenu: [...],
    receipt: {...},
    allReceipts: [...],
    tipStatus: boolean
  }
  ```
- `${receiptId}pu` - Payment updates
  ```javascript
  {
    allitemsParticipants: [...]
  }
  ```

---

## Data Flow Diagrams

### 1. Receipt Creation Flow

```
┌──────────────┐
│  HomeScreen  │
└──────┬───────┘
       │ User taps "Scan receipt"
       ▼
┌──────────────────┐
│ ReceiptPicture   │
│   Screen         │
│                  │
│ 1. Take photo    │
│ 2. Compress      │
│ 3. Upload to     │
│    Veryfi OCR    │
└──────┬───────────┘
       │ OCR Response
       ▼
┌──────────────────┐
│  UserReceipt     │
│   Screen         │
│                  │
│ 1. Display items │
│ 2. Edit items    │
│ 3. Edit tax      │
│ 4. Edit name     │
│ 5. Confirm       │
└──────┬───────────┘
       │ POST /receipt/create
       ▼
┌──────────────────┐
│  Server          │
│                  │
│ 1. Create receipt│
│ 2. Create items   │
│ 3. Add creator    │
│ 4. Upload image  │
│ 5. Return receipt│
└──────┬───────────┘
       │ Receipt data
       ▼
┌──────────────────┐
│ VirtualReceipt   │
│   Screen         │
│                  │
│ 1. Display items │
│ 2. Socket.io     │
│    connection    │
│ 3. Assign items  │
└──────────────────┘
```

### 2. Item Assignment Flow

```
┌──────────────────┐
│ VirtualReceipt   │
│   Screen         │
└──────┬───────────┘
       │ User taps item
       ▼
┌──────────────────┐
│ assignItem()     │
│                  │
│ Check quantity   │
└──────┬───────────┘
       │
   ┌───┴───┐
   │       │
   ▼       ▼
qty > 2  qty <= 2
   │       │
   │       ▼
   │   Direct assign
   │   PUT /menu/addparticipant
   │       │
   │       ▼
   │   Server updates DB
   │       │
   │       ▼
   │   Socket.io broadcast
   │       │
   │       ▼
   │   All clients update
   │
   ▼
Show quantity modal
   │
   ▼
User selects quantity
   │
   ▼
PUT /menu/addparticipant
   │
   ▼
Server updates DB
   │
   ▼
Socket.io broadcast
   │
   ▼
All clients update
```

### 3. Payment Confirmation Flow

```
┌──────────────────┐
│ VirtualReceipt   │
│   Screen         │
└──────┬───────────┘
       │ User taps "Confirm selection"
       ▼
┌──────────────────┐
│ AcceptPayment    │
│   Modal          │
│                  │
│ 1. Show total    │
│ 2. Show items    │
│ 3. Confirm       │
└──────┬───────────┘
       │ User confirms
       ▼
┌──────────────────┐
│ PaymentMethods   │
│   Modal          │
│                  │
│ Select method:   │
│ - Venmo          │
│ - CashApp        │
│ - PayPal         │
│ - Zelle          │
│ - Apple Pay      │
│ - Cash           │
└──────┬───────────┘
       │ User selects
       ▼
┌──────────────────┐
│ confirmPayment() │
│                  │
│ PUT /receipt/    │
│ confirmStatus    │
└──────┬───────────┘
       │
       ▼
┌──────────────────┐
│  Server          │
│                  │
│ 1. Update status │
│ 2. Send email    │
│ 3. Socket.io     │
│    broadcast     │
└──────┬───────────┘
       │
   ┌───┴───┐
   │       │
   ▼       ▼
Creator  Participant
   │       │
   ▼       ▼
Payment   Confirmation
Updates   Screen
Screen
```

---

## Microservices Integration

### Current Architecture (Monolithic)

```
┌─────────────────────────────────────┐
│      React Native App               │
│                                     │
│  ┌──────────────────────────────┐  │
│  │   API Service Layer          │  │
│  │   (api.js)                   │  │
│  └──────────────┬───────────────┘  │
│                 │                  │
│  ┌──────────────▼───────────────┐  │
│  │   Request Wrapper            │  │
│  │   (Request.js)                │  │
│  └──────────────┬───────────────┘  │
└─────────────────┼──────────────────┘
                  │
                  │ HTTPS/WSS
                  │
┌─────────────────▼──────────────────┐
│   Monolithic Server                │
│   (server.celipa.co)               │
│                                    │
│  ┌──────────────────────────────┐ │
│  │   Express Routes             │ │
│  │   - /user/*                  │ │
│  │   - /receipt/*               │ │
│  │   - /menu/*                  │ │
│  │   - /global/*                │ │
│  └──────────────┬───────────────┘ │
│                 │                  │
│  ┌──────────────▼───────────────┐ │
│  │   Controllers                │ │
│  │   - auth.controller          │ │
│  │   - receipt.controller       │ │
│  │   - user.controller          │ │
│  └──────────────┬───────────────┘ │
│                 │                  │
│  ┌──────────────▼───────────────┐ │
│  │   Services                   │ │
│  │   - receipt.service          │ │
│  │   - user.service             │ │
│  └──────────────┬───────────────┘ │
│                 │                  │
│  ┌──────────────▼───────────────┐ │
│  │   MongoDB                    │ │
│  │   (Single Database)          │ │
│  └──────────────────────────────┘ │
│                                    │
│  ┌──────────────────────────────┐ │
│  │   Socket.io                  │ │
│  │   (Real-time updates)        │ │
│  └──────────────────────────────┘ │
└────────────────────────────────────┘
```

### Proposed Microservices Architecture

```
┌─────────────────────────────────────┐
│      React Native App               │
│                                     │
│  ┌──────────────────────────────┐  │
│  │   API Service Layer          │  │
│  │   (api.js)                   │  │
│  └──────────────┬───────────────┘  │
│                 │                  │
│  ┌──────────────▼───────────────┐  │
│  │   Request Wrapper            │  │
│  │   (Request.js)               │  │
│  │                              │  │
│  │   - Adds JWT token           │  │
│  │   - Formats requests         │  │
│  │   - Error handling           │  │
│  └──────────────┬───────────────┘  │
└─────────────────┼──────────────────┘
                  │
                  │ HTTPS/WSS
                  │
┌─────────────────▼──────────────────┐
│      API Gateway                   │
│  (Kong / AWS API Gateway)          │
│                                    │
│  - Request routing                 │
│  - JWT validation                  │
│  - Rate limiting                   │
│  - CORS handling                   │
└──────┬─────────────────────────────┘
       │
       │ Routes to appropriate service
       │
┌──────┴─────────────────────────────┐
│                                    │
│  ┌──────────────────────────────┐ │
│  │  Authentication Service      │ │
│  │  POST /auth/login            │ │
│  │  POST /auth/register         │ │
│  │  POST /auth/verify           │ │
│  └──────────────────────────────┘ │
│                                    │
│  ┌──────────────────────────────┐ │
│  │  User Service               │ │
│  │  GET /users/:id             │ │
│  │  PATCH /users/:id           │ │
│  │  PUT /users/:id/payment     │ │
│  └──────────────────────────────┘ │
│                                    │
│  ┌──────────────────────────────┐ │
│  │  Receipt Service             │ │
│  │  POST /receipts              │ │
│  │  GET /receipts/:id           │ │
│  │  GET /receipts/join/:code    │ │
│  └──────────────────────────────┘ │
│                                    │
│  ┌──────────────────────────────┐ │
│  │  Receipt Items Service       │ │
│  │  GET /receipts/:id/items     │ │
│  │  PUT /receipts/:id/items/:id│ │
│  │  POST /receipts/:id/items   │ │
│  └──────────────────────────────┘ │
│                                    │
│  ┌──────────────────────────────┐ │
│  │  Receipt Participants Service │ │
│  │  POST /receipts/:id/         │ │
│  │      participants            │ │
│  │  PUT /receipts/:id/          │ │
│  │      participants/:id/status │ │
│  └──────────────────────────────┘ │
│                                    │
│  ┌──────────────────────────────┐ │
│  │  OCR Service                 │ │
│  │  POST /ocr/scan              │ │
│  │  GET /ocr/credentials        │ │
│  └──────────────────────────────┘ │
│                                    │
│  ┌──────────────────────────────┐ │
│  │  Calculation Service         │ │
│  │  POST /calculate/tip        │ │
│  │  POST /calculate/total       │ │
│  └──────────────────────────────┘ │
│                                    │
│  ┌──────────────────────────────┐ │
│  │  Notification Service        │ │
│  │  POST /notifications/sms     │ │
│  │  POST /notifications/email   │ │
│  └──────────────────────────────┘ │
│                                    │
│  ┌──────────────────────────────┐ │
│  │  File Storage Service        │ │
│  │  POST /files/upload          │ │
│  │  GET /files/:id              │ │
│  └──────────────────────────────┘ │
│                                    │
│  ┌──────────────────────────────┐ │
│  │  Real-time Service           │ │
│  │  (WebSocket)                 │ │
│  │  ws://realtime.celipa.co     │ │
│  └──────────────────────────────┘ │
│                                    │
│  ┌──────────────────────────────┐ │
│  │  Message Queue               │ │
│  │  (RabbitMQ/Kafka)            │ │
│  │  - Event-driven updates      │ │
│  └──────────────────────────────┘ │
└────────────────────────────────────┘
```

### App Changes Required for Microservices

#### 1. API Service Layer Updates

**Current (api.js)**
```javascript
export function getReceipt(id) {
  return request(`/current/user/${id}`, { method: "GET" });
}
```

**Microservices (api.js)**
```javascript
// Option 1: API Gateway handles routing (no app changes needed)
export function getReceipt(id) {
  return request(`/receipts/${id}`, { method: "GET" });
}

// Option 2: Direct service calls (requires app changes)
const RECEIPT_SERVICE_URL = process.env.RECEIPT_SERVICE_URL;
export function getReceipt(id) {
  return request(`${RECEIPT_SERVICE_URL}/receipts/${id}`, { method: "GET" });
}
```

#### 2. Socket.io Connection Updates

**Current**
```javascript
const SocketEndpoint = SERVER_URL + "/";
const socket = io(SocketEndpoint, { transports: ["websocket"] });
```

**Microservices**
```javascript
// Connect to dedicated Real-time Service
const REALTIME_SERVICE_URL = process.env.REALTIME_SERVICE_URL;
const socket = io(REALTIME_SERVICE_URL, { transports: ["websocket"] });
```

#### 3. Request Wrapper Updates

**Current (Request.js)**
```javascript
export default async function request(url, options) {
  // Single SERVER_URL
  return fetch(SERVER_URL + url, newOptions);
}
```

**Microservices (Request.js)**
```javascript
// Option 1: API Gateway (no changes needed)
export default async function request(url, options) {
  return fetch(API_GATEWAY_URL + url, newOptions);
}

// Option 2: Service discovery
export default async function request(url, options) {
  const serviceUrl = getServiceUrl(url); // Route to correct service
  return fetch(serviceUrl + url, newOptions);
}
```

#### 4. Error Handling Updates

**Microservices Considerations**
- Service-specific error handling
- Retry logic for failed requests
- Circuit breaker pattern
- Fallback mechanisms

#### 5. Caching Strategy

**Recommendations**
- Cache receipt data locally
- Cache user profile
- Cache menu items
- Use Redux Persist for offline support

---

## Key App Features & Implementation

### 1. Receipt Scanning

**Flow:**
1. User opens camera (ReceiptPicture screen)
2. Takes photo or selects from gallery
3. Image compressed
4. Uploaded to Veryfi OCR API
5. Extracted data displayed (UserReceipt screen)
6. User edits/confirms
7. Receipt created via API

**API Calls:**
- `GET /global/credentials` - Get Veryfi credentials
- Direct Veryfi API call - Upload image
- `POST /receipt/create` - Create receipt with extracted data

**Microservices:**
- OCR Service handles scanning
- Receipt Service creates receipt
- File Storage Service stores image

### 2. Real-time Collaboration

**Implementation:**
- Socket.io connection per receipt
- Server broadcasts on any change
- Client updates state on receive
- UI re-renders automatically

**Events:**
- Item assignment
- Participant addition
- Tip/tax changes
- Payment confirmations

**Microservices:**
- Real-time Service manages connections
- Receipt Services emit events
- Message Queue for reliable delivery

### 3. Payment Tracking

**Flow:**
1. User confirms items
2. AcceptPaymentModal shows total
3. User selects payment method
4. Payment status updated
5. Email sent (if confirmed)
6. PaymentUpdates screen shows status

**API Calls:**
- `PUT /receipt/confirmStatus/:userId/:receiptId`
- `PUT /receipt/status/:userId/:receiptId`
- `POST /receipt/venmo/payments/:receiptId`

**Microservices:**
- Receipt Participants Service updates status
- Notification Service sends email
- Payment Service (future) handles payments

### 4. Offline Support

**Current:**
- Redux Persist stores user state
- Receipt data cached in Redux
- Limited offline functionality

**Microservices Enhancement:**
- Service Worker for API caching
- Local database (SQLite) for receipts
- Sync queue for offline actions

---

## Performance Considerations

### Current Optimizations
- Image compression before upload
- Lazy loading of screens
- Redux state caching
- Socket.io connection reuse

### Microservices Optimizations
- API response caching
- Batch API requests
- GraphQL for complex queries
- CDN for static assets

---

## Security Considerations

### Current
- JWT tokens in AsyncStorage
- HTTPS for all API calls
- Token in Authorization header

### Microservices
- Token validation at API Gateway
- Service-to-service authentication (mTLS)
- Token refresh mechanism
- Secure storage (Keychain/Keystore)

---

## Testing Strategy

### Unit Tests
- Calculation functions
- Redux reducers
- Utility functions

### Integration Tests
- API service layer
- Socket.io events
- Navigation flows

### E2E Tests
- Complete receipt flow
- Payment confirmation
- Real-time updates

---

## Deployment Considerations

### Current
- Expo managed workflow
- OTA updates via Expo Updates
- App Store / Play Store distribution

### Microservices
- Feature flags for gradual rollout
- A/B testing capabilities
- Service versioning
- Backward compatibility

---

## Migration Path

### Phase 1: Preparation
1. Update API service layer to use API Gateway
2. Add service discovery client
3. Implement retry logic
4. Add error handling

### Phase 2: Gradual Migration
1. Start with stateless services (Calculation)
2. Migrate OCR service
3. Migrate receipt services
4. Migrate real-time service

### Phase 3: Optimization
1. Implement caching
2. Add offline support
3. Optimize API calls
4. Performance tuning

---

## Conclusion

The Celipa mobile app is a well-structured React Native application with:
- Clear separation of concerns
- Redux for state management
- Socket.io for real-time updates
- Comprehensive receipt splitting functionality

**Key Strengths:**
- Modular component structure
- Real-time collaboration
- Comprehensive user flows
- Good error handling

**Areas for Improvement:**
- API service abstraction for microservices
- Better offline support
- Enhanced caching strategy
- Service discovery integration

The app is well-positioned to integrate with a microservices architecture with minimal changes, primarily in the API service layer and Socket.io connection management.
