# Design Document: Rural AI Ecosystem Mobile Application

## Overview

The Rural AI Ecosystem Mobile Application is a mobile-first platform that connects rural service seekers with service providers through an AI-powered voice interface. The system is designed for low-bandwidth environments and users with varying literacy levels, prioritizing voice interaction over text-based interfaces.

The architecture follows a client-server model with a mobile application (React Native for cross-platform support), a backend API server (Node.js/Express), a cloud-based LLM service for voice processing and intelligent skill tagging, and a geospatial database for ecosystem management. The system uses push notifications for real-time service matching, implements a reputation system to ensure service quality, and includes intelligent features like dynamic geofencing, demand clustering, and offline resilience.

Key design principles:
- **Accessibility First**: Voice-based interaction as the primary interface
- **Intelligent Matching**: NLP-based skill tagging and asset inference
- **Dynamic Adaptation**: Adjustable search radius based on supply and urgency
- **Collaborative Economy**: Demand clustering for group savings
- **Offline Resilience**: Core features work with intermittent connectivity
- **Low Bandwidth**: Optimized for rural network conditions with SMS fallbacks
- **Scalability**: Support for multiple village ecosystems
- **Privacy**: Minimal data collection with user consent

## Architecture

### System Components

```
┌─────────────────┐
│  Mobile App     │
│  (React Native) │
└────────┬────────┘
         │
         │ HTTPS/REST
         │
┌────────▼────────────────────────────────────────────────┐
│         API Gateway (Node.js/Express)                   │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌────────┐ │
│  │   Auth   │  │ Service  │  │  User    │  │Cluster │ │
│  │ Service  │  │ Matching │  │ Service  │  │Service │ │
│  └──────────┘  └──────────┘  └──────────┘  └────────┘ │
└────────┬────────────────────────────────────────────────┘
         │
    ┌────┴────┐
    │         │
┌───▼───┐ ┌──▼──────────┐
│  DB   │ │  Voice LLM  │
│(Mongo)│ │   Service   │
└───────┘ └─────────────┘
```

### Mobile Application Layer

The mobile app is built with React Native to support both iOS and Android from a single codebase. The app includes:

- **Voice Interface Module**: Captures audio, streams to backend, plays responses
- **Offline Storage**: Stores voice recordings locally when connectivity is unavailable
- **Service Request UI**: Visual forms as fallback for voice interaction
- **Notification Handler**: Receives and displays service matches and group offers
- **Location Services**: Tracks user location for ecosystem assignment
- **Offline Queue Manager**: Stores and syncs actions when connectivity returns
- **Connectivity Monitor**: Displays offline indicator and manages sync

### Backend API Layer

The backend provides RESTful APIs and WebSocket connections for real-time features:

- **Authentication Service**: Mobile number verification via SMS
- **User Service**: Profile management and onboarding
- **Intelligent Skill Tagging Service**: NLP-based skill analysis and categorization
- **Service Matching Engine**: Matches requests with providers based on tags, location, and compensation
- **Dynamic Geofencing Service**: Calculates supply index, urgency score, and dynamic radius
- **Demand Clustering Service**: Detects clusters and generates group offers
- **Ecosystem Manager**: Assigns users to ecosystems and handles cross-ecosystem coordination
- **Rating Service**: Manages reputation scores and feedback
- **Notification Service**: Sends push notifications and SMS fallbacks
- **Offline Sync Service**: Processes queued requests from offline users

### Voice LLM Integration

The Voice LLM Service processes natural language interactions:

- **Speech-to-Text**: Converts user audio to text (using services like Google Speech-to-Text or Whisper API)
- **Intent Recognition**: LLM identifies user intent (create request, register as provider, etc.)
- **Entity Extraction**: Extracts structured data (service type, compensation, timing)
- **Skill Analysis**: Analyzes natural language skill descriptions and generates specific tags and broad categories
- **Asset Inference**: Identifies mentioned assets and suggests related services
- **Urgency Detection**: Detects urgency keywords and calculates urgency scores
- **Dialogue Management**: Handles multi-turn conversations to collect missing information
- **Text-to-Speech**: Converts responses to audio (using services like Google Text-to-Speech)

### Data Storage

MongoDB is used for flexible schema and geospatial queries:

- **Users Collection**: User profiles, authentication, provider status, skills, assets
- **Service Requests Collection**: All service requests with status tracking, urgency scores, supply indexes
- **Ecosystems Collection**: Village definitions with geographic boundaries
- **Ratings Collection**: Service ratings and feedback
- **Transactions Collection**: Service completion records
- **Demand Clusters Collection**: Detected clusters and group offers
- **Labor Gangs Collection**: Temporary labor groups
- **Skill Taxonomy Collection**: Mappings between descriptions, tags, and categories

## Components and Interfaces

### User Authentication Component

**Responsibilities:**
- Generate and send SMS verification codes
- Validate verification codes
- Create and manage user sessions
- Handle session expiration and renewal

**Interface:**
```typescript
interface AuthService {
  sendVerificationCode(mobileNumber: string): Promise<{success: boolean, message: string}>
  verifyCode(mobileNumber: string, code: string): Promise<{success: boolean, token: string}>
  validateSession(token: string): Promise<{valid: boolean, userId: string}>
  refreshSession(token: string): Promise<{success: boolean, newToken: string}>
}
```

### Onboarding Component

**Responsibilities:**
- Guide users through profile creation
- Support both voice and manual input modes
- Validate required fields
- Store user profiles

**Interface:**
```typescript
interface OnboardingService {
  startVoiceOnboarding(userId: string): Promise<{sessionId: string}>
  processVoiceInput(sessionId: string, audioData: Buffer): Promise<{
    response: string,
    audioResponse: Buffer,
    complete: boolean,
    extractedData: Partial<UserProfile>
  }>
  submitManualProfile(userId: string, profile: UserProfile): Promise<{success: boolean}>
  completeOnboarding(userId: string): Promise<{success: boolean}>
}

interface UserProfile {
  name: string
  occupation: string
  activities: string[]
  servicesNeeded: string[]
  skillDescriptions: string[]
  location: GeoPoint
}
```

### Service Request Component

**Responsibilities:**
- Create and manage service requests
- Extract requirements through voice interaction
- Validate request completeness
- Update request status through lifecycle

**Interface:**
```typescript
interface ServiceRequestService {
  createRequest(userId: string, voiceSession: boolean): Promise<{requestId: string, sessionId?: string}>
  processVoiceInput(sessionId: string, audioData: Buffer): Promise<{
    response: string,
    audioResponse: Buffer,
    complete: boolean,
    extractedData: Partial<ServiceRequest>
  }>
  confirmAndPublish(requestId: string, request: ServiceRequest): Promise<{success: boolean}>
  updateStatus(requestId: string, status: RequestStatus): Promise<{success: boolean}>
  getRequestsByUser(userId: string, role: 'seeker' | 'provider'): Promise<ServiceRequest[]>
}

interface ServiceRequest {
  id: string
  seekerId: string
  serviceType: string
  specificTags: string[]
  broadCategories: string[]
  description: string
  location: GeoPoint
  timing: {
    startDate: Date
    duration: number
    flexible: boolean
  }
  compensation: number
  urgencyScore: UrgencyScore
  localSupplyIndex: number
  dynamicRadiusKm: number
  status: RequestStatus
  createdAt: Date
}

interface UrgencyScore {
  score: number // 0-100
  level: 'low' | 'medium' | 'high' | 'critical'
  detectedKeywords: string[]
}

type RequestStatus = 
  | 'open'
  | 'pending'
  | 'accepted'
  | 'in_progress'
  | 'completed'
  | 'confirmed'
  | 'cancelled'
```

### Service Provider Component

**Responsibilities:**
- Register users as service providers
- Process natural language skill descriptions
- Generate specific and broad category tags from descriptions
- Infer secondary services from asset ownership
- Support asset-sharing registration
- Manage provider skills and compensation expectations
- Handle provider availability status
- Track provider ratings and reputation

**Interface:**
```typescript
interface ServiceProviderService {
  registerProvider(userId: string, providerData: ProviderProfile): Promise<{success: boolean}>
  processSkillDescription(userId: string, description: string): Promise<{
    specificTags: string[],
    broadCategories: string[],
    inferredAssets: string[],
    suggestedServices: string[]
  }>
  registerAssetSharing(userId: string, asset: Asset): Promise<{success: boolean}>
  updateProviderProfile(userId: string, updates: Partial<ProviderProfile>): Promise<{success: boolean}>
  setAvailability(userId: string, active: boolean): Promise<{success: boolean}>
  getProviderProfile(userId: string): Promise<ProviderProfile>
}

interface ProviderProfile {
  userId: string
  skillDescriptions: string[] // Natural language descriptions
  specificTags: string[] // e.g., "coconut_harvesting"
  broadCategories: string[] // e.g., "agriculture_labor"
  assets: Asset[] // Owned equipment/resources
  assetSharingOnly: boolean // True if only renting assets, not providing labor
  compensationExpectations: Map<string, number> // Tag -> minimum compensation
  rating: number
  totalServices: number
  isNewbie: boolean
  active: boolean
}

interface Asset {
  type: string // e.g., "tractor", "water_pump"
  description: string
  availableForRent: boolean
  rentalRate?: number
  inferredServices: string[] // e.g., ["transport", "tillage"]
}
```

### Intelligent Skill Tagging Component

**Responsibilities:**
- Analyze natural language skill descriptions
- Generate specific skill tags and broad categories
- Infer assets from descriptions
- Suggest secondary services based on assets
- Maintain skill taxonomy and mappings

**Interface:**
```typescript
interface SkillTaggingService {
  analyzeSkillDescription(description: string): Promise<SkillAnalysis>
  inferAssetsFromDescription(description: string): Promise<Asset[]>
  suggestServicesForAsset(asset: Asset): Promise<string[]>
  getSkillTaxonomy(): Promise<SkillTaxonomy>
}

interface SkillAnalysis {
  originalDescription: string
  specificTags: string[]
  broadCategories: string[]
  confidence: number
  detectedAssets: Asset[]
  suggestedServices: string[]
}

interface SkillTaxonomy {
  categories: Map<string, string[]> // Category -> specific skills
  assetServiceMappings: Map<string, string[]> // Asset type -> services
}
```

### Dynamic Geofencing Component

**Responsibilities:**
- Calculate Local Supply Index for service requests
- Calculate Urgency Score from voice keywords
- Determine dynamic search radius based on supply and urgency
- Prevent ecosystem leakage for low-value tasks
- Override default radius for critical requests

**Interface:**
```typescript
interface DynamicGeofencingService {
  calculateLocalSupplyIndex(request: ServiceRequest, radiusKm: number): Promise<number>
  calculateUrgencyScore(transcription: string, keywords: string[]): Promise<UrgencyScore>
  determineDynamicRadius(request: ServiceRequest, supplyIndex: number, urgencyScore: UrgencyScore): Promise<DynamicRadiusResult>
  shouldPreventLeakage(request: ServiceRequest, distance: number): Promise<boolean>
}

interface DynamicRadiusResult {
  radiusKm: number
  reasoning: string
  includeNeighboring: boolean
  maxDistanceKm: number
  preventLeakage: boolean
}
```

### Demand Clustering Component

**Responsibilities:**
- Detect demand clusters for similar services
- Generate group offer notifications
- Manage labor gang formations
- Track cluster participation and savings

**Interface:**
```typescript
interface DemandClusteringService {
  detectClusters(ecosystemId: string, timeWindowHours: number): Promise<DemandCluster[]>
  generateGroupOffer(cluster: DemandCluster): Promise<GroupOffer>
  notifyClusterParticipants(cluster: DemandCluster, offer: GroupOffer): Promise<{notified: number}>
  createLaborGang(leaderId: string, memberIds: string[], requestId: string): Promise<LaborGang>
  acceptJobAsGang(gangId: string, requestId: string): Promise<{success: boolean}>
}

interface DemandCluster {
  id: string
  ecosystemId: string
  serviceType: string
  requestIds: string[]
  userIds: string[]
  detectedAt: Date
  potentialSavings: number
  clusterSize: number
}

interface GroupOffer {
  clusterId: string
  message: string
  participantCount: number
  estimatedSavingsPerUser: number
  expiresAt: Date
}

interface LaborGang {
  id: string
  leaderId: string
  memberIds: string[]
  active: boolean
  createdAt: Date
  completedJobs: number
}
```

### Offline Resilience Component

**Responsibilities:**
- Store voice recordings locally when offline
- Queue requests for upload when connectivity returns
- Send SMS confirmations as low-bandwidth fallback
- Manage offline/online state transitions
- Process queued requests in chronological order

**Interface:**
```typescript
interface OfflineResilienceService {
  storeOfflineVoiceRecording(userId: string, audioData: Buffer, metadata: RequestMetadata): Promise<{queueId: string}>
  getQueuedRequests(userId: string): Promise<QueuedRequest[]>
  uploadQueuedRequests(userId: string): Promise<{uploaded: number, failed: number}>
  sendSMSConfirmation(userId: string, mobileNumber: string, message: string): Promise<{sent: boolean}>
  getConnectivityStatus(): Promise<ConnectivityStatus>
}

interface QueuedRequest {
  queueId: string
  userId: string
  audioData: Buffer
  metadata: RequestMetadata
  queuedAt: Date
  attempts: number
  status: 'pending' | 'uploading' | 'uploaded' | 'failed'
}

interface RequestMetadata {
  requestType: 'service_request' | 'profile_update' | 'rating'
  approximateLocation?: GeoPoint
  timestamp: Date
}

interface ConnectivityStatus {
  online: boolean
  bandwidth: 'none' | 'low' | 'medium' | 'high'
  lastOnline?: Date
  queuedRequestCount: number
}
```

### Service Matching Engine

**Responsibilities:**
- Match service requests with qualified providers
- Filter by skills (including specific tags and broad categories), location, and compensation
- Calculate Local Supply Index and Urgency Score
- Determine dynamic ecosystem scope based on supply and urgency
- Prevent ecosystem leakage for low-value tasks
- Send notifications to matched providers

**Interface:**
```typescript
interface ServiceMatchingEngine {
  findMatches(request: ServiceRequest): Promise<MatchResult>
  evaluateEcosystemScope(request: ServiceRequest): Promise<EcosystemScopeResult>
  calculateLocalSupplyIndex(request: ServiceRequest): Promise<number>
  calculateUrgencyScore(request: ServiceRequest): Promise<UrgencyScore>
  notifyProviders(providerIds: string[], request: ServiceRequest): Promise<{notified: number}>
}

interface MatchResult {
  localProviders: ProviderMatch[]
  neighboringProviders: ProviderMatch[]
  supplyIndex: number
  urgencyScore: UrgencyScore
  dynamicRadius: number
  matchCriteria: {
    skillMatch: boolean
    compensationMatch: boolean
    locationMatch: boolean
  }
}

interface ProviderMatch {
  providerId: string
  matchScore: number
  specificTagMatches: string[]
  broadCategoryMatches: string[]
  distanceKm: number
}

interface EcosystemScopeResult {
  localOnly: boolean
  includeNeighboring: boolean
  dynamicRadiusKm: number
  reasoning: string
  preventLeakage: boolean
}
```

### Ecosystem Manager

**Responsibilities:**
- Define and manage village ecosystems
- Assign users to ecosystems based on location
- Identify neighboring ecosystems
- Handle cross-ecosystem coordination logic

**Interface:**
```typescript
interface EcosystemManager {
  assignUserToEcosystem(userId: string, location: GeoPoint): Promise<{ecosystemId: string}>
  getEcosystem(ecosystemId: string): Promise<Ecosystem>
  findNeighboringEcosystems(ecosystemId: string, maxDistanceKm: number): Promise<Ecosystem[]>
  shouldExtendToNeighbors(request: ServiceRequest, localMatches: number): Promise<boolean>
}

interface Ecosystem {
  id: string
  name: string
  centerPoint: GeoPoint
  radiusKm: number
  userCount: number
  providerCount: number
}

interface GeoPoint {
  latitude: number
  longitude: number
}
```

### Rating and Reputation Component

**Responsibilities:**
- Collect ratings from service seekers
- Calculate and update provider ratings
- Manage newbie status
- Prevent duplicate ratings

**Interface:**
```typescript
interface RatingService {
  submitRating(requestId: string, seekerId: string, providerId: string, rating: number, feedback?: string): Promise<{success: boolean}>
  getProviderRating(providerId: string): Promise<{
    averageRating: number,
    totalRatings: number,
    isNewbie: boolean
  }>
  hasRated(requestId: string, seekerId: string): Promise<boolean>
  updateProviderReputation(providerId: string): Promise<{newRating: number}>
}
```

### Voice LLM Service

**Responsibilities:**
- Convert speech to text
- Process natural language with LLM
- Extract structured data from conversations
- Analyze skill descriptions and generate tags
- Detect urgency keywords and calculate urgency scores
- Infer assets and suggest services
- Manage dialogue state
- Convert text responses to speech

**Interface:**
```typescript
interface VoiceLLMService {
  processVoiceInput(audioData: Buffer, context: ConversationContext): Promise<LLMResponse>
  extractEntities(text: string, schema: EntitySchema): Promise<ExtractedEntities>
  analyzeSkillDescription(description: string): Promise<SkillAnalysis>
  detectUrgencyKeywords(text: string): Promise<UrgencyScore>
  inferAssets(description: string): Promise<Asset[]>
  generateResponse(intent: string, entities: ExtractedEntities, context: ConversationContext): Promise<string>
  textToSpeech(text: string, language: string): Promise<Buffer>
}

interface ConversationContext {
  sessionId: string
  userId: string
  intent: string
  collectedData: Record<string, any>
  missingFields: string[]
  turnCount: number
}

interface LLMResponse {
  transcription: string
  intent: string
  entities: ExtractedEntities
  skillAnalysis?: SkillAnalysis
  urgencyScore?: UrgencyScore
  response: string
  audioResponse: Buffer
  complete: boolean
  confidence: number
}

interface ExtractedEntities {
  serviceType?: string
  compensation?: number
  location?: string
  timing?: {date: string, duration: string}
  description?: string
  skillDescription?: string
  assetMention?: string
}

interface EntitySchema {
  required: string[]
  optional: string[]
  types: Record<string, 'string' | 'number' | 'date' | 'enum'>
}
```

### Notification Service

**Responsibilities:**
- Send push notifications to mobile devices
- Send SMS notifications as fallback
- Handle notification delivery failures
- Track notification status
- Support notification preferences

**Interface:**
```typescript
interface NotificationService {
  sendServiceMatchNotification(providerId: string, request: ServiceRequest): Promise<{sent: boolean}>
  sendGroupOfferNotification(userIds: string[], offer: GroupOffer): Promise<{sent: number}>
  sendStatusUpdateNotification(userId: string, requestId: string, newStatus: RequestStatus): Promise<{sent: boolean}>
  sendRatingReminderNotification(seekerId: string, requestId: string): Promise<{sent: boolean}>
  sendSMSNotification(mobileNumber: string, message: string): Promise<{sent: boolean}>
  registerDeviceToken(userId: string, token: string): Promise<{success: boolean}>
}
```

## Data Models

### User Model

```typescript
interface User {
  _id: string
  mobileNumber: string
  name: string
  occupation: string
  activities: string[]
  servicesNeeded: string[]
  skillDescriptions: string[] // Natural language descriptions
  specificTags: string[] // Generated from LLM analysis
  broadCategories: string[] // Generated from LLM analysis
  assets: Asset[] // Owned equipment/resources
  location: {
    type: 'Point'
    coordinates: [number, number] // [longitude, latitude]
  }
  ecosystemId: string
  isProvider: boolean
  providerProfile?: ProviderProfile
  assetSharingOnly: boolean
  createdAt: Date
  lastActive: Date
  sessionToken?: string
  sessionExpiry?: Date
  deviceTokens: string[]
  offlineQueue: QueuedRequest[]
}
```

### Service Request Model

```typescript
interface ServiceRequestDocument {
  _id: string
  seekerId: string
  serviceType: string // No longer restricted to enum
  specificTags: string[] // Matched against provider tags
  broadCategories: string[] // Matched against provider categories
  description: string
  location: {
    type: 'Point'
    coordinates: [number, number]
  }
  timing: {
    startDate: Date
    duration: number // hours
    flexible: boolean
  }
  compensation: number
  status: RequestStatus
  ecosystemId: string
  crossEcosystem: boolean
  dynamicRadiusKm: number // Calculated based on supply and urgency
  localSupplyIndex: number
  urgencyScore: UrgencyScore
  urgencyKeywords: string[]
  interestedProviders: string[]
  acceptedProviderId?: string
  clusterId?: string // If part of a demand cluster
  completedAt?: Date
  ratedAt?: Date
  createdAt: Date
  updatedAt: Date
}
```

### Demand Cluster Model

```typescript
interface DemandClusterDocument {
  _id: string
  ecosystemId: string
  serviceType: string
  specificTags: string[]
  requestIds: string[]
  userIds: string[]
  detectedAt: Date
  groupOfferSent: boolean
  participantCount: number
  potentialSavings: number
  status: 'active' | 'expired' | 'fulfilled'
  expiresAt: Date
}
```

### Labor Gang Model

```typescript
interface LaborGangDocument {
  _id: string
  leaderId: string
  memberIds: string[]
  ecosystemId: string
  skills: string[]
  active: boolean
  createdAt: Date
  completedJobs: number
  totalEarnings: number
}
```

### Skill Taxonomy Model

```typescript
interface SkillTaxonomyDocument {
  _id: string
  specificTag: string
  broadCategory: string
  relatedTags: string[]
  commonDescriptions: string[] // Examples of how users describe this skill
  assetAssociations: string[] // Assets commonly associated with this skill
  updatedAt: Date
}
```

### Ecosystem Model

```typescript
interface EcosystemDocument {
  _id: string
  name: string
  location: {
    type: 'Point'
    coordinates: [number, number]
  }
  radiusKm: number
  neighboringEcosystems: string[]
  userCount: number
  providerCount: number
  averageCompensation: Map<string, number> // Service type -> average compensation
  createdAt: Date
}
```

### Rating Model

```typescript
interface RatingDocument {
  _id: string
  requestId: string
  seekerId: string
  providerId: string
  rating: number // 1-5
  feedback?: string
  createdAt: Date
}
```

### Transaction Model

```typescript
interface TransactionDocument {
  _id: string
  requestId: string
  seekerId: string
  providerId: string
  serviceType: string
  compensation: number
  completedAt: Date
  ecosystemId: string
  crossEcosystem: boolean
}
```

## Correctness Properties

*A property is a characteristic or behavior that should hold true across all valid executions of a system—essentially, a formal statement about what the system should do. Properties serve as the bridge between human-readable specifications and machine-verifiable correctness guarantees.*

### Authentication and Session Properties

Property 1: Verification code generation
*For any* valid mobile number, when a user requests authentication, the system should generate and send a unique verification code
**Validates: Requirements 1.1**

Property 2: Correct verification grants access
*For any* user with a valid verification code, when the code is submitted, the system should authenticate the user and create a valid session token
**Validates: Requirements 1.2**

Property 3: Invalid verification rejection
*For any* incorrect verification code, when submitted, the system should reject authentication and allow retry without locking the account
**Validates: Requirements 1.3**

Property 4: Mobile number uniqueness
*For any* two users in the system, their mobile numbers should be distinct (no duplicate mobile numbers)
**Validates: Requirements 1.4**

Property 5: Session persistence
*For any* authenticated user, their session token should remain valid for any time period less than 30 days from creation
**Validates: Requirements 1.5**

### Onboarding Properties

Property 6: Required field validation
*For any* onboarding attempt, the system should reject completion if name or occupation fields are empty or contain only whitespace
**Validates: Requirements 2.6**

Property 7: Profile persistence after onboarding
*For any* completed onboarding, the user profile should be stored with all provided fields and the account should be marked as active
**Validates: Requirements 2.5**

Property 8: Missing information prompts
*For any* voice-based interaction (onboarding or service request), when required fields are missing, the system should prompt for the missing information before allowing completion
**Validates: Requirements 2.4, 3.3**

### Service Request Properties

Property 9: Compensation requirement
*For any* service request, the system should reject creation if no compensation amount is specified
**Validates: Requirements 3.6**

Property 10: Request timestamping
*For any* published service request, it should have a creation timestamp that is set to the time of publication
**Validates: Requirements 3.7**

Property 11: Confirmation before posting
*For any* voice-based service request, the system should confirm all collected details with the user before publishing the request
**Validates: Requirements 3.4**

### Intelligent Profile Building Properties

Property 12: No skill restriction
*For any* skill description provided by a user, the system should accept it without requiring selection from a pre-defined list
**Validates: Requirements 4.1**

Property 13: Dual tagging from descriptions
*For any* natural language skill description, the Voice LLM should generate both specific tags and broad category tags
**Validates: Requirements 4.2**

Property 14: Asset inference and service suggestion
*For any* skill description that mentions asset ownership, the system should infer the asset and suggest related secondary services
**Validates: Requirements 4.4**

Property 15: Asset-sharing registration support
*For any* user registering with only assets (no labor skills), the system should allow asset-sharing-only registration
**Validates: Requirements 4.6**

Property 16: Skill or asset requirement
*For any* provider registration, the system should reject completion if neither skills nor assets are specified
**Validates: Requirements 4.10**

Property 17: Newbie tag assignment
*For any* newly registered service provider with zero completed services, their profile should display the "newbie" tag
**Validates: Requirements 4.7**

Property 18: Profile update persistence
*For any* service provider profile update, the changes should be saved and reflected in subsequent profile retrievals
**Validates: Requirements 4.9**

### Dynamic Geofencing Properties

Property 19: Local Supply Index calculation
*For any* incoming service request, the system should calculate a Local Supply Index indicating available provider count within the default radius
**Validates: Requirements 6.3**

Property 20: Urgency Score calculation
*For any* service request with voice transcription, the system should calculate an Urgency Score based on detected keywords
**Validates: Requirements 6.4**

Property 21: Critical expansion logic
*For any* service request where Local Supply Index equals zero and Urgency Score is high, the system should immediately expand the search radius to neighboring ecosystems up to 20km
**Validates: Requirements 6.5**

Property 22: Ecosystem leakage prevention
*For any* service request classified as low-value, the system should prevent expansion beyond the local ecosystem regardless of other factors
**Validates: Requirements 6.6**

Property 23: Ecosystem assignment
*For any* user completing onboarding, they should be assigned to exactly one ecosystem based on their geographic location
**Validates: Requirements 6.1**

Property 24: Location-based reassignment
*For any* user whose location changes to a different ecosystem's boundary, the system should update their ecosystem assignment
**Validates: Requirements 6.7**

### Service Matching Properties

Property 25: Tag-based matching
*For any* service request, the system should notify providers whose specific tags or broad categories match the request's tags or categories
**Validates: Requirements 5.1**

Property 26: Compensation filtering
*For any* service request, the system should exclude providers whose minimum compensation expectations exceed the offered amount
**Validates: Requirements 5.4**

Property 27: Notification content completeness
*For any* service match notification, it should include service type, location, timing, and compensation information
**Validates: Requirements 5.3**

### Demand Clustering Properties

Property 28: Cluster detection
*For any* ecosystem, when multiple service requests for similar services are created within 24 hours, the system should detect and group them as a demand cluster
**Validates: Requirements 9.3**

Property 29: Group offer generation
*For any* detected demand cluster, the system should generate a group offer notification to all participants
**Validates: Requirements 9.4**

Property 30: Labor gang formation
*For any* group of laborers with a designated leader, the system should allow formation of a labor gang that can accept jobs collectively
**Validates: Requirements 9.5**

### Offline Resilience Properties

Property 31: Offline voice recording
*For any* voice input when connectivity is unavailable, the app should store the recording locally
**Validates: Requirements 10.1, 10.2**

Property 32: Automatic upload on reconnection
*For any* queued offline request, when connectivity is restored, the system should automatically upload it to the cloud for processing
**Validates: Requirements 10.3**

Property 33: SMS confirmation fallback
*For any* voice request processed after offline queueing, the system should send an SMS confirmation to the user
**Validates: Requirements 10.4**

Property 34: Offline indicator display
*For any* connectivity loss, the app should display a clear offline indicator to the user
**Validates: Requirements 10.5**

Property 35: Chronological queue processing
*For any* set of queued offline requests, when connectivity returns, they should be processed in chronological order (oldest first)
**Validates: Requirements 10.6**

### Cross-Ecosystem Coordination Properties

Property 36: Time-based expansion
*For any* service request that receives no responses within the local ecosystem after 2 hours, the system should extend to neighboring ecosystems
**Validates: Requirements 11.1**

Property 37: High compensation expansion
*For any* service request where compensation exceeds the local ecosystem average by 20% or more, the system should immediately include neighboring ecosystems
**Validates: Requirements 11.2**

Property 38: Urgent request expansion
*For any* service request marked as urgent, the system should include neighboring ecosystems from the initial posting
**Validates: Requirements 11.3**

Property 39: Cross-ecosystem distance limit
*For any* cross-ecosystem service request, notifications should only be sent to providers in ecosystems within 10 kilometers of the requester
**Validates: Requirements 11.4**

Property 40: Cross-ecosystem notification content
*For any* service provider from a neighboring ecosystem who accepts a request, the notification to the seeker should include the provider's location and estimated travel time
**Validates: Requirements 11.5**

Property 41: Cross-ecosystem tracking
*For any* service completed by a provider from a different ecosystem, the transaction should be flagged as cross-ecosystem for analytics
**Validates: Requirements 11.6**

### Rating and Reputation Properties

Property 42: Rating prompt on completion
*For any* service marked as completed, the system should send a rating prompt to the service seeker
**Validates: Requirements 7.1, 9.6**

Property 43: Rating calculation
*For any* service provider, their displayed average rating should equal the mean of all ratings they have received
**Validates: Requirements 7.2**

Property 44: Newbie to rated transition
*For any* service provider, when they accumulate 5 or more ratings, the system should remove the "newbie" tag and display their average rating
**Validates: Requirements 7.5**

Property 45: Duplicate rating prevention
*For any* service request, a service seeker should not be able to submit more than one rating
**Validates: Requirements 7.6**

### Voice LLM Properties

Property 46: Speech-to-text processing
*For any* voice input, the system should convert the audio to text and extract the user's intent
**Validates: Requirements 8.2**

Property 47: Text-to-speech response
*For any* LLM-generated response, the system should convert it to audio and deliver it to the user
**Validates: Requirements 8.3**

Property 48: Ambiguity handling
*For any* voice input with ambiguous or incomplete information, the system should ask clarifying questions before proceeding
**Validates: Requirements 8.4**

Property 49: Action confirmation
*For any* action the Voice LLM performs on behalf of the user, the system should verbally confirm the action before execution
**Validates: Requirements 8.5, 13.2**

Property 50: Low confidence retry
*For any* voice recognition result with low confidence score, the system should ask the user to repeat their input
**Validates: Requirements 8.7**

### Service Lifecycle Properties

Property 51: Initial status
*For any* newly created service request, its status should be set to "open"
**Validates: Requirements 12.1**

Property 52: Status transition validity
*For any* service request, status transitions should follow the valid sequence: open → pending → accepted → in_progress → completed → confirmed
**Validates: Requirements 12.2, 12.3, 12.4, 12.5, 12.6**

Property 53: Rating trigger on confirmation
*For any* service request that transitions to "confirmed" status, the system should trigger a rating prompt to the service seeker
**Validates: Requirements 12.6**

Property 54: Stale request notification
*For any* service request that remains in "open" status for 24 hours with no interested providers, the system should notify the seeker with suggestions
**Validates: Requirements 12.7**

### Profile Management Properties

Property 55: Profile completeness display
*For any* user profile view, it should display name, mobile number, occupation, activities, skills, and provider status
**Validates: Requirements 13.1**

Property 56: Provider profile additional fields
*For any* user who is a service provider, their profile should additionally display rating, completed services count, and registered skills/assets
**Validates: Requirements 13.4**

Property 57: Manual update validation
*For any* manual profile update, the system should validate that required fields are non-empty before saving
**Validates: Requirements 13.3**

Property 58: Provider status toggle
*For any* service provider, they should be able to toggle their status between active and inactive
**Validates: Requirements 13.5**

Property 59: Inactive provider notification filtering
*For any* service provider with inactive status, they should not receive service request notifications
**Validates: Requirements 13.6**

### Compensation Tracking Properties

Property 60: Compensation storage
*For any* service request, the offered compensation amount should be stored and retrievable throughout the request lifecycle
**Validates: Requirements 14.1, 14.2, 14.4**

Property 61: Transaction recording
*For any* completed service, a transaction record should be created with service details and compensation amount
**Validates: Requirements 14.3**

Property 62: Transaction history completeness
*For any* user's transaction history, each entry should include date, service type, and compensation amount
**Validates: Requirements 14.5**

Property 63: Earnings and spending calculation
*For any* service provider, their total earnings should equal the sum of compensation from all their completed services; for any service seeker, their total spending should equal the sum of compensation from all their requested services
**Validates: Requirements 14.6**

### Privacy and Security Properties

Property 64: Data encryption at rest
*For any* sensitive user data (mobile numbers, location), it should be encrypted in the database storage
**Validates: Requirements 15.2**

Property 65: Location consent requirement
*For any* attempt to collect user location data, the system should first obtain explicit user consent
**Validates: Requirements 15.3**

Property 66: Account deletion compliance
*For any* user account deletion request, all personal information should be removed from the system within 30 days
**Validates: Requirements 15.4**

Property 67: Mobile number privacy
*For any* API response or user interface, user mobile numbers should not be exposed to other users without explicit consent
**Validates: Requirements 15.5**

Property 68: Access audit logging
*For any* access to user personal information, an audit log entry should be created with timestamp and accessor information
**Validates: Requirements 15.6**

Property 69: Voice recording deletion
*For any* voice recording processed by the system, the audio file should be deleted after transcription unless the user has opted in to storage
**Validates: Requirements 15.7**

## Error Handling

### Authentication Errors

- **Invalid Mobile Number**: Return error with message "Invalid mobile number format"
- **SMS Delivery Failure**: Retry up to 3 times, then return error "Unable to send verification code"
- **Expired Verification Code**: Return error "Verification code expired, please request a new one"
- **Session Expired**: Return 401 status, prompt user to re-authenticate

### Voice LLM Errors

- **Speech Recognition Failure**: Ask user to repeat input, fallback to manual input after 3 failures
- **Low Confidence Transcription**: Confirm transcription with user before processing
- **LLM Service Unavailable**: Fallback to manual input forms, queue voice requests for retry
- **Text-to-Speech Failure**: Display text response as fallback
- **Skill Tagging Failure**: Use generic broad category, flag for manual review

### Service Matching Errors

- **No Providers Found**: Notify seeker, suggest adjusting compensation or expanding search radius
- **Ecosystem Assignment Failure**: Use default ecosystem based on administrative region
- **Geolocation Unavailable**: Prompt user to manually select village/ecosystem
- **Supply Index Calculation Failure**: Use default radius without dynamic adjustment

### Data Validation Errors

- **Missing Required Fields**: Return specific error listing missing fields
- **Invalid Service Type**: Accept any string, generate tags via LLM
- **Invalid Compensation**: Return error "Compensation must be a positive number"
- **Invalid Rating**: Return error "Rating must be between 1 and 5"
- **No Skills or Assets**: Return error "Please provide at least one skill or asset"

### Network and Connectivity Errors

- **Request Timeout**: Retry with exponential backoff (1s, 2s, 4s)
- **Offline Mode**: Queue actions locally, sync when connectivity restored
- **Database Connection Failure**: Return 503 status, retry connection
- **Upload Failure**: Keep request in offline queue, retry on next connectivity check

### Notification Errors

- **Push Notification Failure**: Fallback to SMS notification
- **SMS Delivery Failure**: Log failure, mark as undelivered
- **Invalid Device Token**: Remove token from user's device list
- **Rate Limit Exceeded**: Queue notifications for delayed delivery

### Clustering and Gang Errors

- **Cluster Detection Failure**: Process requests individually without clustering
- **Group Offer Generation Failure**: Notify participants individually
- **Labor Gang Formation Failure**: Allow individual job acceptance as fallback

## Testing Strategy

### Unit Testing

Unit tests will focus on specific examples, edge cases, and error conditions:

- **Authentication**: Test specific verification code formats, session token generation
- **Validation**: Test boundary cases (empty strings, whitespace, special characters)
- **Calculations**: Test rating averages, compensation totals, supply index with specific examples
- **State Transitions**: Test specific lifecycle transitions and invalid transitions
- **Error Handling**: Test each error condition with specific inputs
- **Edge Cases**: Test empty lists, zero values, maximum values
- **Skill Tagging**: Test specific skill descriptions and expected tag outputs
- **Urgency Detection**: Test specific keyword combinations and expected scores

### Property-Based Testing

Property-based tests will verify universal properties across all inputs using **fast-check** (for TypeScript/JavaScript) or **Hypothesis** (for Python components). Each test will run a minimum of 100 iterations with randomly generated inputs.

**Test Configuration:**
- Minimum 100 iterations per property test
- Each test tagged with: `Feature: rural-ai-ecosystem, Property {N}: {property_text}`
- Random data generators for: users, service requests, providers, ecosystems, ratings, skill descriptions, assets
- Shrinking enabled to find minimal failing examples

**Property Test Coverage:**
- All 69 correctness properties will be implemented as property-based tests
- Generators will create valid and invalid inputs to test validation
- State machine testing for service request lifecycle
- Geospatial generators for ecosystem testing
- Time-based generators for session expiration and request staleness
- String generators for skill descriptions with various patterns
- Keyword generators for urgency detection testing

**Example Property Test Structure:**
```typescript
// Feature: rural-ai-ecosystem, Property 4: Mobile number uniqueness
test('no two users should have the same mobile number', () => {
  fc.assert(
    fc.property(
      fc.array(userGenerator(), {minLength: 2, maxLength: 100}),
      (users) => {
        const mobileNumbers = users.map(u => u.mobileNumber);
        const uniqueNumbers = new Set(mobileNumbers);
        return mobileNumbers.length === uniqueNumbers.size;
      }
    ),
    {numRuns: 100}
  );
});

// Feature: rural-ai-ecosystem, Property 13: Dual tagging from descriptions
test('skill descriptions should generate both specific tags and broad categories', () => {
  fc.assert(
    fc.property(
      skillDescriptionGenerator(),
      async (description) => {
        const analysis = await skillTaggingService.analyzeSkillDescription(description);
        return analysis.specificTags.length > 0 && analysis.broadCategories.length > 0;
      }
    ),
    {numRuns: 100}
  );
});

// Feature: rural-ai-ecosystem, Property 21: Critical expansion logic
test('zero supply with high urgency should expand to 20km', () => {
  fc.assert(
    fc.property(
      serviceRequestGenerator(),
      async (request) => {
        // Force zero supply and high urgency
        request.localSupplyIndex = 0;
        request.urgencyScore = {score: 85, level: 'high', detectedKeywords: ['emergency']};
        
        const result = await geofencingService.determineDynamicRadius(
          request,
          request.localSupplyIndex,
          request.urgencyScore
        );
        
        return result.radiusKm === 20 && result.includeNeighboring === true;
      }
    ),
    {numRuns: 100}
  );
});
```

### Integration Testing

Integration tests will verify component interactions:

- **Voice LLM Pipeline**: Test audio input → transcription → LLM → skill tagging → TTS → audio output
- **Service Matching Flow**: Test request creation → supply/urgency calculation → dynamic radius → matching → notification
- **Ecosystem Coordination**: Test cross-ecosystem request propagation
- **Rating Flow**: Test service completion → rating prompt → rating submission → average update
- **Authentication Flow**: Test SMS → verification → session creation → API access
- **Offline Sync Flow**: Test offline recording → queue storage → reconnection → upload → SMS confirmation
- **Clustering Flow**: Test multiple requests → cluster detection → group offer → notification

### End-to-End Testing

E2E tests will verify complete user journeys:

- **New User Journey**: Registration → onboarding → first service request
- **Provider Journey**: Registration → skill description → receive notification → accept request → complete service
- **Cross-Ecosystem Journey**: Request with no local matches → expansion → remote provider acceptance
- **Rating Journey**: Service completion → rating submission → provider profile update
- **Offline Journey**: Lose connectivity → record voice request → regain connectivity → auto-upload → SMS confirmation
- **Clustering Journey**: Multiple farmers request fertilizer → cluster detected → group offer → combined order

### Performance Testing

- **Voice Processing Latency**: Target <2 seconds for speech-to-text-to-speech round trip
- **Service Matching Speed**: Target <500ms to identify and notify providers
- **Geospatial Queries**: Target <100ms for ecosystem assignment and neighbor finding
- **Supply Index Calculation**: Target <200ms for counting available providers
- **Skill Tagging Latency**: Target <1 second for LLM-based skill analysis
- **Concurrent Users**: Test system with 1000+ concurrent voice interactions
- **Database Performance**: Test with 100,000+ users and 1,000,000+ service requests
- **Offline Queue Processing**: Test bulk upload of 100+ queued requests

### Accessibility Testing

- **Voice Interface**: Test with various accents, speech patterns, background noise
- **Low Literacy**: Verify all critical functions accessible via voice
- **Network Conditions**: Test with 2G/3G speeds and intermittent connectivity
- **Device Compatibility**: Test on low-end Android devices common in rural areas
- **Language Support**: Test with regional languages and dialects
- **SMS Fallback**: Verify SMS notifications work when push notifications fail

## Implementation Notes

### Technology Stack Recommendations

**Mobile App:**
- React Native for cross-platform development
- React Native Voice for audio capture
- React Native Push Notifications for FCM integration
- AsyncStorage for offline queue
- React Native Maps for location services
- NetInfo for connectivity monitoring

**Backend:**
- Node.js with Express for API server
- MongoDB with geospatial indexes for ecosystem queries
- Redis for session management and caching
- Bull for job queues (notification delivery, cross-ecosystem expansion, cluster detection)
- Socket.io for real-time updates

**Voice LLM:**
- Google Speech-to-Text API or OpenAI Whisper for transcription
- OpenAI GPT-4 or Anthropic Claude for intent recognition, skill tagging, and dialogue
- Google Text-to-Speech API for audio generation
- Support for regional languages (Hindi, Tamil, Telugu, Bengali, etc.)

**Infrastructure:**
- AWS or Google Cloud for hosting
- Firebase Cloud Messaging for push notifications
- Twilio for SMS verification and fallback notifications
- CloudWatch/Datadog for monitoring
- S3 for temporary audio storage (deleted after processing)

### Scalability Considerations

- **Horizontal Scaling**: Stateless API servers behind load balancer
- **Database Sharding**: Shard by ecosystem ID for geographic distribution
- **Caching**: Cache ecosystem boundaries, provider profiles, skill taxonomy
- **Async Processing**: Use job queues for notifications, cross-ecosystem expansion, analytics, cluster detection
- **CDN**: Serve static assets and TTS audio from CDN
- **Read Replicas**: Use MongoDB read replicas for analytics queries

### Security Considerations

- **API Authentication**: JWT tokens with 30-day expiration
- **Rate Limiting**: Limit API calls per user to prevent abuse
- **Input Sanitization**: Validate and sanitize all user inputs
- **SQL Injection Prevention**: Use parameterized queries (MongoDB prevents by default)
- **XSS Prevention**: Sanitize user-generated content before display
- **HTTPS Only**: Enforce TLS 1.3 for all API communication
- **Secrets Management**: Use environment variables and secret managers for API keys
- **Audio Storage**: Delete voice recordings after processing unless user opts in

### Offline Support

- **Request Queue**: Store service requests locally when offline, sync when online
- **Profile Caching**: Cache user profile for offline viewing
- **Notification Queue**: Queue notifications for delivery when connectivity restored
- **Conflict Resolution**: Last-write-wins for profile updates, merge for service requests
- **Sync Indicator**: Show sync status and queued request count
- **Chronological Processing**: Process queued requests in order of creation

### Localization

- **Multi-Language Support**: Support Hindi, English, and regional languages
- **Voice Language Detection**: Auto-detect language from voice input
- **UI Language Selection**: Allow users to choose preferred language
- **Cultural Adaptation**: Adapt service types and terminology to local context
- **Skill Taxonomy Localization**: Maintain skill mappings for each supported language

### Intelligent Features Implementation

**Skill Tagging:**
- Use few-shot prompting with LLM to analyze skill descriptions
- Maintain skill taxonomy database that grows over time
- Use embeddings to find similar skills and suggest tags
- Allow manual correction of tags to improve future tagging

**Dynamic Geofencing:**
- Calculate supply index by counting active providers with matching tags within radius
- Use keyword matching and LLM analysis for urgency detection
- Implement decision tree for radius calculation based on supply, urgency, and service value
- Track ecosystem leakage metrics to tune thresholds

**Demand Clustering:**
- Run periodic job (every 30 minutes) to detect clusters
- Use geospatial queries and time windows to find similar requests
- Calculate potential savings based on shared transportation or bulk discounts
- Expire group offers after 24 hours

**Offline Resilience:**
- Store audio files in app's local storage with metadata
- Monitor connectivity status and trigger sync on reconnection
- Use exponential backoff for failed uploads
- Send SMS confirmation after successful processing to close the loop
