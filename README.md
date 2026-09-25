# E-Commerce Analytics Warehouse

An end-to-end e-commerce analytics project using SQL, ETL, dimensional modeling, optimized reporting marts, and Power BI.

## Project Overview

This project builds an analytics-ready data warehouse from the Brazilian Olist e-commerce dataset found on Kaggle.

The goal is to transform raw transactional marketplace data into a structured analytics system that supports business decisions related to:

- Revenue and sales performance
- Customer behavior
- Seller performance
- Product and category performance
- Delivery and fulfillment efficiency
- Customer satisfaction

Rather than querying raw transactional tables directly for every analysis, this project creates cleaned staging tables, dimensional warehouse models, and optimized reporting marts designed for faster and more reliable business intelligence.

## Business Problem

Olist operates an e-commerce marketplace connecting customers, sellers, and products across Brazil.

The raw data contains information about orders, products, sellers, customers, payments, reviews, and shipping activity across multiple source tables.

While this transaction-level data is valuable, it is not optimized for repeated business reporting.

This project addresses the following question:

> How can raw marketplace transaction data be transformed into an efficient analytics warehouse that allows business teams to monitor revenue, customer experience, seller performance, and fulfillment operations?

## Dataset

The project uses the **Brazilian E-Commerce Public Dataset by Olist**.

The dataset contains approximately 100,000 orders placed between 2016 and 2018 and includes nine related source files.

### Source Tables

| Dataset | Description |
|---|---|
| Customers | Customer identifiers and geographic information |
| Orders | Order status and lifecycle timestamps |
| Order Items | Products, sellers, prices, and freight values |
| Payments | Payment methods, installments, and payment values |
| Reviews | Customer ratings and review information |
| Products | Product categories and physical characteristics |
| Sellers | Seller identifiers and geographic information |
| Geolocation | Brazilian ZIP-code geographic coordinates |
| Category Translation | Portuguese-to-English product category translations |

## Project Architecture

The project follows an analytics engineering workflow:

```text
Raw Source Data
       ↓
Staging Layer
       ↓
Data Cleaning & Validation
       ↓
Dimensional Data Warehouse
       ↓
Analytics / Reporting Marts
       ↓
Advanced SQL Analysis
       ↓
Power BI Dashboard
