# System Design - LLD + HLD

> Backend | Java | Design Patterns.

This repository contains Low Level Design (LLD) and High Level Design (HLD) implementations focused on backend interview patterns, concurrency, and extensibility.

## 🎯 Goal
Build 10+ production-style system designs.

**Tech Stack Used:** Java, OOP, SOLID, Design Patterns (Strategy, Factory, Singleton, Observer)

---

## 📚 Low Level Design (LLD) - Completed

| No | System | Key Patterns Used | Link |
|----|--------|-------------------|------|
| 01 | Parking Lot System | Strategy, Factory, Singleton | [View LLD](./parking-lot/) |
| 02 | BookMyShow | Factory, Strategy, Observer | 
| 03 | Splitwise / Expense Share | Strategy, Observer | 
| 04 | Warehouse Automation Platform (Final Year Major Project) | Kafka, MQTT, Redis, Strategy | 
| 05 | Elevator System | Strategy, State | 
| 06 | Tic-Tac-Toe / Chess | Factory, Strategy | 

## 🏗️ High Level Design (HLD) - Complete
| No | System | Key Concepts | Status |
|----|--------|--------------|--------|
| 01 | URL Shortener (TinyURL) | Hashing, DB Sharding, Redis | 
| 02 | Rate Limiter | Token Bucket, Redis | 
| 03 | Warehouse Real-Time Tracking (50 robots) | MQTT, Kafka, WebSocket | 

---

## 🔍 What I Document in Each Design

I cover:
1.  **Problem Statement & Requirements** (Functional + Non-Functional)
2.  **Class Diagram** (Clean Excalidraw)
3.  **Design Decisions** - Why Strategy/Factory/Singleton? (OCP, SRP)
4.  **Concurrency Handling** - How I handle 2 users booking same slot
5.  **Code Skeleton** - Java interfaces + core classes
6.  **Future Improvements** - How to extend without breaking


