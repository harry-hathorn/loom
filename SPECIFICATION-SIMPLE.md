# Loom - Simple Specification

**What is this document?**
This is a plain English explanation of what Loom does and how it works, written for non-technical readers. No programming knowledge required.

---

## What is Loom?

**Loom** is an AI-powered coding assistant that helps you write and modify code through conversation. Think of it like ChatGPT, but specifically designed for software development and integrated with your actual code files.

### What can it do?

- **Chat about your code:** Have conversations with an AI about your software projects
- **Edit files:** Ask Loom to make changes to your code files
- **Run commands:** Execute terminal commands safely
- **Search code:** Find information across your entire codebase
- **Track history:** See all your past conversations and changes
- **Work remotely:** Connect to a cloud server to run Loom from anywhere
- **Collaborate:** Share conversations with team members

### How do you use it?

You can interact with Loom in three ways:

1. **Command Line:** Type commands in your terminal (like a power user)
2. **Web Interface:** Use a website in your browser
3. **Text-based Interface:** A simple text-only interface

---

## How Loom Works (The Simple Version)

### The Conversation Loop

Here's what happens when you talk to Loom:

```
You type a message
    ↓
Loom reads your message
    ↓
Loom sends it to an AI (like Claude or GPT)
    ↓
The AI thinks about what to do
    ↓
The AI might ask to use a tool (like reading a file)
    ↓
Loom runs the tool safely
    ↓
Loom tells the AI what happened
    ↓
The AI responds to you
    ↓
You see the answer on your screen
```

### The AI's Toolbox

Loom has a set of tools it can use to help you:

1. **Read File:** Look at the contents of a file
2. **Edit File:** Make changes to a file (with your permission)
3. **List Files:** See what files are in a folder
4. **Bash:** Run terminal commands (like compiling code)
5. **Search:** Find text across all your files
6. **Git:** Track changes with version control

**Important:** Loom can only access files in your project folder. It can't reach files elsewhere on your computer.

### Safety Features

Loom has several safety measures:

- **Workspace Boundary:** Loom can only see files in your project folder
- **Permission Checks:** You must be logged in to use Loom
- **Secure Storage:** Passwords and secrets are encrypted
- **Audit Trail:** All important actions are recorded
- **Timeout Protection:** Commands that take too long are stopped

---

## The Main Features

### 1. Conversations (Threads)

A "thread" is a conversation between you and Loom. Each thread:

- Has a title and description
- Contains all messages you've exchanged
- Shows any code changes made
- Can be searched later
- Can be shared with others (if you want)

**Example:**
```
You: "Help me add a login button"
Loom: [Reads your HTML file]
Loom: "I'll add a login button to your header"
Loom: [Edits the file]
Loom: "Done! I added it to the top right corner"
You: "Thanks! Can you make it blue?"
Loom: [Edits the file again]
Loom: "Changed the color to blue"
```

### 2. Multiple AI Options

Loom can work with different AI services:

- **Claude (Anthropic):** The default, good at coding
- **GPT (OpenAI):** Another popular AI
- **Vertex (Google):** Google's AI service

You can choose which one to use, and Loom will switch between them seamlessly.

### 3. Weavers (Remote Workspaces)

A "Weaver" is like a virtual computer in the cloud:

- You can create one with any programming environment
- It runs in a secure container (isolated from others)
- It automatically deletes itself after a set time
- Useful for testing code without affecting your computer

**Use case:** You want to test some code but don't want to mess up your computer. Create a Weaver, run the test there, and when you're done, the Weaver disappears.

### 4. Feature Flags

"Feature flags" are switches that turn features on or off:

- **Gradual Rollout:** Enable a feature for 10% of users, then 50%, then 100%
- **Kill Switch:** Turn off a feature instantly if something breaks
- **Targeting:** Enable features for specific users or teams
- **Testing:** Show different versions to different groups

**Example:** You're launching a new feature. First, show it to 5% of users to make sure it works. If no problems, gradually increase to everyone.

### 5. Analytics

Loom tracks how people use it:

- **Events:** Records actions (like "clicked button" or "ran search")
- **User Identification:** Knows who is doing what (while protecting privacy)
- **Funnels:** See how users move through steps (like signup flow)
- **Retention:** Understand how often users come back

**Privacy:** All tracking is anonymized. Personal information is protected.

---

## Understanding the System Architecture

### The Building Blocks

Think of Loom as a building with several floors:

```
┌─────────────────────────────────┐
│   Top Floor: Interface Layer    │  ← What you see (CLI, Web)
├─────────────────────────────────┤
│   Second Floor: Routing Layer   │  ← Directs requests
├─────────────────────────────────┤
│   Third Floor: Business Logic   │  ← Does the actual work
├─────────────────────────────────┤
│   Fourth Floor: Data Layer      │  ← Stores information
├─────────────────────────────────┤
│   Basement: Common Tools        │  ← Shared utilities
└─────────────────────────────────┘
```

**Each floor has a specific job:**
1. **Interface Layer:** Shows buttons, text, and menus to users
2. **Routing Layer:** Decides where requests should go
3. **Business Logic:** The brains - makes decisions and solves problems
4. **Data Layer:** Saves and retrieves information
5. **Common Tools:** Shared utilities that all floors use

### How Information Flows

When you do something in Loom (like ask a question), here's what happens:

```
1. You click "Send"
      ↓
2. Your message goes to the Interface Layer
      ↓
3. The Routing Layer checks if you're logged in
      ↓
4. The Business Logic Layer figures out what you want
      ↓
5. The Data Layer saves your message to the database
      ↓
6. The Business Logic Layer asks the AI for help
      ↓
7. The AI responds
      ↓
8. The response flows back up through the layers
      ↓
9. You see the answer on your screen
```

---

## Authentication and Security

### How You Log In

Loom offers several ways to log in:

1. **OAuth (Social Login):**
   - Click "Login with GitHub" (or Google, Okta)
   - You're redirected to that service
   - You approve Loom's access
   - You're logged in!

2. **Magic Link:**
   - Enter your email address
   - Loom sends you an email with a special link
   - Click the link
   - You're logged in!

3. **Device Code (For Servers):**
   - Loom shows you a code
   - You go to a website and enter the code
   - You approve it
   - Your server is now logged in

### Keeping Things Secure

Loom uses several security measures:

**Secrets Protection:**
- Passwords and API keys are encrypted (scrambled so no one can read them)
- They're never shown in logs or error messages
- Even developers can't see them in plain text

**Access Control:**
- Each user has specific permissions (what they can and can't do)
- Organizations have separate data (your company's data is private)
- Administrators can see everything, regular users see less

**Audit Trail:**
- Every important action is recorded
- You can see who did what and when
- Useful for security and troubleshooting

**Safe Tool Execution:**
- Commands that could be dangerous have time limits
- File access is restricted to your project folder
- Nothing can modify system files or other users' data

---

## Storing Information

### What Gets Saved

Loom stores several types of information:

**Threads (Conversations):**
- Every message you send
- Every response from the AI
- Files that were read or modified
- Timestamps (when things happened)

**Users and Accounts:**
- Your email and display name
- Your organization membership
- Your login sessions
- Your preferences

**Weavers (Remote Workspaces):**
- Which weavers exist
- Who created them
- Their status (running, stopped, etc.)

**Analytics:**
- How often features are used
- Performance metrics
- Error rates

### Where It's Stored

Loom uses **SQLite** for storage:

- **What is it?** A database that lives in a single file
- **Why use it?** Simple, reliable, no separate database server needed
- **Where is it?** Usually at `/var/lib/loom/loom.db`

**Think of it like this:**
- A big Excel spreadsheet, but much more powerful
- Can search through millions of records quickly
- Keeps data safe even if the computer crashes
- Can handle many users at once

---

## Scaling and Performance

### Handling More Users

As Loom grows, it needs to handle more users. Here's how:

**Horizontal Scaling (More Servers):**
- Run multiple copies of the Loom server
- Put them behind a "traffic director" (load balancer)
- Each server handles some of the users
- If one server has problems, others keep working

**Vertical Scaling (Bigger Server):**
- Use a more powerful computer
- More memory, faster CPU
- Works up to a point, then you need multiple servers

**Caching (Remembering Answers):**
- Frequently accessed data is kept in memory
- Much faster than reading from disk
- Like having a cheat sheet instead of looking everything up

### Speed Optimizations

Loom does several things to be fast:

**Database Indexes:**
- Like an index in a book
- Lets you find things quickly
- Updated automatically when data changes

**Full-Text Search:**
- Special search engine built in
- Can search millions of conversations in under a second
- Understands relevance (best matches first)

**Streaming Responses:**
- AI responses appear as they're generated
- You don't wait for the entire answer
- Like watching a video download instead of waiting for it to finish

**Background Processing:**
- Heavy tasks happen in the background
- You don't wait for them to finish
- Like dropping off laundry and picking it up later

---

## Multi-Tenancy (Multiple Organizations)

### What is Multi-Tenancy?

Multi-tenancy means Loom can serve many organizations (companies, teams) from the same system, while keeping everything separate:

**Like an Apartment Building:**
- Same building (Loom server)
- Multiple apartments (organizations)
- Each apartment has its own stuff (data, settings)
- Tenants can't access each other's apartments

### How Data is Separated

**Data Isolation:**
- Each organization has its own data
- Users only see data from their organization
- No mixing of data between orgs

**Execution Isolation:**
- Weavers (remote workspaces) run in separate containers
- Each org has its own Kubernetes namespace
- Resource limits prevent one org from affecting others

**Configuration Isolation:**
- Each org can have different feature flags
- Different rate limits per org
- Separate secrets and credentials

---

## Monitoring and Troubleshooting

### Watching the System

Loom has several ways to monitor health:

**Health Checks:**
- A special URL (`/health`) that reports if the system is OK
- Checks database connectivity
- Reports version information

**Metrics:**
- How many requests per second
- How fast responses are
- Error rates
- Cache hit rates

**Logging:**
- Records important events
- Helps diagnose problems
- Each log entry has a timestamp and details

### When Things Go Wrong

**Common Issues:**

1. **Database Locked:**
   - Too many people trying to write at once
   - Solution: Connection pooling (take turns)

2. **Slow AI Response:**
   - AI service is busy
   - Solution: Retry automatically

3. **Out of Memory:**
   - Loom is using too much RAM
   - Solution: Add more memory or optimize

4. **Weaver Won't Start:**
   - Container image doesn't exist
   - Solution: Check image name and permissions

---

## Future Improvements

### Short-Term Plans

**Better Database:**
- Option to use PostgreSQL for larger deployments
- Better scaling to millions of users
- Multiple copies of the database (replicas)

**Distributed Cache:**
- Share cache across multiple servers
- Better performance for large deployments
- Using Redis (a popular cache system)

**Job Queue:**
- Dedicated system for background tasks
- Better reliability for email sending, report generation
- Retry failed tasks automatically

### Long-Term Vision

**Multi-Region:**
- Servers in different geographic locations
- Faster access for users worldwide
- Data stays in the same region (legal requirements)

**Custom Tools:**
- Let users create their own tools
- Plugin system for extensions
- Community-contributed tools

**Better Collaboration:**
- Real-time shared editing
- Comments and annotations
- Team workspaces

---

## Glossary

**AI (Artificial Intelligence):** Computer programs that can think and learn

**API:** A way for computer programs to talk to each other

**Authentication:** Proving who you are (logging in)

**Authorization:** What you're allowed to do (permissions)

**Container:** A package that contains everything needed to run a program

**Database:** A system for storing and retrieving data

**Deployment:** Putting software into production use

**Encryption:** Scrambling data so only authorized people can read it

**Feature Flag:** A switch that turns features on or off

**Git:** A system for tracking changes to code

**LLM (Large Language Model):** AI that understands and generates text

**OAuth:** A way to log in using another service (like GitHub)

**Repository:** A storage location for software code

**SQLite:** A simple database that lives in a single file

**Thread:** A conversation in Loom

**Weaver:** A remote workspace that runs in the cloud

**Workspace:** A folder containing your project files

---

## Frequently Asked Questions

**Q: Is Loom free?**
A: Loom is open-source software, but you need your own API keys for AI services (like Claude or GPT).

**Q: Can Loom access my files?**
A: Loom can only access files in your project workspace. It can't see files elsewhere on your computer.

**Q: Is my data private?**
A: Yes. Your data is stored separately from other users'. Organizations cannot see each other's data.

**Q: What happens if the AI makes a mistake?**
A: Loom shows you exactly what changes the AI wants to make. You can review before applying.

**Q: Can I use Loom offline?**
A: You need an internet connection to talk to the AI, but you can view past conversations offline.

**Q: How much does it cost?**
A: Loom itself is free, but you pay the AI provider (Claude, GPT, etc.) for API usage.

**Q: Can I self-host Loom?**
A: Yes! Loom is open-source. You can run it on your own server.

**Q: What programming languages does Loom support?**
A: Loom can work with any programming language because it uses AI, which understands code in any language.

---

## Summary

Loom is an AI-powered coding assistant that:

- **Chats with you** about your code
- **Edits files** safely with your permission
- **Runs commands** in a controlled environment
- **Tracks history** of all your conversations
- **Connects to AIs** like Claude and GPT
- **Keeps data safe** with encryption and access control
- **Scales up** to handle many users
- **Works everywhere** (CLI, web, or text interface)

**Key Takeaway:** Loom makes software development easier by letting you have conversations with AI about your code, while keeping everything safe, organized, and trackable.

---

**For Technical Readers:**
See `SPECIFICATION.md` for the complete technical specification with code examples, API documentation, and architecture diagrams.

---

**Version:** 1.0
**Last Updated:** 2025-01-16
**Purpose:** Non-technical overview of the Loom system

---

**End of Simple Specification**
