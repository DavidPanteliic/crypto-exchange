# Crypto Exchange — System Design Document


## Table of Contents
 
1. [Introduction](#1-introduction)
2. [Product Scope](#2-product-scope)
3. [Functional Requirements](#3-functional-requirements)
4. [High-Level Architecture](#4-high-level-architecture)
5. [Core Concepts](#5-core-concepts)



## 1. Introduction

This document describes the design of **V1 of a centralized cryptocurrency exchange**, similar in overall structure to large exchanges such as Binance. V1 contains two separate trading products inside one exchange:
 
- **Spot Trading** — users submit BUY and SELL orders to the exchange. Orders go into an **Order Book**, a **Matching Engine** matches compatible orders, and trades execute automatically against internal balances.
- **P2P Trading** — users publish **ads(offers)** and trade directly with each other. When an ad is accepted, the seller's crypto moves into **escrow**; the fiat side is settled either off-platform or through an integrated payment provider, and escrow releases the crypto once payment is confirmed.


## 2. Product Scope

### 2.1. Difference beetween Crypto and Fiat

| Concept | What it is | Where it lives |
|---|---|---|
| **Crypto balances** | Custodied by the exchange | Wallet Service and Ledger (database), backed by blockchain holdings |
| **Fiat settlement in P2P** | Fiat moves directly between the two users, or through an external payment provider. The exchange coordinates and verifies it but never holds it | Off-platform, or at the payment provider |

 ### 2.2 Assets in V1 (we will add more in later versions)
 
| Asset | Type | Held in exchange wallets |
|---|---|---|
| BTC | Crypto | Yes |
| ETH | Crypto | Yes |
| USDT | Crypto (stablecoin) | Yes |
| EUR | Fiat | **No** — used only as a pricing/payment currency in P2P |
| USD | Fiat | **No** — used only as a pricing/payment currency in P2P |

### 2.3 Main functionalities in V1
 
- **Account:** registration, login, logout, session/token management, password hashing, 2FA support, basic user profile, basic account security.
- **Wallet:** BTC, ETH and USDT, with a single `available` balance per asset that can never become negative. Internal financial operations are explicit wallet-to-wallet transfers with ledger entries; there is no `reserved` balance field.
- **Spot Trading:** Order Book, Matching Engine, LIMIT and MARKET orders, trading pairs BTC/USDT and ETH/USDT, order and trade history, market data over REST.
- **P2P Trading:** ads, escrow, `OFF_PLATFORM` and `IN_APP` payments, payment provider integration.
- **Deposits and withdrawals** of crypto through Blockchain Integration.
- **Ledger** records all balance changes and checks that they match wallet.



## 3. Functional Requirements

### 3.1 Functional
 
**Account and security**
 
| ID | Requirement |
|---|---|
| FR-1 | A user can register, log in and log out. |
| FR-2 | The system manages sessions/tokens securely (issue, expire, revoke). |
| FR-3 | Passwords are hashed with Argon2id; users can enable 2FA. |
| FR-4 | A user has a basic profile and basic account security settings. |  


**Wallet, deposits and withdrawals**
 
| ID | Requirement |
|---|---|
| FR-5 | A user can view their balance (available, plus a derived view of funds committed to open orders, escrow and pending withdrawals) per asset. | 
| FR-6 | A user can deposit BTC, ETH and USDT: deposit address, detection, confirmations, credit. |
| FR-7 | A user can withdraw crypto to an external address.(User's personal wallet or other exchange app) |
| FR-8 | Every balance change is recorded in the ledger, so any balance can be explained. |  


**Spot Trading**
 
| ID | Requirement |
|---|---|
| FR-9 | A user can list markets and view the order book, ticker and recent trades of a market. |
| FR-10 | A user can place a LIMIT order (limit price). |
| FR-11 | A user can place a MARKET order. |
| FR-12 | Funds are committed when an order is accepted; an order without sufficient funds is REJECTED. |
| FR-13 | Orders are matched with price-time priority; a LIMIT order never executes at a worse price than its limit. |
| FR-14 | Partial fills are supported; the unfilled remainder of a LIMIT order stays in the book. |
| FR-15 | A user can cancel an open or partially filled order; the uncommitted funds are released. |
| FR-16 | A user can view a single order, list orders, and see order and trade history. |
| FR-17 | Every trade settles atomically: balance changes and ledger entries commit together or not at all. |  

 
**P2P Trading**
 
| ID | Requirement |
|---|---|
| FR-18 | A user can publish an ad to buy or sell an asset for fiat, with a price, min/max limits and one or more supported payment methods (`OFF_PLATFORM`, `IN_APP`). |
| FR-19 | A user can browse and filter active ads by asset, fiat currency and payment method. |
| FR-20 | A user can accept an ad, choosing one of its payment methods; this creates a P2P order and immediately transfers the seller's crypto into escrow. |
| FR-21 | For `OFF_PLATFORM` orders, the buyer marks the order as paid and the seller manually confirms receipt before escrow releases. |
| FR-22 | For `IN_APP` orders, the buyer pays through the platform; the platform receives the payment status from the external payment provider and releases escrow automatically once payment is confirmed successful. |
| FR-23 | Escrow is never released before payment is confirmed successful — by the seller's manual confirmation (`OFF_PLATFORM`) or by the payment provider's status (`IN_APP`). |
| FR-24 | Either party can open a dispute on a P2P order regardless of its payment method; the platform resolves it by releasing escrow to the buyer or refunding it to the seller. |
| FR-25 | A P2P order that is never paid (or whose `IN_APP` payment is never completed) expires automatically and refunds escrow to the seller. |  

### 3.2 Non-functional
 
| ID | Requirement |
|---|---|
| NFR-1 | **Correctness:** balances never go negative and no funds are created or destroyed — value is conserved across every reserve, release, trade settlement, escrow transfer and withdrawal. |
| NFR-2 | **Consistency:** every balance change and its ledger entries commit atomically. |
| NFR-3 | **Deterministic matching:** for a given sequence of commands, a market's Matching Engine always produces the same trades in the same order. |
| NFR-4 | **Settlement integrity:** no trade is considered financially settled without durable accounting, and every trade settles exactly once. |
| NFR-5 | **Availability:** backend services can be replicated; a single instance failure must not stop trading. |
| NFR-6 | **Scalability:** stateless services scale horizontally; Spot scales by partitioning markets; new markets are added without redesign. |
| NFR-7 | **Durability:** no acknowledged order, trade, escrow lock, deposit or withdrawal is lost (replication, backups, recovery). |
| NFR-8 | **Security:** credentials, data in transit and sensitive data at rest are protected . |
| NFR-9 | **Auditability:** every financial movement — reserve, trade, escrow, withdrawal, dispute outcome — is traceable through the ledger . |
| NFR-10 | **Observability:** monitoring, logging and health checks detect failures early. |
| NFR-11 | **Escrow integrity:** crypto only ever leaves escrow together with a matching, atomically-committed ledger entry, and never before the order's payment is confirmed successful. |
| NFR-12 | **Market data isolation:** the load or failure of market data distribution never affects order acceptance, matching or settlement. |
 
### 3.3 Design principles
 
- **Never use floating-point numbers for money.** Store amounts as integers in the smallest unit or as fixed-precision decimals.
- **Ledger first.** Balances are derived from, and must always reconcile with, the ledger.
- **Every financial write is a database transaction.** Partial updates are not acceptable.
- **Every financial request is idempotent.** Retrying a request never duplicates its effect.
- **The Wallet Service is the only service that mutates balances.** The Matching Engine, Order Service, Trade Service, P2P Service, Payment Service and Dispute Service never do — they *request* transfers from the Wallet Service.
- **No `reserved` field, anywhere.** A wallet has exactly one balance: `available`. Whenever funds must be "set aside" — committed to a Spot order, locked in P2P escrow, or in flight as a withdrawal — that is modeled as an actual **transfer to a platform-owned system wallet**, written to the ledger like any other movement. Nothing is flagged as locked in place; it visibly moves. See [5.3](#53-wallets).
- **The Order Book is trading state, not financial truth.** It is an in-memory structure the Matching Engine uses to match. The database (orders, trades) and the ledger are the durable source of truth.
- **The Matching Engine never touches balances.** It creates trades; settlement is a separate step performed by the Wallet Service.
- **One serialized matching sequence per market.** Different markets run independently.
- **The platform never custodies fiat.** For `OFF_PLATFORM` orders it moves directly between the two users; for `IN_APP` orders it is processed by an external payment provider. The platform coordinates and verifies the payment but keeps no internal fiat wallet or fiat ledger.


## 4. High-Level Architecture

### 4.1 Components
 
| Component | Responsibility |
|---|---|
| **Frontend** | Web and mobile applications. Present balances, the Spot order form and order book, P2P ads, and the P2P order flow; talk only to the API Gateway. |
| **API Gateway** | Single entry point for clients. Routing, TLS termination, rate limiting, request authentication. |
| **Authentication Service** | Registration, login, logout, session/token management, password hashing, 2FA. |
| **User Service** | Stores and serves user profile, account information and P2P reputation data. |
| **Wallet Service** | Owns every balance mutation across both Spot and P2P: deposits, withdrawals, order fund reservation/release, trade settlement, and escrow lock/release/refund. The only component that writes to the ledger. |
| **Order Service** | Validates incoming Spot orders, requests fund reservation from the Wallet Service, and hands accepted orders to the Matching Engine. Owns order lifecycle state and order/trade history queries. |
| **Matching Engine** | Holds the in-memory Order Book per market, matches BUY and SELL orders with price-time priority, and emits trade events. Never touches balances. |
| **Trade / Settlement Service** | Consumes trade events from the Matching Engine and requests settlement (debit/credit both sides) from the Wallet Service, atomically with the trade record. |
| **Market Data Service** | Publishes order book snapshots, best bid/ask, last price, recent trades and 24h statistics, built from Matching Engine trade/book events. Read-only; never in the order-acceptance or matching path. |
| **P2P Service** | Manages ads and the P2P order lifecycle: accept, payment (`OFF_PLATFORM` or `IN_APP`), confirm, expire. Calls the Wallet Service to move funds and the Payment Service to initiate/track `IN_APP` payments; never mutates a balance itself. |
| **Payment Service** | Initiates `IN_APP` payments, communicates with the external payment provider, receives payment status via webhook, validates success, and notifies the P2P Service. Holds no fiat and maintains no fiat wallet. |
| **Dispute Service** | Reviews appeals on P2P orders and instructs the Wallet Service to release escrow to the buyer or refund it to the seller. |
| **Ledger** | Append-only record of every balance change, from both Spot and P2P. |
| **Blockchain Integration** | Generates deposit addresses, watches blockchains for deposits, tracks confirmations, and signs and broadcasts withdrawals. |
| **Database** | Persistent relational store for users, wallets, markets, orders, trades, P2P ads/orders, disputes and ledger entries. |
 
 
### 4.2 Two trading lanes, one settlement layer
 
Spot and P2P are independent all the way up to fund movement:
 
- **Spot** never creates an ad, never touches escrow, and has no concept of a dispute over payment — its "dispute" equivalent (an incorrect trade) is a data-integrity bug, not a workflow.
- **P2P** never touches the Order Book or the Matching Engine, and has no concept of price-time priority — a P2P "price" is just a number in an ad, discovered by the advertiser, not by matching.
They share exactly one thing: **every balance change either product causes goes through the Wallet Service and is written to the Ledger.** This is what lets both products be added, scaled and audited independently while keeping one, single financially-correct source of truth.


The exchange uses a persistent relational database to store **users**.

## 5. Core Concepts
 
### 5.1 Assets
 
The exchange custodies three crypto assets in V1: **BTC**, **ETH**, **USDT**. EUR exists only as a fiat pricing/payment currency inside P2P ads 
 
### 5.2 Markets
 
There are two different kinds of "market" in this document, and they must not be confused:
 
| | Spot market | P2P market |
|---|---|---|
| Composition | Two custodied crypto assets, e.g. `BTC/USDT` | One custodied crypto asset + one fiat currency, e.g. `BTC/EUR` |
| Has an Order Book | Yes | No |
| Price discovery | By matching | Set by the advertiser |
| V1 examples | `BTC/USDT`, `ETH/USDT` | `BTC/EUR`, `ETH/EUR`, `USDT/EUR` |
 

## 7. Passwords and Protection 

For users passwords we will use argon2id. Making brute-force attacks more expensive
brute-force attacks - automated method where software systematically tries every possible combination of letters, numbers, or symbols to guess login credentials or encryption keys

TLS (Transport Layer Security) protection - cryptographic protocol that securas data in transit between a client and server 
Secure the HTTP connections on the web: 
1) Encryption - data is encrypted in transit
2) Integrity - an attacker can't secretly modify data (without detection) while it is in transit between client and server.
3) Authentication - server authenticate themselves to clients, but clients don't authenticate themselves to servers

TLS can use combination of cryptohraphic mehanism

HTTPS - encrypted version of the HTTP protocol. Using TSL to encrypt all communication, mostly used for banking activities and online services

AES-256-GCM is a secure encryption method that hides data and checks if anyone changed it. 
GCM (Galois/Counter Mode)- Adds an authentication tag to prove the data is real and unchanged.
This metod uses a large 256-bit key to lock data so for hacker it will be very hard to guess. Also this metod is very fast because it work in parallel and uses hardware help 

**Manipulator in the middle (MITM) attack** - A common way for an attacker to execute an MITM attack is to set up a wireless access point in a public place, such as a cafe or an airport, and wait for a victim to connect to it. If a victim does connect, then the attacker will be able to read and modify any data exchanged between the user's browser and any sites they connect to.
