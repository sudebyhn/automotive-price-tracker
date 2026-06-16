# 🚗 Automotive Competitor Price Tracking System

A production-grade automated data pipeline that collects daily vehicle price data from major automotive brands in Turkey and stores it in a centralized dashboard database for competitive intelligence.

> **Status:** Live in production — running daily at Skoda Yuce Auto

---

## 💡 The Problem I Solved

The existing system relied on **RPA (Robotic Process Automation)** to scrape competitor pricing from HTML pages. Every time a brand updated their website layout, the RPA scripts broke — requiring manual intervention and rewriting.

**Root cause:** RPA reads the visual HTML structure. It's fragile by design.

**My approach:** Instead of scraping HTML, I inspected each brand's website using browser DevTools (Network tab) to identify the **internal API endpoints** their pages call to load pricing data. By calling those endpoints directly, I retrieve clean, structured JSON — bypassing the HTML layer entirely.

This makes the system **layout-independent**: as long as the underlying API contract doesn't change, the pipeline runs without maintenance.

---

## 🏗️ Architecture
