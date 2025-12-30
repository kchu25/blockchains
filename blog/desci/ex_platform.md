@def title = "Building a Biology Experiment Platform: Learning from DeSci's UX Problem"
@def published = "30 December 2025"
@def tags = ["desci"]

# Building a Biology Experiment Platform: Learning from DeSci's UX Problem

## The State of DeSci: Why It's Not Working

**BIO Protocol just launched** (Jan 3, 2025) with Binance backing. VitaDAO has funded 10M+ in research since 2021. Market caps look impressive in crypto terms.

**But here's the problem:** You've never heard of them. Neither have 99% of scientists.

### The Brutal Truth

VitaDAO has 6+ bioDAOs, \$20M+ in treasuries, ecosystem market cap over \$200M. Sounds big? The NIA's 2023 budget for aging biology alone was ~\$400M—40x VitaDAO's *entire history*.

**Why scientists don't use it:**


1. Need to: buy crypto → install MetaMask → understand gas fees → navigate Discord → stake tokens → comprehend vesting schedules
2. Scientists unfamiliar with Web3 find blockchain difficult to understand and use
3. Traditional academia views DeSci with skepticism, needing proof beyond hype through projects with real impact
4. No drugs in Phase 2 trials yet to prove the model works sustainably

**The UX gap is enormous.** An NIH grant application is painful, but doesn't require learning about L2 rollups.

## What You Should Actually Build

Forget "another blockchain." The tech exists. What's missing is a bridge between crypto and scientists.

### Direction 1: The Abstraction Layer (Recommended)

**Build: "Stripe for DeSci"**

The platform that makes blockchain invisible to scientists while keeping its benefits:

**Core principle:** Scientists use email. Blockchain runs in the background.

```
Scientist experience:          Behind the scenes:
─────────────────────         ─────────────────
1. Sign in with email   →     Account abstraction wallet created
2. Submit experiment    →     Protocol uploaded to Arweave  
3. Get USD funding      →     Smart contract distributes tokens
4. View results         →     Data pulled from IPFS
5. Receive payment      →     Auto-converted to bank account
```

**Key features:**
- **Fiat on/off ramps**: Credit card payments, ACH transfers, bank withdrawals
- **Email-based identity**: No MetaMask, no seed phrases visible to users
- **Dollar-denominated**: Show "\$5,000 reward pool" not "2.3 ETH"
- **Social login**: Google/ORCID authentication
- **Automatic compliance**: Tax forms, W-9s, institutional approvals handled

**Tech stack:**
- **Account Abstraction**: Biconomy or ZeroDev for gasless transactions
- **Wallet abstraction**: Magic.link or Web3Auth (email → wallet)
- **Fiat rails**: Stripe for onboarding, Coinbase Commerce or Circle for crypto conversion
- **Backend**: Next.js with Privy for authentication
- **Smart contracts**: Simple, audited contracts on Arbitrum
- **Storage**: Arweave for protocols (abstracted via API)

**The pitch to scientists:** "Submit experiments, get paid in dollars. That's it."

### Direction 2: The Hybrid Model

**Build: Traditional platform + blockchain backbone**

Start as a normal web app, gradually reveal blockchain benefits:

**Phase 1 (Months 1-6):** Pure Web2
- Scientists use regular forms and dashboards
- Store everything in PostgreSQL
- Process payments via Stripe
- Build user base, prove value

**Phase 2 (Months 6-12):** Silent blockchain integration
- Mirror critical data to blockchain
- Issue "impact certificates" as NFTs (don't call them NFTs)
- Enable "transparent funding" view (optional)

**Phase 3 (Year 2+):** Optional crypto features
- Let early adopters earn governance tokens
- Enable peer-to-peer funding in crypto
- Launch marketplace for experiment data

**Why this works:**
- No scary crypto UX to start
- Blockchain provides verifiability without friction
- Scientists can ignore crypto entirely if they want
- Appeals to traditional institutions

### Direction 3: Vertical Integration

**Build: Experiment-specific platform, not general DeSci**

Pick ONE biology domain and go deep:

**Example: Longevity Experiments**
- Target: At-home aging biomarker experiments
- Users: Biohackers, citizen scientists, longevity enthusiasts
- Advantage: This audience already understands crypto

**Example: Replication Crisis Platform**
- Target: Psychology/social science replication studies
- Users: Academic labs needing quick replication funding
- Advantage: Clear problem, measurable impact

**Example: Microbiome Studies**
- Target: Gut microbiome interventions with n-of-1 trials
- Users: Health-conscious consumers
- Advantage: Easy to participate, clear metrics

**Why vertical works:**
- Solve one problem really well
- Community is tight-knit, easier to reach
- Build reputation before expanding
- Can customize UX for specific needs

## The Reward Mechanism (Abstracted)

Keep the sophisticated tokenomics, hide the complexity.

### What Scientists See

"Complete this experiment and earn \$500-\$2,000 based on result quality"

**Quality determined by:**
- Completion rate: Did you finish all steps?
- Data completeness: Did you provide all measurements?
- Peer review score: How do reviewers rate your methodology?
- Result consensus: How close are your results to others?

### What Actually Happens (Under the Hood)

$$R_i = B \cdot \left(\frac{Q_i}{Q_{max}}\right) \cdot \left(1 + \frac{C_i}{10}\right) \cdot M$$

Where:
- $B$ = Base reward pool
- $Q_i$ = Quality score (peer-reviewed, 0-10)
- $C_i$ = Consensus score (statistical clustering)
- $M$ = Experiment difficulty multiplier

**Implementation:**
```javascript
// Behind the scenes
async function calculateReward(participantId, experimentId) {
  const quality = await getPeerReviewScore(participantId);
  const consensus = await calculateConsensus(experimentId);
  const multiplier = experiments[experimentId].difficulty;
  
  const reward = BASE_REWARD * (quality/10) * (1 + consensus/10) * multiplier;
  
  // Convert to USD and pay via Stripe
  await payParticipant(participantId, reward, 'USD');
}
```

**Anti-gaming measures:**
- Require institutional email or ORCID verification
- Stake small deposit (returned on completion)
- Ban accounts with repeated low-quality submissions
- Gradual reward increases for consistent participants

## Technical Architecture

### The Minimalist Stack

**Frontend (what scientists interact with):**
- Next.js or SvelteKit
- TailwindCSS for clean, academic-style UI
- No wallet connection UI needed
- Hosted on Vercel/Netlify

**Authentication & Wallets:**
- Privy or Dynamic for email-based auth
- Account abstraction (Biconomy) for gasless transactions
- Each user gets a non-custodial wallet they never see

**Backend:**
- Next.js API routes or Express
- PostgreSQL for user data, metadata
- Redis for caching
- Hosted on Railway, Render, or Fly.io

**Blockchain (invisible to users):**
- Arbitrum or Base for low costs
- Simple smart contracts for escrow and rewards
- The Graph for indexing (optional)
- Events logged to traditional database for speed

**Storage:**
- Arweave via Bundlr for experiment protocols
- IPFS (via Pinata) for result data
- S3 for user uploads initially (mirror to IPFS later)

**Payments:**
- Stripe for collecting fiat
- Circle USDC for stable crypto payments
- Coinbase Commerce for crypto-savvy users
- Auto-convert between fiat and crypto

### Smart Contract Architecture (Simplified)

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.20;

contract ExperimentPlatform {
    struct Experiment {
        address creator;
        string protocolHash;      // Arweave reference
        uint256 rewardPool;
        uint256 minParticipants;
        uint256 deadline;
        bool active;
    }
    
    struct Participant {
        address wallet;
        string resultHash;        // IPFS reference
        uint256 qualityScore;     // Set by peer review
        bool rewarded;
    }
    
    mapping(uint256 => Experiment) public experiments;
    mapping(uint256 => Participant[]) public participants;
    
    // Minimal, auditable functions only
    function createExperiment(string memory _hash, uint256 _minPart) external payable;
    function submitResult(uint256 _expId, string memory _resultHash) external;
    function distributeRewards(uint256 _expId, uint256[] memory _scores) external;
}
```

**Keep contracts simple.** Complex logic lives in backend where it can be updated.

## Implementation Roadmap

### Phase 1: MVP (4-6 weeks)

**Week 1-2: Foundation**
- Set up Next.js app with Privy authentication
- Basic experiment creation form (Web2 style)
- Store data in PostgreSQL
- Deploy to Vercel

**Week 3-4: Core Features**
- Participant registration for experiments
- Simple result submission (text/file upload)
- Admin dashboard for reviewing submissions
- Stripe integration for basic payments

**Week 5-6: Soft Launch**
- Test with 1-2 pilot experiments
- Invite 10-20 scientists to try it
- Collect feedback, iterate UX
- No blockchain yet—prove the concept first

### Phase 2: Blockchain Integration (3-4 weeks)

**Add blockchain invisibly:**
- Deploy simple smart contracts to Arbitrum testnet
- Integrate Privy wallet creation (users don't notice)
- Mirror experiment data to Arweave
- Add "blockchain verified" badges

**Test mixed payments:**
- Let researchers fund in crypto OR fiat
- Pay participants in their preferred currency
- Ensure conversions work smoothly

### Phase 3: Growth Features (Ongoing)

**Month 3-6:**
- Peer review system with reputation scores
- Advanced consensus algorithms
- Automatic quality scoring
- Referral program

**Month 6-12:**
- Governance features (researchers vote on platform changes)
- Data marketplace (sell aggregate results)
- API for institutional integration
- Mobile app for easier participation

## Cost-Effective Launch

**Minimal budget:**
- Domain + hosting: \$50/year
- Stripe fees: 2.9% + \$0.30 per transaction
- Smart contract deployment: ~\$50-200
- No audit needed initially (use OpenZeppelin templates)
- **Total startup cost: <\$500**

**With funding (\$5k-10k):**
- Smart contract audit: \$3k-5k (QuillAudits, Hacken)
- Initial marketing: \$2k
- Better infrastructure: \$1k/year
- First experiments funded by you: \$2k

**Avoid:**
- Building your own blockchain
- Complex tokenomics initially
- Expensive consultants
- Over-engineering

## What Makes This Different from Existing DeSci

| Feature | Existing DeSci | Your Platform |
|---------|----------------|---------------|
| Target user | Crypto natives | Traditional scientists |
| Entry barrier | Install MetaMask | Email login |
| Payment | VITA, BIO tokens | USD (crypto optional) |
| Governance | Token voting | Simple voting, then DAO |
| Documentation | Discord, Twitter | Academic style, clear |
| Success metric | Token price | Papers published |

## Traction Strategy

### Don't compete with VitaDAO or BIO

**Instead:** Position as the normie on-ramp to DeSci

**Partnership opportunities:**
- "Powered by BIO Protocol infrastructure"
- Use VitaDAO's IP-NFT framework behind the scenes
- Refer advanced users to existing platforms
- Become the UX layer on top of their tech

### Target the right users first

**Bad first users:**
- Crypto day traders
- Grant-chasing academics
- Anti-establishment scientists

**Good first users:**
- Citizen scientists (already experimental)
- Undergraduate research programs (need projects)
- Failed grant applicants (looking for alternatives)
- Replication researchers (underserved niche)

### Content strategy

- Publish one high-quality experiment protocol per week
- Write clear methodologies, not crypto jargon
- Get published in open science journals
- Present at academic conferences, not crypto conferences

## Key Success Metrics

**Ignore vanity metrics:**
- Token price
- Twitter followers
- "Total value locked"

**Focus on real impact:**
- Number of completed experiments
- Peer-reviewed papers citing platform data
- Scientists who return for second experiment
- Institutions that adopt platform
- Replication rate of experiments

**Milestone targets:**

- **Month 3:** 50 scientists signed up, 5 experiments completed
- **Month 6:** 200 scientists, 25 experiments, 1 publication
- **Month 12:** 1000 scientists, 100 experiments, partnership with 1 university
- **Year 2:** Self-sustaining, breakthrough finding published

## The Hard Truth About DeSci

The ultimate verdict will be whether DeSci DAOs can develop drugs that slow, stall, or reverse aging and sustain themselves through profitable research.

**Your platform doesn't need to cure aging.** But it needs to:

1. Make life easier for scientists (not harder)
2. Produce at least one replicable, cited finding
3. Be more convenient than current alternatives
4. Prove blockchain adds value (transparency, IP ownership)

If you can't clearly explain why blockchain is necessary for your platform, it probably isn't.

## Recommended Starting Point

**Option A (safest):** Build as Web2 platform first
- Prove scientists want this
- Add blockchain features later
- Lower technical risk
- Faster to market

**Option B (balanced):** Hybrid from day 1
- Use account abstraction for invisible crypto
- Store protocols on Arweave
- Traditional payments and UX
- "Blockchain verified" as feature, not focus

**Option C (bold):** Partner with existing DeSci
- Fork VitaDAO's contracts
- Build the UX layer they're missing
- Leverage their funding and network
- Focus 100% on scientist experience

**My recommendation:** Start with Option A, transition to Option B after proof of concept.

## Next Steps

**This week:**

1. Pick one experiment type (longevity, replication, microbiome)
2. Interview 10 scientists in that niche
3. Ask: "Would you use a platform that does X?"
4. Sketch the simplest possible MVP

**This month:**

1. Build basic web app (no blockchain)
2. Run 1 pilot experiment with friends/colleagues
3. Document everything that breaks
4. Decide if blockchain actually helps

**This quarter:**

1. If pilot works: add account abstraction
2. Launch with 3-5 real experiments
3. Get first publication or citation
4. Reassess: scale up or pivot

**Don't build in a vacuum.** The reason DeSci hasn't taken off isn't the tech—it's that builders aren't talking to scientists.

---

## Should You Tell People You're Building This?

> **Short answer: Yes, but strategically.**
>
> ### The "Stealth Mode" Trap
>
> >> **What most builders think:** "I need to keep this secret until it's perfect, or someone will steal my idea."
> >>
> >> **Reality:** Ideas are worthless. Execution is everything. Here's what actually happens:
> >> - You spend 6 months building in secret
> >> - Launch to crickets because nobody knows it exists
> >> - Realize scientists want something completely different
> >> - Waste months pivoting
> >>
> >> Meanwhile, someone who talked to 50 scientists first builds exactly what they need and wins.
>
> ### Who You Should Tell (And Why)
>
> >> **1. Potential users (scientists) - Tell ASAP**
> >> - They won't steal your idea—they're busy doing science
> >> - They'll tell you if it solves a real problem or not
> >> - Early feedback = avoid building wrong thing
> >> - "I'm building a platform to make experiment collaboration easier. Can I ask you 3 questions?"
> >>
> >> **2. Technical advisors - Tell selectively**
> >> - Other developers can help with architecture decisions
> >> - Web3 people have seen what works and what doesn't
> >> - But don't pitch the "business idea"—ask specific technical questions
> >>
> >> **3. Potential competitors - Don't worry about them**
> >> - BIO Protocol, VitaDAO already exist with millions in funding
> >> - If they wanted to solve the UX problem, they would have
> >> - Your advantage is speed and focus, not secrecy
> >> - By the time they notice you, you'll have users
> >>
> >> **4. Investors/partners - Only after proof of concept**
> >> - Wait until you have 10+ scientists using MVP
> >> - Then leverage traction for funding/partnerships
> >> - "We have 50 completed experiments" > "We have an idea"
>
> ### What to Share vs What to Keep Close
>
> >> **Share freely:**
> >> - The problem you're solving ("experiment collaboration is broken")
> >> - High-level approach ("making DeSci accessible to normal scientists")
> >> - Technical challenges ("how to do protocol inheritance?")
> >> - Early results ("10 scientists tested it, here's feedback")
> >>
> >> **Keep private:**
> >> - Specific algorithm for royalty decay (until it's proven to work)
> >> - Your user acquisition strategy (who you're targeting first)
> >> - Partnerships you're negotiating
> >> - Financial projections / fundraising plans
> >>
> >> **The rule:** Share problems and learning. Keep tactics and relationships private.
>
> ### The Strategic Sharing Approach
>
> >> **Phase 1 (Weeks 1-4): Research mode**
> >> - Interview 20-30 scientists
> >> - Post in r/labrats: "What's your biggest frustration with experiment collaboration?"
> >> - Join DeSci Discord: "I'm exploring solutions to X, what have you tried?"
> >> - **You're not "building" yet, just researching**
> >>
> >> **Phase 2 (Weeks 5-8): Build in semi-public**
> >> - Tweet progress: "Building a tool for experiment protocols. Day 12."
> >> - Show wireframes: "Does this UX make sense for scientists?"
> >> - Get feedback constantly
> >> - **You're building, but it's not launched**
> >>
> >> **Phase 3 (Weeks 9-12): Private beta**
> >> - Invite 10 scientists you interviewed
> >> - Run 2-3 pilot experiments
> >> - Document everything that breaks
> >> - **Still not publicly launched**
> >>
> >> **Phase 4 (Month 4+): Public launch**
> >> - Write launch post: "We ran 10 experiments with 50 scientists. Here's what we learned."
> >> - By now you have proof, testimonials, data
> >> - Much harder for anyone to copy what you've validated
>
> ### The "Someone Will Steal It" Reality Check
>
> >> **Who would steal it?**
> >> - **Big tech companies?** They don't care about niche science platforms
> >> - **Existing DeSci projects?** They're ideologically committed to their approach
> >> - **Other indie builders?** They have their own ideas and lack your domain knowledge
> >> - **Scientists?** They want to use tools, not build them
> >>
> >> **What actually kills startups:**
> >> - Building something nobody wants (70% of failures)
> >> - Running out of money (29% of failures)
> >> - Getting copied (< 1% of failures)
> >>
> >> Even if someone does copy you, they'd need to:
> >>
> >> 1. Understand the problem as deeply
> >> 2. Build as fast as you
> >> 3. Talk to as many scientists
> >> 4. Launch with better UX
> >> 5. Out-market you
> >>
> >> That's incredibly hard. And if someone can do all that, they deserve to win.
>
> ### Exceptions: When NOT to Share
>
> >> **1. You have truly novel tech (rare)**
> >> - Genuinely new consensus algorithm
> >> - Proprietary AI model for protocol analysis
> >> - Patent-worthy innovation
> >> → File provisional patent first, then share
> >>
> >> **2. You're in a cutthroat market**
> >> - Multiple well-funded competitors
> >> - Race to market is critical
> >> - First-mover advantage is massive
> >> → Build fast, launch, then talk
> >>
> >> **3. You have insider access**
> >> - Exclusive data or relationships
> >> - Partnership that would be spoiled by publicity
> >> - Regulatory advantage
> >> → Keep the specific advantage secret
> >>
> >> **But even then:** You can still talk about the *problem* without revealing your *solution*.
>
> ### The Build-in-Public Strategy (Recommended)
>
> >> **Why this works:**
> >> - Builds audience before launch
> >> - Gets feedback early when changes are cheap
> >> - Creates accountability (people are watching)
> >> - Attracts collaborators and early users
> >> - Validates demand before investing heavily
> >>
> >> **How to do it:**
> >> - Tweet/blog weekly progress
> >> - Share mockups and get feedback
> >> - Ask questions in relevant communities
> >> - Be honest about challenges
> >> - Celebrate small wins
> >>
> >> **What it looks like:**
> >> - Week 1: "Talking to scientists about experiment collaboration problems"
> >> - Week 3: "Interviewed 15 scientists. Top pain point: citation system is broken"
> >> - Week 6: "Building protocol inheritance feature. Sketched this flow—thoughts?"
> >> - Week 10: "First experiment completed on the platform! 8 participants, clean data."
> >> - Week 14: "Launching beta. If you're a scientist interested in X, DM me."
> >>
> >> By launch, you have an audience of people who've been following your journey.
>
> ### Bottom Line
>
> >> **Talk to scientists immediately and constantly.** They're your users. Without them, you're building nothing.
> >>
> >> **Share the journey publicly.** Builds credibility, gets feedback, attracts early adopters.
> >>
> >> **Keep specific tactics private.** Don't announce your growth hacks or unfair advantages.
> >>
> >> **The best defense against copycats is speed and relationships.** By the time someone copies your idea, you should already have 100 users who love you.
> >>
> >> Remember: VitaDAO and BIO Protocol have years of head start and millions in funding. They're not failing because someone stole their idea—they're struggling because they didn't solve the UX problem. Your edge is understanding scientists and moving fast, not secrecy.

## Resources

**Learn from:**
- ResearchHub (backed by Brian Armstrong, has scientist adoption)
- protocols.io (traditional experiment protocols, good UX)
- Experiment.com (crowdfunding for science)

**Technical guides:**
- Privy docs: privy.io/docs
- Account abstraction: docs.biconomy.io
- Arweave: docs.arweave.org

**Communities:**
- DeSci Discord servers (for what NOT to do)
- r/labrats (for what scientists actually care about)
- Open science communities

The opportunity is huge. But success means making scientists' lives easier, not teaching them about Web3.

---

## What's Stripe?

> **Stripe is the behind-the-scenes payment system that powers millions of websites.** When you buy something online and enter your credit card, there's a good chance Stripe is handling that transaction.
> 
> Think of it like the plumbing in a building—you don't see it, but it makes everything work. Without Stripe, every company would need to build their own secure payment system, deal with banks directly, handle fraud detection, manage refunds, etc. Stripe does all that boring infrastructure work so developers can just add a few lines of code and boom—they can accept payments.
> 
> **Why I keep mentioning it:** When I say "build the Stripe for DeSci," I mean create something that makes blockchain as easy to use as Stripe makes payments. Scientists shouldn't need to understand crypto any more than online shoppers need to understand how credit card processing works. They just want it to work invisibly.
> 
> Stripe's genius was abstracting away complexity. That's exactly what DeSci needs—hide the blockchain, show the benefits.

---

## The Experiment Referral Economy

> **This is actually brilliant and solves a real problem.**
> 
> Right now, science works like this: publish paper → wait for citations → hope for tenure. It's slow (years), gameable (citation cartels), and gatekept (Nature/Science or bust).
> 
> **Your idea: experiments that reference other experiments, creating a living knowledge graph.**
> 
> Here's how it could work:
> - Experiment A tests "Does cold exposure increase brown fat?"
> - Scientist B sees interesting results, creates Experiment B: "Does brown fat correlate with metabolic health?"
> - Experiment B "forks" or "extends" Experiment A
> - Original experimenter gets ongoing credit + small royalty when their work is built upon
> 
> **Why this is better than citations:**
> - **Immediate feedback**: You know your experiment mattered within weeks, not years
> - **Financial incentive**: Get paid when others build on your work (like open source sponsorship)
> - **No gatekeepers**: Don't need journal approval to validate your contribution
> - **Real replication**: Someone literally running your experiment again is worth 100x more than a citation
> 
> **The mechanism:**
> - Each experiment has a DOI-like permanent ID
> - When creating new experiments, mark which ones you're "extending" or "replicating"
> - Original creators get 5-10% of the reward pool automatically
> - Public graph shows experiment lineages (like GitHub's fork network)
> 
> **This creates a compound effect**: Good experiments become "platforms" that spawn 10+ derivative studies. Bad experiments get ignored. No journal editor needed to decide what's important—the scientific community votes with their participation.
> 
> **The peer review question:** You still need *some* quality filter, but it can be post-hoc:
> - Anyone can submit experiments (no gatekeeping)
> - Community reviews completed experiments for quality
> - High-quality experiments get featured, more referrals, more rewards
> - Low-quality experiments fade into obscurity
> 
> This flips the model: instead of asking permission before publishing (peer review), you prove value after publishing (community validation). Wikipedia editing model, not academic journal model.
> 
> **Bonus**: Each experiment's "impact score" becomes its referral count + completion rate + quality ratings. Way more meaningful than H-index.

---

## The Citation Problem: What If Scientists Don't Link Back?

> **You're hitting on the core weakness: voluntary attribution doesn't work. Scientists forget, omit, or deliberately don't cite things all the time.**
> 
> **But here's where blockchain actually helps—you can make attribution automatic and unavoidable:**
> 
> ### Option 1: Data Dependency Detection
> 
> >> When Scientist B creates a new experiment, the platform analyzes:
> >> - Does it use the same protocol steps? 
> >> - Same reagents or materials list?
> >> - Same population or cohort definition?
> >> - Same measurement techniques?
> >> 
> >> If overlap exceeds ~60%, the system flags: "This appears to build on Experiment A. Link it?" Not optional—you must either link it or explain why you're not.
> >> 
> >> Think: GitHub detecting "this looks like a fork" or plagiarism detectors finding text similarity.
> 
> ### Option 2: Protocol Inheritance (The Smart Move)
> 
> >> Instead of writing protocols from scratch, scientists "clone" existing protocols and modify them.
> >> 
> >> **Example:**
> >> - Experiment A has protocol stored on Arweave with hash `ABC123`
> >> - Scientist B clicks "Use this protocol as template"
> >> - Makes modifications (changes dosage, adds a step)
> >> - New protocol auto-links to parent: hash `ABC123` → `DEF456`
> >> - Smart contract enforces: Parent gets 5% of any rewards automatically
> >> 
> >> You literally can't use someone's protocol without the link being created. It's baked into the data structure.
> 
> ### Option 3: Knowledge Graph Analysis
> 
> >> Run semantic analysis on all experiment descriptions using AI:
> >> - "This experiment mentions 'brown adipose tissue' and 'cold exposure'"
> >> - Search for all prior experiments with those keywords
> >> - Surface them to Scientist B: "Did you know these 3 experiments are related?"
> >> - Incentivize linking: "Link 2+ related experiments and get bonus rewards"
> >> 
> >> The platform becomes smarter than individual scientists at seeing connections.
> 
> ### Option 4: Retroactive Attribution (The Nuclear Option)
> 
> >> If Scientist B doesn't link to Experiment A but should have:
> >> - Community can propose the link via governance
> >> - If vote passes, link is added retroactively
> >> - Original creator gets back-pay for all derivative work
> >> - Scientist B doesn't lose anything, but the record is corrected
> >> 
> >> This is only possible because blockchain creates an immutable history. You can't erase that Experiment B borrowed from Experiment A.
> 
> ### The Key Insight: Make Attribution the Path of Least Resistance
> 
> >> **Bad design:** Scientist must manually search and link related work (they won't)
> >> 
> >> **Good design:** Platform auto-detects relationships, scientist just clicks "yes"
> >> 
> >> **Best design:** Using someone's protocol *requires* linking by default. The only way to avoid it is to write everything from scratch—which is more work than just accepting the attribution.
> >> 
> >> Think about how GitHub handles forks: you can't fork a repo without it showing the parent. It's not optional. That's the model.
> 
> ### Why This Actually Works
> 
> >> Traditional citations fail because:
> >>
> >> 1. They're added at the end (easy to forget)
> >> 2. No one checks if you cited everything
> >> 3. No penalty for omission
> >> 
> >> Your system succeeds because:
> >>
> >> 1. Attribution happens at creation time (can't forget)
> >> 2. Platform detects missing links automatically
> >> 3. Financial incentive to link (you get better visibility + community trust)
> >> 4. Protocols are reusable objects, not text descriptions
> >> 
> >> You're not relying on goodwill—you're changing the incentive structure so that attribution is the easier, more profitable path.

---

## The Power Law Problem: Preventing Protocol Monopolies

> **You're absolutely right—this creates a "rich get richer" problem. The first good protocol for "measure blood glucose" gets 1000 forks, makes 50k in royalties, and dominates forever. New scientists see this and think "why bother?"**
>
> **This kills innovation. Here's how to fix it:**
>
> ### Solution 1: Diminishing Returns on Royalties
>
> >> **The problem with flat 5% royalties:** Protocol A spawns 100 experiments, original creator makes massive passive income, has no incentive to improve.
> >>
> >> **Better model—decay over time or usage:**
> >> ```
> >> First 10 uses:  5% royalty
> >> Uses 11-50:    3% royalty  
> >> Uses 51-100:   1% royalty
> >> Uses 100+:     0.5% royalty
> >> ```
> >>
> >> Or time-based decay:
> >> ```
> >> Year 1: 5% royalty
> >> Year 2: 3% royalty
> >> Year 3: 1% royalty
> >> Year 4+: Protocol becomes "public domain"
> >> ```
> >>
> >> **Why this works:** Early adopters get rewarded fairly, but protocols don't become permanent rent-seeking machines. After saturation, the incentive shifts to creating *new* protocols.
>
> ### Solution 2: Competitive Forking with Bounties
>
> >> **The twist:** Anyone can create an "alternative protocol" for the same question and compete directly.
> >>
> >> **Example:**
> >> - Protocol A: "Measure blood glucose with finger prick" (1000 uses)
> >> - Scientist C creates Protocol B: "Measure blood glucose with continuous monitor" (better, less painful)
> >> - Platform adds "Protocol Battle" feature: Both compete for the same use case
> >> - Scientists choosing a protocol see: "Protocol A (proven, 1000 uses) vs Protocol B (new, potentially better)"
> >>
> >> **Incentive boost for challengers:**
> >> - New protocols get 2x visibility for first 90 days
> >> - Bounties for "improve existing protocols" 
> >> - If your alternative gets more uses than the original, you get a "Protocol Champion" badge
> >>
> >> This way, dominant protocols face constant competitive pressure.
>
> ### Solution 3: Multi-Attribution with Capped Individual Shares
>
> >> **The problem:** One protocol takes 5% of every reward pool forever.
> >>
> >> **Better model:** Multiple protocols can be linked, but total attribution capped at 10-15%
> >>
> >> **Example:**
> >> - Experiment uses Protocol A (5%) + Protocol B (4%) + Protocol C (3%)
> >> - Total: 12% goes to protocol creators, 88% to participants
> >> - No single protocol can take more than 5% ever
> >>
> >> **Why this helps:** Creates a "protocol ecosystem" rather than winner-takes-all. Scientists are incentivized to remix and combine protocols, not just copy the most popular one.
>
> ### Solution 4: "Protocol Innovation Bonus"
>
> >> **Reward originality explicitly:**
> >> - AI/semantic analysis detects how different a protocol is from existing ones
> >> - "Similarity score": 0% (completely novel) to 100% (identical)
> >> - Protocols with <30% similarity get "Innovation Bonus": 2x rewards for first 50 uses
> >>
> >> **Example:**
> >> - Protocol A: Standard blood draw procedure (95% similar to existing)
> >> - Protocol B: Novel saliva-based glucose test (15% similar)
> >> - Protocol B gets innovation bonus even with fewer uses
> >>
> >> This directly incentivizes people to create genuinely new approaches, not just tweak existing ones.
>
> ### Solution 5: Category-Specific Leaderboards
>
> >> **The problem:** A few mega-protocols dominate the global leaderboard.
> >>
> >> **Better approach:** Organize protocols by category/specialty:
> >> - Cardiovascular protocols
> >> - Metabolic protocols  
> >> - Microbiome protocols
> >> - Neuroscience protocols
> >>
> >> **Each category has its own leaderboard and "top protocol" status.**
> >>
> >> New scientists don't compete with the global #1 protocol—they compete within their niche. Much more achievable to become "#1 protocol for gut microbiome sampling" than "#1 protocol on the entire platform."
>
> ### Solution 6: Reputation ≠ Royalties
>
> >> **Separate the incentives:**
> >> - **Royalties:** Financial reward for usage (diminishing returns)
> >> - **Reputation points:** Non-transferable score that grows with impact
> >>
> >> Even after royalties drop to near-zero, you still accumulate reputation forever. This becomes your scientific "credit score" for:
> >> - Applying to new grants
> >> - Getting priority review
> >> - Unlocking governance voting power
> >>
> >> **Why this works:** Scientists care about recognition as much as money. A protocol with 10,000 uses gives you massive reputation even if financial rewards have decayed.
>
> ### The Balanced Model
>
> >> Combine these approaches:
> >>
> >> **For protocol creators:**
> >> - Meaningful early rewards (5% for first 50 uses)
> >> - Diminishing financial returns (decay to 0.5% after 100 uses)
> >> - Permanent reputation accumulation
> >> - Innovation bonuses for novel approaches
> >>
> >> **For new scientists:**
> >> - Clear incentive to create alternatives (2x visibility boost)
> >> - Niche leaderboards (can be #1 in subcategory)
> >> - Bounties for "replace dominant protocol"
> >> - AI-assisted differentiation detection
> >>
> >> **Result:** Healthy ecosystem with turnover. Dominant protocols don't stay dominant forever. Innovation is constantly rewarded. Power law exists but is manageable.
>
> ### Real-World Analogy
>
> >> Think of YouTube or TikTok:
> >> - Early viral creators made millions
> >> - Algorithm constantly promotes new creators
> >> - Categories prevent monopolies (you can't dominate all genres)
> >> - Views matter, but so does engagement, novelty, niche appeal
> >>
> >> Your platform needs similar "algorithmic fairness"—not just "most popular wins forever."
>
> ### The One Thing to Avoid
>
> >> **Don't try to eliminate power law entirely.** Some protocols *should* be more popular because they're genuinely better. The goal isn't equality of outcomes—it's ensuring competition remains possible.
> >>
> >> Bad: Everyone gets equal rewards regardless of quality
> >> Good: Clear path for new protocols to challenge incumbents
> >>
> >> As long as a genuinely better protocol can overtake a worse one within 6-12 months, the system works.