# Graph Report - .  (2026-06-13)

## Corpus Check
- Corpus is ~7,866 words - fits in a single context window. You may not need a graph.

## Summary
- 205 nodes · 327 edges · 17 communities
- Extraction: 98% EXTRACTED · 2% INFERRED · 0% AMBIGUOUS · INFERRED: 8 edges (avg confidence: 0.8)
- Token cost: 0 input · 0 output

## Community Hubs (Navigation)
- [[_COMMUNITY_Device Repository (orchestration)|Device Repository (orchestration)]]
- [[_COMMUNITY_Encrypted Token Storage|Encrypted Token Storage]]
- [[_COMMUNITY_Location Decryption (crypto)|Location Decryption (crypto)]]
- [[_COMMUNITY_Compose UI & Activity|Compose UI & Activity]]
- [[_COMMUNITY_MCS Push Client|MCS Push Client]]
- [[_COMMUNITY_FCMGCM Registration|FCM/GCM Registration]]
- [[_COMMUNITY_Main ViewModel|Main ViewModel]]
- [[_COMMUNITY_HTTP ECE Decryption|HTTP ECE Decryption]]
- [[_COMMUNITY_Nova Find-My-Device API|Nova Find-My-Device API]]
- [[_COMMUNITY_Traccar API Client|Traccar API Client]]
- [[_COMMUNITY_Google OAuth Client|Google OAuth Client]]
- [[_COMMUNITY_Push Notification Service|Push Notification Service]]
- [[_COMMUNITY_SPOT API Client|SPOT API Client]]
- [[_COMMUNITY_Location Worker|Location Worker]]

## God Nodes (most connected - your core abstractions)
1. `DeviceRepository` - 26 edges
2. `TokenStorage` - 20 edges
3. `String` - 16 edges
4. `McsClient` - 12 edges
5. `MainViewModel` - 10 edges
6. `String` - 10 edges
7. `LocationDecryptor` - 10 edges
8. `FcmRegistrationClient` - 8 edges
9. `HttpEceDecryptor` - 8 edges
10. `ByteArray` - 8 edges

## Surprising Connections (you probably didn't know these)
- `DeviceListScreen()` --calls--> `ShimmerListItem()`  [INFERRED]
  app/src/main/java/org/traccar/relay/DeviceListScreen.kt → app/src/main/java/org/traccar/relay/ui/ShimmerListItem.kt
- `DeviceListScreen()` --calls--> `ServerUrlDialog()`  [INFERRED]
  app/src/main/java/org/traccar/relay/DeviceListScreen.kt → app/src/main/java/org/traccar/relay/ui/ServerUrlDialog.kt

## Import Cycles
- None detected.

## Communities (17 total, 0 thin omitted)

### Community 0 - "Device Repository (orchestration)"
Cohesion: 0.12
Nodes (14): DeviceRepository, LocationEntry, LocationResult, Boolean, ByteArray, List, org, String (+6 more)

### Community 1 - "Encrypted Token Storage"
Cohesion: 0.14
Nodes (6): List, String, TokenStorage, Context, MasterKey, SharedPreferences

### Community 2 - "Location Decryption (crypto)"
Cohesion: 0.22
Nodes (9): ByteArray, Int, Long, org, BigInteger, DecryptedLocation, ECPoint, DecryptedLocation (+1 more)

### Community 3 - "Compose UI & Activity"
Cohesion: 0.12
Nodes (10): String, KeySetupScreen(), LoginScreen(), Bundle, ComponentActivity, MainViewModel, DeviceListScreen(), MainActivity (+2 more)

### Community 4 - "MCS Push Client"
Cohesion: 0.24
Nodes (8): Any, ByteArray, Int, String, DataInputStream, DataOutputStream, javax, McsClient

### Community 5 - "FCM/GCM Registration"
Cohesion: 0.28
Nodes (7): AndroidCheckinRequest, AndroidCheckinResponse, Pair, String, EcKeys, FcmCredentials, FcmRegistrationClient

### Community 6 - "Main ViewModel"
Cohesion: 0.24
Nodes (7): AndroidViewModel, ByteArray, String, MainViewModel, UiState, WorkScheduleInfo, StateFlow

### Community 7 - "HTTP ECE Decryption"
Cohesion: 0.29
Nodes (6): ByteArray, Int, String, java, PrivateKey, HttpEceDecryptor

### Community 8 - "Nova Find-My-Device API"
Cohesion: 0.26
Nodes (7): Device, NovaApiClient, Boolean, ByteArray, List, Pair, String

### Community 9 - "Traccar API Client"
Cohesion: 0.25
Nodes (6): TraccarApiClient, Int, Long, String, Double, Float

### Community 10 - "Google OAuth Client"
Cohesion: 0.46
Nodes (4): String, GoogleAuthClient, FormBody, Map

### Community 11 - "Push Notification Service"
Cohesion: 0.29
Nodes (4): String, FirebaseMessagingService, PushNotificationService, RemoteMessage

### Community 12 - "SPOT API Client"
Cohesion: 0.40
Nodes (3): SpotApiClient, ByteArray, String

### Community 13 - "Location Worker"
Cohesion: 0.40
Nodes (3): LocationWorker, Result, Worker

## Knowledge Gaps
- **33 isolated node(s):** `MainViewModel`, `Bundle`, `StateFlow`, `Boolean`, `List` (+28 more)
  These have ≤1 connection - possible missing edges or undocumented components.

## Suggested Questions
_Questions this graph is uniquely positioned to answer:_

- **Why does `DeviceRepository` connect `Device Repository (orchestration)` to `Push Notification Service`, `Location Worker`?**
  _High betweenness centrality (0.038) - this node is a cross-community bridge._
- **Are the 3 inferred relationships involving `DeviceRepository` (e.g. with `.doWork()` and `.onMessageReceived()`) actually correct?**
  _`DeviceRepository` has 3 INFERRED edges - model-reasoned connections that need verification._
- **What connects `MainViewModel`, `Bundle`, `StateFlow` to the rest of the system?**
  _33 weakly-connected nodes found - possible documentation gaps or missing edges._
- **Should `Device Repository (orchestration)` be split into smaller, more focused modules?**
  _Cohesion score 0.11764705882352941 - nodes in this community are weakly interconnected._
- **Should `Encrypted Token Storage` be split into smaller, more focused modules?**
  _Cohesion score 0.13666666666666666 - nodes in this community are weakly interconnected._
- **Should `Compose UI & Activity` be split into smaller, more focused modules?**
  _Cohesion score 0.11764705882352941 - nodes in this community are weakly interconnected._