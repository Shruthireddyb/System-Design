# System Design - LLD + HLD

> Backend | Java | Design Patterns.

This repository contains Low Level Design (LLD) and High Level Design (HLD) implementations focused on backend interview patterns, concurrency, and extensibility.

##  Goal
Build 10+ production-style system designs.

**Tech Stack Used:** Java, OOP, SOLID, Design Patterns (Strategy, Factory, Singleton, Observer)

---

##  Low Level Design (LLD) 

| No | System | Key Patterns Used |
|----|--------|-------------------|
| 01 | Parking Lot System | Strategy, Factory, Singleton |
| 02 | BookMyShow/Ticket Booking System | Factory, Strategy, Observer | 
| 03 | Splitwise/Expense Share | Strategy, Observer | 
| 04 |  ATM Machine | State, Command, Chain of Responsibility | 
| 05 | Chat Application | Observer, Singleton | 
| 06 | Food Delivery Platform | Strategy, Factory, Observer | 
| 07 | Flight Booking System | Strategy, Factory, Observer | 
| 08 | Job Scheduler | Strategy, Factory, Singleton | 
| 09 | Ecommerce Platform | Strategy, Factory, Observer | 
| 10 | OTT Platform | Strategy, Factor | 
| 11 | Notification System | Strategy, Factory, Observer | 


##  High Level Design (HLD) 

| No | System | Key Concepts | 
|----|--------|--------------|
| 01 | URL Shortener (TinyURL) | Hashing, DB Sharding, Redis | 
| 02 | Rate Limiter | Token Bucket, Redis | 
| 03 | Feed System/News Feed | Fanout, Caching, Sharding | 
| 04 | Proximity Search/Nearby Search | Geohash, QuadTree, Redis | 
| 05 | 

---

##  What I Document in Each Design

I cover:
1.  **Problem Statement & Requirements** (Functional + Non-Functional)
2.  **Class Diagram** (Clean Excalidraw)
3.  **Design Decisions** - Why Strategy/Factory/Singleton? (OCP, SRP)
4.  **Concurrency Handling** - How I handle 2 users booking same slot
5.  **Code Skeleton** - Java interfaces + core classes
6.  **Future Improvements** - How to extend without breaking


