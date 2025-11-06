# Research: Telegram Desktop Licensing & Infrastructure Options

**Research Date:** 2025-11-06
**Repository:** tdesktop (Telegram Desktop official client)

## License Summary

### Main License
**GNU General Public License v3 (GPLv3)** with OpenSSL linking exception

**Copyright:** The Telegram Desktop Authors (2014-2025)

### Permissions
✅ **You CAN:**
- Use the software freely
- Study and read the source code
- Modify the code for your purposes
- Distribute copies (original or modified)
- Create derivative works
- Sell your modifications

### Requirements (Copyleft)
If you distribute modified versions, you MUST:
1. Keep the same GPLv3 license
2. Provide source code to recipients
3. Indicate changes clearly
4. Include copyright notices
5. Include LICENSE file

### Key Points
- No warranty (software provided as-is)
- OpenSSL exception for linking
- Third-party dependencies: Qt (LGPL), OpenSSL (Apache 2.0), WebRTC (BSD), etc.

---

## What's Available in This Repository

### ✅ AVAILABLE: Desktop Client Application
- Complete Telegram Desktop client source code
- UI/UX implementation
- MTProto protocol client implementation
- Message encryption/decryption logic
- Local data storage
- Media handling, calls, etc.

### ❌ NOT AVAILABLE: Server Infrastructure
- Telegram's server infrastructure (proprietary)
- Message routing/delivery systems
- User authentication backend
- Database systems
- Cloud storage infrastructure

**Architecture:**
```
[Telegram Desktop Client] → MTProto Protocol → [Telegram's Servers] (PROPRIETARY)
```

---

## Infrastructure Options Research

### Option 1: Open Source MTProto Servers ⭐ RECOMMENDED

**Available Implementations:**

#### 1. Teamgram Server (Go)
- **Status:** Active (2.1k GitHub stars)
- **License:** Apache 2.0
- **Features:**
  - MTProto 2.0 protocol support
  - API Layer 201
  - Private chats, basic groups
  - Contact management, web access
- **Limitations (Free Version):**
  - No stickers, themes, reactions (enterprise only)
  - No channels, megagroups (enterprise only)
  - No voice/video calls (enterprise only)
  - No bot support (enterprise only)

**Infrastructure Requirements:**
- MySQL 5.7
- Redis
- etcd
- Apache Kafka
- MinIO (object storage)
- FFmpeg (media processing)

**Deployment:** Docker Compose or source (Go 1.21+)

#### 2. MyTelegram Server (C#)
- **Status:** Active
- **License:** MIT/Apache 2.0
- **Platforms:** Windows (win-x64), Linux (linux-x64)
- Private deployment supported

#### 3. mini-telegram (Rust)
- **Status:** Active
- Monolithic, idiomatic implementation
- Compatible with all Telegram clients

**PROS:**
- ✅ Works with existing Telegram Desktop code (no modifications needed)
- ✅ Proven compatibility
- ✅ Full data ownership
- ✅ Moderate complexity

**CONS:**
- ❌ Feature gaps compared to official Telegram
- ❌ Some features locked in enterprise editions
- ❌ Complex infrastructure stack
- ❌ Maintenance burden

---

### Option 2: Rocket.Chat

**What is Rocket.Chat:**
- Fully open source messaging platform (TypeScript/Node.js)
- 1000+ contributors, actively maintained
- Google Summer of Code 2025 participant
- ISO 27001 certified

**Protocol:** REST API + WebSocket (NOT MTProto)

**Features:**
- ✅ Complete messaging platform
- ✅ Team collaboration
- ✅ Voice/video calls
- ✅ Screen sharing
- ✅ File sharing
- ✅ Enterprise integrations
- ✅ GDPR, HIPAA, FINRA compliant
- ✅ Self-hosted (on-premise, air-gapped supported)
- ✅ Docker, Kubernetes deployment

**Deployment:** Much simpler than MTProto servers (no Kafka, etc.)

**The Challenge:**
- ⚠️ **Completely incompatible with Telegram Desktop**
- ⚠️ Would require replacing ALL MTProto code
- ⚠️ 6-12 months of engineering effort

**Why This Is Hard:**
- MTProto deeply integrated: 1,085 references across 225+ files
- Total codebase: 2,212 source files
- 64 dedicated MTProto implementation files
- Would essentially be rebuilding entire client from scratch

---

## Telegram Desktop Architecture Analysis

### MTProto Integration Depth
- **Total Source Files:** 2,212
- **MTProto Directory Files:** 64
- **Files Referencing MTProto:** 225+
- **Total MTProto References:** 1,085

### Key Components Using MTProto:
- Authentication & sessions
- Connection management (TCP, HTTP, resolving)
- Encryption (DH key exchange, RSA, AES)
- Message sending/receiving
- File upload/download
- Voice/video calls
- All API interactions

**Conclusion:** MTProto is not a "swappable module" - it's the backbone of the client.

---

## Recommended Approaches

### Approach 1: Telegram Desktop + Teamgram Server ⭐ RECOMMENDED
**Effort:** ⭐⭐ Moderate (1-2 weeks)

**Architecture:**
```
[Telegram Desktop - UNCHANGED]
         ↓ MTProto
[Teamgram Server - YOUR CONTROL]
         ↓
[Your Infrastructure: MySQL, Redis, Kafka, MinIO]
```

**Steps:**
1. Deploy Teamgram server with Docker Compose
2. Configure infrastructure components
3. Build Telegram Desktop pointing to your server
4. Test and deploy

**Best For:** Full data ownership with minimal code changes

---

### Approach 2: Use Rocket.Chat Native Clients
**Effort:** ⭐ Easy (2-3 days)

**Architecture:**
```
[Rocket.Chat Clients - Desktop/Web/Mobile]
         ↓ REST/WebSocket
[Rocket.Chat Server - YOUR CONTROL]
         ↓
[Your Infrastructure - Simpler than MTProto]
```

**Steps:**
1. Deploy Rocket.Chat server
2. Use existing Rocket.Chat clients
3. Customize as needed

**Best For:** Complete, feature-rich, self-hosted messaging with minimal effort

**Note:** Rocket.Chat already has excellent clients - no need to modify Telegram Desktop!

---

### Approach 3: Hybrid System
**Effort:** ⭐⭐⭐ High (several weeks)

Use both systems for different purposes:
- **Telegram Desktop + Teamgram:** Core messaging
- **Rocket.Chat:** Team collaboration, enterprise features

---

### Approach 4: Rebuild Telegram Desktop for Rocket.Chat ❌ NOT RECOMMENDED
**Effort:** ⭐⭐⭐⭐⭐ Massive (6-12 months full-time team)

**Tasks Required:**
1. Remove/replace MTProto code (225+ files)
2. Implement Rocket.Chat REST API client
3. Implement Rocket.Chat WebSocket client
4. Redesign authentication
5. Rewrite encryption/decryption
6. Rewrite file operations
7. Rewrite calls integration
8. Rewrite all network operations
9. Maintain UI compatibility
10. Comprehensive testing

**Why Not Recommended:**
- Essentially building a new client from scratch
- Rocket.Chat already has great clients
- High risk, long timeline
- Must maintain GPL v3 compliance

---

## Final Recommendation

### If Your Goal Is: "Own the entire messaging infrastructure"
**Use:** Telegram Desktop (this repo) + Teamgram Server
**Timeline:** 1-2 weeks
**Complexity:** Moderate

### If Your Goal Is: "Have a complete, feature-rich, self-hosted platform"
**Use:** Rocket.Chat with its native clients
**Timeline:** 2-3 days
**Complexity:** Low

### If Your Goal Is: "Learn and experiment"
**Use:** Start with Teamgram (easier), consider Rocket.Chat integration later

---

## License Compatibility

All options are GPL v3 compatible:
- **Teamgram:** Apache 2.0 (compatible)
- **MyTelegram:** MIT/Apache 2.0 (compatible)
- **Rocket.Chat:** MIT License (compatible)
- **Telegram Desktop:** GPL v3 (must keep this license for modifications)

---

## Resources

### MTProto Servers
- Teamgram: https://github.com/teamgram/teamgram-server
- MyTelegram: https://github.com/itJunky/mytelegram-server
- MTProto Protocol: https://core.telegram.org/mtproto

### Rocket.Chat
- GitHub: https://github.com/RocketChat/Rocket.Chat
- Documentation: https://docs.rocket.chat/
- API Docs: https://developer.rocket.chat/

### Telegram Desktop
- This Repository: https://github.com/telegramdesktop/tdesktop
- API Credentials: https://core.telegram.org/api/obtaining_api_id
- Build Instructions: See docs/ directory

---

## Conclusion

**Yes, the code is available for use and modification** under GPL v3.

**You have viable options for adding infrastructure:**
1. **Easiest:** Use Rocket.Chat with its native clients
2. **Most Compatible:** Use Teamgram server with this Telegram Desktop client
3. **Not Recommended:** Heavily modify Telegram Desktop for Rocket.Chat

Choose based on your specific requirements, timeline, and resources.
